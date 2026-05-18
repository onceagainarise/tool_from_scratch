# EC2 → RDS PostgreSQL Migration: Full Analysis, Detailed Steps & Glossary

---

## PART 1: CORRECTNESS ASSESSMENT

### Overall Verdict: Mostly Correct with a Few Notable Gaps

The document is technically sound in its high-level architecture and strategy. However, there are several inaccuracies, oversimplifications, and missing caveats that are important for a production migration.

---

### Specific Issues Found

#### 1. "Install the test_decoding plugin" — MISLEADING
The document says in Section 4.2:
> *"AWS DMS requires the test_decoding output plugin to be installed if pglogical is not being used."*

**What's wrong:** `test_decoding` is a **built-in plugin** included in every PostgreSQL installation since 9.4. There is nothing to install. You simply need `wal_level = logical` to enable it. Saying "install" could cause a team to spend time looking for a package that doesn't exist.

#### 2. RDS Has No Full Superuser — Critical Gap
The document says in Section 4.2:
> *"AWS recommends a superuser-like account for setup."*

**What's wrong:** On Amazon RDS, there is no true superuser. The master user has the `rds_superuser` role, which is different. Specifically, `rds_superuser` cannot execute `REPLICATION` or certain `pg_hba.conf`-level operations directly. For DMS and logical replication setups, you must use the `rds_superuser` role carefully and grant `rds_replication` explicitly where needed.

#### 3. WAL Retention Configuration — Omitted
The document never mentions `wal_keep_size` (PostgreSQL 13+) or `wal_keep_segments` (pre-13). This is critical: if your EC2 source generates WAL faster than DMS or logical replication can consume it, the replication slot can fall behind and WAL can be recycled before being consumed — causing the slot to become invalid and requiring a full restart of the migration.

#### 4. Sequence Handling — Understated
The document mentions "DDL and sequences are not something to rely on without separate handling" in the logical replication section, but treats it as a minor footnote. In practice, sequences need explicit manual migration for **all four methods** and must be manually reset post-cutover. This is a common cause of post-cutover failures (e.g., duplicate key violations on inserts).

#### 5. Version Compatibility Is One-Directional
The document says native logical replication "works across different major versions." This is partially misleading. Logical replication across major versions is only supported from **lower to higher**. Replicating from PostgreSQL 14 to PostgreSQL 12, for example, is unsupported. Also, column type compatibility must be checked across versions.

#### 6. Connection Poolers Can Break Logical Replication — Omitted
If the EC2 source uses PgBouncer or another connection pooler in transaction-mode pooling, logical replication connections will fail. Replication connections must use a direct connection, not a pooler.

#### 7. Large Object (LOB) Support — Not Mentioned
DMS has known limitations with PostgreSQL large objects (`pg_largeobject`). Native logical replication and pglogical also do not replicate large objects. This must be handled separately if the schema uses them.

#### 8. SSL/TLS Between EC2 and RDS — Omitted
RDS enforces SSL in many configurations. The document never mentions configuring SSL on DMS endpoints or in replication connection strings, which can cause silent connection failures.

---

## PART 2: DETAILED IMPLEMENTATION STEPS

---

## Method 1: AWS DMS Full Load + CDC

### Phase 0: Pre-Flight Checks

1. Confirm source PostgreSQL version is 9.4 or higher. Run:
   ```sql
   SELECT version();
   ```

2. Confirm source has logical replication capacity. Run:
   ```sql
   SHOW wal_level;
   SHOW max_replication_slots;
   SHOW max_wal_senders;
   ```
   You need `wal_level = logical`, `max_replication_slots >= 2` (one extra for safety), and `max_wal_senders >= 2`.

3. Identify all tables without a primary key:
   ```sql
   SELECT schemaname, tablename
   FROM pg_tables
   WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
   AND tablename NOT IN (
       SELECT c.relname FROM pg_index i
       JOIN pg_class c ON c.oid = i.indrelid
       WHERE i.indisprimary
   );
   ```
   For each such table, decide whether to add a primary key before migration or set `REPLICA IDENTITY FULL` as a workaround.

