# Percona XtraDB Cluster 8.0: 6+1 跨VPC高可用部署指南 (v8 - Corrected)

## 0. 方案簡介

本文檔提供了一個在**兩個 VPC** 的 **6 台 ENS 主機**上部署 Percona XtraDB Cluster (PXC) 8.0，並在**第三地點**部署一台**仲裁器 (Arbiter)** 的完整、生產級高可用方案的詳細步驟。

本指南**特別整合了經過實戰檢驗的、用於規避常見安裝陷阱的詳細流程**，並提供了一個經過充分調優且註釋精簡的配置文件範本。

---

## 階段零：環境清理 (可選，用於部署失敗後)

此腳本用於在部署失敗後，將節點恢復到絕對純淨的狀態，以便重新部署。**僅在需要時執行。**

```bash
# Execute on a failed PXC data node
#
# Defines a function to safely kill the mysqld process
kill_mysqld_process() {
    MYSQL_PID=$(ps aux | grep '[m]ysqld' | awk '{print $2}')
    if [ -n "$MYSQL_PID" ]; then
        echo "Found mysqld process (PID: $MYSQL_PID), force killing..."
        sudo kill -9 "$MYSQL_PID"
    fi
}

# Run cleanup
kill_mysqld_process

# Rename post-removal scripts that might cause errors, forcing the purge to succeed
sudo mv /var/lib/dpkg/info/percona-xtradb-cluster-server.postrm /var/lib/dpkg/info/percona-xtradb-cluster-server.postrm.bak 2>/dev/null || true
sudo mv /var/lib/dpkg/info/percona-xtradb-cluster-server.postinst /var/lib/dpkg/info/percona-xtradb-cluster-server.postinst.bak 2>/dev/null || true

# Force dpkg state repair and completely purge packages
sudo dpkg --configure -a
sudo apt-get -f install -y
sudo apt-get purge -y 'percona*'
sudo apt-get autoremove -y --purge

# Clean up all remaining directories and file contents
sudo rm -rf /etc/mysql /var/log/mysql
# Critical: Only clear the contents, do not delete the directory itself
sudo mkdir -p /var/lib/mysql && sudo rm -rf /var/lib/mysql/*
```

---

## 階段一：主機初始化 (在所有7個節點上執行)

### 1.1 設置主機名

```bash
# === On the three hosts in VPC Macau 1, execute respectively ===
sudo hostnamectl set-hostname vpc-mo-1-node001
sudo hostnamectl set-hostname vpc-mo-1-node002
sudo hostnamectl set-hostname vpc-mo-1-node003

# === On the three hosts in VPC Macau 2, execute respectively ===
sudo hostnamectl set-hostname vpc-mo-2-node001
sudo hostnamectl set-hostname vpc-mo-2-node002
sudo hostnamectl set-hostname vpc-mo-2-node003

# === On the arbiter host, execute ===
sudo hostnamectl set-hostname arbiter-hk-node001
```

### 1.2 配置 /etc/hosts

在**所有7個節點**上，編輯 `/etc/hosts` 文件。**請將以下 IP 地址替換為您的真實內網 IP**。

```
# /etc/hosts

# --- VPC Macau 1 ---
10.1.1.1  vpc-mo-1-node001
10.1.1.2  vpc-mo-1-node002
10.1.1.3  vpc-mo-1-node003

# --- VPC Macau 2 ---
10.2.2.1  vpc-mo-2-node001
10.2.2.2  vpc-mo-2-node002
10.2.2.3  vpc-mo-2-node003

# --- Arbiter ---
10.3.3.1  arbiter-hk-node001
```

### 1.3 禁用 SWAP (推薦)

