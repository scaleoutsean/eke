# Eke

A CLI for doing stuff with BeeGFS storage pools and Hive Index.

Eke was built for version 8.3. Older or newer versions may not work exactly the same.

## Features

Eke does stuff with BeeGFS pools and [Hive Index](https://doc.beegfs.io/latest/hive/hive_index.html)

## v0.1 Commands

- `show-pools` - enhanced wrapper for BeeGFS
- `migrate-safe` - thin wrapper for BeeGFS `entry migrate` sub-commmand
- `migrate-smart` - fancy wrapper for BeeGFS `entry migrate`
- `migrate-up` - limited wrapper for migrating data "up" (archive-to-default)
- `hive-import` - consolidates Hive index for easy reporting and analysis
- `migrate-ledger` - checks manifest and status of migrate jobs
- `migrate-verify` - checks status of input files post-migration

## Build

```sh
go build -o bin/eke ./cmd/eke
```

Version:

```sh
./bin/eke --version
```

## `eke` sub-commands

If you encounter errors that `eke` can't handle, simply fall back to BeeGFS CLI. `eke` doesn't use any "secret" sub-commands, switches or arguments that `beegfs` doesn't already have.

BeeGFS `manager` note: when a command supports `--manager`, you can pass either `host` or `host:port`.

### show-pools

Merge `beegfs pool list`, `beegfs target list`, and `beegfs health capacity` into one view.

Use the per-pool `max_used` / `max_used_percent` value as the safety signal, not only the pool average. BeeGFS target selection inside a pool can be significantly skewed depending on workload and `tuneTargetChooser`, so a pool may look mostly empty on average while one target is close to full.

```sh
./bin/eke show-pools
./bin/eke show-pools -o json
./bin/eke show-pools --detail
./bin/eke show-pools --manager b1 -o json
```

Use `--detail` (or alias `--health`) to include inode utilization and a metadata-target section. This keeps the default view compact while still exposing the other important capacity dimension for small-file workloads.

### migrate-up

Narrow exception workflow for temporary cold-to-hot moves.

- File-list input only (Parquet).
- Dry-run by default.
- Conservative defaults: `--limit-files=100`, `--max-target-used=70`.
- `--update-directories` is off by default.
- Optional `--pending-ledger` output for submitted upward-migration tracking.
- Optional webhook notification after successful execute.
- Stripe-fit precheck warns when estimated required per-file targets exceed destination pool target count; add `--enforce-stripe-fit` to hard-fail.

`migrate-up` requires a Parquet input with `source_dir`, `size`, and `type`, plus either `path` or `name`. Only file rows are selected (`type` = `f`/`file`).

Stripe-fit estimation uses candidate metadata (`num_targets`, `chunk_size`, `size`) when available. It is a practical precheck and may be conservative compared to BeeGFS runtime target assignment for small files.

```sh
./bin/eke migrate-up \
	--input ./out/candidates_hotfix.parquet \
	--source-pool archive \
	--target-pool storage_pool_default
```

```sh
./bin/eke migrate-up \
	--manager b1:9022 \
	--input ./out/candidates_hotfix.parquet \
	--source-pool archive \
	--target-pool storage_pool_default \
	--limit-files 50 \
	--limit-bytes 20GiB \
	--pending-ledger ./out/migrate_up_pending.json \
	--notify-webhook https://ops.example.net/eke/events \
	--webhook-extra-payload @./job.json \
	--webhook-no-verify-ssl \
	--execute
```

Use this as a short-lived exception tool; schedule a later down-tier run (`migrate-safe` or `migrate-smart`) to reverse temporary uplift.

Execution is batched (currently up to 256 paths per `beegfs entry migrate` call). If a batch contains stale paths, `eke` retries per-path and skips missing files with warnings.

Webhook behavior:

- Event schema is versioned: `schema=eke.webhook.v1`, `schema_version=1`.
- `migrate-up --notify-webhook` sends a submission event payload with pool info, selected counts/bytes, and the associated pending-ledger entry (when `--pending-ledger` is set).
- Completion notification is handled by `migrate-ledger --notify-webhook` when entries become `done`.
- Webhook URL accepts both `http://` and `https://` schemes, including explicit ports (for example `http://ops-gw:8080/hooks/eke`).
- For HTTPS webhooks with self-signed certificates, use `--webhook-no-verify-ssl` (or alias `--webhook-no-verify-tls`).
- Optional `action` payload can be attached with `--webhook-extra-payload` (inline JSON object or `@file`).
- `--webhook-extra-payload` must be a JSON object and must be <= 256 bytes after JSON compaction. It is up to Webhook service to validate extra payload (if any) against an agreed upon schema and check for malicious commands.
- Use a valid port and path on your Webhook service. Get these details from Webhook service administrator.

### migrate-safe

Guarded wrapper around `beegfs entry migrate`:

- Blocks migration when destination pool utilization is above threshold.
- Runs dry-run by default.
- Executes only with `--execute`.
- Supports BeeGFS pass-through flags for `--filter-files`, `--update-directories`, and `--rebalance`.

`migrate-safe` guards on the destination pool's hottest target (`max_used_percent`), not the pool average. This matters because BeeGFS may place migrated data unevenly across targets in the same pool.

```sh
./bin/eke migrate-safe --path /mnt/beegfs/project1 --from-pool s:1 --to-pool s:2
./bin/eke migrate-safe --path /mnt/beegfs/project1 --from-pool s:1 --to-pool s:2 --execute
./bin/eke migrate-safe --manager b1 --path /mnt/beegfs/project1 --from-pool s:1 --to-pool s:2
./bin/eke migrate-safe --path /mnt/beegfs/project1 --from-pool storage_pool_default --to-pool archive
./bin/eke migrate-safe --path /mnt/beegfs --from-pool s:2 --to-pool s:2 --recurse --rebalance --execute
./bin/eke migrate-safe --path /mnt/beegfs/project1 --from-pool s:1 --to-pool s:2 --filter-files="mtime > 14d and type == file"
```

Pool flags accept either BeeGFS pool IDs (for example `s:1`, `s:2`) or pool aliases (for example `storage_pool_default`, `archive`).

`--filter-files` is passed through to BeeGFS as-is, so the full BeeGFS expression syntax is available (including helpers like `glob(...)` and `regex(...)`).

`--rebalance` uses BeeGFS background migration mode (BeeGFS 8.2+ enterprise feature). In execute mode, monitor progress with `beegfs stats rebalance`.

### migrate-smart

Read a Parquet file of candidate files, then plan a hot-to-cold move that tries to bring the source tier back under a target fullness while staying below the archive ceiling.

The initial version requires a Parquet file with at least these columns:

- `source_dir`
- `size`

Additionally, either of these must be present:

- `path`
- or `name` (so `eke` can derive `path` from `source_dir || '/' || name`)

For future batching and scoring refinements, the recommended export is still `SELECT *` from the candidate Hive-derived table and then let `eke` use the subset it currently understands.

Do not export from `hive_summary_counts` for `migrate-smart`; that table is useful for reporting, but it does not contain per-file `path` or `size` data.

```sh
./bin/eke migrate-smart \
	--input ./out/candidates.parquet \
	--source-pool storage_pool_default \
	--target-pool archive \
	--path-rewrite-from /mnt/index \
	--path-rewrite-to /mnt/beegfs \
	--filter /mnt/projects/2021 \
	--max-default 65 \
	--max-archive 95
```

```sh
./bin/eke migrate-smart \
	--input ./out/candidates.parquet \
	--source-pool storage_pool_default \
	--target-pool archive \
	--pending-ledger ./out/migrate_pending.json \
	--min-move-capacity 1Gi \
	--execute
```

In v0.1 this is intentionally file-only and dry-run by default. It reads pool state from BeeGFS, reads the candidate list from Parquet using DuckDB, and then plans the smallest safe move set it can find.

`migrate-smart` also supports BeeGFS pass-through flags for migration execution: `--recurse`, `--update-directories`, `--filter-files`, and `--rebalance`.

Execution is batched (currently up to 256 paths per `beegfs entry migrate` call) to avoid oversized command lines on very large candidate sets.

If a batch fails, `eke` retries each path in that batch individually; stale paths (for example files removed after Hive index snapshot) are skipped with warnings while other valid paths continue.

`migrate-smart` includes the same stripe-fit precheck behavior as `migrate-up`; add `--enforce-stripe-fit` to fail before migration when estimated required targets exceed destination pool target count.

`--limit-bytes` is optional. If used, it accepts plain bytes or common suffixes such as `1Gi`, `10GiB`, `500MB`, or `2TiB`.

`--min-move-capacity` is optional. If set, `migrate-smart` will try to select at least that amount (for example `1Mi`, `10GiB`) even when the source tier is already below `--max-default`, while still respecting target ceiling and other limits.

`--min-move-capacity` is a floor, not an override. It cannot reduce threshold-driven required bytes. Effective required bytes are `max(threshold_need, min_move_capacity)`.

Planning formula used by `migrate-smart`:

- `threshold_need = max(0, source_used_bytes - source_capacity_bytes * (max_default / 100))`
- `need_bytes = max(threshold_need, min_move_capacity)`

This means `--min-move-capacity` **can only increase** the requested move amount, never decrease it.

Sanity checks:

- `--min-move-capacity` cannot be negative.
- If `--limit-bytes` is set, `--min-move-capacity` must be less than or equal to `--limit-bytes`.
- If `--min-move-capacity` is greater than 25% of source pool capacity, `migrate-smart` prints a warning and continues.
- Add `--enforce-safe-move-capacity` to turn that >25% condition into a hard error.

If your candidate input originates from Hive index paths (for example `/mnt/index/...`) and your BeeGFS mount path differs (for example `/mnt/beegfs/...`), use `--path-rewrite-from` and `--path-rewrite-to`. Users who use in-tree Hive index don't have to do that.

When the source tier is already below `--max-default`, `migrate-smart` reports that no migration is needed and does not select files.

`migrate-smart` also prints `Candidates matched (after input/filter): N` so it is clear whether zero selected files is due to filtering or simply because no bytes need to be moved.

If you run with `--execute` while `Need to move` is 0, `eke` prints an explicit no-op reason and exits without calling BeeGFS migration.

`--filter-files` is passed through to BeeGFS as-is, so full BeeGFS filter syntax can be used when you want an extra server-side guard in addition to Parquet candidate selection.

`--rebalance` uses BeeGFS background migration mode (BeeGFS 8.2+ enterprise feature). In execute mode, monitor progress with `beegfs stats rebalance`.

Pending ledger mode:

- `--pending-ledger <file>` enables in-flight accounting for repeated runs.
- During planning, pending bytes from prior entries are subtracted from source-used and added to target-used before computing the plan.
- On `--execute`, a successful run records a new pending ledger entry with selected bytes and file count.
- Each new pending entry stores filesystem identity and hybrid verification samples (tail-heavy plus random paths).
- If active entries already exist for the same source/target pair, `--execute` is refused (single-flight safety); dry-run still works.
- A lock file (`<ledger>.lock`) is used to prevent concurrent execute writers.

### migrate-ledger

Reconcile pending migration entries and optionally verify sampled paths.

```sh
./bin/eke migrate-ledger \
	--pending-ledger ./out/migrate_pending.json
```

```sh
./bin/eke migrate-ledger \
	--manager b1 \
	--pending-ledger ./out/migrate_pending.json \
	--source-pool storage_pool_default \
	--target-pool archive \
	--notify-webhook https://ops.example.net/eke/events \
	--webhook-extra-payload @./job.json \
	--webhook-no-verify-ssl \
	--verify-samples
```

Behavior:

- Verifies filesystem identity matches each pending entry (hard-stop on mismatch).
- Uses source/target capacity reconciliation gates to decide completion. Small batches may finish before `migrate-ledger` is executed in which case `--verify-samples` is the only way to get a positive response for job completion status
- Optionally verifies sampled paths via `beegfs entry refresh` + `beegfs entry info`.
- For tiny entries (<= 1 MiB), if sample verification passes, reconcile can mark complete even when capacity counters do not visibly change. 
- If counters lag in your environment, add `--complete-on-samples` to allow sampled-path verification to mark entries complete for larger moves.
- Marks completed entries as `done` with `completed_at_utc`.
- `--notify-webhook` is sent only for entries that become newly `done` in the current run; already-done entries are not re-notified.

Example candidate export:

```sql
COPY (
	SELECT *
	FROM hive_entries
	WHERE snapshot_id = '20260702'
		AND type = 'f'
		AND source_dir LIKE '/mnt/projects/%'
) TO 'candidates.parquet' (FORMAT PARQUET);
```

`--filter` applies an additional path or source directory filter when `migrate-smart` reads the Parquet file. Plain text is treated as a prefix, while SQL wildcard patterns like `%` and `_` are passed through as-is. When path rewrite flags are set, filtering is evaluated against both raw index paths and rewritten BeeGFS paths.

### migrate-verify

A helper to verify candidate paths are currently placed on an expected storage pool.

This is useful after migration to confirm whether entries from `candidates.parquet` are actually on archive (or still on default), especially when pool capacity counters are lagging.

```sh
./bin/eke migrate-verify \
	--input ./out/candidates.parquet \
	--expected-pool archive \
	--path-rewrite-from /mnt/index \
	--path-rewrite-to /mnt/beegfs
```

Behavior:

- Checks each candidate with `beegfs entry info`.
- Reports summary counts: `checked`, `on_expected`, `mismatched`, `missing`, `errors`.
- Emits periodic progress lines with checked/total, rate, and ETA (default every 10,000 paths; configurable with `--progress-every`).
- Prints sample mismatch and missing paths for triage.
- Exits non-zero when mismatches or errors are found.
- Missing paths are reported but not fatal by default; add `--strict-missing` to fail on missing candidates.

### hive-import

**NOTE:** `--output-db` should be outside of both BeeGFS mount path and `index-root` path (which can be omitted if in-tree Hive Index is used, i.e. actual paths match index file paths).

Import all `.bdm.db` files under index root into a consolidated DuckDB snapshot path. With BeeGFS mounted to `/mnt/beegfs` and indexes under `/mnt/index`, we pick a different location for database artifacts.

```sh
cd /mnt/filesystem-analytics; mkdir out # avoid using /mnt/beegfs or named Hive index path
./bin/eke hive-import --index-root /mnt/index --output-db ./out/hive_snapshot.duckdb
./bin/eke hive-import --index-root /mnt/index --output-format both --parquet-dir ./out/parquet
./bin/eke hive-import --index-root /mnt/index --output-db ./out/hive_snapshot.duckdb --snapshot 20260702
./bin/eke hive-import --manager b1 --index-root /mnt/index --output-db ./out/hive_snapshot.duckdb --snapshot 20260702
./bin/eke hive-import --index-root /mnt/index --output-db ./out/hive_snapshot.duckdb --snapshot 20260702 --destination ./publish/20260702
./bin/eke hive-import --index-root /mnt/index --output-db ./out/hive_snapshot.duckdb --snapshot 20260702 --destination s3://storage/reports/20260702
./bin/eke hive-import --index-root /mnt/index --output-db ./out/hive_snapshot.duckdb --snapshot 20260702 --destination https://s3.example.com/storage/reports/20260702 --s3-endpoint https://s3.example.com
./bin/eke hive-import --index-root /mnt/index --output-db ./out/hive_snapshot.duckdb --snapshot 20260702 --destination s3://storage/reports/20260702 --post-step './scripts/post_process_manifests.sh'
```

Remote manager execution:

- Use `--manager <host>` (or `-m <host>`) to run BeeGFS and DuckDB commands through SSH on that node.
- This is useful when the local machine does not have direct BeeGFS access or when index files only exist on a cluster node.

Destination publishing:

- `--destination` publishes a Parquet snapshot plus `manifest.json` after the local DuckDB import completes.
- Local DuckDB output remains the canonical local history artifact.
- The destination can be a local directory, an `s3://bucket/prefix` URI, or an HTTPS S3-compatible endpoint URL of the form `https://host/bucket/prefix`.
- `--post-step` runs an optional shell command after import/publish. This is useful for custom non-clobbering manifest/index workflows (for example Delta/metadata API integration).
- `--post-step` receives environment variables: `EKE_SNAPSHOT_ID`, `EKE_PUBLISH_DIR`, `EKE_OUTPUT_DB`, `EKE_PARQUET_DIR`, `EKE_DESTINATION`.
- For S3-compatible endpoints, pass `--s3-endpoint` when the endpoint is not the default AWS S3 service.
- Publishing to S3 uses the AWS Go SDK and sets request checksum calculation to `when_required` in code, which avoids the `MissingContentLength` problem you hit without needing `AWS_REQUEST_CHECKSUM_CALCULATION`.
- The SDK still relies on the local AWS profile/config or explicit endpoint flags for credentials and endpoint selection.
- Recommended layout for time-series snapshots: publish each run to a snapshot-specific prefix (for example `.../snapshot=20260702/`) and treat `manifest.json` as run-local metadata.
- If you need a consolidated manifest/index layer, implement it in `--post-step` instead of overwriting a single global manifest.

Depending on needs, you may have at least three files per prefix as a basic result (without `output_db` from `manifest.json` uploaded to S3).

```sh
$ aws s3 ls s3://beegfs/snapshot=20260702/
2026-07-03 07:17:45       7316 hive_entries_snapshot=20260702.parquet
2026-07-03 07:17:19       1233 hive_summary_counts_snapshot=20260702.parquet
2026-07-03 07:29:30        588 manifest.json
```

Querying published snapshots:

- DuckDB can query the Parquet snapshots directly from S3 or an S3-compatible bucket with `httpfs`.
- Eke does not bundle DuckDB; use the DuckDB CLI directly for ad hoc analysis.
- The move to the AWS Go SDK does not change AWS credential precedence: CLI flags or code-level options take priority, then environment variables, then shared config and credentials files.

```sql
INSTALL httpfs;
LOAD httpfs;

CREATE OR REPLACE SECRET secret (
	TYPE s3,
	PROVIDER config,
	KEY_ID '...access-key-id...'
	SECRET '...secret-access-key...'
	ENDPOINT '192.168.1.3',
	URL_STYLE 'path',
	USE_SSL 'true',
	VERIFY_SSL 'false',
	REGION 'us-east-1'
);

SELECT *
FROM 's3://beegfs/hive_summary_counts_snapshot=20260702.parquet';

SELECT summary_count
FROM 's3://beegfs/hive_summary_counts_snapshot=20260702.parquet'
WHERE source_dir LIKE '/mnt/index/project1%';
```

`hive-import` creates:

- `hive_entries`: flattened entries table with source db metadata.
- `hive_summary_counts`: per-db row counts for entries and summary.

Snapshot behavior:

- `--snapshot` is a logical run identifier stored in the `snapshot_id` column.
- Running `hive-import` repeatedly with different snapshot IDs appends historical snapshots into the same DuckDB file.
- Running with the same snapshot ID is idempotent: existing rows for that snapshot are replaced before insert.

Flag style:

- Help text shows GNU-style flags like `--snapshot`.
- Short form with a single dash (for example `-snapshot`) is also accepted by Go's flag parser.

