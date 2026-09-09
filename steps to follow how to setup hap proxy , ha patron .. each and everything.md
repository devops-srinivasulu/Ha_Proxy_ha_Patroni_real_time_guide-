Absolutely. Let's build this **from completely fresh EC2 servers**, step by step, so that if you execute the commands in order, you end up with:

```text
                         AWS VPC
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│                    ┌──────────────────┐                         │
│                    │     BACKEND      │                         │
│                    │                  │                         │
│                    │ Backend App      │                         │
│                    │ HAProxy          │                         │
│                    │ etcd #1          │                         │
│                    └────────┬─────────┘                         │
│                             │                                   │
│                 etcd 2379/2380                                │
│                       ┌─────┴─────┐                             │
│                       │           │                             │
│                       ▼           ▼                             │
│              ┌──────────────┐ ┌──────────────┐                │
│              │     DB1      │ │     DB2      │                │
│              │              │ │              │                │
│              │ PostgreSQL   │ │ PostgreSQL   │                │
│              │ Patroni      │ │ Patroni      │                │
│              │ etcd #2      │ │ etcd #3      │                │
│              │              │ │              │                │
│              │  PRIMARY     │ │  REPLICA     │                │
│              └──────┬───────┘ └──────┬───────┘                │
│                     │                 │                        │
│                     └───────┬─────────┘                        │
│                         WAL replication                         │
│                           TCP 5432                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

                    Normal operation

Backend App
     │
     ▼
  HAProxy :5000
     │
     ▼
  DB1 :5432  ← PRIMARY
  DB2 :5432  ← REPLICA
```

Then we will deliberately kill DB1:

```text
                    DB1 FAILURE
                        ❌
                        │
                        ▼
                 Patroni detects
                        │
                        ▼
                  etcd coordinates
                        │
                        ▼
              DB2 promoted PRIMARY
                        │
                        ▼
                HAProxy detects
                DB2 /primary = 200
                        │
                        ▼
             Backend → DB2 :5432
```

And finally we'll recover DB1:

```text
DB1 comes back
      │
      ▼
Patroni sees DB2 is leader
      │
      ▼
DB1 is re-synchronized
      │
      ▼
DB1 becomes REPLICA
      │
      ▼
DB2 PRIMARY
DB1 REPLICA
```

This is the complete lab.

---

# PART 0 — What we're going to install

Before touching the servers, understand exactly what each component does.

| Server  | Software            | Purpose                    |
| ------- | ------------------- | -------------------------- |
| BACKEND | Backend application | Your application           |
| BACKEND | HAProxy             | Stable PostgreSQL endpoint |
| BACKEND | etcd                | DCS / cluster coordination |
| DB1     | PostgreSQL          | Database node              |
| DB1     | Patroni             | PostgreSQL HA management   |
| DB1     | etcd                | DCS member                 |
| DB2     | PostgreSQL          | Database node              |
| DB2     | Patroni             | PostgreSQL HA management   |
| DB2     | etcd                | DCS member                 |

We'll use:

```text
PostgreSQL 16
Patroni
etcd 3.7.1
HAProxy
```

Patroni uses a distributed configuration store such as etcd to coordinate leadership, while its REST API can expose `/primary` for HAProxy health checking. PostgreSQL streaming replication carries WAL from the primary to the standby.

---

# PART 1 — Start with three fresh EC2 instances

We'll assume Ubuntu 24.04 LTS.

Create:

```text
BACKEND
DB1
DB2
```

All three should be:

```text
Same VPC
Same private network/subnet or routable subnets
Private IP addresses
SSM managed
```

You do **not** need a bastion host.

---

# PART 2 — Security Groups

You already have:

```text
backend_sg
db1_sg
db2_sg
```

Use the rules we just finalized.

## `backend_sg`

```text
TCP 2379    Source: db1_sg
TCP 2379    Source: db2_sg

TCP 2380    Source: db1_sg
TCP 2380    Source: db2_sg
```

No public access.

---

## `db1_sg`

```text
TCP 5432    Source: backend_sg
TCP 5432    Source: db2_sg

TCP 8008    Source: backend_sg

TCP 2379    Source: backend_sg
TCP 2379    Source: db1_sg
TCP 2379    Source: db2_sg

TCP 2380    Source: backend_sg
TCP 2380    Source: db1_sg
TCP 2380    Source: db2_sg
```

---

## `db2_sg`

```text
TCP 5432    Source: backend_sg
TCP 5432    Source: db1_sg

TCP 8008    Source: backend_sg

TCP 2379    Source: backend_sg
TCP 2379    Source: db1_sg
TCP 2379    Source: db2_sg

TCP 2380    Source: backend_sg
TCP 2380    Source: db1_sg
TCP 2380    Source: db2_sg
```

Assume default outbound:

```text
All traffic → 0.0.0.0/0
```

If you have restricted outbound rules, we'll need to add appropriate outbound access.

---

# PART 3 — Get the private IP addresses

Before installing anything, record:

```text
BACKEND_PRIVATE_IP = ?
DB1_PRIVATE_IP      = ?
DB2_PRIVATE_IP      = ?
```

For example, imagine:

```text
BACKEND = 10.0.1.10
DB1     = 10.0.2.10
DB2     = 10.0.2.20
```

**Do not use these example IPs.**

Use your actual private IPs.

