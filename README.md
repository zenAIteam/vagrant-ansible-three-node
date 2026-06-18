# Vagrant Ansible 三節點實驗環境

> 三節點自動化部署實驗環境 — Vagrant · Ansible · Docker · Prometheus · Grafana · Firewalld

目標是在本機模擬接近生產的多服務分層架構，練習從零部署到監控可視化的完整流程，並記錄實際遇到的問題與排查過程。

以 Vagrant 自動建立三台 VirtualBox 虛擬機，透過 Ansible Playbook 完整部署 Docker、WordPress 與 Prometheus/Grafana 監控系統，並以宣告式防火牆規則實作最小權限原則的實作練習專案。

---

## 架構圖

```mermaid
graph TD
    subgraph HOST["本機（VirtualBox + Vagrant）"]
        CN["control-node<br/>192.168.56.10<br/>ansible-core"]
        WS["web-server<br/>192.168.56.11<br/>Docker<br/>WordPress:80<br/>Node Exporter:9100"]
        MS["monitor-server<br/>192.168.56.12<br/>Prometheus:9090<br/>Grafana:3000"]
    end

    CN -- "ansible-playbook" --> WS
    CN -- "ansible-playbook" --> MS
    WS -- "抓取 :9100（僅限 56.12）" --> MS
```

---

## 節點說明

| 節點 | IP | 角色 |
|---|---|---|
| `control-node` | 192.168.56.10 | Ansible 控制節點 |
| `web-server` | 192.168.56.11 | Docker + WordPress + Node Exporter |
| `monitor-server` | 192.168.56.12 | Prometheus + Grafana |

---

## 使用技術

**Vagrant + VirtualBox** — 宣告式多 VM 佈建工具。三個節點皆定義於單一 `Vagrantfile`，執行 `vagrant up` 即可同時建立所有虛擬機。

**Ansible** — 基於 SSH 的無代理程式自動化工具。Playbook 具備冪等性：重複執行兩次結果相同。透過 Inventory 檔案（`hosts`）將 Playbook 對應至目標節點。

**Docker Compose** — Web 伺服器上的容器編排工具。WordPress 與 MySQL 資料庫皆以服務形式定義於 `docker-compose.yml`，由 Compose 統一管理。

**Prometheus** — 拉取式（Pull）指標收集系統。依排程從 Node Exporter 的 `/metrics` 端點抓取資料（預設每 15 秒），並以時間序列格式儲存於本機。

**Grafana** — 視覺化儀表板工具。以 Prometheus 為資料來源，呈現 CPU、記憶體、磁碟及網路等系統指標圖表。

**Node Exporter** — Prometheus 官方匯出器，將 Web 伺服器的硬體與作業系統指標暴露於 port 9100，供 Prometheus 抓取。

**firewalld + ansible.posix** — 以 Ansible 宣告式管理各節點防火牆規則，實現最小權限原則與來源 IP 限制。

---

## 前置需求

