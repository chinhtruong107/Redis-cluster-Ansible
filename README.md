# Redis Cluster Ansible Automation

Automated deployment of a 6-node Redis Cluster on AWS EC2 using Ansible.

---

## Requirements

- Control node: Ubuntu 22.04 with Ansible installed
- 6 managed nodes: Ubuntu 22.04 EC2 instances
- SSH key pair for EC2 access
- AWS Security Group with ports open:
  - 22/tcp
  - 6379/tcp
  - 16379/tcp
- Dedicated data volume mounted at `/data`

---

## Project Structure

```text
Redis-Cluster/
├── ansible.cfg
├── inventory/
│   └── inventory.ini
├── group_vars/
│   └── redis-cluster.yml
├── playbooks/
│   └── playbook.yml
├── roles/
│   ├── common/                 # Common host configuration
│   └── installation/           # Redis installation and configuration
├── templates/
│   ├── disable-thp.service.j2
│   ├── 99-redis.conf.j2
│   ├── redis.conf.j2
│   └── users.acl.j2
└── README.md
```

---

## Architecture

Cluster topology:

```text
redis-01  ─┐
redis-02  ─┼── Primary Nodes
redis-03  ─┘

redis-04  ─┐
redis-05  ─┼── Replica Nodes
redis-06  ─┘
```

Deployment target:

```text
3 Primary Nodes
3 Replica Nodes
Replication Factor = 1
16384 Hash Slots
```

---

## Configuration

Edit `group_vars/redis-cluster.yml`:

```yaml
# SSH
ansible_user: ubuntu
ansible_ssh_private_key_file: /home/ubuntu/.ssh/redis-cluster.pem

# Redis
redis_client_port: 6379
redis_cluster_port: 16379

# Memory
redis_maxmemory: 512mb

# Replication
redis_master_user: repl
redis_master_password: RedisRepl@2026
```

Edit `inventory/inventory.ini`:

```ini
[redis]
redis-01 ansible_host=172.31.27.194
redis-02 ansible_host=172.31.21.147
redis-03 ansible_host=172.31.30.70
redis-04 ansible_host=172.31.24.51
redis-05 ansible_host=172.31.23.131
redis-06 ansible_host=172.31.16.78
```

---

## What This Playbook Configures

### System Configuration

- Update package cache
- Install required packages
- Configure `/etc/hosts`
- Disable Transparent Huge Pages (THP)
- Configure kernel tuning parameters

### Sysctl Tuning

```text
vm.overcommit_memory = 1
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
```

### Redis Configuration

- Install Redis from the official Redis repository
- Configure Redis Cluster mode
- Configure persistence:
  - RDB
  - AOF
- Configure replication authentication
- Configure ACL authentication
- Configure data directory

### Storage Validation

- Verify mounted `/data`
- Verify write access
- Verify ownership and permissions

---

## Usage

### Verify Connectivity

```bash
ansible redis -i inventory/inventory.ini -m ping
```

Expected:

```text
SUCCESS => pong
```

---

### Run Deployment

```bash
ansible-playbook \
-i inventory/inventory.ini \
playbooks/playbook.yml \
--become
```

---

### Dry Run

```bash
ansible-playbook \
-i inventory/inventory.ini \
playbooks/playbook.yml \
--check \
--become
```

---

## Verify Installation

### Redis Version

```bash
redis-server --version
```

### Service Status

```bash
systemctl status redis-server
```

### Verify Listening Ports

```bash
ss -lntp | grep -E '6379|16379'
```

Expected:

```text
6379  -> Client Port
16379 -> Cluster Bus Port
```

### Verify Data Directory

```bash
redis-cli config get dir
```

Expected:

```text
dir
/data/redis/6379
```

---

## Create Redis Cluster

Run from any node:

```bash
redis-cli --cluster create \
172.31.27.194:6379 \
172.31.21.147:6379 \
172.31.30.70:6379 \
172.31.24.51:6379 \
172.31.23.131:6379 \
172.31.16.78:6379 \
--cluster-replicas 1
```

Confirm:

```text
Can I set the above configuration? (type 'yes' to accept)
```

Type:

```text
yes
```

---

## Verify Cluster

### Cluster Status

```bash
redis-cli -c -h 172.31.27.194 cluster info
```

Expected:

```text
cluster_state:ok
cluster_slots_assigned:16384
```

### Cluster Nodes

```bash
redis-cli -c -h 172.31.27.194 cluster nodes
```

Expected node format:

```text
172.31.x.x:6379@16379
```

---

## Functional Testing

### Read / Write Test

```bash
redis-cli -c -h 172.31.27.194
```

```redis
SET test:key1 hello
GET test:key1
```

Expected:

```text
hello
```

---

### Replication Verification

```bash
redis-cli info replication
```

---

### Memory Verification

```bash
redis-cli info memory
```

---

### Slot Coverage Verification

```bash
redis-cli cluster info
```

Expected:

```text
cluster_slots_ok:16384
```

---

## Failover Test

Stop a primary node:

```bash
systemctl stop redis-server
```

Verify cluster:

```bash
redis-cli -c -h 172.31.21.147 cluster nodes
```

Expected:

- Replica promoted to Primary
- Cluster remains healthy
- Automatic failover occurs

Restart node:

```bash
systemctl start redis-server
```

Expected:

- Node rejoins cluster
- Replication restored

---

## Scale Out

### Add New Primary Node

```bash
redis-cli --cluster add-node \
172.31.0.104:6379 \
172.31.27.194:6379
```

Rebalance slots:

```bash
redis-cli --cluster rebalance \
172.31.27.194:6379 \
--cluster-use-empty-masters
```

### Add Replica Node

```bash
redis-cli --cluster add-node \
172.31.0.110:6379 \
172.31.27.194:6379 \
--cluster-slave \
--cluster-master-id <MASTER_ID>
```

---

## Security Group Rules

| Port | Protocol | Purpose |
|--------|----------|----------|
| 22 | TCP | SSH |
| 6379 | TCP | Redis Client Port |
| 16379 | TCP | Redis Cluster Bus |

All Redis nodes must be able to communicate with each other on both Redis ports.

---

## Roles

| Role | Responsibility |
|--------|---------------|
| `common` | Common host configuration and prerequisites |
| `installation` | Redis installation, tuning, storage validation, configuration deployment |

---

## Troubleshooting

### Check Redis Logs

```bash
journalctl -u redis-server -f
```

### Check Cluster Health

```bash
redis-cli --cluster check <NODE_IP>:6379
```

### Check Redis Ports

```bash
ss -lntp | grep redis
```

### Check Open File Limits

```bash
cat /proc/$(pidof redis-server)/limits | grep "Max open files"
```

Expected:

```text
Max open files 100000 100000 files
```