You can get them on each server with:

## RUN ON: BACKEND

```bash
hostname
hostname -I
```

## RUN ON: DB1

```bash
hostname
hostname -I
```

## RUN ON: DB2

```bash
hostname
hostname -I
```

Write them down.

We'll refer to them as:

```text
BACKEND_IP
DB1_IP
DB2_IP
```

---

# PART 4 — Connect using SSM (if you connect your all db and backned server from ssm seperately you no ned to run these kind of commands "aws ssm start-session --target <BACKEND_INSTANCE_ID>" ) its only for connect to the db1, db2 server those are already you connected thorugh ssm .

From your local computer:

## BACKEND

```bash
aws ssm start-session --target <BACKEND_INSTANCE_ID>
```

## DB1

```bash
aws ssm start-session --target <DB1_INSTANCE_ID>
```

## DB2

```bash
aws ssm start-session --target <DB2_INSTANCE_ID>
```

You can open three terminal windows.

I strongly recommend:

```text
Terminal 1 → BACKEND
Terminal 2 → DB1
Terminal 3 → DB2
```

That will make this lab much easier.

---

# PART 5 — Basic server preparation

Do this on **ALL THREE SERVERS**.

## RUN ON: BACKEND

```bash
sudo apt update
sudo apt upgrade -y
```

## RUN ON: DB1

```bash
sudo apt update
sudo apt upgrade -y
```

## RUN ON: DB2

```bash
sudo apt update
sudo apt upgrade -y
```

Then verify OS.

### ALL SERVERS

```bash
cat /etc/os-release
```

You should see Ubuntu 24.04 LTS.

---

# PART 6 — Set hostnames

This makes the cluster easier to understand.

## RUN ON: BACKEND

```bash
sudo hostnamectl set-hostname backend
```

## RUN ON: DB1

```bash
sudo hostnamectl set-hostname db1
```

## RUN ON: DB2

```bash
sudo hostnamectl set-hostname db2
```

Reconnect your SSM sessions after changing the hostname if necessary.

Check:

```bash
hostname
```

Expected:

```text
backend
```

or:

```text
db1
```

or:

```text
db2
```

---

# PART 7 — Configure `/etc/hosts`

This is useful because we can refer to servers by name instead of constantly typing IP addresses.

Do this on **ALL THREE SERVERS**.

Replace the IPs with your actual private IPs.

Example:

```bash
sudo tee -a /etc/hosts > /dev/null <<EOF
10.0.1.10 backend
10.0.2.10 db1
10.0.2.20 db2
EOF
```

Then:

```bash
getent hosts backend
getent hosts db1
getent hosts db2
```

You should get the corresponding IP addresses.

---

# PART 8 — Test network connectivity

Before installing PostgreSQL, etcd or Patroni, make sure the network is correct.

We'll use `nc`.

Install it on all three.

## ALL SERVERS

```bash
sudo apt install -y netcat-openbsd
```

---

## Test BACKEND → DB1 PostgreSQL

### RUN ON: BACKEND

```bash
nc -vz -w 3 db1 5432
```

At this moment PostgreSQL isn't running yet.

So you may get:

```text
Connection refused
```

That is **okay**.

It means:

```text
BACKEND can reach DB1
but nothing is listening on 5432 yet
```

If you get:

```text
Connection timed out
```

that's different.

That usually indicates a network/SG/routing problem.

---

# PART 9 — Install PostgreSQL

Now we start building the database layer.

We will install PostgreSQL **only on DB1 and DB2**.

## RUN ON: DB1

```bash
sudo apt update
sudo apt install -y postgresql-16 postgresql-client-16
```

## RUN ON: DB2

```bash
sudo apt update
sudo apt install -y postgresql-16 postgresql-client-16
```

Check:

### DB1

```bash
psql --version
```

### DB2

```bash
psql --version
```

Expected something similar to:

```text
psql (PostgreSQL) 16.x
```

Check clusters:

### DB1

```bash
pg_lsclusters
```

### DB2

```bash
pg_lsclusters
```

---

# PART 10 — Important: Patroni will control PostgreSQL

Normally Ubuntu starts PostgreSQL using:

```text
postgresql.service
```

But our architecture is different.

We want:

```text
                 Patroni
                    │
                    ▼
              PostgreSQL
```

Patroni needs to control whether PostgreSQL is:

```text
PRIMARY
```

or:

```text
REPLICA
```

and perform failover/recovery.

Therefore we stop the normal PostgreSQL service.

## RUN ON: DB1

```bash
sudo systemctl stop postgresql
```

Then:

```bash
sudo systemctl disable postgresql
```

## RUN ON: DB2

```bash
sudo systemctl stop postgresql
```

Then:

```bash
sudo systemctl disable postgresql
```

Check:

```bash
sudo systemctl status postgresql
```

It should be stopped/disabled.

**Do not start PostgreSQL manually from this point onward.**

Later:

```text
Patroni → PostgreSQL
```

---

# PART 11 — Install etcd

Now we're going to create a **3-node etcd cluster**.

```text
BACKEND = etcd #1
DB1     = etcd #2
DB2     = etcd #3
```

Why three?

Because etcd needs a quorum.

```text
3 members
   ↓
majority = 2
```

So if one etcd member dies:

```text
2 remaining
     ↓
quorum maintained
```