4. Check if `test_decoding` is available (it requires only `wal_level = logical`; no install needed):
   ```sql
   SELECT * FROM pg_create_logical_replication_slot('test_slot', 'test_decoding');
   SELECT pg_drop_replication_slot('test_slot');
   ```

5. Estimate database size and WAL generation rate to right-size the DMS replication instance and the migration window:
   ```sql
   SELECT pg_size_pretty(pg_database_size(current_database()));
   SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0'));
   ```

### Phase 1: Prepare the EC2 Source

6. Edit `/etc/postgresql/<version>/main/postgresql.conf` and set:
   ```
   wal_level = logical
   max_replication_slots = 5
   max_wal_senders = 5
   wal_keep_size = 1024   # MB; use wal_keep_segments on PostgreSQL < 13
   idle_in_transaction_session_timeout = 0  # Do NOT set this during DMS migration
   ```

7. Reload PostgreSQL (if only `wal_keep_size`/`max_wal_senders` changed, a reload is enough; `wal_level` requires a **full restart**):
   ```bash
   sudo systemctl restart postgresql
   ```

8. Edit `/etc/postgresql/<version>/main/pg_hba.conf` to allow the DMS replication instance IP:
   ```
   host    replication    dms_user    <DMS_replication_instance_IP>/32    md5
   host    all            dms_user    <DMS_replication_instance_IP>/32    md5
   ```

9. Reload pg_hba.conf:
   ```bash
   sudo systemctl reload postgresql
   ```

10. Create the DMS database user:
    ```sql
    CREATE USER dms_user WITH PASSWORD 'strongpassword' REPLICATION;
    GRANT CONNECT ON DATABASE yourdb TO dms_user;
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO dms_user;
    ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO dms_user;
    -- For rds_replication on the target RDS:
    -- GRANT rds_replication TO dms_user;  (only needed on RDS target)
    ```

### Phase 2: Prepare the Target RDS

11. Create the RDS PostgreSQL instance in the AWS Console or via Terraform/CLI. Ensure:
    - Same or higher PostgreSQL major version as source.
    - Sufficient storage (add 25% buffer).
    - Multi-AZ if production.
    - A custom parameter group (not the default) to allow modifications.

12. In the RDS parameter group, verify:
    - `rds.logical_replication = 1` (needed only if RDS will be a logical replication source later; not strictly required when RDS is a DMS target, but good practice).

13. Create the target schema on RDS. Because DMS does not reliably replicate secondary objects, do this **manually before starting DMS**:
    ```bash
    # Schema-only dump from source
    pg_dump -h <ec2_host> -U postgres -s -n public yourdb > schema_only.sql
    # Apply to RDS
    psql -h <rds_endpoint> -U masteruser -d yourdb -f schema_only.sql
    ```

14. On RDS, disable foreign key checks and drop indexes that will slow the bulk load (re-add after full load):
    ```sql
    -- Note all constraint names first, then drop
    ALTER TABLE orders DISABLE TRIGGER ALL;
    ```

### Phase 3: Configure DMS

15. In the AWS DMS console, create a **replication instance**. Use `dms.t3.medium` or larger depending on DB size. Place it in the same VPC as both the EC2 source and RDS target.

16. Create a **source endpoint**:
    - Engine: PostgreSQL
    - Server: EC2 private IP
    - Port: 5432
    - Database, username, password
    - Extra connection attributes: `pluginName=test_decoding;slotName=dms_slot`
    - Test the connection.

17. Create a **target endpoint**:
    - Engine: PostgreSQL (Amazon RDS)
    - Point to the RDS endpoint
    - Test the connection.

