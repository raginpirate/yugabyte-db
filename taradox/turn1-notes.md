# Database lane, turn 1 notes (2026-10-17): read and estimate, no code

Sources: /home/raginpirate/src/yugabyte-db at tag 2026.1.2.0-b137 (c5defa6484e, 2026-09-18) and master (07b99390b25, 2026-10-17).
Merge base 7f59219a01a (2026-04-21); master is 3,507 commits past it, the tag 1,032.
Tag files were extracted with `git show` to /tmp/claude-1000/-home-raginpirate-src-taradox-hosted/dc513407-4aea-4622-9029-73500bca3002/scratchpad/database/tag/
"T:" means the line is at the tag. "M:" means it is at master. Anything not read in source is marked (not read).

## How LISTEN/NOTIFY works today (T: src/postgres/src/backend/commands/async.c)
- NOTIFY inserts a row into pg_yb_notifications in the yb_system database, then deletes it in the same transaction (ybInsertPendingNotifiesToTable, T:2969-3069).
- The first LISTEN on a node registers itself under NotifyQueueLock (T:1379-1435). It then drops and remakes the slot yb_notifications_<tserver> outside the lock (T:1451, ybCreateNotifsReplicationSlot T:3074-3091) and starts the poller bgworker (T:1452, T:3097-3168).
- The poller reads the virtual WAL (YBCReadRecord, T:3260-3318) and acks every committed transaction (YBCCalculatePersistAndGetRestartLSN, T:3365). Any error makes it set has_runtime_error, signal every listener and exit (T:3296-3318).
- A LISTEN that is not the node's first is refused while has_runtime_error is set (T:1384-1406). Only a new poller clears the flag (T:3110).
- The last listener's unregister (T:1592-1617) calls ybCleanupListenState (T:1615). That drops the slot by RPC (ReplicationSlotDrop, T:1715) while still holding NotifyQueueLock exclusively.
- The only difference at master is in ybNotifsPollerInit: ReplicationSlotAcquire nowait=false (M:3226-3235).
- The master drops a node's slot when that tserver restarts (T: master_heartbeat_service.cc:1661-1668) or is removed (T: master_cluster_handler.cc:213-219, by DeleteNotificationsReplicationSlot at xrepl_catalog_manager.cc:2169-2200). It does not drop the slot when only a session dies.
- Cost: every NOTIFY is a Raft write. Every node that has a listener pulls every notification over its own CDC stream, and every listener on the node scans every entry. That is O(W·N_listening) on the network and O(W·S) deliveries.

## (a) Where every replica applies a committed write
- Leader and follower both apply through ReplicaState::ApplyPendingOperationsUnlocked (T: consensus/replica_state.cc:940). Followers call it from UpdateReplica when they learn the commit index (T: raft_consensus.cc:2206, :2399). That leads to OperationDriver::ReplicationFinished → ApplyTask (T: operation_driver.cc:332-380).
- Single-shard and non-transactional writes go WriteOperation::DoReplicated (T: write_operation.cc:123) → Tablet::ApplyRowOperations (T: tablet.cc:1936) → ApplyOperation (T:1966) → ApplyKeyValueRowOperations (T:2082). From there NonTransactionalBatchWriter::Apply (T: docdb/rocksdb_writer.cc:1323) writes each write pair through HandleRegularRecord (T:1346).
- Distributed transactions: UpdateTxnOperation::DoReplicated (T: update_txn_operation.cc:59-74) → TransactionParticipant ProcessReplicated (APPLYING at T: transaction_participant.cc:904) → ProcessApply (T:983) → applier_.ApplyIntents (T:1049) → Tablet::ApplyIntents (T: tablet.cc:2673-2711). That runs ApplyIntentsContext::Entry per intent (T: rocksdb_writer.cc:788-880, the regular-DB put at T:873).
- Precedent for a per-key hook on every replica: VectorIndexesUpdater::Feed is called on both paths, at T: rocksdb_writer.cc:877 (intents) and :1343 (non-transactional); at M: :938 and :1431. snapshot_coordinator_->ApplyWritePair (T: tablet.cc:2132) is another.
- Decoding is done by CDC already: SubDocKey::DecodeFrom, doc_key().hashed_group() and range_group() against the tablet schema (AddPrimaryKey T: cdc/cdcsdk_producer.cc:249-277), columns (AddColumnToMap T:177-229), packed rows (T:660-727), write records (T:1385-1470), intent records (T:981). pggate converts those to PG datums (T: ybc_pggate.cc:2924, PBToDatum about 3018 and 3037).
- For us: entries, participants and dispositions hash on conversation_id (server/schema.sql:353-411). A watch key is (table id, hashed group). participants_stays (user_id, …) is its own index table with its own tablets, so it can be watched by user_id.
- Follower apply lag: when the commit index advances, the leader signals its peers at once (T: raft_consensus.cc:1525-1539). A request that carries only a commit-index advance is not treated as a heartbeat, so it is sent at once (T: consensus_peers.cc:370-378). Followers also get propagated safe time on every request (T: raft_consensus.cc:2224-2238; mvcc.h:98).