- [VirtualBox](https://www.virtualbox.org/) ≥ 6.1
- [Vagrant](https://www.vagrantup.com/) ≥ 2.3
- 可用 RAM 約 6 GB（每台 VM 約佔用 512 MB～1 GB）
- 可用磁碟空間約 20 GB

---

## 啟動方式

```bash
# 1. 複製專案
git clone https://github.com/zenAIteam/vagrant-ansible-three-node.git
cd vagrant-ansible-three-node

# 2. 啟動所有虛擬機
vagrant up

# 3. SSH 進入控制節點
vagrant ssh control

# 4. 依序執行 Ansible Playbook
cd ~/ansible-project
ansible-playbook deploy_docker.yml       # 在 web-server 安裝 Docker
ansible-playbook deploy_app.yml          # 以 Docker Compose 部署 WordPress
ansible-playbook deploy_monitor.yml      # 部署 Prometheus + Grafana
ansible-playbook deploy_exporter.yml     # 在 web-server 部署 Node Exporter
ansible-playbook deploy_firewall.yml     # 強化各節點防火牆規則（Phase 2）
```

### 瀏覽器驗證

| 服務 | 網址 | 預設帳密 |
|---|---|---|
| WordPress | http://192.168.56.11 | （安裝時設定） |
| Prometheus | http://192.168.56.12:9090 | — |
| Grafana | http://192.168.56.12:3000 | `admin` / `admin` |

---

## 執行成果

### Grafana Dashboard

> 截圖待補（建議於 `vagrant up` 完成並執行壓測後擷取）

<!-- 請將 Grafana dashboard 截圖存為 screenshots/grafana-dashboard.png 後取消下列註解 -->
<!-- ![Grafana Dashboard](screenshots/grafana-dashboard.png) -->

### 壓力測試結果（ab -n 50000 -c 100）

> 截圖待補

<!-- 請將 ab 輸出截圖存為 screenshots/ab-result.png 後取消下列註解 -->
<!-- ![ab 壓測輸出](screenshots/ab-result.png) -->

---

## 核心概念

### 冪等性（Idempotency）

Ansible 的任務設計為可安全重複執行。例如「確認 Docker 已安裝」這類任務，會先檢查目前狀態，若 Docker 已存在則略過，不做任何更動。因此，即使 Playbook 中途失敗，重新執行也不會產生副作用。

### SSH 金鑰認證

Ansible 透過 SSH 與受控節點溝通，不需輸入密碼。在 control-node 上執行 `ssh-keygen` 生成金鑰對，再透過 `ssh-copy-id` 將 `id_rsa.pub` 分發至 web-server 與 monitor-server 的 `~/.ssh/authorized_keys`。如此一來，Ansible 執行 Playbook 時便能以 control-node 的私鑰完成認證，無需輸入密碼。

### Prometheus 拉取模型

與推送式（Push）監控不同，Prometheus 依排程主動向目標抓取指標。`prometheus.yml` 設定檔列出所有目標（如 `web-server:9100`）。若目標無法連線，Prometheus 會將其標記為 `DOWN`，本身不會崩潰，使監控管線對節點臨時故障具備較高的容錯性。

### firewalld 連接埠管理

Rocky Linux 9 預設啟用 `firewalld`。任何監聽在非標準埠（如 Node Exporter 的 9100）的服務，若未明確放行，封包將被靜默丟棄。使用 Ansible 的 `ansible.posix.firewalld` 模組可冪等地管理連接埠，比手動執行 `firewall-cmd` 更具可重複性與可審計性。

### Ansible Inventory 與 Become

`hosts` Inventory 檔案將節點分組（如 `[webservers]`、`[monitoring]`），Playbook 依群組名稱選擇目標。`become: true` 會透過 `sudo` 提升至 root 權限，安裝套件或重載防火牆規則等任務皆需此權限。

---

## 防火牆強化（Firewall Hardening）

### 設計目標

將第一階段中臨時以 `firewall-cmd` 手動開放 port 的做法，改為由 Ansible 統一管理的宣告式防火牆規則，實現：

- **最小權限原則（Least Privilege）** — 每個節點只開放自身服務所需的 port，其餘一律 DROP
- **來源 IP 限制（Source Restriction）** — Node Exporter :9100 僅允許 monitor-server 存取，防止系統指標對外洩漏
- **冪等部署（Idempotency）** — 規則即程式碼，可重複執行、納入版本控制

### 各節點防火牆規則

| 節點 | 開放 Port | 來源限制 | 說明 |
|---|---|---|---|
| `control-node` | SSH :22 | any | 僅作為控制節點，不對外提供服務 |
| `web-server` | SSH :22 | any | 管理用途 |
| `web-server` | HTTP :80 | any | WordPress 對外服務 |
| `web-server` | Node Exporter :9100 | **僅限 192.168.56.12** | 防止指標資訊對外暴露 |
| `monitor-server` | SSH :22 | any | 管理用途 |
| `monitor-server` | Prometheus :9090 | any | 實驗環境；正式環境應限內網 |
| `monitor-server` | Grafana :3000 | any | 實驗環境；正式環境應限內網 |

> **正式環境注意事項：** Prometheus :9090 與 Grafana :3000 在本專案中允許 any source，方便實驗存取。正式環境應限制為內部管理網段，或加上 VPN 才能存取，避免監控後台對外暴露。

### 執行方式

```bash
# 安裝所需 collection（僅需執行一次）
ansible-galaxy collection install ansible.posix

# 部署防火牆規則
ansible-playbook deploy_firewall.yml -i hosts

# 驗證 web-server 的 rich rule 是否生效
ansible webservers -m shell \
  -a "firewall-cmd --list-rich-rules" \
  -i hosts --become
```

### 預期輸出

```
# web-server — firewall-cmd --list-rich-rules 應看到：
rule family="ipv4" source address="192.168.56.12" port port="9100" protocol="tcp" accept

# ansible-playbook 執行結束時自動列印各節點規則摘要：
192.168.56.10 | SUCCESS
  "fw_result.stdout_lines": ["public (active)", "  services: ssh", ...]

192.168.56.11 | SUCCESS
  "fw_result.stdout_lines": ["public (active)", "  services: ssh http", "  ports: ", "  rich rules: rule family=ipv4 ...", ...]

192.168.56.12 | SUCCESS
  "fw_result.stdout_lines": ["public (active)", "  services: ssh", "  ports: 9090/tcp 3000/tcp", ...]
```

---

## 問題排查記錄

### 問題一 — port 9100 出現 `No route to host`

**現象：** Prometheus 目標頁面顯示 `web-server:9100` 狀態為 `DOWN`。從 monitor-server 手動執行 `curl http://192.168.56.11:9100/metrics` 回傳 `No route to host`。

**根本原因：** Web 伺服器上的 `firewalld` 預設封鎖 port 9100。Node Exporter 正在執行並監聽，但防火牆在封包抵達程序前就將其丟棄。

**診斷方式：**
```bash
# 在 web-server 上 — 確認 Node Exporter 正在監聽
ss -tlnp | grep 9100

# 在 monitor-server 上 — 測試連線
curl -v http://192.168.56.11:9100/metrics

# 在 web-server 上 — 查看防火牆規則
sudo firewall-cmd --list-ports
```

**解法一 — Ansible 臨時指令：**
```bash
ansible webservers -m firewalld \
  -a "port=9100/tcp state=enabled permanent=yes immediate=yes" \
  --become -i hosts
```

**解法二 — Playbook 任務（建議，可重複執行）：**
```yaml
- name: 在 firewalld 放行 Node Exporter 連接埠
  ansible.posix.firewalld:
    port: 9100/tcp
    permanent: yes
    immediate: yes
    state: enabled
  become: yes
```

**為何 `permanent: yes` 與 `immediate: yes` 都需要？**
`permanent` 將規則寫入 firewalld 設定（重開機後仍有效）；`immediate` 立即套用至執行中的防火牆，不需重新載入。兩者缺一不可：省略 `immediate` 則連接埠在下次重開機前仍為關閉狀態。

---

### 問題二 — SSH 金鑰互通失敗

**現象：** `ansible -m ping all` 回傳 `UNREACHABLE! ... Permission denied (publickey,gssapi-keyex,gssapi-with-mic)`。

**根本原因：** control-node 尚未產生自己的金鑰對（`id_rsa` / `id_rsa.pub`）。正確的設定流程是：在 control-node 執行 `ssh-keygen` 生成金鑰對後，再將 `id_rsa.pub` 分發至 web-server 與 monitor-server 的 `~/.ssh/authorized_keys`，Ansible 才能以私鑰完成無密碼認證。缺少這個步驟就會出現 `Permission denied (publickey)`。

**診斷方式：**
```bash
# 在 control-node 上 — 確認金鑰對是否存在
ls -la ~/.ssh/

# 在 web-server 上 — 確認 authorized_keys 內容
cat ~/.ssh/authorized_keys
```

**解法：**
```bash
# 第一步：在 control-node 產生金鑰對（全部按 Enter 使用預設值）
ssh-keygen -t rsa -b 4096

# 第二步：將公鑰分發至各受控節點
ssh-copy-id vagrant@192.168.56.11
ssh-copy-id vagrant@192.168.56.12

# 第三步：驗證 Ansible 可正常連線
ansible all -m ping -i hosts
```

**修復後的預期輸出：**
```
192.168.56.11 | SUCCESS => { "ping": "pong" }
192.168.56.12 | SUCCESS => { "ping": "pong" }
```

**提示：** `ssh-copy-id` 首次執行時需輸入目標節點密碼（`vagrant`），這屬正常現象。完成後 Ansible 即可使用金鑰進行無密碼連線。

---

### 問題三 — 找不到 `ansible` 指令

**現象：** `vagrant ssh control` 進入後，執行 `ansible` 或 `ansible-playbook` 回傳 `command not found`。

**根本原因：** Rocky Linux 9 的預設套件集不含 `ansible-core`，且基本 `dnf` 來源也不包含此套件，需先啟用 EPEL（Enterprise Linux 額外套件）來源。

**診斷方式：**
```bash
# 確認 ansible 是否已安裝
which ansible

# 查看目前啟用的套件來源
dnf repolist
```

**解法：**
```bash
# 第一步：啟用 EPEL 來源
sudo dnf install epel-release -y

# 第二步：從 EPEL 安裝 ansible-core
sudo dnf install ansible-core -y

# 第三步：確認安裝成功
ansible --version
```

**為何要先安裝 `epel-release`？** EPEL 是由社群維護的套件來源，收錄官方 RHEL/Rocky 來源沒有的套件，`ansible-core` 即由此維護。未啟用 EPEL 直接執行 `dnf install ansible-core` 會回傳 `No match for argument`。

---

### 問題四 — `ansible.posix.firewalld` 模組找不到

**現象：** 執行 `ansible-playbook deploy_firewall.yml` 時報錯：

```
ERROR! couldn't resolve module/action 'ansible.posix.firewalld'
```

**根本原因：** `ansible.posix` 是獨立維護的 collection，不隨 `ansible-core` 一起安裝，需要手動加裝。

**診斷方式：**
```bash
# 確認目前已安裝的 collections
ansible-galaxy collection list

# 確認 ansible-core 版本（2.9 以下不支援 FQCN 格式）
ansible --version
```

**解法：**
```bash
ansible-galaxy collection install ansible.posix
```

**為何使用 `ansible.posix.firewalld` 而非舊式 `firewalld`？** FQCN（Fully Qualified Collection Name）格式明確指定模組來源，避免與同名模組衝突，是 Ansible 2.10 以後的標準寫法，也符合 RHCE 考試要求。

---

### 問題五 — rich rule 設定後 :9100 仍無法從 monitor-server 存取

**現象：** `deploy_firewall.yml` 執行成功，但從 monitor-server 執行 `curl http://192.168.56.11:9100/metrics` 仍回傳 `No route to host`。

**根本原因：** `ansible.posix.firewalld` 的 `rich_rule` 參數在某些版本有空白字元解析問題，導致規則寫入格式錯誤，`firewall-cmd --list-rich-rules` 顯示的規則與預期不符。

**診斷方式：**
```bash
# 在 web-server 上確認 rich rule 實際寫入內容
sudo firewall-cmd --list-rich-rules

# 從 monitor-server 測試（應可連線）
curl -v http://192.168.56.11:9100/metrics

# 從 control-node 測試（應被拒絕，確認來源限制有效）
curl http://192.168.56.11:9100/metrics
```

**解法：** 改用 `shell` 模組直接呼叫 `firewall-cmd`，確保格式完全正確：

```bash
ansible webservers -m shell -a \
  "firewall-cmd --permanent \
   --add-rich-rule='rule family=ipv4 source address=192.168.56.12 port port=9100 protocol=tcp accept' \
   && firewall-cmd --reload" \
  -i hosts --become
```

**兩種方式的取捨：** `ansible.posix.firewalld` 模組具備冪等性、輸出結構化、適合 Playbook；`shell + firewall-cmd` 除錯時更直觀、格式可控。實務上先用 shell 確認規則語法正確，再移植回 Playbook。

---

### 問題六 — `dhcpv6-client` service 移除時報錯

**現象：** Playbook 執行到移除 `dhcpv6-client` 的 task 時回傳：

```
Error: 'dhcpv6-client' not among existing services in zone 'public'
```

**根本原因：** 不同的 Rocky Linux 9 安裝環境或映像版本，預設開放的 firewalld service 清單不盡相同。部分環境預設沒有 `dhcpv6-client`，移除時自然找不到。

**解法：** 在該 task 加上 `ignore_errors: yes`，讓 Playbook 在此情況下繼續執行而不中斷：

```yaml
- name: 移除 dhcpv6-client（若不存在則略過）
  ansible.posix.firewalld:
    service: dhcpv6-client
    permanent: yes
    immediate: yes
    state: disabled
  ignore_errors: yes
```

**為何不直接刪除這個 task？** 保留此 task 可確保在預設開放 `dhcpv6-client` 的環境中也能正確收斂至安全狀態，`ignore_errors` 只是讓它在已收斂的環境中優雅跳過。

---

## 專案結構

```
vagrant-ansible-three-node/
├── Vagrantfile                    # 定義所有三台虛擬機
├── ansible-project/
│   ├── hosts                      # Inventory：IP 對應群組
│   ├── ansible.cfg                # Ansible 設定（選用）
│   ├── deploy_docker.yml          # 在 web-server 安裝 Docker
│   ├── deploy_app.yml             # 以 Compose 部署 WordPress
│   ├── deploy_monitor.yml         # 部署 Prometheus + Grafana
│   ├── deploy_exporter.yml        # 部署 Node Exporter
│   └── deploy_firewall.yml        # 強化各節點防火牆規則
└── vagrant-test/                  # 實驗用暫存目錄
```

---

## Ansible 常用指令

```bash
# 測試所有節點連線
ansible all -m ping -i hosts

# 執行單一 Playbook
ansible-playbook deploy_docker.yml -i hosts

# 乾跑模式（Check Mode）— 顯示將會變更的內容，但不實際執行
ansible-playbook deploy_app.yml --check -i hosts

# 顯示詳細輸出以便除錯
ansible-playbook deploy_monitor.yml -vvv -i hosts

# 只執行帶有特定標籤的任務
ansible-playbook deploy_exporter.yml --tags "firewall" -i hosts

# 臨時指令：查看所有節點的磁碟使用量
ansible all -m shell -a "df -h" -i hosts --become
```

---

## Vagrant 常用指令

```bash
vagrant up              # 啟動所有虛擬機
vagrant halt            # 正常關閉所有虛擬機
vagrant reload          # 重啟虛擬機（重新讀取 Vagrantfile）
vagrant destroy -f      # 刪除所有虛擬機（不可復原）
vagrant status          # 顯示各虛擬機狀態
vagrant ssh control     # SSH 進入 control-node
vagrant ssh web         # SSH 進入 web-server
vagrant ssh monitor     # SSH 進入 monitor-server
```

---

## 快速排障索引

| 現象 | 可能原因 | 對應章節 |
|---|---|---|
| Prometheus 目標顯示 `DOWN` | firewalld 封鎖 port 9100 | 問題一 |
| `Permission denied (publickey)` | SSH 公鑰未分發 | 問題二 |
| `ansible: command not found` | ansible-core 未安裝 | 問題三 |
| `couldn't resolve module 'ansible.posix.firewalld'` | collection 未安裝 | 問題四 |
| rich rule 設定後 :9100 仍無法存取 | rich rule 格式解析錯誤 | 問題五 |
| `dhcpv6-client` 移除時報錯 | 該環境預設未開放此 service | 問題六 |