If you had only one etcd server and it died, Patroni would lose its DCS.

---

# PART 12 — Install etcd on BACKEND

## RUN ON: BACKEND

First:

```bash
sudo apt update
sudo apt install -y curl tar
```

Download etcd:

```bash
ETCD_VER=v3.7.1
```

Then:

```bash
curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd.tar.gz
```

Create directory:

```bash
sudo mkdir -p /opt/etcd
```

Extract:

```bash
sudo tar -xzf /tmp/etcd.tar.gz -C /opt/etcd --strip-components=1
```

Create symlinks:

```bash
sudo ln -sf /opt/etcd/etcd /usr/local/bin/etcd
sudo ln -sf /opt/etcd/etcdctl /usr/local/bin/etcdctl
sudo ln -sf /opt/etcd/etcdutl /usr/local/bin/etcdutl
```

Check:

```bash
etcd --version
```

and:

```bash
etcdctl version
```

---

# PART 13 — Install etcd on DB1

## RUN ON: DB1

```bash
sudo apt update
sudo apt install -y curl tar
```

```bash
ETCD_VER=v3.7.1
```

```bash
curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd.tar.gz
```

```bash
sudo mkdir -p /opt/etcd
```

```bash
sudo tar -xzf /tmp/etcd.tar.gz -C /opt/etcd --strip-components=1
```

```bash
sudo ln -sf /opt/etcd/etcd /usr/local/bin/etcd
sudo ln -sf /opt/etcd/etcdctl /usr/local/bin/etcdctl
sudo ln -sf /opt/etcd/etcdutl /usr/local/bin/etcdutl
```

Check:

```bash
etcd --version
```

---

# PART 14 — Install etcd on DB2

## RUN ON: DB2

```bash
sudo apt update
sudo apt install -y curl tar
```

```bash
ETCD_VER=v3.7.1
```

```bash
curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd.tar.gz
```

```bash
sudo mkdir -p /opt/etcd
```

```bash
sudo tar -xzf /tmp/etcd.tar.gz -C /opt/etcd --strip-components=1
```

```bash
sudo ln -sf /opt/etcd/etcd /usr/local/bin/etcd
sudo ln -sf /opt/etcd/etcdctl /usr/local/bin/etcdctl
sudo ln -sf /opt/etcd/etcdutl /usr/local/bin/etcdutl
```

Check:

```bash
etcd --version
```

The etcd release/install approach above follows the official release distribution; v3.7.1 is currently listed as a release.

---

# PART 15 — Create etcd user and directories

Do this on **ALL THREE**.

## BACKEND

```bash
sudo useradd --system --home /var/lib/etcd --shell /usr/sbin/nologin etcd
```

```bash
sudo mkdir -p /var/lib/etcd
```

```bash
sudo chown -R etcd:etcd /var/lib/etcd
```

## DB1

Same commands:

```bash
sudo useradd --system --home /var/lib/etcd --shell /usr/sbin/nologin etcd
```

```bash
sudo mkdir -p /var/lib/etcd
```

```bash
sudo chown -R etcd:etcd /var/lib/etcd
```

## DB2

Same:

```bash
sudo useradd --system --home /var/lib/etcd --shell /usr/sbin/nologin etcd
```

```bash
sudo mkdir -p /var/lib/etcd
```

```bash
sudo chown -R etcd:etcd /var/lib/etcd
```

---

# PART 16 — Create the etcd systemd service

Now we tell Linux how to start etcd.

This is where the three-node cluster is actually defined.

---

## RUN ON: BACKEND

Create:

```bash
sudo nano /etc/systemd/system/etcd.service
```

Put:

```ini
[Unit]
Description=etcd
Documentation=https://etcd.io/docs/
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
User=etcd
Group=etcd

ExecStart=/usr/local/bin/etcd \
  --name etcd-backend \
  --data-dir /var/lib/etcd \
  --listen-client-urls http://0.0.0.0:2379 \
  --advertise-client-urls http://BACKEND_IP:2379 \
  --listen-peer-urls http://0.0.0.0:2380 \
  --initial-advertise-peer-urls http://BACKEND_IP:2380 \
  --initial-cluster etcd-backend=http://BACKEND_IP:2380,etcd-db1=http://DB1_IP:2380,etcd-db2=http://DB2_IP:2380 \
  --initial-cluster-state new \
  --initial-cluster-token postgres-ha-cluster

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Replace:

```text
BACKEND_IP
DB1_IP
DB2_IP
```

with the real private IPs.

---

# PART 17 — DB1 etcd service

## RUN ON: DB1

```bash
sudo nano /etc/systemd/system/etcd.service
```

Put:

```ini
[Unit]
Description=etcd
Documentation=https://etcd.io/docs/
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
User=etcd
Group=etcd

ExecStart=/usr/local/bin/etcd \
  --name etcd-db1 \
  --data-dir /var/lib/etcd \
  --listen-client-urls http://0.0.0.0:2379 \
  --advertise-client-urls http://DB1_IP:2379 \
  --listen-peer-urls http://0.0.0.0:2380 \
  --initial-advertise-peer-urls http://DB1_IP:2380 \
  --initial-cluster etcd-backend=http://BACKEND_IP:2380,etcd-db1=http://DB1_IP:2380,etcd-db2=http://DB2_IP:2380 \
  --initial-cluster-state new \
  --initial-cluster-token postgres-ha-cluster

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Again replace the IP placeholders.

