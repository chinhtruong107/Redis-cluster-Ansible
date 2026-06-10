# Redis Cluster Ansible Automation

Automated deployment of a 6-node Redis Cluster on AWS EC2 using Ansible.

---

## Requirements

- Control node: Ubuntu 22.04 with Ansible installed
- 6 managed nodes: Ubuntu 22.04 EC2 t2.micro instances
- SSH key pair for EC2 access
- AWS Security Group with ports open:
  - 22/tcp
  - 6379/tcp
  - 16379/tcp

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
│   ├── playbook.yml
│   ├── backup.yml
│   ├── restore.yml
│   └── scale.yml
├── roles/
│   ├── common/
│   ├── installation/
│   ├── cluster/
│   ├── testing/
│   ├── backup/
│   ├── restore/
│   └── scale/
└── README.md
```

---

## Architecture

Cluster topology:

```text
redis-01  ─┐
redis-02  ─┼── Primary Nodes (Master)
redis-03  ─┘

redis-04  ─┐
redis-05  ─┼── Replica Nodes
redis-06  ─┘
```

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
redis_client_port: 6379
redis_cluster_port: 16379
redis_maxmemory: 512mb
redis_master_user: repl
redis_master_password: <REPLICATION_PASSWORD>
redis_admin_password: <ADMIN_PASSWORD>
redis_cluster_replicas: 1
backup_timestamp: <TIMESTAMP>
```

Edit `inventory/inventory.ini`:

```ini
[redis]
redis-01 ansible_host=172.31.x.x
redis-02 ansible_host=172.31.x.x
redis-03 ansible_host=172.31.x.x
redis-04 ansible_host=172.31.x.x
redis-05 ansible_host=172.31.x.x
redis-06 ansible_host=172.31.x.x

[redis:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/redis-cluster.pem
```

---

## What This Playbook Does

### common role
- Update package cache
- Install required packages

### installation role
- Configure `/etc/hosts`
- Install Redis from official Redis repository
- Disable Transparent Huge Pages (THP)
- Kernel tuning: `vm.overcommit_memory`, `somaxconn`, `tcp_max_syn_backlog`
- Create data directory and set permissions
- Configure `redis.conf`, `users.acl`, systemd override
- Configure tmpfiles.d for PID directory
- Enable and start Redis service
- Verify port, ping, cluster nodes, data directory

### cluster role
- Reset cluster data (`reset_cluster` tag)
- Create Redis Cluster (`init_cluster` tag)
- Verify cluster: info, nodes, health check
- Test read/write
- Verify replication, memory, slot coverage

### testing role
- Stop redis-01 to trigger failover
- Verify replica is promoted to master
- Restart redis-01 and verify rejoin
- Test read/write after failover
- Verify replication after recovery

### backup role
- Trigger AOF rewrite (`bgrewriteaof`) and wait for completion
- Trigger RDB snapshot (`bgsave`) and wait for completion
- Copy AOF directory and RDB file to `/data/backup/redis/`
- Compress backup into `.tar.gz` per node with timestamp

### restore role
- Stop Redis service
- Clear existing data (AOF, RDB, cluster config)
- Extract latest backup from `/data/backup/redis/`
- Restore AOF and RDB files
- Fix ownership and start Redis service
- Verify connectivity and replication status

### scale role
- Add new master node to existing cluster
- Rebalance slots across all masters
- Verify cluster health after scale

---

## Usage

### Verify Connectivity

```bash
ansible redis -i inventory/inventory.ini -m ping
```

### Deploy Cluster

```bash
ansible-playbook -i inventory/inventory.ini playbooks/playbook.yml --become
```

### Dry Run

```bash
ansible-playbook -i inventory/inventory.ini playbooks/playbook.yml --check --become
```

### Reset and Recreate Cluster

```bash
# Reset data on all nodes
ansible -i inventory/inventory.ini redis -m shell \
  -a "systemctl stop redis-server && \
      rm -f /data/redis/6379/nodes-6379.conf && \
      rm -rf /data/redis/6379/appendonlydir/* && \
      rm -f /data/redis/6379/dump.rdb && \
      systemctl start redis-server" \
  --become

# Recreate cluster
ansible-playbook -i inventory/inventory.ini playbooks/playbook.yml \
  --become --tags init_cluster
```

---

## Tags