18. Create a **replication task**:
    - Migration type: **Full load and change data capture (CDC)**
    - Table mappings: select the schemas/tables to migrate
    - Under Task Settings, set:
      - `TargetTablePrepMode = DO_NOTHING` (since you've already created the schema)
      - Enable CloudWatch logging
      - Enable validation

### Phase 4: Run and Monitor

19. Start the replication task. Monitor in CloudWatch and the DMS console:
    - Full load rows loaded vs. total
    - CDC latency (source latency and target latency should both approach 0 over time)
    - Source CPU and I/O on EC2
    - Replication slot size: `SELECT slot_name, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) FROM pg_replication_slots;`

### Phase 5: Cutover

20. When CDC latency is consistently near zero, schedule the cutover window.

21. At cutover time:
    - Stop all application writes to the EC2 source (kill app connections or redirect traffic to a maintenance page).
    - Wait for DMS CDC lag to reach exactly zero.
    - Stop the DMS task.

22. Manually migrate sequences:
    ```sql
    -- On source: get current values
    SELECT sequence_name, last_value FROM information_schema.sequences;
    -- On target: advance each sequence past the source value
    SELECT setval('public.orders_id_seq', <source_last_value> + 1000);
    ```

23. Run validation queries:
    ```sql
    -- Row count comparison (run on both source and target)
    SELECT COUNT(*) FROM orders;
    SELECT COUNT(*) FROM customers;
    -- Checksum spot checks
    SELECT md5(array_agg(t.*)::text) FROM (SELECT * FROM orders ORDER BY id LIMIT 1000) t;
    ```

24. Update the application's database connection string to point to RDS.

25. Keep the EC2 source running (read-only) for 24–72 hours as a rollback option.

---

## Method 2: Hybrid (Native Full Load + DMS CDC)

### Phase 0: Pre-Flight Checks (same as Method 1, Steps 1–5)

### Phase 1: Schema Migration

6. Dump the schema with all objects, in dependency order:
   ```bash
   pg_dump -h <ec2_host> -U postgres -s --schema=public yourdb > schema.sql
   ```

7. Review and clean `schema.sql` manually. Remove objects that DMS will create conflicts with (e.g., foreign keys that will fail during bulk load, triggers that fire on insert).

8. Apply schema to RDS:
   ```bash
   psql -h <rds_endpoint> -U masteruser -d yourdb -f schema.sql
   ```

### Phase 2: Full Data Load (Native Tools)

9. Record the current LSN on the source **before** starting the data dump. This is the CDC start point for DMS:
   ```sql
   SELECT pg_current_wal_lsn();
   -- Example result: 0/A1234500 — save this value
   ```

10. Perform the data-only dump:
    ```bash
    pg_dump -h <ec2_host> -U postgres -a --schema=public yourdb > data_only.sql
    ```

11. Restore to RDS:
    ```bash
    psql -h <rds_endpoint> -U masteruser -d yourdb -f data_only.sql
    ```

12. Re-enable constraints and indexes on RDS after the bulk load.

### Phase 3: DMS CDC Only (Starting from Saved LSN)

13. Create the DMS source and target endpoints (same as Method 1, Steps 16–17).

14. Create the replication task with:
    - Migration type: **Change data capture (CDC) only**
    - Under Custom CDC Start Configuration, set the CDC start position to the LSN you saved in Step 9:
      `--cdc-start-position "0/A1234500"`

15. Start the CDC task. Monitor lag until it catches up to real-time.

### Phase 4: Cutover (same as Method 1, Steps 20–25)

---

## Method 3: Native PostgreSQL Logical Replication

### Phase 0: Pre-Flight Checks

1. Confirm PostgreSQL 10.0 or higher on both source and target:
   ```sql
   SELECT version();
   ```

2. Confirm `wal_level = logical` on source:
   ```sql
   SHOW wal_level;
   ```

3. List tables without a primary key (logical replication cannot capture UPDATE/DELETE on these without REPLICA IDENTITY FULL):
   ```sql
   SELECT relname FROM pg_class c
   WHERE c.relkind = 'r'
   AND NOT EXISTS (
       SELECT 1 FROM pg_index i WHERE i.indrelid = c.oid AND i.indisprimary
   );
   ```

4. Confirm `rds.logical_replication = 1` in the RDS parameter group (needed if the RDS instance will subscribe to the EC2 source).

### Phase 1: Configure the EC2 Source

5. Set in `postgresql.conf`:
   ```
   wal_level = logical
   max_replication_slots = 5
   max_wal_senders = 5
   wal_keep_size = 1024
   ```
   Restart PostgreSQL if `wal_level` changed.

6. In `pg_hba.conf`, allow replication connections from the RDS instance's IP:
   ```
   host    replication    repl_user    <rds_nat_ip>/32    md5
   ```

7. Create the replication user:
   ```sql
   CREATE USER repl_user WITH REPLICATION PASSWORD 'strongpassword';
   GRANT SELECT ON ALL TABLES IN SCHEMA public TO repl_user;
   ```

### Phase 2: Prepare the Target RDS

8. Create the target schema on RDS (logical replication does NOT transfer schema):
   ```bash
   pg_dump -h <ec2_host> -U postgres -s --schema=public yourdb | \
     psql -h <rds_endpoint> -U masteruser -d yourdb
   ```

9. Verify all target tables have the correct structure:
   ```sql
   SELECT table_name, column_name, data_type
   FROM information_schema.columns
   WHERE table_schema = 'public'
   ORDER BY table_name, ordinal_position;
   ```

### Phase 3: Create the Publication and Subscription

10. On the EC2 source, create the publication:
    ```sql
    -- Replicate all tables
    CREATE PUBLICATION ec2_to_rds_pub FOR ALL TABLES;
    -- Or specify tables:
    CREATE PUBLICATION ec2_to_rds_pub FOR TABLE orders, customers, products;
    ```

11. On the RDS target, create the subscription:
    ```sql
    CREATE SUBSCRIPTION ec2_to_rds_sub
    CONNECTION 'host=<ec2_private_ip> port=5432 dbname=yourdb user=repl_user password=strongpassword sslmode=require'
    PUBLICATION ec2_to_rds_pub
    WITH (copy_data = true, create_slot = true, enabled = true);
    ```
    The `copy_data = true` triggers an initial table copy for all tables in the publication.

### Phase 4: Monitor Sync Progress

12. On the source, check replication progress:
    ```sql
    SELECT application_name, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
           write_lag, flush_lag, replay_lag
    FROM pg_stat_replication;
    ```

13. On the target, check subscription status:
    ```sql
    SELECT subname, subenabled, subslotname FROM pg_subscription;
    SELECT * FROM pg_stat_subscription;
    ```

14. Check table-level sync state:
    ```sql
    SELECT srrelid::regclass, srsubstate FROM pg_subscription_rel;
    -- 'r' = ready (synced), 'i' = initial copy, 'd' = data copy
    ```

### Phase 5: Cutover

15. When all tables show `srsubstate = 'r'` and `replay_lag` is near zero:
    - Stop writes to the EC2 source.
    - Wait for `replay_lag` to reach zero.
    - On RDS, disable the subscription:
      ```sql
      ALTER SUBSCRIPTION ec2_to_rds_sub DISABLE;
      ```
    - Manually migrate sequences (same as Method 1, Step 22).
    - Run row count validation.
    - Switch application connection strings to RDS.

16. After validation, clean up on the source:
    ```sql
    -- On source, drop the publication
    DROP PUBLICATION ec2_to_rds_pub;
    ```
    And on the target:
    ```sql
    DROP SUBSCRIPTION ec2_to_rds_sub;
    ```

---

## Method 4: pglogical

### Phase 0: Pre-Flight Checks (same as Method 3, Steps 1–4, plus verify pglogical is installable)

```bash
# On EC2 source (Ubuntu/Debian example for PostgreSQL 14)
apt-cache search pglogical
sudo apt-get install postgresql-14-pglogical
```

### Phase 1: Configure the EC2 Source

5. In `postgresql.conf`:
   ```
   wal_level = logical
   shared_preload_libraries = 'pglogical'
   max_replication_slots = 10
   max_wal_senders = 10
   max_worker_processes = 10
   wal_keep_size = 1024
   ```
   **Restart PostgreSQL** (required for `shared_preload_libraries`).

6. In `pg_hba.conf`, add the RDS IP for replication and regular connections.

7. Create the replication user with superuser or specific pglogical permissions:
   ```sql
   CREATE USER pglogical_user WITH REPLICATION PASSWORD 'strongpassword';
   GRANT USAGE ON SCHEMA public TO pglogical_user;
   GRANT SELECT ON ALL TABLES IN SCHEMA public TO pglogical_user;
   GRANT SELECT ON ALL SEQUENCES IN SCHEMA public TO pglogical_user;
   ```

### Phase 2: Install and Configure pglogical on Source

8. Create the pglogical extension:
   ```sql
   CREATE EXTENSION IF NOT EXISTS pglogical;
   ```

9. Create the provider (publisher) node:
   ```sql
   SELECT pglogical.create_node(
       node_name := 'ec2_provider',
       dsn := 'host=<ec2_private_ip> port=5432 dbname=yourdb user=pglogical_user password=strongpassword'
   );
   ```

10. Add all tables in the public schema to the default replication set:
    ```sql
    SELECT pglogical.replication_set_add_all_tables('default', ARRAY['public']);
    SELECT pglogical.replication_set_add_all_sequences('default', ARRAY['public']);
    ```

### Phase 3: Configure pglogical on the RDS Target

11. In the RDS parameter group:
    - `rds.logical_replication = 1`
    - `shared_preload_libraries = pglogical`
    - Reboot the RDS instance to apply `shared_preload_libraries`.

12. On RDS, create the pglogical extension:
    ```sql
    CREATE EXTENSION IF NOT EXISTS pglogical;
    ```

13. Create the subscriber node on RDS:
    ```sql
    SELECT pglogical.create_node(
        node_name := 'rds_subscriber',
        dsn := 'host=<rds_endpoint> port=5432 dbname=yourdb user=masteruser password=password'
    );
    ```

14. Create the target schema on RDS (same as Method 3, Step 8).

### Phase 4: Create the Subscription

15. On RDS, create the subscription:
    ```sql
    SELECT pglogical.create_subscription(
        subscription_name := 'ec2_to_rds_sub',
        provider_dsn := 'host=<ec2_private_ip> port=5432 dbname=yourdb user=pglogical_user password=strongpassword sslmode=require',
        replication_sets := ARRAY['default'],
        synchronize_data := true,
        synchronize_structure := false  -- We handle schema separately
    );
    ```

### Phase 5: Monitor Sync Progress

16. Check subscription status:
    ```sql
    SELECT * FROM pglogical.show_subscription_status('ec2_to_rds_sub');
    -- Status should progress: 'down' → 'initializing' → 'replicating'
    ```

17. Check per-table sync:
    ```sql
    SELECT * FROM pglogical.local_sync_status;
    ```

### Phase 6: Cutover

18. Once replication status is `'replicating'` and LSN lag is near zero:
    - Stop application writes to EC2.
    - Wait for full LSN catch-up.
    - Disable the subscription on RDS:
      ```sql
      SELECT pglogical.alter_subscription_disable('ec2_to_rds_sub', immediate := true);
      ```
    - Migrate sequences manually.
    - Validate row counts.
    - Switch application to RDS.

19. After sign-off, drop the subscription:
    ```sql
    SELECT pglogical.drop_subscription('ec2_to_rds_sub', ifexists := true);
    ```

---

## PART 3: EDGE CASES TO KEEP IN MIND

### Across All Methods

| Edge Case | Risk | Mitigation |
|---|---|---|
| Tables without primary keys | UPDATE/DELETE not captured or incorrect | Add PKs before migration or set `REPLICA IDENTITY FULL` |
| Sequences not replicated | Post-cutover inserts fail with duplicate key errors | Manually migrate sequences; set target value well above source |
| DDL changes during migration | Replication breaks or mismatches schema | Freeze DDL for the migration window; document and replay any required DDL manually |
| Connection pooler (PgBouncer) in front of source | Replication connections fail | Replication must use a direct connection, not via a pooler |
| Large objects (`pg_largeobject`) | Not replicated by any method | Dump and restore large objects separately using `pg_dump --large-objects` |
| Partitioned tables | Some methods cannot replicate all partitions automatically | Add all child partitions explicitly to publications/replication sets |
| Very large tables (> 1 TB) | Full load takes longer than WAL retention window | Increase `wal_keep_size`, use parallel dump/restore, pre-split large tables |
| Foreign data wrappers (FDWs) | Not replicated | Recreate FDW definitions manually on RDS after cutover |
| Row-level security (RLS) | Policies not replicated | Recreate RLS policies manually after schema migration |
| Extensions not available on RDS | Functionality missing post-migration | Verify all source extensions are supported by RDS before starting |
| UNLOGGED and TEMPORARY tables | Not replicated by logical replication or pglogical | Handle separately; these are usually transient data |
| Materialized views | Not refreshed during replication | Refresh manually post-cutover |
| User-defined types (UDTs) | Schema migration may silently omit them | Verify `pg_dump -s` captures all custom types |
| Triggers on target tables | Can cause data duplication or constraint violations during bulk load | Disable triggers on RDS target during initial load; re-enable after |
| Vacuum/autovacuum on source | WAL bloat during heavy replication load | Monitor bloat during migration; tune autovacuum if needed |
| SSL enforcement on RDS | Connections refused if SSL is not configured | Add `sslmode=require` to all connection strings |
| Timezone settings mismatch | Timestamp data stored or interpreted differently | Verify `TimeZone` parameter is identical on source and target |
| `pg_hba.conf` restrictive rules | DMS or replication connections rejected | Verify by testing connections from the DMS replication instance or RDS IP before starting |
| Replication slot growth | Source disk fills up if consumer falls behind | Monitor slot lag with `pg_replication_slots`; set an alert on `pg_wal_lsn_diff` |
| RDS maintenance window | RDS may reboot during migration | Schedule migrations outside the RDS maintenance window; disable auto minor version upgrade temporarily |

### Method 1 (DMS) Specific

- DMS idle transaction timeout: Never set `idle_in_transaction_session_timeout` during a DMS migration; it will terminate DMS sessions.
- DMS LOB mode: If tables have `text`, `bytea`, or `json` columns over a certain size, configure DMS LOB mode (Limited or Full LOB) appropriately. Incorrect LOB settings silently truncate data.
- DMS task restart after failure: After a failure, DMS can resume from CDC but may miss rows if the replication slot has been dropped. Always check the slot still exists before resuming.
- DMS replication instance in same AZ: For best performance and to avoid cross-AZ data transfer charges, place the DMS replication instance in the same Availability Zone as the source EC2.

### Method 2 (Hybrid) Specific

- LSN drift: Between taking the LSN snapshot (Step 9) and starting DMS CDC, the source will continue writing. DMS must start CDC from the **exact LSN at the time of the dump start**, not the dump end time.
- `pg_dump` consistency: `pg_dump` takes a consistent snapshot internally (via `REPEATABLE READ`). Do not use `--no-synchronized-snapshots` with parallel dumps unless you understand the implications.

### Method 3 (Native Logical Replication) Specific

- `copy_data = true` vs. `false`: If you manually pre-load data via `pg_dump`, set `copy_data = false` on the subscription to avoid double-loading.
- Subscription on RDS needs the EC2 source to be reachable over the network during the subscription lifetime — not just at creation.
- Conflict handling: If any data is written to the target during replication (e.g., test inserts), it will cause a replication conflict that halts the subscription silently. Always ensure the target is write-quiesced.
- The subscriber does not replicate from other subscribers — only from the original publisher.

### Method 4 (pglogical) Specific

- pglogical and RDS: pglogical on RDS requires `shared_preload_libraries = pglogical` in the parameter group and a reboot. Plan this downtime for the RDS instance in advance.
- Version mismatch: pglogical version on source and target must be compatible. Mixed major versions of the pglogical extension itself (not PostgreSQL) can cause issues.
- pglogical's `synchronize_structure` option copies some DDL, but it is incomplete — it does not handle all constraints and secondary objects. Always build the target schema independently.
- `UNLOGGED` and `TEMPORARY` tables are explicitly excluded from pglogical replication sets and must be handled manually.

---

## PART 4: KEYWORD DEFINITIONS

---

### CDC — Change Data Capture
A technique that captures every individual row-level change (INSERT, UPDATE, DELETE) made to a database and streams them to a downstream target in near-real-time. CDC allows the target to stay continuously synchronized with the source without needing another full data dump. In PostgreSQL, CDC is implemented via logical decoding of the WAL.

---

### WAL — Write-Ahead Log
PostgreSQL's durability mechanism. Before any data change is written to the data files on disk, it is first written to the WAL. This ensures data is never lost on a crash. For replication, the WAL acts as the stream of all changes; logical replication and DMS both read from the WAL to capture changes.

---

### LSN — Log Sequence Number
A unique, monotonically increasing pointer within the WAL stream. Every WAL record has an LSN. In migration and replication contexts, the LSN is used to mark the exact position from which CDC should start. A subscriber's "lag" is measured by comparing its current replay LSN to the source's current WAL LSN.

---

### Logical Decoding
A PostgreSQL feature that interprets the WAL and converts raw binary write-ahead log records into a human-readable or structured stream of row-level changes. Plugins like `test_decoding` and `pglogical` decode the WAL. This is what powers logical replication and DMS CDC for PostgreSQL.

---

### Replication Slot
A named, persistent cursor into the WAL stream on the source PostgreSQL instance. It ensures the source retains WAL data until the consumer (DMS, subscriber, etc.) has consumed it — even if the consumer disconnects temporarily. Danger: an abandoned or lagging slot causes WAL to accumulate on disk, potentially filling the source disk.

---

### test_decoding
A **built-in** PostgreSQL output plugin (included since PostgreSQL 9.4) that decodes WAL records into a simple text format. AWS DMS uses this plugin to read changes from a PostgreSQL source. No installation is needed — it is always present when `wal_level = logical`.

---

### wal_level
A PostgreSQL parameter that controls how much information is written into the WAL. Valid values:
- `minimal` — only what's needed for crash recovery
- `replica` — enough for physical replication
- `logical` — enough for logical decoding and logical replication (required for all methods in this document)

---

### max_replication_slots
A PostgreSQL parameter that controls how many replication slots can exist simultaneously. Each logical subscriber (DMS task, native subscription, pglogical subscription) consumes one slot. Set this higher than the number of active consumers.

---

### max_wal_senders
A PostgreSQL parameter that limits the number of simultaneous WAL sender processes. Each replication connection to the source uses one sender process. Set at least as high as `max_replication_slots`.

---

### wal_keep_size / wal_keep_segments
Parameters that ensure a minimum amount of WAL is retained on disk even after it would normally be recycled, as a buffer for replication consumers that fall behind. `wal_keep_size` (PostgreSQL 13+) is measured in megabytes; `wal_keep_segments` (pre-13) counts WAL file segments (each 16 MB by default).

---

### pg_hba.conf
The PostgreSQL host-based authentication configuration file. It controls which IP addresses and users can connect to PostgreSQL, over which network interface, and with which authentication method. During migration setup, rules must be added to allow the DMS replication instance or RDS subscriber to connect to the EC2 source.

---

### pg_stat_replication
A PostgreSQL system view that shows one row per active replication connection. Key columns:
- `sent_lsn` — how far the source has sent WAL to this consumer
- `write_lsn` — how far the consumer has written it to its WAL
- `flush_lsn` — how far the consumer has flushed to disk
- `replay_lsn` — how far the consumer has applied (replayed) changes
- `write_lag`, `flush_lag`, `replay_lag` — time-based lag measurements

---

### pg_replication_slots
A PostgreSQL system view listing all active and inactive replication slots, their consumers, and how far behind each slot is. The column `pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)` tells you how much WAL is being held back by each slot.

---

### Publication (PostgreSQL Logical Replication)
A named object on a PostgreSQL source database that defines a set of tables (and optionally sequences) whose changes will be broadcast to subscribers. Created with `CREATE PUBLICATION`. Think of it as a "channel" of changes.

---

### Subscription (PostgreSQL Logical Replication)
A named object on a PostgreSQL target database that connects to a publication on a source, copies the initial data, and then continuously applies incoming changes. Created with `CREATE SUBSCRIPTION`.

---

### pglogical
A third-party PostgreSQL extension (originally developed by 2ndQuadrant, now Enterprisedb) that implements logical streaming replication with more flexibility than native logical replication. It supports replication sets (granular control over which tables/sequences to replicate), can replicate across major versions, and provides better conflict handling. Requires installation as a shared library.

---

### DDL — Data Definition Language
SQL commands that define or modify the structure of database objects: `CREATE TABLE`, `ALTER TABLE`, `DROP INDEX`, etc. (as opposed to DML — Data Manipulation Language — which is INSERT/UPDATE/DELETE). Native logical replication and DMS CDC do **not** replicate DDL changes. Any schema alterations during migration must be replayed manually on the target.

---

### CRUD
Acronym for the four basic database operations: **C**reate (INSERT), **R**ead (SELECT), **U**pdate (UPDATE), **D**elete (DELETE). A "live CRUD workload" means the database is actively receiving all four types of operations during the migration — the most complex scenario.

---

### Replica Identity
A PostgreSQL table-level setting that controls what information is written to the WAL for UPDATE and DELETE operations. Options:
- `DEFAULT` — only the primary key columns are logged (default)
- `FULL` — the entire old row is logged (needed for tables without a primary key)
- `NOTHING` — no old row information (UPDATE and DELETE cannot be captured)
- `USING INDEX` — a specific unique index is used as the identity

---

### DMS — AWS Database Migration Service
An AWS managed service that automates database migrations. For PostgreSQL, it can perform a full initial data load, then switch to CDC mode to replicate ongoing changes until cutover. It uses a replication instance (a managed EC2 VM) to read from the source and write to the target.

---

### RDS Parameter Group
An AWS configuration container that holds PostgreSQL parameter settings for an RDS instance. Unlike on-premise PostgreSQL where you edit `postgresql.conf` directly, on RDS you modify the parameter group and associate it with the instance. Changes to static parameters require a reboot; dynamic parameters take effect immediately.

---

### CloudWatch (AWS)
AWS's monitoring and observability service. AWS DMS publishes metrics (CDC latency, full load progress, errors) and logs to CloudWatch. These should be actively monitored during the migration window.

---

### rds_superuser
The highest-privilege role available on Amazon RDS. It is **not** equivalent to PostgreSQL's native `SUPERUSER`. The `rds_superuser` role can manage other users, grant `rds_replication`, and perform most administrative tasks, but cannot execute operations that require OS-level or direct file system access (which true `SUPERUSER` can do on self-managed PostgreSQL).

---

### REPLICA IDENTITY FULL
A table setting that instructs PostgreSQL to log the entire old row (all columns) in the WAL before every UPDATE or DELETE. This is required for logical replication to capture changes on tables that lack a primary key, but it significantly increases WAL volume.