## (b) RPC between tablet servers
- Services are registered in TabletServer::RegisterServices (T: tablet_server.cc:693-805; M:822). The existing services are TabletServerService, ConsensusService (consensus.proto T:630), CDCService and others.
- yb::rpc is unary only: no streaming methods were found in src/yb/rpc. A call can be answered later because RpcContext is movable (T: rpc_context.h:180-215), so a long poll is possible. MultiRaftHeartbeatBatcher (T: consensus/multi_raft_batcher.h:45) is the per-destination batching model.

## (c) Locations, leaders, moves, splits
- Remote replicas come from the tserver's YBClient MetaCache: LookupTabletByKey (T: meta_cache.cc:2264), ProcessTabletLocations (T:1039), MarkReplicaFailed (T:479) and MarkAsSplit (T:469). A split is detected by the TABLET_SPLIT error (T: tablet_rpc.cc:435).
- Local replicas: consensus state changes come through TSTabletManager::ApplyChange/MarkTabletDirty (T: ts_tablet_manager.cc:2863-2878). Splits are applied on every replica through SplitOperation::DoReplicated (T: split_operation.cc:123), which calls TSTabletManager::ApplyTabletSplit (T: ts_tablet_manager.cc:1234). Deletions go through DeleteTablet (T:1925).
- The master pushes only the tserver list in its heartbeat answer (T: master_heartbeat_service.cc, resp->add_tservers in TSHeartbeat). It pushes no locations.

## (d) Backend to its tserver
- PgClientService (T: tserver/pg_client.proto:33…). Today pggate calls CDCService directly for the virtual WAL: InitVirtualWALForCDC, GetConsistentChanges and DestroyVirtualWALForCDC (T: pggate/pg_client.cc:1770-1895).
- The default deadline is ysql_client_read_write_timeout_ms + pg_client_extra_timeout_ms (T: pg_client.cc:150-205).
- When a session expires, PgClientServiceImpl::SessionsRemoved runs (T: pg_client_service.cc:2977-2983) from SessionRegistryBase::CleanupSessions (T: session_registry.cc:211-243).