```bash
sudo swapoff -a
# Permanently disable by commenting out the swap line in /etc/fstab
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

---

## 階段二：安裝所需軟件 (在對應節點上執行)

### 2.1 添加 Percona 軟件源 (在所有7個節點上執行)

```bash
sudo apt-get update
sudo apt-get install -y wget curl gnupg2 lsb-release
wget https://repo.percona.com/apt/percona-release_latest.generic_all.deb
sudo dpkg -i percona-release_latest.generic_all.deb
sudo percona-release setup pxc80
sudo apt-get update
```

### 2.2 安裝 PXC 資料庫節點 (在6台數據主機上執行)

在澳門 VPC 1 和 VPC 2 的 **6 台 ENS 主機**上安裝 PXC 伺服器、xtrabackup 和工具包。

```bash
# Ensure the default directory exists with correct permissions
sudo mkdir -p /var/lib/mysql && sudo chown mysql:mysql /var/lib/mysql

# Execute installation
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y \
percona-xtradb-cluster \
percona-toolkit \
percona-xtrabackup-80
```

### 2.3 安裝 Galera 仲裁器 (僅在仲裁主機上執行)

在位於第三地點的**仲裁主機**上，僅安裝 `garb`。

```bash
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y percona-xtradb-cluster-garbd
```

---

## 階段三：配置集群 (核心步驟)

### 3.1 配置 PXC 資料庫節點 (在6台數據主機上)

#### 完整配置文件範例 (用於 `vpc-mo-1-node001`)

將以下內容完整複製到 `vpc-mo-1-node001` 的 `/etc/mysql/mysql.conf.d/mysqld.cnf`。對於其他5個節點，請以此為範本，僅修改 `Node Specific Settings` 部分。

```ini
[mysqld]
# =================================================================
# Base Settings
# =================================================================
require_secure_transport        = ON
user                            = mysql
port                            = 3306
datadir                         = /data/mysql
pid-file                        = /var/run/mysqld/mysqld.pid
socket                          = /var/run/mysqld/mysqld.sock
tmpdir                          = /tmp
skip-name-resolve
explicit_defaults_for_timestamp
default_authentication_plugin   = mysql_native_password
character-set-server            = utf8mb4
collation-server                = utf8mb4_0900_ai_ci
init-connect                    = 'SET NAMES utf8mb4'

# =================================================================
# Connection & Limits
# =================================================================
max_connections                 = 2048
max_connect_errors              = 100000
back_log                        = 300
thread_cache_size               = 64
open_files_limit                = 65535
table_open_cache                = 1024
interactive_timeout             = 28800
wait_timeout                    = 28800

# =================================================================
# Buffer & Cache
# =================================================================
max_allowed_packet              = 500M
binlog_cache_size               = 1M
max_heap_table_size             = 8M
tmp_table_size                  = 128M
read_buffer_size                = 2M
read_rnd_buffer_size            = 8M
sort_buffer_size                = 8M
join_buffer_size                = 8M
key_buffer_size                 = 4M
bulk_insert_buffer_size         = 8M

# =================================================================
# Logging
# =================================================================
log-error                       = /data/logs/mysql/mysqld-error.log
slow_query_log                  = 1
long_query_time                 = 1
slow_query_log_file             = /data/logs/mysql/mysql-slow.log
log_queries_not_using_indexes   = ON
expire_logs_days                = 7

# =================================================================
# SSL/TLS Settings
# =================================================================
ssl_ca                          = /etc/mysql/ssl/ca.pem
ssl_cert                        = /etc/mysql/ssl/server-cert.pem
ssl_key                         = /etc/mysql/ssl/server-key.pem

# =================================================================
# InnoDB
# =================================================================
default_storage_engine          = InnoDB
innodb_file_per_table           = 1
innodb_open_files               = 500
# Sized for 4GB RAM
innodb_buffer_pool_size         = 2G
innodb_log_file_size            = 512M
innodb_flush_log_at_trx_commit  = 2
# Use O_DIRECT on Linux to avoid double buffering
innodb_flush_method             = O_DIRECT
innodb_write_io_threads         = 4
innodb_read_io_threads          = 4
innodb_thread_concurrency       = 0
innodb_purge_threads            = 1
innodb_log_buffer_size          = 2M
innodb_max_dirty_pages_pct      = 90
innodb_lock_wait_timeout        = 120

