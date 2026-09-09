Exactly. 👍 Let's make the **complete Security Group rule list** for your 3-server setup.

Your architecture is:

```text
BACKEND
 ├── Backend application
 ├── HAProxy
 └── etcd #1

DB1
 ├── PostgreSQL
 ├── Patroni
 └── etcd #2

DB2
 ├── PostgreSQL
 ├── Patroni
 └── etcd #3
```

Below is the clean rule set I recommend for the lab/production-style setup.

---

# 1. `backend_sg` — Inbound Rules

```text
TCP 2379    Source: db1_sg       # DB1/Patroni → BACKEND etcd client API
TCP 2379    Source: db2_sg       # DB2/Patroni → BACKEND etcd client API

TCP 2380    Source: db1_sg       # DB1 etcd → BACKEND etcd peer communication
TCP 2380    Source: db2_sg       # DB2 etcd → BACKEND etcd peer communication

TCP 5000    Source: backend_sg   # Backend app → local HAProxy PostgreSQL endpoint (only if needed as SG rule)
```

### Why no 5432 on `backend_sg`?

Because **HAProxy connects OUT from BACKEND to DB1/DB2 on port 5432**.

The inbound rule belongs on the **DB security groups**, not on `backend_sg`.

### Why no 8008?

Same reason.

HAProxy on BACKEND connects **outbound** to DB1/DB2's Patroni API on `8008`.

So DB1/DB2 allow:

```text
TCP 8008 ← backend_sg
```

---

# 2. `db1_sg` — Inbound Rules

```text
TCP 5432    Source: backend_sg   # HAProxy on BACKEND → PostgreSQL on DB1
TCP 5432    Source: db2_sg       # PostgreSQL replication/DB connectivity between DB2 and DB1

TCP 8008    Source: backend_sg   # HAProxy → Patroni health API on DB1

TCP 2379    Source: backend_sg   # BACKEND etcd → DB1 etcd client API
TCP 2379    Source: db1_sg       # Local DB1 components → DB1 etcd client API
TCP 2379    Source: db2_sg       # DB2/Patroni → DB1 etcd client API

TCP 2380    Source: backend_sg   # BACKEND etcd → DB1 etcd peer communication
TCP 2380    Source: db1_sg       # DB1 etcd peer communication
TCP 2380    Source: db2_sg       # DB2 etcd → DB1 etcd peer communication
```

---

# 3. `db2_sg` — Inbound Rules

```text
TCP 5432    Source: backend_sg   # HAProxy on BACKEND → PostgreSQL on DB2

TCP 5432    Source: db1_sg       # PostgreSQL replication/DB connectivity between DB1 and DB2

TCP 8008    Source: backend_sg   # HAProxy → Patroni health API on DB2

TCP 2379    Source: backend_sg   # BACKEND etcd → DB2 etcd client API
TCP 2379    Source: db1_sg       # DB1/Patroni → DB2 etcd client API
TCP 2379    Source: db2_sg       # Local DB2 components → DB2 etcd client API

TCP 2380    Source: backend_sg   # BACKEND etcd → DB2 etcd peer communication
TCP 2380    Source: db1_sg       # DB1 etcd → DB2 etcd peer communication
TCP 2380    Source: db2_sg       # DB2 etcd peer communication
```

---

# Important: One correction about port 5000

If HAProxy is configured like this:

```text
bind 127.0.0.1:5000
```

then **you do NOT need a `TCP 5000` inbound rule in `backend_sg`**.

Why?

Because the backend application and HAProxy are on the **same machine**:

```text
Backend application
       │
       │ localhost:5000
       ▼
    HAProxy
       │
       ├──► DB1:5432
       └──► DB2:5432
```

AWS Security Groups don't control this localhost traffic.

So for our setup, I would actually **leave 5000 out of the SG** initially.

---

# Final clean version

### `backend_sg`

```text
Inbound:

TCP 2379    Source: db1_sg       # DB1/Patroni → BACKEND etcd
TCP 2379    Source: db2_sg       # DB2/Patroni → BACKEND etcd

TCP 2380    Source: db1_sg       # DB1 etcd → BACKEND etcd peer
TCP 2380    Source: db2_sg       # DB2 etcd → BACKEND etcd peer
```

### `db1_sg`

```text
Inbound:

TCP 5432    Source: backend_sg   # HAProxy → DB1 PostgreSQL
TCP 5432    Source: db2_sg       # DB2 → DB1 PostgreSQL/replication

TCP 8008    Source: backend_sg   # HAProxy → DB1 Patroni API

TCP 2379    Source: backend_sg   # BACKEND etcd → DB1 etcd
TCP 2379    Source: db1_sg       # DB1 local etcd/client traffic
TCP 2379    Source: db2_sg       # DB2/Patroni → DB1 etcd

TCP 2380    Source: backend_sg   # BACKEND etcd → DB1 etcd peer
TCP 2380    Source: db1_sg       # DB1 etcd peer traffic
TCP 2380    Source: db2_sg       # DB2 etcd → DB1 etcd peer
```

### `db2_sg`

```text
Inbound:

TCP 5432    Source: backend_sg   # HAProxy → DB2 PostgreSQL
TCP 5432    Source: db1_sg       # DB1 → DB2 PostgreSQL/replication

TCP 8008    Source: backend_sg   # HAProxy → DB2 Patroni API

TCP 2379    Source: backend_sg   # BACKEND etcd → DB2 etcd
TCP 2379    Source: db1_sg       # DB1/Patroni → DB2 etcd
TCP 2379    Source: db2_sg       # DB2 local etcd/client traffic

TCP 2380    Source: backend_sg   # BACKEND etcd → DB2 etcd peer
TCP 2380    Source: db1_sg       # DB1 etcd → DB2 etcd peer
TCP 2380    Source: db2_sg       # DB2 etcd peer traffic
```

### And for all three SGs

**Do not add:**

```text
TCP 22    0.0.0.0/0
```

because we're using **AWS SSM Session Manager** for administration.

Also, assuming your outbound rules are still the default **Allow all outbound**, you don't need separate outbound rules for these ports. The inbound rules above control who can initiate connections to each server.