## Defect readings (tag versus master)
1. Heartbeat circle: PopulatePgCatalogVersionInfo is called at T: master_heartbeat_service.cc:563, after the report is processed. It reads every row (T:302-336), and the cache is off by default (T: catalog_manager.cc:590). M: the default is true (catalog_manager.cc:602, #31551 9a7de4294d0; the backport acd0ce9cced kept false on 2026.1). M: when the cache is empty the read still falls through (catalog_manager.cc:11381-11397). Not fixed.
2. Write with no deadline: the local call has no timer. M f7768590c8a [#33277] (2026-10-17) adds a local-call timeout, but only for PgClientSession Perform: "other YBSession users keep waiting" (commit message). Not in the tag. It touches 22 files (+360/-49).
3. Pre-vote denial: on becoming leader it withholds votes forever (T: raft_consensus.cc:1133) and denies every other candidate (T:2486-2490); M:1165 and :2544 are the same. No check-quorum exists at the tag or master. Not fixed.
4. Synchronous cdc_state write under the lock: T: cdc_service.cc:5771-5800. M 753607d5149 [#33853] (2026-09-11) moves the write outside mutex_ (M:5788-5825; not in the tag). The write is still synchronous on the IO-pool thread (TODO #12173 at M: cdc_state_table.cc:642, :661). Issue #12173 has been open since 2022.
5. Barrier kept: CDCMasterBgTask skips when no stream is in cdc_state (T: cdc_service.cc:3624-3630). M:3638 is the same. Not fixed. Mitigated by our flag (start.rs:884, T18).
6. Stream outliving a killed session: the drop runs at exit (T: async.c:1715). "Shutdown connection" is the reactor's message (T: rpc/reactor.cc:119). Open issue #31910 "Drop notifications replication slot when YSQL operation lease expires". Not fixed.
7. Poller exits on a timeout: retryable patterns (T: cdcsdk_virtual_wal.h:47-53; M the same). By design (d94d4130c45 #31484) every other error is fatal. Not fixed.
8 and 9. Refused LISTEN, and LISTEN accepted with no poller: T: async.c:1384-1406, 1435-1452, 1688-1720. M is the same. Not fixed.
10. The 60 s alter backoff: retrying_ts_rpc_max_delay_ms (T: async_rpc_tasks_base.cc:50, backoff :334-341). The throttler is max_concurrent_alter_table_rpcs=10 (T: xrepl_catalog_manager.cc:163). M 510bcefc936 [#33260] (2026-10-16) sets the throttler to 0. The failure backoff is unchanged.
+. The slot drop runs under NotifyQueueLock: T: async.c:1592-1617 → 1715. M is the same.

Related upstream issues found (read only): #35212 (open, 2026-10-17) reports that NOTIFY in an autocommit, read-only statement fails with 40001; our conversations.rs:532 hits the same. #33803 (active_pid cleared). #31480. #34795. #34863/#34872.

## Build (read only)
- Default compiler clang21 (T: build-support/common-build-env.sh:567-573). Release builds use full LTO for clang (T: build-support/jenkins/common-lto.sh:76-84). The LTO link needs 17 GiB per process (T: yb-jenkins-build.sh:349).
- Native only: CMake refuses when YB_TARGET_ARCH differs from the host (T: CMakeLists.txt:84-89).
- Third-party archives: thirdparty_archives.yml, sha c46477f491. x86_64 clang21 full-lto is v20260826163332-…, 1,307 MiB. aarch64 is v20260826163331-…, 258 MiB. Both come from github.com/yugabyte/yugabyte-db-thirdparty releases (T: python/yugabyte/thirdparty_releases.py:30).
- LLVM comes through llvm-installer==1.5.2 (requirements_frozen.txt; common-build-env.sh:1362). Its URL was not read.
- The build image is not pinned in the tree (only `yugabyteci/yb_build_infra_almalinux9_x86_64:latest` in .devcontainer). Docker Hub: yugabyteci/yb_build_infra_almalinux8_x86_64 v2026-09-10T10_14_48 is sha256:6a9a7d74f78e58a15de514e49b6c9df213f654e38be8ff63d6f3d96a055bd0cb (3.1 GB). aarch64 is sha256:cce0b1a674bcc4574f1c05ae38a774773d7e94031c913194cf0c3bae04586ba0.
- Upstream's image unpacks yugabyte*$(uname -m)*.tar.gz into /home/yugabyte and strips it (T: docker/images/yugabyte/Dockerfile). Our database/Dockerfile takes that layout from `FROM ${YUGABYTE} AS upstream`.