---

# PART 18 — DB2 etcd service

## RUN ON: DB2

```bash
sudo nano /etc/systemd/system/etcd.service
```

Put:

```ini
[Unit]
Description=etcd
Documentation=https://etcd.io/docs/
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
User=etcd
Group=etcd

ExecStart=/usr/local/bin/etcd \
  --name etcd-db2 \
  --data-dir /var/lib/etcd \
  --listen-client-urls http://0.0.0.0:2379 \
  --advertise-client-urls http://DB2_IP:2379 \
  --listen-peer-urls http://0.0.0.0:2380 \
  --initial-advertise-peer-urls http://DB2_IP:2380 \
  --initial-cluster etcd-backend=http://BACKEND_IP:2380,etcd-db1=http://DB1_IP:2380,etcd-db2=http://DB2_IP:2380 \
  --initial-cluster-state new \
  --initial-cluster-token postgres-ha-cluster

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

---

# PART 19 — Start etcd

Because this is a new cluster, start all three.

## BACKEND

```bash
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```

## DB1

```bash
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```

## DB2

```bash
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```

---

# PART 20 — Verify etcd

Now comes an important checkpoint.

From **BACKEND**:

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=http://backend:2379,http://db1:2379,http://db2:2379 \
  endpoint health
```

You should see all three endpoints healthy.

Then:

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=http://backend:2379,http://db1:2379,http://db2:2379 \
  member list
```

You should see:

```text
etcd-backend
etcd-db1
etcd-db2
```

### STOP HERE if etcd is not healthy.

Don't continue to Patroni until this works.

---

# PART 21 — Install Patroni dependencies

Now we're ready for the PostgreSQL HA layer.

Patroni will manage:

```text
PostgreSQL
     │
     ▼
Primary / Replica state
     │
     ▼
Failover
     │
     ▼
Reinitialization
```

Install on **DB1 and DB2 only**.

---

## DB1

```bash
sudo apt update
sudo apt install -y python3-venv python3-dev libpq-dev gcc curl
```

## DB2

```bash
sudo apt update
sudo apt install -y python3-venv python3-dev libpq-dev gcc curl
```

Create virtual environment.

### DB1

```bash
sudo python3 -m venv /opt/patroni
```

### DB2

```bash
sudo python3 -m venv /opt/patroni
```

Upgrade pip.

### DB1

```bash
sudo /opt/patroni/bin/pip install --upgrade pip
```

### DB2

```bash
sudo /opt/patroni/bin/pip install --upgrade pip
```

Install Patroni:

### DB1

```bash
sudo /opt/patroni/bin/pip install 'patroni[etcd3,psycopg3]'
```

### DB2

```bash
sudo /opt/patroni/bin/pip install 'patroni[etcd3,psycopg3]'
```

Verify:

### DB1

```bash
/opt/patroni/bin/patroni --version
```

### DB2

```bash
/opt/patroni/bin/patroni --version
```

The Patroni documentation currently supports PostgreSQL 9.3–18 and provides etcd/etcd3 extras for DCS connectivity.

---

# PART 22 — Create Patroni directories

## DB1

```bash
sudo mkdir -p /etc/patroni
sudo chown postgres:postgres /etc/patroni
```

## DB2

```bash
sudo mkdir -p /etc/patroni
sudo chown postgres:postgres /etc/patroni
```

---

# PART 23 — Create the Patroni configuration

This is the most important configuration file in the project.

It tells Patroni:

```text
Who am I?
Where is etcd?
Where is PostgreSQL?
How should replication work?
What are the passwords?
How should failover happen?
```

---

# DB1 Patroni configuration

## RUN ON: DB1

```bash
sudo nano /etc/patroni/patroni.yml
```

We'll create a production-style starting configuration.

```yaml
scope: postgres-ha
namespace: /service/
name: db1

restapi:
  listen: 0.0.0.0:8008
  connect_address: DB1_IP:8008

etcd3:
  hosts: BACKEND_IP:2379,DB1_IP:2379,DB2_IP:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576

    postgresql:
      use_pg_rewind: true
      use_slots: true

      parameters:
        wal_level: replica
        hot_standby: "on"
        max_wal_senders: 10
        max_replication_slots: 10
        wal_keep_size: 256MB

  initdb:
    - encoding: UTF8
    - data-checksums

  pg_hba:
    - host replication replicator BACKEND_IP/32 scram-sha-256
    - host replication replicator DB1_IP/32 scram-sha-256
    - host replication replicator DB2_IP/32 scram-sha-256
    - host all all 0.0.0.0/0 scram-sha-256

  users:
    admin:
      password: CHANGE_THIS_ADMIN_PASSWORD
      options:
        - createrole
        - createdb

    replicator:
      password: CHANGE_THIS_REPLICATION_PASSWORD
      options:
        - replication

postgresql:
  listen: 0.0.0.0:5432
  connect_address: DB1_IP:5432
  data_dir: /var/lib/postgresql/16/main

  authentication:
    superuser:
      username: postgres
      password: CHANGE_THIS_POSTGRES_PASSWORD

    replication:
      username: replicator
      password: CHANGE_THIS_REPLICATION_PASSWORD

  parameters:
    unix_socket_directories: '/var/run/postgresql'

  pg_rewind:
    username: postgres
    password: CHANGE_THIS_POSTGRES_PASSWORD

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

