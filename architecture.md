Great topic. Let me explain TiDB's architecture using an options trading platform as the use case — since that fits what you're learning.Think of TiDB as a distributed SQL database built for workloads that a single MySQL server couldn't survive — which is exactly the workload an options trading platform creates. Imagine you're building something like Zerodha or Interactive Brokers: millions of options orders flowing in every minute during market hours, every order must be durably saved (you cannot lose a buy/sell), analysts need to run SQL like `SELECT * FROM orders WHERE strike = 25000 AND expiry = '2026-05-29'`, and the system must scale horizontally as users multiply. A single MySQL box would buckle, and manual sharding is painful. TiDB's three-part architecture solves this by splitting responsibilities into three layers — each does exactly one thing.

<img width="1440" height="920" alt="image" src="https://github.com/user-attachments/assets/5650aea0-372c-44e1-9995-b833b4f2d62a" />


**TiDB** is the SQL frontend. It speaks the MySQL protocol, so your trading app connects to it like any normal MySQL database. It parses queries, plans them, and figures out where the data actually lives — but it stores no data itself. It is stateless. You can run ten of them behind a load balancer and they all share the same view of the world.

**TiKV** is where data actually lives. It's a distributed key-value store. Your `orders` table gets chopped into chunks called *regions* (roughly 96 MB each), and each region is stored on three different TiKV nodes using the Raft consensus protocol. If one server's disk dies in the middle of market hours, your order isn't lost — the other two replicas still have it.

**PD** (Placement Driver) is the brain. It does three jobs: it hands out globally unique timestamps for every transaction so the cluster has a consistent sense of time (called TSO — timestamp oracle), it remembers which TiKV node holds which region, and it rebalances regions when you add or remove nodes. Nothing happens in the cluster without PD's coordination.Now walk through what happens when a user taps `BUY 50 lots NIFTY 25000 CE`:

1. The trading app sends `INSERT INTO orders (...) VALUES (...)` to TiDB.
2. TiDB asks PD for a transaction timestamp (this is the TSO — every transaction in the cluster gets a unique monotonic number from PD).
3. TiDB asks PD which TiKV node currently holds the region for this user's orders.
4. PD responds with the region's leader TiKV node.
5. TiDB sends the write to that leader.
6. The leader replicates the write to two followers via Raft. As soon as a majority (2 of 3) confirms, the write is durable.
7. TiDB commits and returns success to the trading app.

The whole flow typically takes single-digit milliseconds. And because PD is constantly rebalancing regions in the background, hot tables like `orders` and `option_chain` during the 9:15 AM market open get spread across TiKV nodes automatically — no manual sharding ever, no DBA-on-call moment when a single shard melts.

Three properties fall out of this design that matter for trading data specifically. First, ACID transactions across the whole cluster — a trade that debits a margin balance and inserts a position row either both commit or both roll back, even if those rows live on different TiKV nodes. Second, no data loss on hardware failure — losing one TiKV node loses zero orders because every region has 3 replicas. Third, you can `JOIN`, `GROUP BY`, and run analytics queries the same way you do on MySQL today — there's no separate query engine to learn.

Good question — the previous diagram shows the *roles*, but the on-disk reality is very different for each layer. Let me break it down node by node, then show the deployment side.

**TiDB nodes — stateless, almost nothing on disk.** TiDB is just a Go process (`tidb-server` binary). When it starts, it doesn't touch the disk for any user data. It opens a gRPC connection to PD, learns where regions live, parses SQL, and serves results. Everything is in RAM — query plans, prepared statements, connection state, transaction buffers. On a TiDB machine you'll find only the binary, a config file `tidb.toml` (listen address, PD endpoints, slow-query threshold), and `tidb.log`. That's it. Restart a TiDB node and zero rows are lost. In your trading platform you'd run 3-4 TiDB nodes behind an HAProxy/Nginx load balancer — they're treated like web servers: cattle, not pets.

**PD nodes — small metadata, big responsibility.** PD runs an embedded etcd internally to store cluster metadata using Raft for consistency among the 3 PD nodes themselves. PD's disk holds the *store registry* (list of every TiKV node, its address, capacity, last heartbeat), the *region map* (every region's key range and which 3 TiKV nodes hold its replicas), the *TSO checkpoint* (last allocated timestamp, advanced in 50ms batches), and etcd's own WAL and snapshot files. The data directory (default `pd-data/` or `/var/lib/pd/`) looks like this:

```
pd-data/
├── member/
│   ├── wal/         ← etcd write-ahead log (.wal files)
│   └── snap/        ← etcd snapshots (.snap files)
└── last_resign      ← leader election state
```

Total footprint stays small — usually under 10 GB even for huge clusters. You run **3 PD nodes** (odd number for Raft majority); one is the elected leader, the others are followers. If the PD leader crashes, another takes over within seconds and no transactions are permanently lost.

**TiKV nodes — where 99% of your storage lives.** This is the actual database. Every order row, every position, every option-chain quote eventually lands on a TiKV node's disk. Under the hood TiKV runs **two separate RocksDB instances** per node:

1. **kv RocksDB** — holds user data: your `orders` table, `positions`, `option_chain`, every actual row.
2. **raft RocksDB** — holds the Raft log: every recent write proposal, used for replicating to followers and recovering after crashes.

The TiKV data directory looks like this:

```
tikv-data/
├── db/                  ← kv RocksDB (user data)
│   ├── *.sst            ← Sorted String Tables (the actual rows, LSM levels)
│   ├── *.log            ← RocksDB write-ahead log
│   ├── MANIFEST-*       ← bookkeeping
│   └── CURRENT
├── raft/                ← raft RocksDB (consensus log)
│   └── (same structure)
├── snap/                ← region snapshots, used to catch up new replicas
└── last_tikv.toml
```

The `.sst` files are where the bulk of your data actually sits. RocksDB writes go first to the WAL, then to a memtable in RAM, then are flushed to immutable `.sst` files, then compacted in the background. As your `orders` table grows, hundreds or thousands of `.sst` files accumulate on each TiKV node.

The important mental shift: **one TiKV node does not hold one "orders table file."** Instead it holds many *regions* — each region being a contiguous slice of the global key space, ~96 MB by default. So a single TiKV node might be holding:

- Region 1: `orders` rows where `user_id` is 100 to 9,800
- Region 47: `option_chain` rows for NIFTY 2026-05-29
- Region 122: `positions` rows for users 50,000 to 58,000
- ...and 800 more

When a region grows beyond 144 MB, TiKV splits it in two and PD reassigns one half to a less-loaded node. This is the auto-sharding that you'd otherwise have to do by hand.**A minimum production deployment for an options trading platform.** Each node type has very different resource needs, which is why they're typically run on separate machines:

- **3 PD nodes** — small machines (4 vCPU, 8 GB RAM, 100 GB SSD). Odd count for Raft majority. One is the elected leader.
- **3 TiDB nodes** — CPU-heavy (16 vCPU, 32–64 GB RAM, modest disk for logs). Scale this tier when queries become CPU-bound at market open.
- **3+ TiKV nodes** — disk-heavy (16 vCPU, 64 GB RAM, NVMe SSDs sized for your data + 3× replication overhead). Scale this tier when you're running out of disk.

So an 8-machine cluster is a reasonable production minimum. Each tier scales independently. If you double users and queries get slow, you add TiDB nodes — no data migration needed because they're stateless. If your `order_history` table balloons after a year of trading, you add TiKV nodes and PD will automatically migrate regions onto them to balance load. If a TiKV node dies overnight, the other two replicas of every region keep serving live trades, and PD spawns fresh replicas on healthy nodes to restore the count of 3.

There's also an optional fourth component called **TiFlash** for analytics workloads. TiFlash nodes asynchronously replicate from TiKV but store data in *columnar* format instead of row format. So a query like "total volume of all PE options traded today grouped by strike" runs 10–100× faster on TiFlash than on TiKV, without slowing down the live order-write path. You write to TiDB normally; TiDB's optimizer decides which engine to read from per query.

A way to think about the disk-size hierarchy: PD holds *kilobytes-to-gigabytes* (metadata), TiDB holds *almost zero* (logs only), TiKV holds *terabytes* (the actual rows × 3 replicas).

Want to go further on any of these — for example, the exact key encoding TiKV uses for an `orders` row (so you understand why regions split on `user_id` ranges), how the LSM-tree write path inside RocksDB actually persists a trade, or how to model the schema for `option_chain` so hot rows don't pile up on one TiKV node?
A mnemonic that sticks: **PD plans, TiDB processes, TiKV persists.**

Want me to go deeper on any one piece next — how Raft actually makes a write durable (it's surprisingly intuitive once you see it), how a single SQL transaction stays consistent across multiple TiKV nodes using TSO, or how you'd design the `orders` and `option_chain` schema to keep hot rows well-distributed?