# =================================================================
# MyISAM
# =================================================================
myisam_sort_buffer_size         = 64M
myisam_max_sort_file_size       = 10G

# =================================================================
# Binary Log & GTID
# =================================================================
# PXC requires ROW format
binlog_format                   = ROW
log_bin                         = /data/logs/mysql/mysql-bin
log_slave_updates               = ON
gtid_mode                       = ON
enforce_gtid_consistency        = ON

# =================================================================
# Galera Cluster
# =================================================================
wsrep_on                        = ON
wsrep_provider                  = /usr/lib/galera4/libgalera_smm.so
wsrep_sst_method                = xtrabackup-v2
pxc_strict_mode                 = ENFORCING
pxc-encrypt-cluster-traffic     = ON
# Match the number of CPU cores
wsrep_slave_threads             = 2
wsrep_retry_autocommit          = 10
wsrep_provider_options          ="gcs.fc_limit=512; gcs.fc_factor=1.0"
transaction-isolation           = READ-COMMITTED

# Cluster topology. MUST be identical across all nodes.
wsrep_cluster_name              = "pxc-mo-cluster"
wsrep_cluster_address           = "gcomm://vpc-mo-1-node001,vpc-mo-1-node002,vpc-mo-1-node003,vpc-mo-2-node001,vpc-mo-2-node002,vpc-mo-2-node003,arbiter-hk-node001"

# =================================================================
# Performance Schema
# =================================================================
performance_schema_consumer_events_waits_current=ON
performance_schema_consumer_events_waits_history=ON
performance_schema_consumer_events_waits_history_long=ON
performance_schema_instrument='wait/%=ON'

# =================================================================
# Node Specific Settings - MUST BE UNIQUE PER NODE
# =================================================================
wsrep_node_name                 = "vpc-mo-1-node001"
wsrep_node_address              = "10.1.1.1"
server-id                       = 101
```

#### 其他節點的修改指南

* **vpc-mo-1-node002**: `wsrep_node_name` 改為 "vpc-mo-1-node002", `wsrep_node_address` 改為其 IP, `server-id` 改為 `102`。
* **vpc-mo-1-node003**: `wsrep_node_name` 改為 "vpc-mo-1-node003", `wsrep_node_address` 改為其 IP, `server-id` 改為 `103`。
* **vpc-mo-2-node001**: `wsrep_node_name` 改為 "vpc-mo-2-node001", `wsrep_node_address` 改為其 IP, `server-id` 改為 `201`。
* **vpc-mo-2-node002**: `wsrep_node_name` 改為 "vpc-mo-2-node002", `wsrep_node_address` 改為其 IP, `server-id` 改為 `202`。
* **vpc-mo-2-node003**: `wsrep_node_name` 改為 "vpc-mo-2-node003", `wsrep_node_address` 改為其 IP, `server-id` 改為 `203`。

### 3.2 配置仲裁器節點 (僅在仲裁主機上)

在**仲裁主機**上，編輯 `/etc/default/garb` 文件。

```bash
# /etc/default/garb - Configuration for Galera Arbitrator Daemon

# Cluster connection address.
# The startup script adds the "gcomm://" prefix automatically.
# This list MUST BE IDENTICAL to wsrep_cluster_address on data nodes.
GALERA_NODES="vpc-mo-1-node001,vpc-mo-1-node002,vpc-mo-1-node003,vpc-mo-2-node001,vpc-mo-2-node002,vpc-mo-2-node003,arbiter-hk-node001"

# Cluster name. MUST be IDENTICAL to wsrep_cluster_name.
GALERA_GROUP="pxc-mo-cluster"