| Tag | Description |
|---|---|
| `init_cluster` | Run cluster creation tasks only |
| `reset_cluster` | Clear old data and restart Redis on all nodes |
| `test_failover` | Run failover test (stop/start redis-01) |

Example — skip failover test:
```bash
ansible-playbook -i inventory/inventory.ini playbooks/playbook.yml \
  --become --skip-tags test_failover
```

---

## Backup

Run backup on all nodes:

```bash
ansible-playbook -i inventory/inventory.ini playbooks/backup.yml --become
```

Backup files are stored at `/data/backup/redis/` on each node:

```text
/data/backup/redis/
├── appendonlydir_<timestamp>/
├── dump_<timestamp>.rdb
└── backup_<hostname>_<timestamp>.tar.gz
```

Each node backs up its own data independently. Backup includes both AOF and RDB files.

---

## Restore

Run restore on all nodes:

```bash
ansible-playbook -i inventory/inventory.ini playbooks/restore.yml --become
```

The restore playbook will automatically:
1. Stop Redis service
2. Clear existing data
3. Extract the latest backup matching the node's hostname
4. Restore AOF and RDB files
5. Fix ownership and restart Redis
6. Verify connectivity and replication

---

## Scale Out

### Prepare

Add the new node to `inventory/inventory.ini`:

```ini
[redis_new]
redis-07 ansible_host=172.31.x.x
```

Reset the new node's data before adding to cluster:

```bash
ansible -i inventory/inventory.ini redis_new -m shell \
  -a "systemctl stop redis-server && \
      rm -f /data/redis/6379/nodes-6379.conf && \
      rm -rf /data/redis/6379/appendonlydir/* && \
      rm -f /data/redis/6379/dump.rdb && \
      systemctl start redis-server" \
  --become
```

### Run Scale Playbook

```bash
ansible-playbook -i inventory/inventory.ini playbooks/scale.yml \
  --become --skip-tags init_cluster
```

The scale playbook will automatically:
1. Setup the new node (install Redis, configure, start service)
2. Add the node to the existing cluster
3. Rebalance slots across all masters
4. Verify cluster health after scale

---

## Troubleshooting

### Redis fails to start

```bash
# Check logs
sudo journalctl -xeu redis-server.service | tail -30

# Test config directly
sudo redis-server /etc/redis/redis.conf 2>&1 | head -20

# Recreate PID directory (cleared on reboot)
sudo mkdir -p /run/redis && sudo chown redis:redis /run/redis
sudo chmod 755 /data
sudo systemctl start redis-server
```

### Cluster state is fail

```bash
# Check cluster status
redis-cli --user admin -a '<PASSWORD>' -h <NODE_IP> -p 6379 cluster info

# Check node list
redis-cli --user admin -a '<PASSWORD>' -h <NODE_IP> -p 6379 cluster nodes

# Check cluster health
redis-cli --cluster check <NODE_IP>:6379 --user admin -a '<PASSWORD>'
```

### Ghost node in cluster (disconnected)

```bash
# Remove ghost node from all nodes
ansible -i inventory/inventory.ini redis -m shell \
  -a "redis-cli --user admin -a '<PASSWORD>' cluster forget <NODE_ID>" \
  --become
```

### Missing slot

```bash
# Reassign missing slot
redis-cli --user admin -a '<PASSWORD>' -h <NODE_IP> -p 6379 cluster addslots <SLOT_NUMBER>
```

### Check file descriptor limit

```bash
cat /proc/$(pidof redis-server)/limits | grep "Max open files"
# Expected: 100000
```

---

## Security Group Rules

| Port | Protocol | Purpose |
|---|---|---|
| 22 | TCP | SSH |
| 6379 | TCP | Redis Client Port |
| 16379 | TCP | Redis Cluster Bus |

Both Redis ports must be open between all nodes (self-referencing Security Group rule).

---

## Notes

- `maxmemory` is set to 512mb to fit within t2.micro (1GB RAM)
- `/run/redis` is cleared on reboot — handled automatically via `tmpfiles.d`
- Cluster creation only runs when `cluster_state:ok` and `cluster_known_nodes:6` are not yet satisfied
- Use `--skip-tags init_cluster` when re-running the playbook against an existing cluster
- Each node stores its own backup independently — restore uses the latest available backup per node
- New node must have clean data before being added to an existing cluster

---

## License

MIT License