Replace:

```text
DB1_IP
BACKEND_IP
```

and **change every password**.

---

# DB2 Patroni configuration

## RUN ON: DB2

```bash
sudo nano /etc/patroni/patroni.yml
```

Use:

```yaml
scope: postgres-ha
namespace: /service/
name: db2

restapi:
  listen: 0.0.0.0:8008
  connect_address: DB2_IP:8008

etcd3:
  hosts: BACKEND_IP:2379,DB1_IP:2379,DB2_IP:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576

    postgresql:
      use_pg_rewind: true
      use_slots: true

      parameters:
        wal_level: replica
        hot_standby: "on"
        max_wal_senders: 10
        max_replication_slots: 10
        wal_keep_size: 256MB

  initdb:
    - encoding: UTF8
    - data-checksums

  pg_hba:
    - host replication replicator BACKEND_IP/32 scram-sha-256
    - host replication replicator DB1_IP/32 scram-sha-256
    - host replication replicator DB2_IP/32 scram-sha-256
    - host all all 0.0.0.0/0 scram-sha-256

  users:
    admin:
      password: CHANGE_THIS_ADMIN_PASSWORD
      options:
        - createrole
        - createdb

    replicator:
      password: CHANGE_THIS_REPLICATION_PASSWORD
      options:
        - replication

postgresql:
  listen: 0.0.0.0:5432
  connect_address: DB2_IP:5432
  data_dir: /var/lib/postgresql/16/main

  authentication:
    superuser:
      username: postgres
      password: CHANGE_THIS_POSTGRES_PASSWORD

    replication:
      username: replicator
      password: CHANGE_THIS_REPLICATION_PASSWORD

  parameters:
    unix_socket_directories: '/var/run/postgresql'

  pg_rewind:
    username: postgres
    password: CHANGE_THIS_POSTGRES_PASSWORD

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

Replace the IPs and passwords.

**Important:** For a real production deployment, passwords should come from a proper secret-management system rather than being casually stored in shell history/configuration. For this training lab, we're keeping the configuration explicit so you can understand it.

---

# PART 24 — Make Patroni configuration readable only by postgres

## DB1

```bash
sudo chown postgres:postgres /etc/patroni/patroni.yml
sudo chmod 600 /etc/patroni/patroni.yml
```

## DB2

```bash
sudo chown postgres:postgres /etc/patroni/patroni.yml
sudo chmod 600 /etc/patroni/patroni.yml
```

---

# PART 25 — Create Patroni systemd service

## DB1

```bash
sudo nano /etc/systemd/system/patroni.service
```

Put:

```ini
[Unit]
Description=Patroni PostgreSQL HA
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=postgres
Group=postgres

ExecStart=/opt/patroni/bin/patroni /etc/patroni/patroni.yml

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

---

## DB2

Do the same:

```bash
sudo nano /etc/systemd/system/patroni.service
```

Use the same service file.

---

# PART 26 — Start Patroni

This is where PostgreSQL HA actually comes alive.

Start **DB1 first**.

## RUN ON: DB1

```bash
sudo systemctl daemon-reload
```

```bash
sudo systemctl enable patroni
```

```bash
sudo systemctl start patroni
```

Watch logs:

```bash
sudo journalctl -u patroni -f
```

You should eventually see Patroni starting PostgreSQL.

---

# PART 27 — Check DB1

Open another DB1 terminal or stop the log view with `Ctrl+C`.

Run:

```bash
sudo -u postgres /opt/patroni/bin/patronictl -c /etc/patroni/patroni.yml list
```

At this point you should see DB1 as:

```text
Leader
```

because DB2 isn't running Patroni yet.

---

# PART 28 — Start DB2 Patroni

## RUN ON: DB2

```bash
sudo systemctl daemon-reload
```

```bash
sudo systemctl enable patroni
```

```bash
sudo systemctl start patroni
```

Watch:

```bash
sudo journalctl -u patroni -f
```

DB2 should discover the existing cluster through etcd and initialize itself as a replica.

---

# PART 29 — Verify the cluster

Run from DB1:

```bash
sudo -u postgres /opt/patroni/bin/patronictl -c /etc/patroni/patroni.yml list
```

You want something conceptually like:

```text
+ Cluster: postgres-ha --------+
| Member | Role    | State     |
+--------+---------+-----------+
| db1    | Leader  | running   |
| db2    | Replica | running   |
+--------+---------+-----------+
```

This is your first major success point.

---

# PART 30 — Verify PostgreSQL roles directly

## DB1

```bash
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
```

Expected:

```text
 f
```

`false` = primary.

---

## DB2

```bash
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
```

Expected:

```text
 t
```

`true` = standby/replica.

---

# PART 31 — Verify streaming replication

On DB1:

```bash
sudo -u postgres psql -c "SELECT client_addr,state,sync_state,write_lsn,flush_lsn,replay_lsn FROM pg_stat_replication;"
```

You should see DB2.

On DB2:

```bash
sudo -u postgres psql -c "SELECT status,received_lsn,latest_end_lsn FROM pg_stat_wal_receiver;"
```

You should see the WAL receiver active.

