```bash
useradd -m -s /bin/bash redis
usermod -aG sudo redis

# Set password
passwd redis

# Setup default Gateway for NAT and VPN communication.

# Update system
sudo apt update
sudo apt upgrade -y

# Install Redis
sudo apt install redis-server redis-tools -y

# Verify installation
redis-server --version

# Check available disks
lsblk
sudo mkdir -p /mnt/HC_Volume_102831732/redis-data
sudo chown -R redis:redis /mnt/HC_Volume_102831732/redis-data/
sudo chmod 755 /mnt/HC_Volume_102831732/redis-data

sudo mkdir -p /mnt/HC_Volume_102831748/redis-data
sudo chown -R redis:redis /mnt/HC_Volume_102831748/redis-data/
sudo chmod 755 /mnt/HC_Volume_102831748/redis-data

sudo mkdir -p /mnt/HC_Volume_102832334/redis-data
sudo chown -R redis:redis /mnt/HC_Volume_102832334/redis-data/
sudo chmod 755 /mnt/HC_Volume_102832334/redis-data


sudo cp /etc/redis/redis.conf /etc/redis/redis.conf.backup


# Redis configuration file for cluster setup
# Replace the entire /etc/redis/redis.conf with this content

################################## NETWORK #####################################
# Bind to all interfaces for cluster communication
bind 0.0.0.0

# Protected mode disabled for private network cluster
protected-mode no

# Default Redis port
port 6379

# TCP keepalive
tcp-keepalive 300

################################# GENERAL #####################################
# Run as daemon
daemonize yes

# PID file
pidfile /run/redis/redis-server.pid

# Log level
loglevel notice

# Log file
logfile /var/log/redis/redis-server.log

# Number of databases
databases 16

################################ SNAPSHOTTING  ################################
# RDB persistence settings
save 900 1
save 300 10
save 60 10000

# Stop writes on save error
stop-writes-on-bgsave-error yes

# Compress RDB files
rdbcompression yes

# Checksum RDB files
rdbchecksum yes

# RDB filename
dbfilename dump.rdb

# Data directory (using your SSD mount)
dir /mnt/HC_Volume_102831732/redis-data/

################################# REPLICATION #################################
# No replication settings needed for all-master cluster

############################## MEMORY MANAGEMENT ################################
# Memory limit (adjust based on your available RAM)
maxmemory 2gb

# Eviction policy
maxmemory-policy allkeys-lru

################################ REDIS CLUSTER  ###############################
# Enable cluster mode
cluster-enabled yes

# Cluster configuration file (will be created automatically)
cluster-config-file /mnt/HC_Volume_102831732/redis-data/nodes-6379.conf

# Cluster node timeout
cluster-node-timeout 15000

# Allow cluster to operate with missing slots (important for all-master setup)
cluster-require-full-coverage no

############################## APPEND ONLY MODE ###############################
# Enable AOF persistence
appendonly yes

# AOF filename
appendfilename "appendonly.aof"

# AOF directory
appenddirname "appendonlydir"

# AOF sync policy
appendfsync everysec

# AOF rewrite settings
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# Don't fsync during rewrites
no-appendfsync-on-rewrite no

# Load truncated AOF files
aof-load-truncated yes

# Use RDB preamble in AOF rewrites (more efficient)
aof-use-rdb-preamble yes

################################### CLIENTS ####################################
# Max clients
maxclients 10000

############################### ADVANCED CONFIG ###############################
# Hash settings
hash-max-listpack-entries 512
hash-max-listpack-value 64

# List settings
list-max-listpack-size -2
list-compress-depth 0

# Set settings
set-max-intset-entries 512

# Sorted set settings
zset-max-listpack-entries 128
zset-max-listpack-value 64

# HyperLogLog settings
hll-sparse-max-bytes 3000

# Stream settings
stream-node-max-bytes 4096
stream-node-max-entries 100

# Active rehashing
activerehashing yes

# Client output buffer limits
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60

# Background task frequency
hz 10
dynamic-hz yes

# Incremental fsync for AOF and RDB
aof-rewrite-incremental-fsync yes
rdb-save-incremental-fsync yes

# Lazy freeing
lazyfree-lazy-eviction no
lazyfree-lazy-expire no
lazyfree-lazy-server-del no
replica-lazy-flush no
lazyfree-lazy-user-del no
lazyfree-lazy-user-flush no

# Jemalloc background thread
jemalloc-bg-thread yes

# Disable transparent huge pages for better performance
disable-thp yes



sudo ufw status
# Enable and start Redis
sudo mkdir -p /etc/systemd/system/redis-server.service.d/

sudo tee /etc/systemd/system/redis-server.service.d/override.conf << EOF
[Service]
PrivateUsers=false
PrivateTmp=false
ProtectSystem=false
ProtectHome=false
NoNewPrivileges=false
EOF

sudo systemctl daemon-reload
sudo systemctl enable redis-server
sudo systemctl start redis-server

# Check status
sudo systemctl status redis-server
sudo systemctl reset-failed redis-server

# Test Redis
redis-cli ping

sudo netstat -tlnp | grep 6379

redis-cli --cluster create \
  10.1.102.1:6379 \
  10.1.102.2:6379 \
  10.0.102.1:6379 \
  --cluster-replicas 0

# Press 'Yes'

# Connect to remote Redis and test
redis-cli -h 10.1.102.1 -p 6379
SET remote_test "Connected to remote Redis"
GET remote_test
DEL remote_test
exit

```