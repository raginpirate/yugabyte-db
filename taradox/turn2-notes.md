# Database lane, turn 2 notes (2026-10-17): the first two branches and the builder, code only

## Fork branches (local only, /home/raginpirate/src/yugabyte-db; nothing pushed)
| branch | base | commits |
|---|---|---|
| fix/notifications-poller-timeout-retryable | master 07b99390b25 | 00da85a78a3 [#31484] CDC: Retry a virtual WAL GetChanges that timed out, and answer before the walsender's deadline |
| backport/2026.1.2/fix/notifications-poller-timeout-retryable | 2026.1.2.0-b137 | 8c4e98238d7 [BACKPORT 2026.1.2][#31484] ... (clean cherry-pick) |
| fix/cdc-state-write-off-io-thread | master 07b99390b25 (has 753607d5149) | 575e48c11ab [#12173] CDC: Do not wait for the cdc_state write in the virtual WAL's expired-session cleanup |
| backport/2026.1.2/fix/cdc-state-write-off-io-thread | 2026.1.2.0-b137 | 14b6c4ba73d [BACKPORT 2026.1.2][#33853] (753607d5149, clean), c1a35476d00 [BACKPORT 2026.1.2][#12173] ... |

Worktrees (scratch, not branches of their own):
- /tmp/claude-1000/-home-raginpirate-src-taradox-hosted/dc513407-4aea-4622-9029-73500bca3002/scratchpad/database/yb-master (master-based branches)
- /tmp/claude-1000/-home-raginpirate-src-taradox-hosted/dc513407-4aea-4622-9029-73500bca3002/scratchpad/database/yb-backport (backport branches)
- /tmp/claude-1000/-home-raginpirate-src-taradox-hosted/dc513407-4aea-4622-9029-73500bca3002/scratchpad/database/yb-release: detached trial of the release tree, 2026.1.2.0-b137 + 8c4e98238d7 + 14b6c4ba73d + c1a35476d00 = 12c0280fa2f. All three apply with no conflict (the two tests were placed apart and the shared include line is identical, so they combine).

## Fix 7: what the logs said, and the change
- T24/T25's poller exits come through ONE branch of CDCSDKVirtualWAL::GetChangesInternal, the response-error branch (tag cdcsdk_virtual_wal.cc:1032), in both forms:
  - "GetCDCStream timed out after deadline expired, passed 1.509s of 1.500s" (client_master_rpc.cc:138), the stream metadata lookup;
  - "XClusterGetChangesRpc ... tablet_id 00000000000000000000000000000000 ... passed its deadline (passed 21.099s of 10.005s)" (rpc.cc:224): the poll of the master's catalog tablet (detect_publication_changes_implicitly) forwarded by TabletLeaderGetChanges (serve_as_proxy defaults true) to the dead master leader, with the walsender's own 10 s deadline, so pggate's call timed out first (outage180/H1 node 1 tserver WARNING lines 2090-2092: "took 21103ms (client timeout 10000ms)").
- Change: kTimedOut retryable for any message (cdcsdk_virtual_wal.h), and GetConsistentChanges gives the virtual WAL deadline - min(cdc_read_rpc_timeout_ms * cdc_read_safe_deadline_ratio = 3 s, half the time left) (cdc_service.cc GetVirtualWALDeadline), the pattern CDCServiceImpl::GetChanges already uses.
- Not changed: the RPC-level failure branch of GetChangesInternal (`!s.ok()`). With cdc_enable_local_rpc_in_virtual_wal (default true) the call is local and has no timer, so it cannot time out there. With the flag off it can, and stays fatal.

## Fix 4: the change
- DestroyVirtualWALBatchForCDC (the only caller is PgClientServiceImpl::SessionsRemoved, IO thread pool) writes active_pid=0 through a new CDCStateTable::UpdateEntriesAsync and does not wait; failure is a WARNING from the callback. DestroyVirtualWALForCDC (RPC handler) unchanged, still synchronous outside mutex_ (753607d5149).
- Ordering against a later InitVirtualWALForCDC is unchanged: the write is issued at the same moment; only the thread no longer waits.

## The builder (taradox v2-database 7d154fd)
- database/builder/compose.yaml, database/builder/build; database/Dockerfile YUGABYTE_SOURCE=image|release; .gitignore /database/release/.
- Pins: image yugabyteci/yb_build_infra_almalinux8_x86_64:v2026-09-10T10_14_48@sha256:6a9a7d74f78e58a15de514e49b6c9df213f654e38be8ff63d6f3d96a055bd0cb (checked with docker buildx imagetools inspect).
  third-party v20260826163332-c46477f491-almalinux8-x86_64-clang21-full-lto 7b797cbc827562bef7cc5c9e007c841f02fdcf5b420f7fce7c129769794fdf0a (1,371,227,316 bytes);
  master's v20261001194835-2c1852ba51-almalinux8-x86_64-clang21-full-lto 189c8fad89d56dc4c71defd7bb0fe79f5844302e270bb2f0b035a8bf83ecb3aa;
  LLVM v21.1.1-yb-2-1776376910-efca861c-almalinux8-x86_64 db68c3eab4b0f827ced2fe9d4438fd60b84eb0195e71b24852de46f4098ee5cd (333,637,439 bytes), the tag llvm-installer 1.5.2's release_tags.json selects (PyPI sdist sha256 db76a13f99adb9bafa353ea38f59fee290171ca9186375a4d7c8cf115eaab0ef, read, never run).
  PyPI: requirements_frozen.txt pins 47 packages by version only; no hashes.
- Not built: Java, yugabyted UI, Odyssey, YBC, pg_parquet. No PGO data, no BOLT (Yugabyte's CI adds them when YB_PGO_DATA_FILE/YB_PGO_BOLT are set; we have neither), so our yb-server is not byte-for-byte the official one's optimisation.
- Release name will be yugabyte-2026.1.2.0-<commit>-release-clang21-centos-x86_64.tar.gz (release_util.py maps almalinux x86_64 to "centos"); build number 137 from the nearest tag.

## Run plan, for when the lead frees the machine (not run)
All from /home/raginpirate/src/taradox-hosted/.claude/worktrees/v2-database, TARADOX_YUGABYTE_SOURCE=/home/raginpirate/src/yugabyte-db:
1. The release: flock -w 7200 /home/raginpirate/src/taradox-hosted/.git/taradox-run.lock timeout <hold> docker compose -f database/builder/compose.yaml run --rm -T builder release 12c0280fa2f
   (first run also pulls the 3.1 GB image and 1.7 GB of archives; estimate 60-90 min compile at 16 jobs plus 20-40 min for yb-server's LTO link: over the 3600 s hold).
2. Red first: the new tests at a scratch commit holding only the tests on the tag, expected to fail (TestTimeoutsNotSentToWalsenderWhileMasterIsDown: the 10 s RPC times out or the response errors; TestVirtualWALBatchCleanupDoesNotWaitForCDCState: about 60 s elapsed).
3. Green: builder test 12c0280fa2f integration-tests_cdcsdk_consumption_consistent_changes-test 'CDCSDKConsumptionConsistentChangesTest.TestRetryableErrorsNotSentToWalsender:CDCSDKConsumptionConsistentChangesTest.TestTimeoutsNotSentToWalsenderWhileMasterIsDown:CDCSDKConsumptionConsistentChangesTest.TestVirtualWALBatchCleanupDoesNotWaitForCDCState'
4. The image: database/Dockerfile defaults set to release + name + sha256, environment/agent database, the lane's test environment up, QUORUM x5 with the platform lane's forcer (scratchpad/timeout15/run-G.sh's leader_stepdown of the slots' cdc_state tablet onto node 1), then stacks stopped.