PostgreSQL's streaming replication works through WAL sender/receiver processes, with the standby continuously receiving and replaying WAL from the primary.

---

# PART 32 — Create test data

This proves we're not just looking at status.

## DB1

```bash
sudo -u postgres psql
```

Then:

```sql
CREATE DATABASE ha_test;
```

Exit:

```sql
\q
```

Now:

```bash
sudo -u postgres psql -d ha_test
```

Create a table:

```sql
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name TEXT,
    department TEXT
);
```

Insert data:

```sql
INSERT INTO employees (name, department)
VALUES
('Srinivas', 'DevOps'),
('Ravi', 'Development'),
('Kiran', 'Testing');
```

Check:

```sql
SELECT * FROM employees;
```

Exit:

```sql
\q
```

---

# PART 33 — Confirm DB2 received the data

## RUN ON: DB2

```bash
sudo -u postgres psql -d ha_test -c "SELECT * FROM employees;"
```

You should see:

```text
 id | name     | department
----+----------+------------
  1 | Srinivas | DevOps
  2 | Ravi     | Development
  3 | Kiran    | Testing
```

Congratulations.

At this point:

```text
DB1
PRIMARY
   │
   │ WAL
   ▼
DB2
REPLICA
```

is working.

---

# PART 34 — Install HAProxy on BACKEND

Now we're going to create the stable database endpoint.

Without HAProxy:

```text
Backend → DB1
```

If DB1 dies:

```text
Backend → DB1 ❌
```

With HAProxy:

```text
Backend
   │
   ▼
HAProxy :5000
   │
   ├── DB1 :5432
   └── DB2 :5432
```

Patroni tells HAProxy which node is primary.

---

## RUN ON: BACKEND

```bash
sudo apt update
sudo apt install -y haproxy
```

Back up configuration:

```bash
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.backup
```

---

# PART 35 — Configure HAProxy

First get your IPs ready:

```text
DB1_IP
DB2_IP
```

Edit:

```bash
sudo nano /etc/haproxy/haproxy.cfg
```

For the first lab, use:

```text
global
    log /dev/log local0
    log /dev/log local1 notice
    daemon

defaults
    log global
    mode tcp
    timeout connect 5s
    timeout client 30s
    timeout server 30s

frontend postgres
    bind 127.0.0.1:5000
    default_backend postgres_primary

backend postgres_primary
    option httpchk GET /primary
    http-check expect status 200

    server db1 DB1_IP:5432 check port 8008
    server db2 DB2_IP:5432 check port 8008
```

Replace the IP placeholders.

Why this works:

```text
HAProxy TCP connection
        │
        ▼
PostgreSQL :5432

Health check
        │
        ▼
Patroni :8008/primary
```

Patroni's `/primary` endpoint returns success only when that node currently holds the leader lock and is primary, making it suitable for HAProxy health checks.

---

# PART 36 — Validate HAProxy

## BACKEND

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
```

You want:

```text
Configuration file is valid
```

Then:

```bash
sudo systemctl enable haproxy
```

```bash
sudo systemctl restart haproxy
```

Check:

```bash
sudo systemctl status haproxy
```

---

# PART 37 — Test Patroni health endpoints

From BACKEND:

```bash
curl -i http://db1:8008/primary
```

You should get:

```text
HTTP/1.1 200 OK
```

because DB1 is primary.

Now:

```bash
curl -i http://db2:8008/primary
```

DB2 should return a non-200 response because it is currently a replica.

This is how HAProxy knows:

```text
DB1 = use me
DB2 = don't send writes here
```

---

# PART 38 — Test HAProxy itself

On BACKEND:

```bash
sudo -u postgres psql -h 127.0.0.1 -p 5000 -d ha_test -c "SELECT pg_is_in_recovery();"
```

Expected:

```text
 f
```

This means:

```text
Backend
   │
   ▼
HAProxy :5000
   │
   ▼
DB1 :5432
   │
   ▼
PRIMARY
```

---

# PART 39 — Test through HAProxy with actual data

## BACKEND

```bash
sudo -u postgres psql -h 127.0.0.1 -p 5000 -d ha_test -c "SELECT * FROM employees;"
```

You should see the records.

Now insert through HAProxy:

```bash
sudo -u postgres psql -h 127.0.0.1 -p 5000 -d ha_test -c "INSERT INTO employees (name, department) VALUES ('HAProxy-Test', 'DevOps');"
```

Check:

```bash
sudo -u postgres psql -h 127.0.0.1 -p 5000 -d ha_test -c "SELECT * FROM employees;"
```

---

# PART 40 — Now comes the most important test: FAILOVER

We are going to intentionally kill Patroni on DB1.

Current:

```text
DB1 = PRIMARY
DB2 = REPLICA

Backend
   │
   ▼
HAProxy
   │
   ▼
DB1
```

---

## Before failure — check cluster

Run from DB1:

```bash
sudo -u postgres /opt/patroni/bin/patronictl -c /etc/patroni/patroni.yml list
```

Confirm:

```text
db1 → Leader
db2 → Replica
```

---

# PART 41 — Simulate DB1 failure

For the first test, stop Patroni.

## RUN ON: DB1

```bash
sudo systemctl stop patroni
```

This simulates the PostgreSQL HA manager on DB1 disappearing.

---

# PART 42 — Watch DB2

## RUN ON: DB2

```bash
sudo journalctl -u patroni -f
```

Patroni should detect that the leader is gone and participate in leader election.

After the election, DB2 should become:

```text
Leader
```

Check:

```bash
sudo -u postgres /opt/patroni/bin/patronictl -c /etc/patroni/patroni.yml list
```

Expected:

```text
db2 → Leader
```

---

# PART 43 — Verify DB2 became primary

## DB2

```bash
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
```

Expected:

```text
 f