# Optional: Galera internal options string (e.g. SSL settings)
# see https://galeracluster.com/library/documentation/galera-parameters.html
GALERA_OPTIONS="socket.ssl=YES; socket.ssl_key=/etc/mysql/ssl/server-key.pem; socket.ssl_cert=/etc/mysql/ssl/server-cert.pem; socket.ssl_ca=/etc/mysql/ssl/ca.pem; socket.ssl_cipher=AES128-SHA256"

# Optional: Log file for garbd. By default, logs to syslog.
# Deprecated for systemd services, use "journalctl -u garb" instead.
# LOG_FILE=""
```

---

## 階段四：啟動集群 (遵循安全流程)

集群的首次啟動必須嚴格遵循引導流程。

### 4.1 首次引導流程

1. **在任意一個 PXC 數據節點上啟動引導 (例如 vpc-mo-1-node001)**

   ```bash
   sudo systemctl start mysql@bootstrap.service
   ```
2. **在其餘 5 個 PXC 數據節點上啟動並加入集群**

   ```bash
   sudo systemctl start mysql
   ```
3. **在仲裁主機上啟動並啟用仲裁器**
   `enable` 命令確保主機重啟後服務會自動運行。

   ```bash
   # On the arbiter host (arbiter-hk-node001)
   sudo systemctl start garb
   sudo systemctl enable garb
   ```

   您可以檢查其狀態以確保它正在運行並已連接到集群：

   ```bash
   sudo systemctl status garb
   ```
4. **回到引導節點 (`vpc-mo-1-node001`) 上停止引導服務**

   ```bash
   sudo systemctl stop mysql@bootstrap.service
   ```
5. **(安全保障) 在引導節點上清理殘留進程**

   ```bash
   sudo pkill mysqld
   sleep 3
   ```
6. **在引導節點上作為普通成員啟動**

   ```bash
   sudo systemctl start mysql
   ```
7. **最終驗證**
   等待約一分鐘，登錄任意一個 PXC 節點，檢查集群大小。

   ```bash
   mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
   ```

   您應該看到結果為 **7**。

---

## 階段五：安全設置 (關鍵步驟)

### 5.1 重置 Root 密碼

在集群**首次啟動成功後**，選擇**任意一個數據節點**執行以下操作。

```bash
# 1. Stop the mysql service on the target node
sudo systemctl stop mysql

# 2. Start mysqld manually in the background, skipping grant tables and disabling wsrep
sudo mysqld --skip-grant-tables --wsrep-on=OFF &
sleep 5 # Wait for mysqld to start

# 3. Use a heredoc to automatically input commands to reset the root password
# Replace <YOUR_STRONG_PASSWORD> with your actual password
mysql -u root <<EOSQL
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY '<YOUR_STRONG_PASSWORD>';
FLUSH PRIVILEGES;
EOSQL

# 4. Kill the mysqld process we started manually
sudo pkill mysqld
sleep 10 # Wait for the process to shut down completely

# 5. Restart the node in normal cluster mode
sudo systemctl start mysql
```

---

## 階段六：從完全關閉狀態下恢復集群（含 TLS 與仲裁器）

當所有節點皆關閉或重啟時，**切勿**同時在所有節點上直接啟動 MySQL。必須選定一台作為引導節點（bootstrap）啟動，其餘節點依序加入。以下流程同時適用於已啟用叢集內部 TLS 的情況。

### 6.1 前置檢查（所有數據節點）

```bash
sudo grep -E 'safe_to_bootstrap|seqno' /data/mysql/grastate.dat || sudo grep -E 'safe_to_bootstrap|seqno' /var/lib/mysql/grastate.dat
```

- 若有節點顯示 `safe_to_bootstrap: 1`，優先選該節點作為引導節點。
- 若全部為 `safe_to_bootstrap: 0`，選擇 `seqno` 最大的節點作為引導節點。

同時確認 TLS 檔一致（所有數據節點）：

```bash
ls -l /etc/mysql/ssl
sha256sum /etc/mysql/ssl/ca.pem
```

- `ca.pem` 必須一致；每台的 `server-cert.pem`、`server-key.pem` 各自對應本機主機名/IP 的 SAN。
- 每台均需設定一致的 `pxc-encrypt-cluster-traffic = ON` 與相同的 `ssl_ca/ssl_cert/ssl_key` 路徑。

### 6.2 關閉所有服務

```bash
# 所有 6 台數據節點
sudo systemctl stop mysql

