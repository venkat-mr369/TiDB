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

A mnemonic that sticks: **PD plans, TiDB processes, TiKV persists.**

Want me to go deeper on any one piece next — how Raft actually makes a write durable (it's surprisingly intuitive once you see it), how a single SQL transaction stays consistent across multiple TiKV nodes using TSO, or how you'd design the `orders` and `option_chain` schema to keep hot rows well-distributed?