```

DB2 is now primary.

---

# PART 44 — Check HAProxy automatically

On BACKEND:

```bash
curl -i http://db1:8008/primary
```

DB1 should no longer return the primary success response.

Now:

```bash
curl -i http://db2:8008/primary
```

DB2 should return:

```text
HTTP/1.1 200 OK
```

HAProxy should therefore select DB2.

---

# PART 45 — Test application connection after failover

This is the real test.

The application still uses:

```text
127.0.0.1:5000
```

It does **not** need to know DB1 or DB2 changed roles.

Run on BACKEND:

```bash
sudo -u postgres psql -h 127.0.0.1 -p 5000 -d ha_test -c "SELECT pg_is_in_recovery();"
```

Expected:

```text
 f
```

But this time:

```text
f = DB2
```

The application endpoint hasn't changed.

That's the entire point of HAProxy + Patroni.

---

# PART 46 — Insert data after failover

This proves writes are now going to DB2.

## BACKEND

```bash
sudo -u postgres psql -h 127.0.0.1 -p 5000 -d ha_test -c "INSERT INTO employees (name, department) VALUES ('Failover-Test', 'DevOps');"
```

Then:

```bash
sudo -u postgres psql -h 127.0.0.1 -p 5000 -d ha_test -c "SELECT * FROM employees;"
```

You should see:

```text
HAProxy-Test
Failover-Test
```

DB2 is now accepting writes.

---

# PART 47 — IMPORTANT: Don't simply start DB1

This is extremely important.

After:

```text
DB1 ❌
DB2 PRIMARY
```

**Do not immediately do:**

```bash
sudo systemctl start postgresql
```

and don't manually make DB1 writable.

Why?

Because you could create:

```text
DB1 → thinks it is PRIMARY
DB2 → PRIMARY
```

That is **split brain**.

Instead, Patroni must safely bring DB1 back as a replica.

---

# PART 48 — Recover DB1

Now start Patroni again.

## DB1

```bash
sudo systemctl start patroni
```

Watch:

```bash
sudo journalctl -u patroni -f
```

Patroni should recognize:

```text
DB2 = current leader
DB1 = old member
```

With `use_pg_rewind: true`, Patroni can use PostgreSQL's rewind mechanism when the old primary can be safely rewound; otherwise the node may need to be reinitialized from the current leader. Patroni/PostgreSQL documentation supports this style of recovery.

Check:

```bash
sudo -u postgres /opt/patroni/bin/patronictl -c /etc/patroni/patroni.yml list
```

Eventually you want:

```text
db2 → Leader
db1 → Replica
```

---

# PART 49 — Verify DB1 is replica

## DB1

```bash
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
```

Expected:

```text
 t
```

DB1 is now replica.

---

# PART 50 — Verify replication in the new direction

Now the direction has reversed:

```text
             WAL
DB2 PRIMARY ───────► DB1 REPLICA
```

Check DB2:

```bash
sudo -u postgres psql -c "SELECT client_addr,state,sync_state FROM pg_stat_replication;"
```

You should see DB1.

---

# PART 51 — Test the opposite failover

Now you can test the other direction.

Current:

```text
DB2 = PRIMARY
DB1 = REPLICA
```

Stop Patroni on DB2.

## DB2

```bash
sudo systemctl stop patroni
```

Watch DB1:

```bash
sudo journalctl -u patroni -f
```

Then:

```bash
sudo -u postgres /opt/patroni/bin/patronictl -c /etc/patroni/patroni.yml list
```

DB1 should become:

```text
Leader
```

And HAProxy should automatically route to DB1 again.

You've now demonstrated **failover in both directions**.

---

# PART 52 — Your final working architecture

After completing everything, your system looks like this:

![Image](https://images.openai.com/static-rsc-4/vG0fdBaKXNvgarW0DLz7eEJf5dzVyhxW_g9cNCP-g9jw7vyL1uxsScXhFr_SC8vJpBKb4_aQ5Qbd6qgJGqZwFi84U8k10xaHLM5YuBAodrQ1BehLGlXs2ZanOV2Dsfv8nWcG3Us6WLVdUZTT8X4392cYHyP8z_tvK6J4uc-vnhmjdqomU6bAg-H1GA7ZCaug?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/KUSfLud_8kILTe_wKilY-8pPrD4NHmalVMpAZZS16Z-CH7an3Q5GnA4c4K-3dLysCCHb2SeiwRavibgtX510ds2RZqLRm8ibcaAziwYTIDIxw1eo9Jfyau9Lck2Qdb4jXoOn-UafEp7B8_0NkviXTaB9SxkndebQoJaYHKA5cnFXJ-SLw9Y7wgXnPqYkvr9v?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Mo01yW0wr4MvtOiK3xbhlptTgWdHjZOwZ3CTbVja9n-tKejVC4SJ-nQ8aG0xPqKfdGONDo6cq7xxCeRVoxiLPBkG3zJX-flgNn_cEO5k0Ppd3GfdcHPJ2ZcBlT-m669fgm96rHkuanqVL_ahW-wvszT-FBir4p1pl1_EqGTII9YuWzsr7R5QbBKkN7y9eHGU?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/9DMmPjQifjXk7RK9MXukDpA2WLdSGPAaQM8iybkvoE2DkCvukrY-rGDwXZ-Dm5upi4qBjwH7VTXDEbFXsuJ6AXSTMsN7_s3VUqRKq72D22D_Ycf0G7ojNHU6Eyrk7dPFLDoVhGE25tZpVUOGtQL9GWpX1yWLUEpHbbd3LsYdCbMW3J3_SwZjRutHbjkDx9bf?purpose=fullsize)

```text
                         ┌───────────────────┐
                         │     BACKEND       │
                         │                   │
                         │ Backend App       │
                         │       │           │
                         │       ▼           │
                         │    HAProxy        │
                         │     :5000         │
                         │       │           │
                         │    etcd #1        │
                         └───────┬───────────┘
                                 │
                ┌────────────────┼────────────────┐
                │                │                │
              etcd             etcd             etcd
              2379              2379              2379
                │                │                │
        ┌───────▼───────┐                ┌───────▼───────┐
        │      DB1      │◄─── WAL ──────►│      DB2      │
        │               │                │               │
        │  PostgreSQL   │                │  PostgreSQL   │
        │  Patroni      │                │  Patroni      │
        │  etcd #2      │                │  etcd #3      │
        │               │                │               │
        │   PRIMARY     │                │   REPLICA     │
        └───────────────┘                └───────────────┘