# 仲裁器
sudo systemctl stop garb
```

### 6.3 設定並啟動引導節點

若選定的引導節點其 `safe_to_bootstrap` 為 0，請先設為 1（請先備份檔案）：

```bash
sudo cp /data/mysql/grastate.dat /data/mysql/grastate.dat.bak.$(date +%s) 2>/dev/null || true
sudo cp /var/lib/mysql/grastate.dat /var/lib/mysql/grastate.dat.bak.$(date +%s) 2>/dev/null || true
sudo sed -i 's/^safe_to_bootstrap: .*/safe_to_bootstrap: 1/' /data/mysql/grastate.dat 2>/dev/null || true
sudo sed -i 's/^safe_to_bootstrap: .*/safe_to_bootstrap: 1/' /var/lib/mysql/grastate.dat 2>/dev/null || true
```

在引導節點啟動叢集：

```bash
sudo systemctl start mysql@bootstrap.service
mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size';"   # 預期為 1
```

若啟動失敗，請檢查 `/data/logs/mysql/mysqld-error.log` 是否有 TLS 相關的 `unknown ca / wrong version number`，代表對端節點尚未以 TLS 正確加入；此時引導節點已啟動為 1 成員，繼續下一步讓其他節點依序加入。

### 6.4 依序啟動其餘數據節點（逐台）

```bash
sudo systemctl start mysql
mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size';"   # 觀察由 1 → 2 → 3 ... 直至 6
```

每加入一台，觀察 `wsrep_cluster_size` 遞增；若某台加入失敗，檢視該台錯誤日誌並排除（見 6.7 排錯）。

### 6.5 啟動仲裁器

資料節點穩定形成主成員後，於仲裁器主機啟動 `garb`：

```bash
sudo systemctl start garb
sudo systemctl status garb | cat
```

### 6.6 引導節點恢復為一般成員

引導完成後，將引導節點從 bootstrap 模式切回一般模式：

```bash
sudo systemctl stop mysql@bootstrap.service
sudo pkill mysqld || true
sleep 3
sudo systemctl start mysql
```

### 6.7 最終驗證

在任一數據節點驗證：

```bash
mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size';"           # 預期 7（含仲裁器）
mysql -e "SHOW VARIABLES LIKE 'pxc_encrypt_cluster_traffic';"  # 預期 ON（如已啟用）
mysql -e "SHOW VARIABLES LIKE 'have_ssl';"                  # 預期 YES
```

### 6.8 常見故障排查

- `wrong version number / unknown ca / invalid CA certificate`
  - 確認所有數據節點與仲裁器均使用相同 `ca.pem`，且各節點的 `server-cert.pem` 含本機主機名/IP 的 SAN。
  - 確認每台的 `ssl_ca/ssl_cert/ssl_key` 路徑與權限正確（建議：資料節點檔案屬主 `mysql:mysql`，仲裁器檔案可由 `garb` 讀取）。
- `failed to reach primary view (pc.wait_prim_timeout)`
  - 確認依「引導 → 逐台加入 → 啟動仲裁器」順序；不要同時啟動全部節點。
- `Unrecognized parameter` 類錯誤
  - 移除不支援或已棄用的 `wsrep_provider_options` 參數（例如 `gcert.pc_lazy_deserialization`、`gcs.fc_master_slave`）。

