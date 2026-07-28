# ES Rollback Plan

# **ES9 → ES6 Rollback Plan**

### **Technical Design & Operational Procedure**

|  |  |
| --- | --- |
| **Scenario** | Blue-green cutover ES6 → ES9 must be reversed |
| **Mechanism** | One-shot delta sync + ID-diff reconciliation (`scripts/es_rollback_py.sh`) |
| **Environment** | Self-managed Elasticsearch in Docker on GCP VMs |
| **Snapshot repository** | **None configured on either cluster** |
| **Downtime** | **Accepted** — ES9 frozen for the duration |

---

## **Table of Contents**

1. [Overview](https://git+.vscode-resource.vscode-cdn.net/d%3A/Workspace/es-migrate/docs/ES-ROLLBACK-PLAN.md?%7B%22path%22%3A%22d%3A%5C%5CWorkspace%5C%5Ces-migrate%5C%5Cdocs%5C%5CES-ROLLBACK-PLAN.md%22%2C%22ref%22%3A%22~%22%7D#1-overview) · 2. [Solutions Under Consideration](https://git+.vscode-resource.vscode-cdn.net/d%3A/Workspace/es-migrate/docs/ES-ROLLBACK-PLAN.md?%7B%22path%22%3A%22d%3A%5C%5CWorkspace%5C%5Ces-migrate%5C%5Cdocs%5C%5CES-ROLLBACK-PLAN.md%22%2C%22ref%22%3A%22~%22%7D#2-solutions-under-consideration) ·
2. [Finalized Solution](https://git+.vscode-resource.vscode-cdn.net/d%3A/Workspace/es-migrate/docs/ES-ROLLBACK-PLAN.md?%7B%22path%22%3A%22d%3A%5C%5CWorkspace%5C%5Ces-migrate%5C%5Cdocs%5C%5CES-ROLLBACK-PLAN.md%22%2C%22ref%22%3A%22~%22%7D#3-finalized-solution) · 4. [Delta Sync Script Design](https://git+.vscode-resource.vscode-cdn.net/d%3A/Workspace/es-migrate/docs/ES-ROLLBACK-PLAN.md?%7B%22path%22%3A%22d%3A%5C%5CWorkspace%5C%5Ces-migrate%5C%5Cdocs%5C%5CES-ROLLBACK-PLAN.md%22%2C%22ref%22%3A%22~%22%7D#4-delta-sync-script-design) ·
3. [Rollback Procedure](https://git+.vscode-resource.vscode-cdn.net/d%3A/Workspace/es-migrate/docs/ES-ROLLBACK-PLAN.md?%7B%22path%22%3A%22d%3A%5C%5CWorkspace%5C%5Ces-migrate%5C%5Cdocs%5C%5CES-ROLLBACK-PLAN.md%22%2C%22ref%22%3A%22~%22%7D#5-rollback-procedure) · 6. [Validation & Verification](https://git+.vscode-resource.vscode-cdn.net/d%3A/Workspace/es-migrate/docs/ES-ROLLBACK-PLAN.md?%7B%22path%22%3A%22d%3A%5C%5CWorkspace%5C%5Ces-migrate%5C%5Cdocs%5C%5CES-ROLLBACK-PLAN.md%22%2C%22ref%22%3A%22~%22%7D#6-validation--verification) ·
4. [Risks & Limitations](https://git+.vscode-resource.vscode-cdn.net/d%3A/Workspace/es-migrate/docs/ES-ROLLBACK-PLAN.md?%7B%22path%22%3A%22d%3A%5C%5CWorkspace%5C%5Ces-migrate%5C%5Cdocs%5C%5CES-ROLLBACK-PLAN.md%22%2C%22ref%22%3A%22~%22%7D#7-risks--limitations) · 8. [Roles](https://git+.vscode-resource.vscode-cdn.net/d%3A/Workspace/es-migrate/docs/ES-ROLLBACK-PLAN.md?%7B%22path%22%3A%22d%3A%5C%5CWorkspace%5C%5Ces-migrate%5C%5Cdocs%5C%5CES-ROLLBACK-PLAN.md%22%2C%22ref%22%3A%22~%22%7D#8-roles--responsibilities) ·
5. [Future Improvements](https://git+.vscode-resource.vscode-cdn.net/d%3A/Workspace/es-migrate/docs/ES-ROLLBACK-PLAN.md?%7B%22path%22%3A%22d%3A%5C%5CWorkspace%5C%5Ces-migrate%5C%5Cdocs%5C%5CES-ROLLBACK-PLAN.md%22%2C%22ref%22%3A%22~%22%7D#9-future-improvements) · 10. [Appendix](https://git+.vscode-resource.vscode-cdn.net/d%3A/Workspace/es-migrate/docs/ES-ROLLBACK-PLAN.md?%7B%22path%22%3A%22d%3A%5C%5CWorkspace%5C%5Ces-migrate%5C%5Cdocs%5C%5CES-ROLLBACK-PLAN.md%22%2C%22ref%22%3A%22~%22%7D#10-appendix--operational-reference)

---

## **1. Overview**

### **1.1 Background**

ES6 served all traffic until cutover, when its contents were copied to ES9 by a one-time reindex-from-remote and all traffic moved across. **ES6 has received no writes since.**

Reversing the cutover is therefore not a matter of redirecting traffic. ES9 has since accepted creates, updates, and hard deletes that ES6 knows nothing about; pointing traffic back today would serve stale data and resurrect deleted documents. ES6 must first be brought to parity.

### **1.2 Objectives and Scope**

- **Correct:** ES6's live document set and contents identical to ES9's — no missing creates, no stale updates, no documents ES9 deleted.
- **Reversible:** if the rollback goes wrong or is withdrawn midway, ES6 must be restorable to its exact pre-rollback state.
- **Auditable:** one execution path, one log, one state directory per index.
- **Bounded:** a window sized from measured numbers, with a defined behavior on overrun (resume, not restart).

**Scope.** One ES9 index or alias → one ES6 index or alias per invocation. Multiple indices run the tool once each with its own state directory; the index → `TS_FIELD` map must be fixed before execution.

**Non-objectives.** Near-zero-downtime rollback; continuous replication; resolving ES6/ES9 mapping incompatibilities (see [MAPPING-DIFFERENCES.md](https://git+.vscode-resource.vscode-cdn.net/d%3A/Workspace/es-migrate/docs/MAPPING-DIFFERENCES.md) — an input to this plan); the traffic-flip mechanism itself.

### **1.3 Constraints and Assumptions**

|  | Constraint | Consequence |
| --- | --- | --- |
| **C1** | No snapshot repository, and none can be created retroactively | The tool must supply its own recovery path — there is no `_restore` fallback |
| **C2** | Downtime accepted; ES9 frozen throughout | A delta window and an ID diff are both meaningless against a moving source, so the freeze is *verified*, not trusted |
| **C3** | Self-managed Docker on GCP VMs | No orchestrator restarts a failed process — long work must checkpoint and be re-enterable by hand |
| **C4** | Target is Elasticsearch 6.8 | No Point-in-Time API; the two sides of a comparison must use different snapshot mechanisms |

**Assumption A1 — every write on ES9 bumped its timestamp field.** Required for a timestamp-windowed delta to be complete. Not trusted blindly: §4.3.4's reverse diff detects and repairs any create that violated it, and `TS_FIELD=all` removes the assumption entirely.

---

## **2. Solutions Under Consideration**

Six approaches evaluated. Four are ruled out structurally — recorded explicitly because several are the obvious answers a reviewer would ask about.

| Option | Verdict |
| --- | --- |
| **Snapshot & restore** | **Impossible twice over.** No repository exists (C1), and even with one, snapshots restore *forward* only — a 9.x snapshot cannot be restored into 6.8 under any configuration. |
| **Cross-cluster replication** | **Rejected.** Requires a Platinum license this deployment lacks; requires the follower to be same-or-newer than the leader (here it is seven majors older); and a follower index is read-only. |
| **Reindex-from-remote (ES9→ES6)** | **Rejected.** ES 6.8 does not support a 9.x source. Independently unsuitable: cannot detect hard deletes, no resume, no pre-image capture. |
| **Dual write** | **Rejected on timing.** A preventive design that must precede cutover; it cannot recover writes ES6 has already missed. Correct answer for next time (§9). |
| **Logstash delta + separate delete script** | **Viable, rejected.** The original PoC — demonstrates both mechanisms but was never built to carry operational guarantees. Compared below. |
| **Purpose-built delta sync script** | **Chosen (§3).** |

The two viable options compared:

|  | Logstash + delete script | Purpose-built script |
| --- | --- | --- |
| Moving parts | 2 tools invoked independently, 1 ephemeral GKE cluster | 1 script, 1 state directory |
| Ordering safety | **Operator-enforced** — nothing stops the delete pass running on an incomplete sync | **Tool-enforced** — a hard gate makes that sequence unreachable |
| Resume | Not designed for it; the delete script has no resume concept | Every phase checkpoints; `resume` continues from the exact page |
| Recovery under C1 | **None** — no pre-image, no undo | Write-ahead journal; `undo` restores ES6 exactly |
| Infrastructure | Ephemeral GKE cluster (quota, provisioning, teardown) | `curl`, `jq`, `python3` on one host |
| After a mid-run crash | Undefined — pipeline and script state can disagree | Defined — cursor advances only after confirmed writes |

---

## **3. Finalized Solution**

**Decision:** `scripts/es_rollback_py.sh`**.** Two environment facts make this a conclusion rather than a judgment call.

**C1 makes undo the deciding criterion.** With no snapshot repository, whichever tool is used must supply its own recovery path or the rollback becomes a one-way door. Only the purpose-built script captures a pre-image before modifying anything. A rollback that cannot itself be rolled back is not acceptable where no backup layer exists.

**C2 removes the argument for a streaming tool.** Logstash's value is low-latency propagation against a live source. With ES9 frozen there is no latency to optimize; a one-shot batch pass is the natural fit and is simpler to verify and recover.

Secondarily, dropping the ephemeral GKE dependency removes a provisioning and teardown step, its quota requirements, and its own failure modes from a time-boxed window on infrastructure with no orchestration layer (C3).

**Architecture.** One process on one host: reads ES9 via PIT + `search_after` (and a sliced PIT walk for full ID sets), writes ES6 via byte-capped `_bulk`, captures pre-images via realtime `_mget` into a local write-ahead journal, and persists phase, cursor, counters, journal, and dead letters under one `STATE_DIR`. No external system participates.

---

## **4. Delta Sync Script Design**

### **4.1 Requirements and Guarantees**

| \# | Requirement | Driver |
| --- | --- | --- |
| R1 | Apply documents **created** on ES9 after cutover | §1.1 |
| R2 | Apply documents **updated** on ES9 after cutover | §1.1 |
| R3 | Apply **hard deletes**, which leave no trace to query | §1.1 |
| R4 | **Resumable** after any interruption, without restarting | C3 |
| R5 | **Reversible** — restore ES6 exactly, with no snapshot available | C1 |
| R6 | **Verifiable** — prove parity before traffic moves | §1.2 |

Two implicit requirements are load-bearing: the run must be **idempotent** (R4 is worthless if re-processing a page corrupts data), and it must **fail closed** — every ambiguous condition stops the run, because under C1 a wrong decision is not recoverable from infrastructure.

The design meets these with the following guarantees, which §6 tests against:

| Guarantee | Basis |
| --- | --- |
| **Idempotent** | All writes keyed by `_id`; whole-document overwrite, never partial update |
| **No lost creates or updates** | Timestamp window plus the reverse-diff repair pass backstopping A1 |
| **No surviving deletes** | Live ID-set diff, gated on a complete sync |
| **No duplicates** | `_id`-keyed writes; source IDs preserved |
| **Crash-safe** | Cursor advances only after confirmed writes; state written atomically |
| **Reversible** | Write-ahead journal with lowest-seq-wins reduction |
| **No silent partial success** | Per-item bulk parsing, dead-letter capture, and a gate that refuses to continue past them |

**Not guaranteed:** consistency against a *live* ES9 — the design depends on C2's freeze, which is why it is verified. Nor anything about documents ES6's mapping rejects; those are dead-lettered and surfaced, not coerced.

### **4.2 Change Detection Strategy**

The conceptual core. Elasticsearch has no change log, so the three change types need two different detection mechanisms.

**Creates and updates (R1, R2) — timestamp window.** Both are detected identically: any document whose `TS_FIELD` exceeds the cutover timestamp has changed since ES6 last saw it. Both are applied identically too, as a full-document overwrite keyed by `_id` — no need to distinguish new from modified, which is what makes the write path naturally idempotent. The cutover boundary is not supplied from memory: `reindex_remote.sh` records it as `_meta.cutover_at` on the ES9 index, and preflight reads it there.

**Hard deletes (R3) — live ID set diff.** A deleted document leaves nothing to query: no visible tombstone, no timestamp to range over. A timestamp-windowed sync structurally *cannot* find deletes — the gap any naive "just sync the delta" approach silently leaves open. The solution compares the two clusters' complete live `_id` sets:

- `ids(ES6) − ids(ES9)` → deleted on ES9 → **delete from ES6**
- `ids(ES9) − ids(ES6)` → missed by the delta → **repair into ES6**

The second direction is the safety net for A1: a create that failed to bump `TS_FIELD` is invisible to the window but caught here. It also recovers documents dead-lettered earlier in the run. Both directions are always computed, since equal counts do not imply equal ID sets — N creates plus N deletes leave the totals matching while both sets differ.

**The dependency this creates.** The delete decision is derived from comparing ID sets, so it is only valid once ES6 holds every create and update. Run the diff against a half-finished sync and the operator is left believing a partial rollback succeeded. This is why the phases have a fixed order and why §4.3.3 exists as a hard gate between them.

**No-timestamp fallback.** For an index with no trustworthy write timestamp, `TS_FIELD=all` replaces the window with a full copy of every live document — correct by construction, at a cost proportional to index size rather than delta size. Delete detection is unchanged.

### **4.3 Execution Model and Phases**

A linear state machine persisted to `$STATE_DIR/state.env`:

```
(new) → DELTA_SYNC → DELTA_DONE → GATE_PASSED → RECONCILE_DONE → DONE
```

`run` drives it to `DONE`; `resume` re-enters at the recorded phase; `status` reports it without taking the lock. **No transition reaches** `RECONCILE_DONE` **without passing** `GATE_PASSED` — §4.2's ordering dependency is enforced by the machine, not by operator discipline.

**Checkpoint granularity** is one search page (Phase 1) or one bulk batch (Phase 3), not one phase, so a crash costs at most the current page — which makes `resume` the normal response to an interruption rather than a last resort (R4, C3).

**Checkpoint ordering** is deliberate: the cursor advances only *after* a page's writes are confirmed. A crash in between re-processes one page, which is harmless given `_id`-keyed writes; the reverse ordering would let a crash skip a page permanently.

**Resume identity guard.** `resume` refuses if `SRC_INDEX`, `DST_INDEX`, or `TS_FIELD` differ from the recorded run — `SEEN` and `TOTAL_HITS` were measured against one window, and continuing against another would invalidate the gate.

**Durable vs. scratch state.** `$STATE_DIR` holds what resume and undo depend on; `$STATE_DIR/work/` is scratch any run may delete. The separation lets scratch be cleared to reclaim disk mid-incident without destroying recoverability.

The five phases follow, each described as purpose, mechanism, and the guard that can stop it.

**4.3.1 Phase 0 —** `PREFLIGHT`

*Read-only; safe to re-run repeatedly before the window opens.*

- **Index resolution.** Aliases resolve to concrete indices. An alias spanning more than one index is fatal rather than guessed.
- **Write availability.** ES6's `index.blocks.*` are inspected — after a long idle period a node may have crossed its flood-stage watermark and gone read-only, which would otherwise fail every bulk write individually. The exact remediation `curl` is printed.
- **Cutover boundary.** `_meta.cutover_at`, or an explicit `SINCE`. Values unparseable or in the future are fatal.
- **Freeze verification** — the check C2 depends on. Two probes, `FREEZE_WAIT` apart, must match:

| Mode | Probe |
| --- | --- |
| Timestamp | `_count` and `max(TS_FIELD)` |
| `TS_FIELD=all` | `_count` and the primary shards' `indexing.index_total` / `delete_total` |

The `all`-mode probe differs because `_count` alone cannot see an in-place update; the indexing counters advance on every index and delete operation. Primaries only, as replica counters move independently during recovery. Note the deliberate bias: a shard relocation inside the window resets these counters and reads as "still writing," aborting the run — a false positive chosen over the alternative of operating on a moving source.

**4.3.2 Phase 1 —** `DELTA_SYNC`

*Applies R1 and R2.*

**Pagination.** ES9 is read inside a PIT (`keep_alive=15m`) and paged with `search_after` at `PAGE_SIZE` (default and ceiling 10 000 — `index.max_result_window` caps `size`, and `search_after` does not lift it). Sort key `(TS_FIELD asc, _shard_doc asc)`; `_shard_doc` is the cheapest total order ES offers and exists only inside a PIT. The PIT, rather than a plain paged search, is what keeps the source snapshot stable across pages.

**Per-page pipeline:** `fetch → chunk → journal pre-image → bulk write → checkpoint`

1. **Chunk** into `_bulk` bodies capped at 5 MB, preserving `_routing`.
2. **Journal** ES6's current copy of every `_id` via `_mget`, **before any write** (R5). `_mget` is used because it is realtime — it reads the translog — so it returns the true current value even for a document written moments earlier; a search-based read could journal a stale pre-image and silently corrupt the undo path.
3. **Write** via `_bulk`.
4. **Checkpoint** cursor and counters.

`TS_FIELD=all` **mode.** Query becomes `match_all` sorted by `_shard_doc` alone. With no watermark to resume from, a PIT expiry restarts the walk and resets `SEEN` with it — otherwise the gate would weigh a two-pass total against a one-pass `TOTAL_HITS` and pass a walk that never finished. Restarting is safe, not merely wasteful: every write is a whole-document overwrite of a frozen source.

**4.3.3 Phase 2 —** `DELTA_GATE`

*The correctness boundary. No data movement — a decision only.*

Two conditions: `SEEN >= TOTAL_HITS` (the walk reached the end of the cursor rather than merely stopping — `SEEN` may exceed the total after a PIT-restart overlap, but must never fall short), and **zero dead letters** unless `ALLOW_PARTIAL=true`.

**Why it exists.** Per §4.2, Phase 3's delete decision is only meaningful once ES6 holds every create and update. Without this gate, a sync that stopped early flows straight into a delete pass acting on an incomplete picture — and the operator sees a run that reached the end and reasonably concludes it succeeded. Under C1 there is no snapshot to compare against later, so that misreading is not recoverable by infrastructure. The gate converts a silent, plausible-looking failure into an explicit stop. This is exactly what §2.1's alternative cannot provide, and the strongest single reason for the §3 decision.

`ALLOW_PARTIAL` is legitimate only when the operator has read the dead letters, knows which documents will be absent, and accepts that Phase 3 may then delete them outright — not a way to unblock undiagnosed failures.

**4.3.4 Phase 3 —** `RECONCILE`

*Applies R3.*

**ID export.** Both indices refreshed, counted, then fully walked for live `_id` sets. Per C4 the mechanisms differ by necessity:

| Side | Mechanism |
| --- | --- |
| ES9 (9.x) | Sliced PIT + `search_after`, one PIT shared across slices for a single consistent snapshot |
| ES6 (6.8) | Sliced scroll — no PIT API; each slice carries its own `scroll_id` |

Both are consistent snapshots. Plain `search_after` on `_doc` would not be: `_doc` ordering shifts as segments merge, silently skipping or duplicating IDs — unacceptable as input to a delete decision.

**Completeness guard.** Each side's exported ID count is checked against its own `_count` before any diff. A truncated export is indistinguishable from "these documents no longer exist" and would translate directly into deleting live data. Mismatch is fatal.

**Diff.** `comm` over the sorted lists under `LC_ALL=C`. The locale is not cosmetic — a UTF-8 collation ignores punctuation, so two distinct IDs can compare equal and be paired wrongly, which here means deleting the wrong documents. Deletes journal their pre-image unconditionally first, being the only irreversible operation; repairs are fetched from ES9 by `_mget`, carrying `_routing`.

**Delete-magnitude guard.** A delete set exceeding `MAX_DELETE_RATIO` of ES6's count (default 10%) stops the run and prints a sample unless `ASSUME_YES=true`. Not redundant with the completeness guard: that catches a *truncated* export, this catches a *complete* export of an unexpected reality — an unannounced bulk deletion on ES9, for instance.

**4.3.5 Phase 4 —** `VERIFY`

Executes the automated half of §6; read-only, and advances to `DONE` only on a clean pass. Detail and acceptance criteria in §6.1.

### **4.4 Journal and Undo**

The component C1 forces into existence (R5). Every mutating operation appends a row *before* the mutation:

```
seq \t id \t op \t found \t source
```

`found=0` records that the document did not exist on ES6 beforehand, so undoing it is a *delete*, not a restore. Repairs are journaled directly as `found=0`, since those IDs are absent from ES6 by construction.

**Lowest-**`seq`**-wins.** `undo` first reduces the journal to the lowest `seq` per ID. This is essential: a `resume` after a partial failure can journal an ID twice, and the later row holds what is really a *post*-sync image. Taking the minimum recovers the true original however many times the run was interrupted.

**Per-run isolation.** A new `run` archives any existing journal rather than appending — sequence numbers restart at 0 each run, so mixing two runs would make "lowest seq" ambiguous and let `undo` pick a post-sync image as the original. Operationally: `undo` **reaches back one run only**; an archived journal requires manual replay.

**Residual gap.** The journal protects against a bad rollback, not against loss of the journal itself, and under C1 there is no second layer. A cold copy of ES6's data volume immediately before the run is cheap insurance — outside the script's scope, but recommended:

```bash
docker run --rm -v <es6_data_volume>:/data -v "$PWD":/backup busybox tar czf /backup/es6-preop-$(date -u +%Y%m%dT%H%M%SZ).tgz /data
```

With the container stopped or quiesced. It converts "the journal is the only backup" into "the journal is the only *online* backup."

### **4.5 Error Handling**

**Transport.** Connect and max timeouts; retry on `429`/`502`/`503`/`504` and connection failure with exponential backoff plus jitter, capped at `MAX_RETRY` (6) and 60s per sleep. Jitter matters because a pressured cluster would otherwise receive synchronized retry waves from every slice.

**Bulk responses.** Elasticsearch returns HTTP 200 for a `_bulk` in which individual items failed, so per-item results are always parsed — the outer status is never treated as success on its own:

| Class | Examples | Handling |
| --- | --- | --- |
| Transient | `429`, `503`, `es_rejected_execution`, circuit breaker | Retried with backoff up to `MAX_RETRY` |
| Permanent | mapping conflict, malformed document | Dead-lettered immediately; run continues |
| Retry-exhausted | transient, still failing after `MAX_RETRY` | Dead-lettered, flagged distinctly |

Dead letters keep status, error body, and the original action and source lines — enough to re-apply by hand once the cause is fixed. They are what the gate refuses to pass.

**PIT expiry.** Detected as `search_context_missing`; the script reopens a PIT and resumes from the last watermark, switching the range predicate from exclusive (`gt`) to **inclusive (**`gte`**)**. Load-bearing: a page boundary can fall inside a group of documents sharing one timestamp, and resuming with `gt` would skip the rest of that group permanently. Re-reading it is idempotent; skipping it is data loss. Bounded at 10 restarts before aborting with a tuning recommendation.

**Scroll loss (ES6).** A scroll cannot resume from the middle, and a truncated export understates ES6's ID set — which becomes wrong deletes. The slice fails outright and `resume` restarts the export.

**Fail-closed on missing pre-images.** If the pre-image `_mget` fails, the run aborts *before* writing. Writing without a journal is unacceptable, not degraded (C1).

### **4.6 Performance Design**

Dominant costs are the Phase 1 delta walk and the two Phase 3 ID walks:

- `search_after` **over** `from`**/**`size` — flat cost per page instead of quadratic degradation, and not capped by `max_result_window`.
- **PIT over repeated searches** — one stable snapshot, avoiding both query re-execution per page and the consistency problems of paging a changing index.
- **Byte-capped bulk chunks (5 MB)** rather than document-count batching, since document sizes vary and a fixed count yields unpredictable heap pressure on the target.
- `MGET_BATCH` **decoupled from** `PAGE_SIZE` — an `_mget` returns full `_source` for every ID requested, so batching pre-image reads at page size would hand the parser a response orders of magnitude larger than needed. Different resources, tuned independently.
- **Sliced ID walks**, one per shard by default. Exceeding shard count is a trap, not a knob: beyond it Elasticsearch splits *within* a shard using a per-document hash filter — O(N) plus a bitset per slice — which can be slower than not slicing. The script warns past that boundary.
- **External sort with a bounded memory budget**, so the diff scales beyond available RAM.

**Two standard optimizations deliberately skipped.** *Disabling* `refresh_interval` is the usual bulk-load advice, but explicit `_refresh` calls at the end of Phases 1, 3 and 4 already make writes visible to the ID export, whereas a periodic refresh would add a segment per interval across all of Phase 1 and feed the merge queue on the disk that is already the bottleneck. *Dropping replicas to zero* would speed writes but removes redundancy from the very index being restored, at the moment losing it would be least recoverable (C1).

---

## **5. Rollback Procedure**

Downtime is accepted (C2), so there is no staged traffic-shifting choreography — a straight sequential pass with explicit stop-and-decide points rather than automation past a failure.

### **5.1 Preconditions**

The window must be approved and communicated, and every ES9 writer — application traffic, batch jobs, ingest pipelines — identified in advance. Step 0 stops them, but the inventory is not something to discover live.

`TS_FIELD` confirmed per index (or `all` where no trustworthy timestamp exists). The execution host needs reach to both `ES6_URL` and `ES9_URL` — internal VPC addresses (`10.146.0.10` / `10.146.0.11`) if run from a bastion or either ES VM — plus `curl`, `jq`, `python3`, `awk`, `sort`, `comm`, `gzip`, `split`; `jq` in particular is absent from a stock Ubuntu image.

`ELASTIC_PW` should come through the existing secret-handling process, never hardcoded into shell history or a checked-in file — worth restating because this deployment runs plain HTTP with a shared `elastic` superuser, which makes the credential easy to treat casually.

Two things must already be true: §6.4's rehearsal has passed recently enough that neither script nor environment has drifted, and a rollback owner and traffic-flip approver (§8) are named. As extra margin under C1, take §4.4's cold volume backup immediately before Step 0.

### **5.2 Execution**

**Step 0 — Freeze ES9.** Stop or fence every writer. Phase 0 will catch a missed one and refuse to proceed, but catching it there costs a wasted preflight cycle rather than a graceful mid-sync abort. Confirm at the infrastructure level — scale writers to zero, revoke the write credential, or block at a proxy — not via an application-level "pause" flag.

**Step 1 — Preflight** *(Phase 0 only)*

```bash
export ES6_URL=http://10.146.0.10:9200
export ES9_URL=http://10.146.0.11:9200
export SRC_INDEX=bench-es9
export DST_INDEX=bench-es6
export TS_FIELD=updated_at        # or "all"
export ELASTIC_PW='<from secret store>'
export STATE_DIR=/data/rollback/bench

./es_rollback_py.sh preflight
```

Expect a log ending in the freeze confirmation and resolved lower bound. Writes nothing, so re-run as needed. Do not proceed on a failed preflight.

**Step 2 — Execute** *(Phases 0 → 4)*

```bash
./es_rollback_py.sh run
```

Tail `$STATE_DIR/run.log` in a second terminal — the `PROGRESS_EVERY` heartbeat is what distinguishes a long ID walk from a hang.

**Step 3 — Monitor**

```bash
./es_rollback_py.sh status
```

Safe from any host with access to `STATE_DIR`, including against a running process — it reads state and never takes the lock.

**Step 4 — Resume, if interrupted**

```bash
./es_rollback_py.sh resume
```

Any crash or killed session leaves state checkpointed. `resume` validates run identity first (§4.3), so a mistakenly changed env var stops the run rather than corrupting it.

**Step 5 — Decision points.** If the run stops before `DONE`:

- **Gate failure** (`SEEN < TOTAL_HITS`, or dead letters) is *not* a decision point — it is fix-and-resume. Investigate `deadletter.ndjson`, resolve the cause (typically an ES6 mapping conflict), `resume`. Do not look for a way around it; §4.3.3 explains what it protects.
- **Delete-magnitude guard** *is* a decision point. Review the printed `to_delete` sample against what is actually known to have happened on ES9. Use `ASSUME_YES=true` only after that review, never as a reflexive unblock.

**Step 6 — Confirm verification.** `VERIFY PASSED` in the log, or `status` reporting `phase: DONE`, is the only accepted condition. Exit code `2` (dead letters) or any verify warning is a hold — escalate per §8. Full acceptance criteria: §6.

**Step 7 — Flip traffic to ES6.** External to this script: DNS, load balancer, or application config, mirroring whatever originally cut traffic to ES9. Re-enable ES6 writes if fenced.

**Step 8 — Business validation** per §6.3, before declaring the window closed.

### **5.3 Undo — Reverting the Rollback**

At any point before Step 7 — or after, if Step 7 proved premature:

```bash
./es_rollback_py.sh undo
```

Restores ES6 to exactly its pre-run state from the journal (§4.4). Two constraints: it covers only the *current* run's journal, and if a second `run` has started since, the earlier journal was archived and requires manual replay. Note the archive path when it is created.

If `undo` itself cannot produce a trustworthy ES6 — corrupted journal, disk failure, anything outside the tool's failure model — the fallback is unconditional: **stay on ES9**. Under C1 there is nothing further to fall back on, so traffic must never be flipped onto an ES6 that has not met §6's acceptance criteria.

### **5.4 Cleanup and Close-Out**

```bash
./es_rollback_py.sh reset
```

Clears `state.env` and scratch but deliberately preserves the journal and dead-letter file; remove those by hand only after the run is audited.

**Close-out record:** `RUN_ID`; final `SYNCED`, `DELETED`, `REPAIRED`, `DL_COUNT`; whether `ALLOW_PARTIAL` or `ASSUME_YES` was used and why; the verification result; traffic-flip timestamp; `STATE_DIR` path. This is what answers "what actually happened" after the window closes.

---

## **6. Validation & Verification**

Parity is not assumed from a clean exit code. Three levels, all required.

**Acceptance criteria — the rollback is complete when, and only when:** Phase 4 reports `VERIFY PASSED`; `status` reports `phase: DONE`; the exit code was `0` (not `2`); §6.2's independent review agrees; and §6.3's business validation passes after the flip. Anything short of all five is a hold escalated per §8 — not an operator judgment call.

### **6.1 Automated (Phase 4)**

| Check | Method | Catches |
| --- | --- | --- |
| **Count parity** | `_count` on both indices | Gross failure — an aborted phase, a wholesale miss |
| **Exact ID set** | Expected `(ids(ES6) − to_delete) + to_repair`, compared byte-for-byte against `ids(ES9)` | Reconcile arithmetic errors, partially applied batches |
| **Content sample** | `SAMPLE_N` docs (default 1000, fixed seed) compared on full `_source` | Documents present with wrong content — a stale update the window missed |

The ID-set check is computed from files already on disk, so it is nearly free, and it validates the reconcile *arithmetic* rather than re-measuring the outcome; on mismatch, ES6's IDs are re-exported into `verify_extra` and `verify_missing`. The content sample uses a fixed seed so re-runs compare the same documents — a shifting sample cannot distinguish a fixed defect from a lucky draw. `VERIFY PASSED` requires all three; verification never mutates and never rolls back automatically.

### **6.2 Operator Verification**

Before flipping traffic, independently confirm counts:

```bash
curl -s -u elastic:$ELASTIC_PW "$ES9_URL/$SRC_INDEX/_count"
curl -s -u elastic:$ELASTIC_PW "$ES6_URL/$DST_INDEX/_count"
```

Then check the `STATE_DIR` artifacts are plausible: `to_delete` and `to_repair` should have magnitudes consistent with the freeze period, and `verify_extra` / `verify_missing` should be absent or empty. A `to_delete` far larger than known deletion activity is a signal to stop even if Phase 4 passed. Finally, spot-check documents *known* to have been mutated on ES9 — a handful of known cases is a stronger signal than a random sample, because it tests the specific behavior the rollback was for.

### **6.3 Business Validation**

Index parity does not prove application correctness, and the script does not attempt to assess it. Before closing the window: exercise the application's real search paths against ES6 (the queries users actually run, not `match_all`); confirm writes succeed and become visible; check dashboards, reports, and downstream consumers; and verify a sample of records the business knows changed during the ES9 period end-to-end through the application rather than by querying Elasticsearch directly.

### **6.4 Pre-Production Rehearsal**

Scripts are syntax-checked locally but exercised for real on the GCP VM (`es9-dest`), never validated from a Windows dev box alone.

**Unit:** `test_es_rollback.sh` against `fake_es.py`, an in-memory ES stand-in — no cluster needed; run on every script change.

**Functional**, mirroring README §5: seed a small ES6 dataset, migrate to ES9 (writing the cutover marker), apply `simulate_es9_mutations.py` for creates, updates, and hard deletes. Then confirm a clean `VERIFY PASSED` with ES6 matching the mutated ES9 state; interrupt mid-`DELTA_SYNC` and confirm `resume` completes cleanly (the behavior production leans on most); deliberately trip both the delete-magnitude guard and the dead-letter gate so their real output is seen once beforehand; and **run** `undo` **and confirm ES6 returns exactly to its pre-run state** — under C1 this drill is the closest thing to proof that the only safety net works, and is mandatory before sign-off.

**Scale**, against the 8M-document benchmark fixture — for throughput numbers and `SLICES`/`PAGE_SIZE` tuning under load, not to re-prove correctness.

**Window sizing.** Take `TOTAL_HITS` from `preflight`; take docs/min for Phase 1 and IDs/min for Phase 3 from the scale rehearsal's heartbeat lines. Phase 3's ID walks scale with *total* live document count, not delta size, so they dominate once the delta is small relative to the index. Budget all phases plus one full contingency pass, so a single `resume` cycle does not overrun the window.

---

## **7. Risks & Limitations**

| Risk | Trigger | Mitigation in design | Residual owner action |
| --- | --- | --- | --- |
| ES9 not actually frozen | Missed writer | Two-sample freeze probe blocks preflight | Confirm the writer inventory before retrying |
| Rollback exceeds the window | Large index, or `TS_FIELD=all` | Fully resumable; `PAGE_SIZE`/`SLICES` tunable | Size from rehearsal data; budget a contingency pass |
| PIT or scroll expiry mid-walk | Long export under load | Auto-reopen, resume from watermark, bounded restarts | Raise `PAGE_SIZE` or reduce load if recurring |
| Oversized delete set | Real mass deletion, or truncated export | Completeness guard plus `MAX_DELETE_RATIO` | Review `to_delete` before any override |
| Dead-lettered documents | Mapping incompatibility | Gate blocks reconcile; full context captured | Fix root cause; override only with informed acceptance |
| ES6 disk exhaustion | Idle period plus catch-up volume | Preflight detects the block and prints the fix | Free headroom before starting |
| Clock skew on `cutover_at` | Marker written by a drifted host | Marker written server-side, not by an operator | If skew is suspected, set `SINCE` earlier — overlap is idempotent |
| Journal loss or corruption | Disk failure on the execution host | None — the journal *is* the safety net (C1) | Take §4.4's volume backup; put `STATE_DIR` on durable storage |
| Credential or network exposure | Plain HTTP, shared superuser | None in the script | Treat `ELASTIC_PW` as a secret; keep the path inside the VPC |
| Operator error — wrong index or variable | Manual command entry | Lock blocks concurrent runs; `resume` validates identity | One documented invocation per index; no ad hoc mid-run changes |

**Known limitations**, stated plainly:

- **The freeze is mandatory.** Every correctness argument depends on C2; there is no partial-freeze or online mode.
- **Delete detection enumerates all IDs**, so Phase 3 scales with total index size regardless of delta size. For a large index with a small delta, reconciliation dominates the window.
- `undo` **reaches back one run only.** Archived journals need manual replay.
- **Mapping incompatibilities are surfaced, not solved.** A document ES6 rejects is dead-lettered; the script has no opinion on coercing it.
- **The** `TS_FIELD=all` **freeze probe has a known false positive** — a shard relocation during sampling reads as ongoing writes and aborts the run, a deliberate bias toward refusing to start.
- `_id` **must be stable across both clusters.** An index whose IDs were regenerated during the original migration is out of scope.

---

## **8. Roles & Responsibilities**

| Activity | Responsible | Accountable | Consulted / Informed |
| --- | --- | --- | --- |
| Freeze ES9 writers, confirm complete | Execution engineer | Rollback owner | Application/service owners |
| Preflight, execute, monitor | Execution engineer | Rollback owner | — |
| Diagnose gate failure or dead letters | Execution engineer | Rollback owner | ES cluster owner |
| Approve `ALLOW_PARTIAL` / `ASSUME_YES` | Rollback owner | Rollback owner | Execution engineer (supplies evidence) |
| Approve traffic flip after §6's criteria | Traffic approver | Rollback owner | Stakeholders |
| Decide `undo` vs. proceed on ambiguity | Rollback owner | Rollback owner | Execution engineer |
| Business validation (§6.3) | Application owner | Rollback owner | Stakeholders |
| Communication and close-out | Rollback owner | Rollback owner | All stakeholders |

*(Assign names before finalizing for a specific execution date.)*

---

## **9. Future Improvements**

- **Dual write at the next cutover** — §2's correct preventive answer; removes the need for a delta sync entirely, at the cost of an application change made *before* the migration.
- **Configure a snapshot repository** — the highest-value change to this environment. It would not enable ES9 → ES6 restore, but it gives every future operation a recovery layer independent of the tool performing it, removing the largest residual risk in §7.
- **Persist checkpoints off-host** — `STATE_DIR` on the execution host is a single point of failure for recoverability.
- **Soft deletes at the application layer** — a `deleted_at` field makes deletes visible to a timestamp window, collapsing Phase 3's full ID enumeration into Phase 1 and removing the dominant cost for large indices.
- **Reduce the freeze requirement** — with soft deletes plus a monotonic version field, a second convergence pass could narrow the freeze to a short final cutover. Relevant only if C2 stops holding.

---

## **10. Appendix — Operational Reference**

### **10.1 Commands and Exit Codes**

| Command | Purpose | Writes? |
| --- | --- | --- |
| `preflight` | Phase 0 only — validate and stop | No |
| `run` | Fresh run, Phases 0 → 4 | Yes |
| `resume` | Continue from checkpoint | Yes |
| `status` | Phase and counters; safe against a running process | No |
| `verify` | Re-run Phase 4 alone | No |
| `undo` | Restore ES6 from the journal | Yes |
| `reset` | Drop state; keeps the journal | No |

**Exit codes:** `0` clean · `1` fatal, needs investigation · `2` finished with dead letters · `130` interrupted, safe to `resume`.

### **10.2 Environment Variables**

| Group | Variable | Default | Purpose |
| --- | --- | --- | --- |
| Connection | `ES6_URL` / `ES9_URL` | `http://localhost:9200` | Cluster endpoints |
|  | `SRC_INDEX` / `DST_INDEX` | `bench-es9` / `bench-es6` | Source and target index or alias |
|  | `TS_FIELD` | `updated_at` | Delta window field; `all` disables the window |
|  | `ES6_USER`/`PW`, `ES9_USER`/`PW` | `elastic` / `$ELASTIC_PW` | Authentication |
|  | `STATE_DIR` | `./.rollback-state` | Checkpoint and journal location |
| Performance | `PAGE_SIZE` | `10000` | Search page size (ES ceiling) |
|  | `MGET_BATCH` | `1000` | IDs per pre-image `_mget` and delete batch |
|  | `SLICES` | `auto` | ID-walk slices; `auto` = one per shard, `1` disables |
|  | `MAX_RETRY` | `6` | HTTP and bulk-item retry attempts |
|  | `PROGRESS_EVERY` | `50` | Pages between heartbeat lines; `0` silences |
| Safety | `MAX_DELETE_RATIO` | `0.10` | Blocks reconcile above this share of ES6 |
|  | `FREEZE_WAIT` | `10` | Seconds between freeze probes |
|  | `SAMPLE_N` | `1000` | Documents compared in Phase 4 |
|  | `SINCE` | *(none)* | Manual cutover override, ISO-8601 |
|  | `ALLOW_PARTIAL` | `false` | Gate passes with dead letters — informed override only |
|  | `ASSUME_YES` | `false` | Skips delete-magnitude confirmation — informed override only |

### **10.3 State Directory Layout**

| Path | Tier | Contents |
| --- | --- | --- |
| `state.env` | Durable | Phase, counters, run identity, watermark |
| `journal.tsv.gz` | Durable | Write-ahead pre-images — the basis of `undo` |
| `journal.<ts>.tsv.gz` | Durable | Archived journal from a previous run |
| `deadletter.ndjson` | Durable | Rejected documents with full context |
| `run.log` | Durable | Full run log |
| `pit_id.txt`, `search_after.json` | Durable | Phase 1 cursor — survives a crash |
| `es9_ids.sorted`, `es6_ids.sorted` | Durable | Phase 3 ID exports |
| `to_delete`, `to_repair` | Durable | Computed diff sets |
| `verify_*` | Durable | Phase 4 failure diagnostics |
| `lock/` | Durable | Single-writer lock with PID |
| `work/` | Scratch | Request bodies, bulk chunks, split parts — safe to delete |

### **10.4 Troubleshooting**

| Symptom | Cause | Action |
| --- | --- | --- |
| `ES9 is still taking writes` | A writer was missed | Stop it; re-run `preflight` |
| `<index> is write-blocked` | ES6 disk watermark | Free disk, apply the printed `curl`, re-run |
| `_meta.cutover_at missing` | Index not migrated by `reindex_remote.sh` | Pass `SINCE=<ISO-8601>` or `TS_FIELD=all` |
| `gate FAILED ... only N were read` | Delta sync incomplete | `resume` — never override |
| `gate FAILED ... rejected by ES6` | Mapping conflict or similar | Inspect `deadletter.ndjson`, fix, `resume` |
| `id export incomplete` | Truncated export | `resume` to restart it |
| `N deletes is more than X of ES6` | Delete-magnitude guard | Review `to_delete`; override only after review |
| `another es_rollback is running` | Concurrent invocation or stale lock | Check the PID; a dead one is reclaimed automatically |
| `state belongs to TS_FIELD=...` | Env var changed between run and resume | Restore the original, or `reset` and start over |

### **10.5 Related Documents**

`docs/ES-ROLLBACK.md` — full phase internals, Vietnamese operator guide · [design spec](https://git+.vscode-resource.vscode-cdn.net/d%3A/Workspace/es-migrate/docs/superpowers/specs/2026-07-21-es9-es6-rollback-design.md) — original rationale · `MAPPING-DIFFERENCES.md` — ES6/ES9 mapping differences · `scripts/es_rollback_py.sh` — source and usage header · `README.md` — environment topology.
