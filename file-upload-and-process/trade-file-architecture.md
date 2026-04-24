# Trade File Processing Architecture

> **Non-XA · JPA Transaction Manager · IBM MQ CLIENT_ACKNOWLEDGE · Idempotent · DB-backed DLQ**

---

## Table of Contents

1. [Overview](#overview)
2. [Database Schema](#database-schema)
3. [Upload Flow](#upload-flow)
4. [The Two Upload Gaps](#the-two-upload-gaps)
5. [IBM MQ Listener Flow](#ibm-mq-listener-flow)
6. [Critical Implementation Traps](#critical-implementation-traps)
7. [IBM MQ Backout Queue](#ibm-mq-backout-queue)
8. [Corner Cases](#corner-cases)
9. [Status State Machine](#status-state-machine)
10. [Complete Failure Matrix](#complete-failure-matrix)
11. [Key Design Decisions Summary](#key-design-decisions-summary)

---

## Overview

This architecture achieves **XA-level atomicity** between a PostgreSQL/MySQL database and IBM MQ **without using XA/JTA transactions**. Instead it relies on:

- **Single `@Transactional` (JPA)** wrapping both DB save and MQ send on the upload side
- **`CLIENT_ACKNOWLEDGE` mode** on the listener side — MQ message acknowledged only after DB commits
- **Idempotency guards** (atomic SQL lock + DB unique constraint) to make every operation safe to retry
- **DB-backed DLQ table** for messages that cannot be processed after all retries
- **IBM MQ Backout Queue** for permanent failures (corrupt files etc.)

---

## Database Schema

```sql
-- Primary file table
CREATE TABLE trade_file (
    id                BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id           VARCHAR(100)  NOT NULL,
    correlation_id    VARCHAR(100)  NOT NULL,          -- client idempotency key
    file_name         VARCHAR(255),
    file_content      BLOB,                            -- or S3 key if stored externally
    status            VARCHAR(20)   DEFAULT 'SUBMITTED',
    last_processed_row INT          DEFAULT 0,         -- checkpoint for large files
    created_at        TIMESTAMP,
    updated_at        TIMESTAMP,

    UNIQUE KEY uq_correlation (correlation_id)         -- prevents duplicate uploads
);

-- Each parsed row from the file
CREATE TABLE trade_instruction (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    file_id     BIGINT        NOT NULL,
    row_number  INT           NOT NULL,                -- position in file (0-indexed)
    payload     VARCHAR(2000),
    status      VARCHAR(20)   DEFAULT 'PENDING',
    created_at  TIMESTAMP,

    UNIQUE KEY uq_file_row (file_id, row_number)       -- idempotency guard on row level
);

-- DB-backed dead letter queue
CREATE TABLE trade_file_dlq (
    id           BIGINT PRIMARY KEY AUTO_INCREMENT,
    file_id      BIGINT,
    user_id      VARCHAR(100),
    reason       VARCHAR(500),
    raw_message  VARCHAR(2000),
    retry_count  INT           DEFAULT 5,
    status       VARCHAR(20)   DEFAULT 'PENDING',      -- PENDING, REPROCESSED, DISCARDED
    created_at   TIMESTAMP,
    updated_at   TIMESTAMP
);
```

---

## Upload Flow

### Happy Path

```
Client (POST /upload + correlationId)
    │
    ▼
FileUploadController
    │
    ▼
@Transactional (JPA local tx — single method boundary)
    │
    ├── ① SELECT by correlationId
    │       └── if found → return existing fileId (idempotency)
    │
    ├── ② fileRepo.save()
    │       └── status = SUBMITTED, written but NOT committed yet
    │
    └── ③ jmsTemplate.send(TRADE.FILE.QUEUE, {fileId, userId})
            └── fires INSIDE open transaction
    │
    ▼
JPA commits DB ← method returns here
    │
    ▼
HTTP 200 — return fileId to client
```

### Service Code

```java
@Service
@RequiredArgsConstructor
public class TradeFileService {

    private final TradeFileRepository fileRepo;
    private final JmsTemplate jmsTemplate;

    @Transactional   // JPA only — no XA coordinator
    public Long uploadFile(MultipartFile file, String userId, String correlationId)
            throws IOException {

        // ① Idempotency check
        return fileRepo.findByCorrelationId(correlationId)
            .map(TradeFile::getId)
            .orElseGet(() -> {
                TradeFile f = TradeFile.builder()
                    .userId(userId)
                    .correlationId(correlationId)
                    .fileName(file.getOriginalFilename())
                    .fileContent(file.getBytes())   // or store to S3 and save key
                    .status(FileStatus.SUBMITTED)
                    .build();

                f = fileRepo.save(f);  // ② DB write (uncommitted)

                jmsTemplate.convertAndSend(           // ③ MQ send inside tx
                    "TRADE.FILE.QUEUE",
                    new TradeFileMessage(f.getId(), userId)
                );

                return f.getId();
            });
        // JPA commits here — if MQ send threw, @Transactional already rolled back DB
    }
}
```

---

## The Two Upload Gaps

Because we are not using XA, there are two gaps between DB and MQ that must be explicitly handled.

### Gap 1 — DB Commit Fails After MQ Send

```
save()   ✅  written in tx
send()   ✅  message delivered to IBM MQ
commit() ❌  DB throws (timeout / deadlock / network)
             → DB rolls back
             → MQ message cannot be unsent

Result: Listener receives fileId → queries DB → record does not exist
```

**Mitigation:** Listener retries 5 times with exponential backoff. If all fail → insert to `trade_file_dlq` via `REQUIRES_NEW` tx → ACK the MQ message → ops review.

---

### Gap 2 — Race: Listener Faster Than DB Commit

```
save()           written, tx still open
send()           IBM MQ receives message instantly
                 └── Fast consumer picks up message
                         └── SELECT trade_file WHERE id=?
                                 └── 0 rows — DB not committed yet!
commit()         DB commits (too late for first listener attempt)
```

**Mitigation:** `fetchWithRetry()` called **outside** any `@Transactional` boundary. Attempt #2 after 500ms catches the committed row. Total retry window is ~7.5 seconds — far longer than any realistic DB commit latency.

```java
// Retry delays: 0ms, 500ms, 1000ms, 2000ms, 4000ms
private TradeFile fetchWithRetry(Long fileId) throws InterruptedException {
    long[] delays = {0, 500, 1000, 2000, 4000};
    for (int i = 0; i < delays.length; i++) {
        if (delays[i] > 0) Thread.sleep(delays[i]);
        Optional<TradeFile> file = fileRepo.findById(fileId);
        if (file.isPresent()) return file.get();
        log.warn("fileId={} not found, attempt {}/5", fileId, i + 1);
    }
    throw new FileNotFoundException("fileId=" + fileId + " not found after 5 retries");
}
```

---

## IBM MQ Listener Flow

### Configuration

```java
@Configuration
public class IbmMqConfig {

    @Bean
    public DefaultJmsListenerContainerFactory jmsListenerContainerFactory(
            ConnectionFactory connectionFactory) {

        DefaultJmsListenerContainerFactory factory = new DefaultJmsListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);

        // CLIENT_ACKNOWLEDGE — we control exactly when message is acknowledged
        factory.setSessionAcknowledgeMode(Session.CLIENT_ACKNOWLEDGE);

        // One message per session — prevents bulk-ACK JMS side effect
        factory.setMaxMessagesPerTask(1);

        return factory;
    }
}
```

### Listener Steps

```
Step 1 — Message arrives (CLIENT_ACKNOWLEDGE, not acked yet)
    │
    ▼
Step 2 — fetchWithRetry(fileId) ← OUTSIDE @Transactional
    │           └── 5 attempts with backoff (Gap 2 handler)
    │           └── Exhausted → DLQ path (Step 6)
    │
    ▼
Step 3 — Atomic lock (1 SQL)
    │   UPDATE trade_file
    │   SET status = 'IN_PROGRESS'
    │   WHERE id = ? AND status = 'SUBMITTED'
    │           └── 1 row updated → this instance owns it → proceed
    │           └── 0 rows updated → already taken/done → skip + ACK
    │
    ▼
Step 4 — @Transactional(REQUIRES) — TradeFileProcessorService
    │   ├── Read file content from trade_file
    │   ├── Parse CSV → list of rows
    │   ├── INSERT trade_instruction per row
    │   │       └── UNIQUE(file_id, row_num) → duplicate → catch → skip
    │   └── UPDATE trade_file SET status = 'COMPLETED'
    │   └── ← DB COMMITS HERE
    │
    ▼
Step 5 — acknowledgeWithRetry() — 3 attempts (immediate, 500ms, 1s)
    │           └── ACK ok → message removed from IBM MQ ✅
    │           └── All fail → IBM MQ redelivers
    │                   └── status=COMPLETED → Step 3 returns 0 → skip → safe ✅
    │
    ▼ (only if Step 2 fetch retries exhausted)
Step 6 — DLQ path
    ├── @Transactional(REQUIRES_NEW) → INSERT trade_file_dlq
    └── jmsMessage.acknowledge() → remove from queue → ops review
```

### Listener Code

```java
@Component
@RequiredArgsConstructor
public class TradeFileListener {

    private final TradeFileProcessorService processorService; // separate @Service ②
    private final TradeFileDlqRepository dlqRepo;

    @JmsListener(destination = "TRADE.FILE.QUEUE",
                 containerFactory = "jmsListenerContainerFactory")
    public void onMessage(TradeFileMessage message, Message jmsMessage) {
        Long fileId = message.getFileId();
        try {
            // Step 2 — OUTSIDE @Transactional ③
            TradeFile file = processorService.fetchWithRetry(fileId);

            // Steps 3 + 4 — inside @Transactional(REQUIRES)
            boolean processed = processorService.process(file);
            if (!processed) log.warn("fileId={} already handled, skipping", fileId);

            // Step 5
            acknowledgeWithRetry(jmsMessage, fileId);

        } catch (FileNotFoundException e) {
            // Step 6 — Gap 1: file never appeared in DB
            saveToDlq(message, e.getMessage());
            silentAck(jmsMessage);

        } catch (Exception e) {
            log.error("Processing failed fileId={}", fileId, e);
            // No ACK → IBM MQ redelivers → clean retry
        }
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveToDlq(TradeFileMessage msg, String reason) {
        dlqRepo.save(TradeFileDlq.builder()
            .fileId(msg.getFileId())
            .userId(msg.getUserId())
            .reason(reason)
            .rawMessage(msg.toString())
            .status(DlqStatus.PENDING)
            .build());
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class TradeFileProcessorService {

    private final TradeFileRepository fileRepo;
    private final TradeInstructionRepository instructionRepo;

    // ③ MUST be outside @Transactional — fresh read per attempt
    public TradeFile fetchWithRetry(Long fileId) throws InterruptedException {
        long[] delays = {0, 500, 1000, 2000, 4000};
        for (int i = 0; i < delays.length; i++) {
            if (delays[i] > 0) Thread.sleep(delays[i]);
            Optional<TradeFile> f = fileRepo.findById(fileId);
            if (f.isPresent()) return f.get();
        }
        throw new FileNotFoundException("fileId=" + fileId);
    }

    // ④ REQUIRES (default) — markInProgress joins this tx, rolls back together
    @Transactional
    public boolean process(TradeFile file) {
        // Step 3 — atomic lock
        int updated = fileRepo.markInProgress(file.getId());
        if (updated == 0) return false; // already taken or completed

        // Step 4 — row processing
        List<String[]> rows = CsvParser.parse(file.getFileContent());
        for (int i = 0; i < rows.size(); i++) {
            insertRowSafely(file.getId(), i, rows.get(i));
        }

        fileRepo.markCompleted(file.getId());
        return true;
        // DB commits here — markInProgress is part of this tx
    }

    private void insertRowSafely(Long fileId, int rowNum, String[] data) {
        try {
            instructionRepo.save(TradeInstruction.builder()
                .fileId(fileId)
                .rowNumber(rowNum)
                .payload(Arrays.toString(data))
                .status("PENDING")
                .build());
        } catch (DataIntegrityViolationException e) {
            // UNIQUE(file_id, row_number) — duplicate on retry → skip silently
            log.warn("Row already exists fileId={} row={}, skipping", fileId, rowNum);
        }
    }
}
```

```java
// Repository — atomic lock query
@Modifying
@Transactional
@Query("""
    UPDATE TradeFile f
    SET f.status = 'IN_PROGRESS', f.updatedAt = CURRENT_TIMESTAMP
    WHERE f.id = :fileId AND f.status = 'SUBMITTED'
""")
int markInProgress(@Param("fileId") Long fileId);
```

---

## Critical Implementation Traps

### ② Spring AOP Self-Invocation Trap

```java
// ❌ WRONG — @Transactional proxy bypassed, no transaction
@Component
public class TradeFileListener {
    @Transactional
    private void processFile(Long fileId) { ... }

    @JmsListener(...)
    public void onMessage(...) {
        processFile(fileId); // self-call → bypasses Spring proxy → no tx
    }
}

// ✅ CORRECT — separate @Service, injected via Spring
@Service
public class TradeFileProcessorService {
    @Transactional
    public boolean process(TradeFile file) { ... }
}
```

### ③ fetchWithRetry Isolation Trap

```java
// ❌ WRONG — REPEATABLE_READ caches first empty result, all retries see same empty
@Transactional
public void onMessage(...) {
    TradeFile file = fetchWithRetry(fileId); // retries are useless inside tx
    ...
}

// ✅ CORRECT — called before any @Transactional boundary
public void onMessage(...) {
    TradeFile file = fetchWithRetry(fileId); // fresh read each attempt
    processorService.process(file);           // @Transactional starts here
}
```

### ④ markInProgress Propagation Trap

```java
// ❌ WRONG — REQUIRES_NEW commits independently
// App crashes mid-row → status stuck IN_PROGRESS forever → file never retried
@Transactional(propagation = Propagation.REQUIRES_NEW)
int markInProgress(Long fileId) { ... }

// ✅ CORRECT — default REQUIRES joins parent tx
// App crash → entire tx rolls back → status reverts to SUBMITTED → clean retry
@Transactional // default = REQUIRES
int markInProgress(Long fileId) { ... }
```

---

## IBM MQ Backout Queue

Without backout configuration, a corrupt file causes an infinite redeliver loop.

```
# IBM MQ queue definition
DEFINE QLOCAL(TRADE.FILE.QUEUE)
  BOQNAME(TRADE.FILE.BACKOUT)    ← backout queue name
  BOTHRESH(5)                    ← move after 5 redelivery failures
```

```java
@JmsListener(destination = "TRADE.FILE.BACKOUT")
public void onBackout(TradeFileMessage message, Message jmsMessage) {
    // Runs in REQUIRES_NEW — independent of any other tx
    fileRepo.markFailed(message.getFileId());
    dlqRepo.save(TradeFileDlq.builder()
        .fileId(message.getFileId())
        .reason("IBM MQ backout threshold exceeded")
        .status(DlqStatus.PENDING)
        .build());
    silentAck(jmsMessage);
}
```

---

## Corner Cases

### ① Client Idempotency Key (correlationId)

| Scenario | Without correlationId | With correlationId |
|---|---|---|
| Client times out, retries | Duplicate trade_file + 2 MQ messages | Idempotency check → same fileId returned |
| Network glitch on response | Duplicate processing | Deduped at DB level |

### Horizontal Scaling

IBM MQ queues are **point-to-point** — each message is delivered to exactly one consumer. However, on redelivery, a different app instance may receive the message:

- **Atomic UPDATE WHERE status=SUBMITTED** ensures only one instance can own a file
- **UNIQUE(file_id, row_number)** ensures no duplicate rows even if two instances somehow process the same file concurrently

### IBM MQ CLIENT_ACKNOWLEDGE Bulk-ACK Risk

JMS spec: calling `acknowledge()` on message N acknowledges **all messages** received in that session up to N.

**Mitigation:** `factory.setMaxMessagesPerTask(1)` — one message per session, one ACK per message.

### MQ Session Timeout During Long Processing

If file processing takes longer than IBM MQ's HBINT (heartbeat) timeout, the session is dropped and IBM MQ redelivers the message to another instance while the first is still processing.

**Mitigation:**
1. Set `HBINT` > expected max processing time
2. For large files: chunk into batches of 500 rows, each with its own `REQUIRES_NEW` tx, tracking `last_processed_row` as a checkpoint

### Large File Handling

```sql
ALTER TABLE trade_file ADD COLUMN last_processed_row INT DEFAULT 0;
```

```java
// On each batch commit
fileRepo.updateCheckpoint(fileId, batchEndRow);

// On redelivery — resume from checkpoint
int startRow = file.getLastProcessedRow() + 1;
```

---

## Status State Machine

```
                        SUBMITTED
                       /         \
           Atomic UPDATE wins    5 fetch retries fail
                     /                   \
               IN_PROGRESS          trade_file_dlq
              /           \              (PENDING → REPROCESSED)
   All rows done    IBM MQ BackoutThreshold=5
          |                   |
      COMPLETED            FAILED
```

### Key invariant — IN_PROGRESS cannot be orphaned

`markInProgress()` uses **default REQUIRES propagation** → joins the same `@Transactional` as the row inserts. If any step fails:

- Entire transaction rolls back
- `IN_PROGRESS` status update is rolled back too
- Status reverts to `SUBMITTED`
- IBM MQ redelivers
- Next attempt starts clean

If `REQUIRES_NEW` were used instead, `IN_PROGRESS` would commit independently. A crash mid-row would leave the file stuck in `IN_PROGRESS` forever — never retried, never completed.

---

## Complete Failure Matrix

| # | Scenario | DB State | MQ State | Recovery |
|---|---|---|---|---|
| 1 | MQ send() throws RuntimeException inside @Transactional | Rolled back ✅ | No message sent | Clean. @Transactional rolls back DB. Client retries upload. |
| 2 | DB commit fails after MQ send — Gap 1 | Nothing committed | Message in MQ ❌ | fetchWithRetry 5x → trade_file_dlq (REQUIRES_NEW) → ops review |
| 3 | Race: IBM MQ delivers before DB commits — Gap 2 | Committing... | Delivered early | Retry #2 after 500ms catches committed row |
| 4 | Client retries upload (timeout / network) | Duplicate row risk | Duplicate msg risk | correlationId UNIQUE constraint → idempotency check → existing fileId |
| 5 | Two app instances race on same message | SUBMITTED | Duplicate delivery | Atomic UPDATE WHERE SUBMITTED → one wins, one skips |
| 6 | App crash before jmsTemplate.send() | Rolled back | No message | Clean. No orphan state. |
| 7 | App crash after send(), before JPA commit | Rolled back | Message in MQ | Same as Gap 1 → fetch retries → DLQ table |
| 8 | App crash mid-row insert (tx uncommitted) | Rolled back → SUBMITTED | Not ACKed → redelivered | Fresh retry. UNIQUE skips already-inserted rows. |
| 9 | App crash after processFile commits, before ACK | COMPLETED persisted | Redelivered on reconnect | markInProgress=0 → skip → ACK succeeds |
| 10 | Corrupt file — parseCsv always throws | SUBMITTED (tx rolls back) | Redelivered N times | IBM MQ BackoutThreshold=5 → BACKOUT queue → FAILED |
| 11 | DB commits, ACK throws exception | COMPLETED | Redelivered | ACK retry 3x. If all fail: COMPLETED guard skips on redelivery. |
| 12 | All 3 ACK retries fail | COMPLETED | Keeps redelivering | status=COMPLETED → markInProgress=0 → skip forever. Safe. |
| 13 | All 5 fetch retries exhausted (Gap 1 confirmed) | Missing | Delivered | REQUIRES_NEW saves to trade_file_dlq → ACK → ops |
| 14 | **BUG**: processFile() in same class as listener | No tx at all | Delivered | AOP proxy bypassed. Fix: separate @Service ② |
| 15 | **BUG**: fetchWithRetry inside @Transactional | Committing... | Delivered | REPEATABLE_READ caches empty result. Fix: call before tx ③ |
| 16 | **BUG**: markInProgress uses REQUIRES_NEW | IN_PROGRESS committed | Crash mid-row | Status orphaned. Fix: use default REQUIRES ④ |
| 17 | Large file: 100k rows in single @Transactional | Lock escalation / timeout | Redelivered | Chunk 500 rows/batch + last_processed_row checkpoint |
| 18 | MQ session timeout during long processing | IN_PROGRESS (tx open) | Redelivered to other instance | Atomic lock → one winner. UNIQUE handles partial overlap. |

> **Rows 14–16 are implementation bugs, not runtime failures.** They only occur if the code is written incorrectly. The ②③④ markers throughout this document show exactly where each is prevented.

---

## Key Design Decisions Summary

| Decision | Choice | Reason |
|---|---|---|
| Transaction manager | `DataSourceTransactionManager` (JPA only) | No XA/JTA overhead, simpler config |
| MQ acknowledge mode | `CLIENT_ACKNOWLEDGE` | Manual control — ACK only after DB commits |
| Messages per session | `maxMessagesPerTask=1` | Prevents JMS bulk-ACK side effects |
| Listener separation | `@Service` for processing logic | Avoids Spring AOP self-invocation trap |
| fetchWithRetry placement | Outside `@Transactional` | Avoids REPEATABLE_READ caching empty results |
| markInProgress propagation | Default `REQUIRES` | Rolls back with parent tx — no orphan status |
| DLQ type | DB table `trade_file_dlq` | Survives MQ outages, queryable, auditable |
| DLQ tx propagation | `REQUIRES_NEW` | Commits independently — never silently lost |
| Duplicate row guard | `UNIQUE(file_id, row_number)` | DB-enforced idempotency on row level |
| Duplicate upload guard | `UNIQUE(correlation_id)` | Client retry safety |
| Permanent failure | IBM MQ BackoutThreshold + BACKOUT queue | Prevents infinite loop on corrupt data |
| Large file | Chunked batches + `last_processed_row` | Avoids long tx, lock escalation, OOM |