```

---

# PART 53 — What happens during normal operation?

```text
Application
     │
     ▼
HAProxy :5000
     │
     ▼
Patroni /primary
     │
     ├──── DB1 → 200 → PRIMARY
     │
     └──── DB2 → non-200 → REPLICA

DB1 PostgreSQL
     │
     │ WAL
     ▼
DB2 PostgreSQL
```

---

# PART 54 — What happens during failure?

```text
             DB1
           PRIMARY
              ❌
              │
              ▼
          Patroni detects
              │
              ▼
             etcd
              │
              ▼
        leader election
              │
              ▼
        DB2 becomes PRIMARY
              │
              ▼
       Patroni /primary
              │
              ▼
          HAProxy
              │
              ▼
        DB2 :5432
```

The application still uses:

```text
127.0.0.1:5000
```

It doesn't need to change its database configuration.

---

# PART 55 — What each technology is responsible for

This is extremely important for your DevOps understanding.

| Component             | Responsibility                                    |
| --------------------- | ------------------------------------------------- |
| PostgreSQL            | Stores database data                              |
| WAL                   | Records database changes for recovery/replication |
| Streaming replication | Sends WAL from primary to replica                 |
| Patroni               | Manages PostgreSQL HA and failover                |
| etcd                  | Stores cluster state/leader information           |
| HAProxy               | Gives application a stable DB endpoint            |
| Backend app           | Uses HAProxy rather than DB1/DB2 directly         |

Remember:

```text
etcd       = coordination
Patroni    = PostgreSQL HA management
PostgreSQL = database
WAL        = replication/recovery mechanism
HAProxy    = traffic routing
```

**HAProxy does not perform replication.**

**etcd does not store your PostgreSQL database.**

**Patroni does not replace PostgreSQL.**

They each have a different job.

---

# PART 56 — The checkpoints I want you to use

Don't execute this entire thing blindly in one sitting.

Treat it as checkpoints:

```text
CHECKPOINT 1
    ↓
3 EC2 servers + SSM
    ↓
CHECKPOINT 2
    ↓
Security Groups
    ↓
CHECKPOINT 3
    ↓
Network connectivity
    ↓
CHECKPOINT 4
    ↓
PostgreSQL installed
    ↓
CHECKPOINT 5
    ↓
3-node etcd healthy
    ↓
CHECKPOINT 6
    ↓
Patroni installed
    ↓
CHECKPOINT 7
    ↓
DB1 PRIMARY
DB2 REPLICA
    ↓
CHECKPOINT 8
    ↓
Streaming replication verified
    ↓
CHECKPOINT 9
    ↓
HAProxy installed
    ↓
CHECKPOINT 10
    ↓
Application → HAProxy → PRIMARY
    ↓
CHECKPOINT 11
    ↓
Kill DB1
    ↓
CHECKPOINT 12
    ↓
DB2 automatically becomes PRIMARY
    ↓
CHECKPOINT 13
    ↓
HAProxy automatically routes to DB2
    ↓
CHECKPOINT 14
    ↓
DB1 safely rejoins as REPLICA
    ↓
CHECKPOINT 15
    ↓
Test reverse failover
```

### One important recommendation

**Don't start entering the Patroni YAML yet if you haven't completed the earlier checkpoints.** The most common beginner mistake with this type of setup is trying to debug Patroni while the underlying etcd/network/PostgreSQL configuration isn't healthy.

Start with **Parts 1–10**, verify the three servers and PostgreSQL are clean, then proceed to etcd. When you reach the etcd checkpoint, verify `member list` and `endpoint health` before moving on. That way, if something fails, we know exactly which layer is responsible.
