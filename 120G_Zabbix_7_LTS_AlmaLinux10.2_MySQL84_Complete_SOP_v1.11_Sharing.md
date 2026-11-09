# Zabbix 7.0 LTS 正式環境建置與維運 SOP

1.11 對外分享版  AlmaLinux 10.2  MySQL 8.4 LTS  Nginx  Hyper V

| 項目 | 內容 |
| --- | --- |
| 用途 | 少於 30 台伺服器與 Cisco 網路設備的正式監控平台 |
| 建議資源 | 4 vCPU  16 GB RAM  120 GB OS 磁碟  300 GB DB 磁碟  專用備份儲存依第二十二章計算 |
| 部署模式 | Zabbix Server  Web  MySQL 安裝於同一台虛擬機 |
| 文件基準日 | 技術基準 2026 年 11 月 8 日  分享版整理 2026 年 11 月 9 日 |
| 建置原則 | 先完成可驗證的基本平台  再逐步加入 Agent  SNMP  Oracle 與告警 |
| 作業系統 | AlmaLinux 10.2 x86_64 Minimal ISO |
| Hyper V | Generation 2  4 vCPU  16 GB 固定記憶體  Secure Boot 使用 Microsoft UEFI Certificate Authority |
| 磁碟策略 | 120 GB OS 磁碟僅供系統安裝  300 GB 固定 DB 磁碟安裝後掛載至 /var/lib/mysql  另增獨立 /backup 與故障範圍外副本 |
| 文件版本 | 1.11 對外分享版  環境參數與認證由部署者提供 |

本手冊假設使用全新 Hyper V 第二代虛擬機，安裝 AlmaLinux 10.2 x86_64 Minimal，並可連線至 AlmaLinux、MySQL 與 Zabbix 官方套件庫。安裝時只允許 120 GB OS 磁碟參與系統分割，300 GB DB 磁碟必須保留到作業系統完成後，再獨立掛載至 /var/lib/mysql。

## 文件管制與修訂履歷

本文件為 Zabbix 7.0 LTS SOP 1.11 對外分享版。帳號、密碼、內部主機名稱、網域、IP、PSK 身分及組織路徑均以占位符或填寫提示取代；本文件不提供任何可直接使用的環境認證。請由部署單位自行提供資料、核准後再執行。

| 版本 | 日期 | 主要變更 | 核准狀態 |
| --- | --- | --- | --- |
| 1.7 | 2026-11-06 | 完整建置、英文語系保留與安全排錯 | 依原紀錄 |
| 1.8 | 2026-11-06 | 自動註冊、PSK、EXE、TLS 與服務告警 | 依原紀錄 |
| 1.9 | 2026-11-07 | 審查修訂、備份還原、通知、權限與更新流程；分享版不含認證值 | 文件待審核，實機變更待驗收 |
| 1.10 | 2026-11-07 | 備份／外送／輪替分離；補 RTO、TCP、EDR、Cisco、第二通道與驗收；保留快速部署；認證由部署者提供 | 文件修訂完成；正式實機驗收待執行 |
| 1.11 | 2026-11-08 | 損壞副本隔離、容量預檢、續傳與逾時、備份範圍、看門狗來源、MSI 核心驗證及暫存驗收；認證改由部署者提供並保留快速部署流程 | 程式與文件檢查完成；Windows／Linux 實機待驗收 |
| 1.11 分享版 | 2026-11-09 | 完整去識別；帳號與秘密改提示、環境參數改占位符；保留操作與快速部署流程 | 接收單位須自行填參數並完成實機驗收 |

| 管制欄位 | 填寫內容 |
| --- | --- |
| 文件擁有人與維護人 | 請自行提供或填寫核准紀錄 |
| 技術複核人與日期 | 請自行提供或填寫核准紀錄 |
| 資安例外核准人與期限 | 請自行提供或填寫核准紀錄 |
| 上線核准人與變更單號 | 請自行提供或填寫核准紀錄 |
| 文件存放與讀取權限 | 請自行提供或填寫核准紀錄 |

所有 `__大寫名稱__` 都是待填欄位，不是可直接使用的設定。平台 FQDN 在 OS hostname、Nginx server_name、憑證 SAN 與平台 Hostname 使用同一核准值；端點則使用自己的 FQDN。變更文件不會自動更新伺服器、腳本或部署 EXE。

### 1.11 執行順序與停止點

既有平台先做設定與權限盤點，不重新格式化磁碟、不重建 Schema、不重跑 CREATE USER。先核對 MySQL 3306／33060、備份目的掛載、HTTPS 與主機命名，再啟用備份排程、通知及隔離還原。第六至十二章屬新建流程；既有平台只執行標示為檢查或經核准的變更。

所有新補腳本均須先在測試環境驗證。文件已做結構、語法與排版檢查；AlmaLinux 的 systemd、SELinux、SMTP、遠端 SSH、MySQL 備份與 Windows 離線部署仍須依附錄 B 逐項完成實機驗收。未通過的項目不可勾選為完成。

## 分享版使用與資料填寫規則

本版保留 1.11 的建置、除錯、備份還原、Windows Agent 快速部署、Cisco 與驗收流程。所有 `__名稱__` 都是必須自行提供的資料，不能原樣執行，也不保證被服務或工具拒絕。先完成第二章與下表，核准工作副本後再開始。密碼與密鑰只填在受控設定或秘密檔，不回填至分享文件。

root、mysql、nginx、apache、zabbix、SYSTEM 與 Administrators 在服務、檔案權限或 ACL 範例中是作業系統／套件角色；不是本文件提供的登入認證。MySQL 連線帳號、SNMP 帳號、LDAP Bind DN、SMTP 認證與日常管理帳號仍必須自行提供。localhost、127.0.0.1、0.0.0.0、標準埠及 MIB OID 保留其技術意義。

| 占位符或資料區域 | 請自行提供的內容 |
| --- | --- |
| `__ZABBIX_FQDN__`  `__ZABBIX_IPV4__` | 平台核准 FQDN 與固定 IPv4；DNS、憑證 SAN、Nginx 與平台 Host 同步。 |
| `__ORG_DOMAIN__`  `__VM_NAME__` | 本單位網域與 Hyper V VM 顯示名稱。 |
| `__HOST_FQDN__`  `__HOST_IPV4__` | 每台 Agent 主機的實際 FQDN／主要 IPv4；由腳本偵測並核對 CMDB。 |
| `__SWITCH_FQDN__`  `__SWITCH_IPV4__` | 交換機核准名稱與管理 IP。 |
| `__DNS_IPV4__`  `__GATEWAY_IPV4__`  `__IPV4_PREFIX__` | DNS、Gateway 與 IPv4 Prefix；依核准網路配置填寫。 |
| `__MGMT_CIDR__`  `__AGENT_CIDR__` | 管理來源與 Agent／Proxy 來源 CIDR；不能直接填全網。 |
| `__WATCHDOG_IPV4__`  `__FIREWALL_ZONE__` | 看門狗實際來源 IPv4（確認 NAT）與作用中 zone；依章節補 /32。 |
| `__SNMP_SOURCE_IPV4__` | 網路設備實際看見的監控端來源 IP。 |
| `__DB_NAME__`  `__DB_ADMIN_USER__` | 自行命名的 DB 與具必要權限的 MySQL 管理帳號；新裝初始化依官方程序。 |
| `__DB_APP_USER__`  `__DB_APP_PASSWORD__` | 應用帳號與獨立強密碼；SQL、Server 與 Web Setup 同步，秘密存保管庫。 |
| `__DB_BACKUP_USER__`  `__DB_BACKUP_PASSWORD__` | 備份帳號與獨立密碼；defaults file 的 user／password 同步。 |
| MySQL 管理密碼區域 | 自行設定、存保管庫；不與應用或備份密碼相同，命令列使用互動 -p。 |
| `__BOOTSTRAP_PSK_IDENTITY__` | 批次自動註冊身分；前端與管理 NB 設定一致，不是 PSK 本文。 |
| `__HOST_PSK_IDENTITY__` | 逐台 PSK 身分；每台唯一，與 Agent、Host Encryption 一致。 |
| PSK 本文與 API Token 區域 | 由部署單位安全產生／取得；只存秘密檔或保管庫，禁止填入共享 SOP。 |
| `__WINDOWS_METADATA__`  `__LINUX_METADATA__` | 批次納管 metadata；Action 條件與腳本值相同。 |
| `__MANUAL_METADATA__` | 手動納管 metadata；不得包含可匹配批次 Action 的條件字串。 |
| `__WINDOWS_GROUP__`  `__LINUX_GROUP__`  `__NETWORK_GROUP__` | 接收單位核准的 Host groups；建立後再連結正確官方 Template。 |
| `__SNMP_USER__` | SNMPv3 最小權限 Security name；網路設備與 Zabbix 一致。 |
| `__SNMP_AUTH_PASSWORD__`  `__SNMP_PRIV_PASSWORD__` | SNMPv3 Auth／Priv 獨立密碼；由保管庫取用，注意 Shell History 與診斷輸出。 |
| `__DEVICE_LOCATION__`  `__DEVICE_CONTACT__` | 設備位置與維運聯絡窗口；分享前移除真實內容。 |
| `__LDAP_FQDN__` 與 LDAP 欄位區域 | LDAPS FQDN、Base DN、Bind DN、查詢帳號與密碼；驗證公司 CA。 |
| `__SMTP_HOST__` 與通知欄位區域 | SMTP 主機、連接埠、Sender、HELO、認證與收件人；獨立驗證通知鏈。 |
| `__ORG_TAG__` | 部署路徑／檔名的組織代碼；須與管理員已核准的實際套件一致。 |
| `__TIMEZONE__` | 組織核准 IANA 時區；OS、PHP、Zabbix 顯示時間一致。 |
| 憑證、SSH、備份與責任欄位區域 | CA／憑證／私鑰、SSH key 與指紋、遠端帳號及路徑、RTO／RPO、核准人由本單位提供。 |

### 部署工具與快速操作的適用前提

本分享版僅提供文件。文中 ZIP、MSI、EXE 與腳本名稱表示應準備的工具，不代表已隨附。接收單位的管理員須先核准工具來源、固定版本與 SHA256，將 Server／PSK identity／metadata／Host groups 及 `__ORG_TAG__` 路徑套入自己的工具，並完成首台實測。不得直接派送其他單位已打包、可能內含共享 PSK 的 EXE。MSI 的公開 SHA256 與原廠簽章不是秘密，保留供驗證。

管理員完成一次性設定並持有本單位的既有 PSK 後，在管理 NB 右鍵 Build-EXE.cmd 產生核准 EXE；工程師端仍維持複製到 C:\Temp 或 D:\Temp、右鍵管理員執行，不需輸入 PSK 或修改設定。首次使用的接收單位必須先產生自己的 PSK；既有環境更新工具時沿用已核准密鑰，不任意重建。

參數替換完成的工作副本須留在受控位置；交付其他人時重新產生去識別副本，不含密碼檔、PSK、Token、含密 EXE、原始 Log 或內部截圖。已被修改的占位符不代表實機已通過驗收。

## 一 建置結論與執行邊界

這套組合在本手冊的備份、通知、隔離還原與安全檢核全部驗收後才可正式上線。建置重點不是把套件裝完，而是確保 MySQL 版本鎖定在 8.4 LTS、資料庫使用正確字元集、SELinux 維持 Enforcing、服務可被驗證，以及備份能實際還原。完成本手冊後，平台可先承載目前少於 30 台伺服器與 Cisco 設備，並保留合理成長空間。

本次不納入：Zabbix HA、MySQL InnoDB Cluster、跨廠區 Proxy、Oracle 深度 Template、Teams 通知與正式憑證簽發。這些應在核心平台驗收後分階段導入。

### 最容易失敗的三個原因

| 風險 | 常見結果 | 預防方式 |
| --- | --- | --- |
| MySQL Repository 選錯 | 實際裝到 9.7 或混入 MariaDB | 安裝前檢查 Enabled Repo 與 Candidate Version  確認為 8.4 後才繼續 |
| Schema 或字元集錯誤 | Web Setup 可開但資料表異常或升級失敗 | 使用 utf8mb4 與 utf8mb4_bin  並依官方 server.sql.gz 匯入 |
| 權限與服務未驗證 | Nginx 連不上 DB  Zabbix Server 反覆停止 | 保留 SELinux  安裝官方 Policy  每階段檢查 Log 與 Service Status |
| AlmaLinux 安裝時選錯磁碟 | 300 GB DB 磁碟被自動分割或清除 | Installation Destination 只勾選 120 GB OS 磁碟；300 GB 磁碟不得出現勾選記號 |

## 二 參數準備

執行前先填妥本章參數表。主機、帳號、網域、IP 與路徑由部署單位提供；密碼、PSK、Token、私鑰只存於核准保管庫，文件只記錄保管庫編號。先在受控工作副本替換占位符並檢查，分享版不得填入秘密值。

| 占位符 | 範例 | 實際值 |
| --- | --- | --- |
| ZABBIX FQDN | `__ZABBIX_FQDN__` | 請自行提供或填寫核准紀錄 |
| ZABBIX IP | `__ZABBIX_IPV4__` | 請自行提供或填寫核准紀錄 |
| PREFIX | `__IPV4_PREFIX__` | 請自行提供或填寫核准紀錄 |
| GATEWAY | `__GATEWAY_IPV4__` | 請自行提供或填寫核准紀錄 |
| DNS | `__DNS_IPV4__` | 請自行提供或填寫核准紀錄 |
| 管理網段 | `__MGMT_CIDR__` | 請自行提供或填寫核准紀錄 |
| Agent 或 Proxy 來源網段 | `__AGENT_CIDR__` | 請自行提供或填寫核准紀錄 |
| MySQL Root 密碼 | 請自行設定，僅存核准密碼保管庫 | 不可與應用或備份帳號共用，不回填分享文件 |
| Zabbix DB 密碼 | 自訂至少 16 碼並符合公司密碼政策 | 已存密碼庫  是  否 |
| VM 名稱 | `__VM_NAME__` | 請自行提供或填寫核准紀錄 |
| OS 磁碟 | 120 GB  SCSI 0 | 請自行提供或填寫核准紀錄 |
| DB 磁碟 | 300 GB 固定 VHDX  SCSI 1 | 請自行提供或填寫核准紀錄 |
| ISO | AlmaLinux 10.2 x86_64 Minimal | 請自行提供或填寫核准紀錄 |

## 三 架構與連線需求

| 來源 | 目的 | 通訊埠 | 用途 |
| --- | --- | --- | --- |
| IT 管理網段 | Zabbix Web | TCP 443  80 僅導向，不登入 | 管理介面 |
| Agent Active 與 Proxy | Zabbix Server | TCP 10051 | 主動檢查與 Proxy 資料 |
| Zabbix Server | Windows Linux Agent | TCP 10050 | 被動檢查  若啟用 |
| Zabbix Server | Cisco 與其他設備 | UDP 161 | SNMP Polling |
| 被監控設備 | Zabbix Server 或 Trap Receiver | UDP 162 | SNMP Trap  第二階段 |
| Zabbix Server | DNS NTP SMTP | 依公司標準 | 解析 時間同步 郵件告警 |
| 本機 Zabbix Server | 本機 MySQL | 應用服務 TCP 127.0.0.1:3306；root 維護可用 socket | 資料庫連線  不對外開放 |

## 四 建置前檢查與回復點

☐ VM 規格為 4 vCPU、16 GB 固定 RAM、120 GB OS VHDX、300 GB 固定 DB VHDX

☐ SCSI 0 為 120 GB OS 磁碟，SCSI 1 為 300 GB DB 磁碟；安裝時只選 OS 磁碟

☐ 已配置固定 IP  DNS  Gateway  FQDN 與反向解析

☐ 可以解析並連線 repo.almalinux.org  repo.mysql.com  repo.zabbix.com

☐ 時間來源已確認  Zabbix 與被監控主機不可有明顯時間偏差

☐ Hyper V 自動檢查點已關閉；後續只在本手冊指定的回復點建立 Production Checkpoint

☐ 變更單已記錄建置時間  執行人  回復條件與服務窗口

☐ 已規劃獨立 /backup 備份儲存與故障範圍外副本，容量依第二十二章估算；安裝 OS 時不選任何資料／備份磁碟。

## 五 Hyper V 虛擬機與安裝前確認

本節是第一次開機前的停止點。所有設定完成並核對後才可啟動 VM。若虛擬硬碟容量、SCSI 位置或 ISO 架構與下表不一致，先停止並修正。

| 項目 | 設定值 | 核對重點 |
| --- | --- | --- |
| VM 名稱 | `__VM_NAME__` | 依公司命名規範；設定檔位於 D:\Hyper-V\`__VM_NAME__` |
| 世代 | Generation 2 | 不得選 Generation 1 |
| CPU | 4 vCPU | CPU Compatibility for Migration 維持關閉，除非確有跨舊世代主機遷移需求 |
| 記憶體 | 16384 MB | Dynamic Memory 關閉 |
| OS 磁碟 | 120 GB VHDX  SCSI 0 | 只供 AlmaLinux 系統安裝 |
| DB 磁碟 | 300 GB 固定 VHDX  SCSI 1 | 檔名 `__VM_NAME__`-DB.vhdx；安裝階段不得選取 |
| DVD | SCSI 2 | 掛載 AlmaLinux-10.2-x86_64-minimal.iso |
| Secure Boot | 啟用 | Template 選 Microsoft UEFI Certificate Authority |
| 開機順序 | DVD  網路  OS  DB | 安裝完成後改為 OS 優先並卸載 ISO |
| 網路 | 正式環境外部 vSwitch | 確認 VLAN 與 IP 規劃 |
| 備份儲存 | 專用 VHDX SCSI 3 或核准 NAS | 安裝後獨立掛載 /backup；容量按實測單份大小計算，另有外部副本 |

### 安裝媒體核對

ISO 必須是 AlmaLinux 10.2 x86_64 Minimal，而不是 Boot、DVD 或 x86_64_v2。下載後依 AlmaLinux 官網公布值核對 SHA 256；核對失敗不得繼續。

Hyper V 設定完成後先開啟 VM Console，再啟動 VM。若顯示 Press any key to boot from CD or DVD，立即按任意鍵進入安裝媒體。

## 六 AlmaLinux 10.2 安裝與 OS 磁碟規劃

核心原則：Anaconda 安裝程式只能使用 120 GB OS 磁碟。300 GB DB 磁碟即使已被偵測，也不得勾選、不得 Reclaim Space、不得建立任何掛載點。

### 啟動安裝程式

1. 在開機選單選 Install AlmaLinux 10.2，直接進入 Anaconda。若懷疑 ISO 完整性，先選 Test this media and install AlmaLinux 10.2。

2. 語言建議選 English United States，方便日後依原廠文件與錯誤訊息排查；鍵盤可另外加入 Chinese Traditional。

3. 進入 Installation Summary 後，先設定 Time and Date、Software Selection、Network and Host Name，再進行 Installation Destination。

| 安裝項目 | 建議設定 | 完成條件 |
| --- | --- | --- |
| Time and Date | `__TIMEZONE__`  NTP 開啟 | 時間與時區正確 |
| Installation Source | Local Media | 沒有來源錯誤 |
| Software Selection | Minimal Install | 不安裝 GUI 與不必要角色 |
| Network and Host Name | 網卡開啟  設定固定 IPv4  Gateway  DNS  FQDN | 畫面顯示 Connected，Hostname 為實際 FQDN |
| Root Password | 依公司密碼政策設定 | 不得允許 Root 以密碼直接遠端 SSH |
| User Creation | 建立具名管理帳號並加入 wheel | 安裝後可使用 sudo |
| Security Profile | 維持預設 | 若公司已有指定基線再套用；不要臨時選未知 Profile |

### Installation Destination 磁碟選擇

1. 開啟 Installation Destination，Local Standard Disks 應看到約 120 GB 與約 300 GB 兩顆磁碟。

2. 只點選約 120 GB 的 OS 磁碟，使其出現勾選記號。約 300 GB 的 DB 磁碟必須保持未勾選。

3. 在 Storage Configuration 選 Custom，不選 Automatic；Encryption 本次不啟用。若公司政策要求 LUKS，需另行設計開機解鎖與 Recovery Key 保管。

4. 按 Done 進入 Manual Partitioning。若畫面要求 Reclaim Space 或顯示既有資料，先停止，重新確認是否選錯磁碟。

| 掛載點 | 容量 | Device Type | 檔案系統 | 用途 |
| --- | --- | --- | --- | --- |
| /boot/efi | 600 MiB | Standard Partition | EFI System Partition  VFAT | UEFI 開機 |
| /boot | 1 GiB | Standard Partition | XFS | Kernel 與開機檔案 |
| swap | 8 GiB | LVM | swap | 短暫記憶體壓力緩衝 |
| / | 其餘全部 | LVM | XFS | 作業系統、套件與應用程式 |

本配置不建立獨立 /home，因為本機是專用伺服器；MySQL 資料不放在 OS 磁碟，而是在安裝後將獨立 300 GB 磁碟掛載至 /var/lib/mysql。完成分割後再次確認所有新分割區都位於 120 GB OS 磁碟。

### 開始安裝與第一次重新啟動

1. 回到 Installation Summary，所有必要項目都不得顯示紅色警示，再按 Begin Installation。

2. 安裝完成後按 Reboot System。若 VM 再次從 ISO 啟動，立即關機，在 Hyper V 卸載 ISO 並將 OS 硬碟移到開機順位第一。

3. 第一次登入後確認核心、版本、UEFI、Secure Boot 與磁碟辨識。

```text
cat /etc/almalinux-release
uname -r
[ -d /sys/firmware/efi ] && echo UEFI || echo Legacy
mokutil --sb-state
lsblk -e7 -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL,SERIAL
findmnt /
```

成功判定：版本為 AlmaLinux 10.2、開機模式為 UEFI、Secure Boot 顯示 enabled、根目錄位於 120 GB OS 磁碟，300 GB DB 磁碟仍沒有分割區與檔案系統。

## 七 DB 磁碟初始化與永久掛載

本節會清除並重新建立 300 GB DB 磁碟的 Partition Table，屬於破壞性操作。只要磁碟大小、裝置名稱、既有 Signatures 或掛載狀態有任何疑問，就停止，不要執行 parted 或 mkfs。

### 辨識並確認目標磁碟

```text
dnf -y install parted xfsprogs
lsblk -e7 -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL,SERIAL
findmnt /
wipefs -n /dev/sdb
```

本機預期 OS 為 /dev/sda、DB 為 /dev/sdb，但不得只依名稱判斷。必須同時用容量、findmnt / 的結果與 wipefs -n 輸出確認 /dev/sdb 確實是尚未使用的 300 GB DB 磁碟。若 /dev/sdb 已有分割區、檔案系統或掛載點，停止並查明來源。

### 建立 GPT 分割區與 XFS

```text
DB_DISK=/dev/sdb
DB_PART=/dev/sdb1
parted "$DB_DISK" --script mklabel gpt mkpart primary xfs 1MiB 100%
partprobe "$DB_DISK"
udevadm settle
mkfs.xfs -L MYSQL_DATA "$DB_PART"
lsblk -f "$DB_DISK"
```

### 設定永久掛載

```text
mkdir -p /var/lib/mysql
DB_UUID=$(blkid -s UUID -o value /dev/sdb1)
echo "UUID=$DB_UUID /var/lib/mysql xfs defaults,noatime 0 0" >> /etc/fstab
findmnt --verify --verbose
mount -a
restorecon -Rv /var/lib/mysql
findmnt /var/lib/mysql
df -hT /var/lib/mysql
```

若 /etc/fstab 已出現相同 UUID 或 /var/lib/mysql 掛載列，不得重複追加。應先以 sudoedit /etc/fstab 修正，再執行 findmnt --verify --verbose 與 mount -a。

### 重新開機驗證

```text
reboot
# 重新登入後執行
findmnt /var/lib/mysql
mountpoint /var/lib/mysql
df -hT /var/lib/mysql
lsblk -f
```

停止條件：重新開機後 /var/lib/mysql 未掛載、檔案系統不是 XFS、容量不是約 300 GB，或 mountpoint 回傳失敗。這些問題修正前不得安裝 MySQL。

## 八 AlmaLinux 10.2 安裝後基礎設定

### 設定主機名稱與時區

```text
hostnamectl set-hostname __ZABBIX_FQDN__
timedatectl set-timezone __TIMEZONE__
hostnamectl
timedatectl
```

執行前：把 `__ZABBIX_FQDN__` 換成實際 FQDN。IP 建議透過 NetworkManager 或既有 VM Template 設定，不在未知介面名稱下直接覆寫連線設定。

### 更新作業系統並安裝必要工具

```text
dnf -y update
dnf -y install dnf-plugins-core chrony firewalld policycoreutils-python-utils \
  curl wget vim unzip openssl
systemctl enable --now chronyd firewalld
chronyc tracking
reboot
```

回復點一：重新登入後確認系統、DB 磁碟掛載、網路、DNS 與時間正常，再建立 Hyper V Production Checkpoint，名稱 Base OS Ready。Checkpoint 只供短期變更回復，不取代備份。

### 檢查是否已存在資料庫套件

```text
rpm -qa | grep -Ei 'mariadb|mysql'
dnf list installed '*mariadb*' '*mysql*'
```

全新 VM 不應已有 MariaDB Server 或其他 MySQL Server。若已有套件，不要直接移除，先確認是否被其他應用程式使用。AlmaLinux 10 已不再提供以 Modularity 發布的 AppStream 內容，因此不需要執行 dnf module disable mysql；改以已安裝套件與 Enabled Repository 判斷是否可能與 MySQL Community Repository 衝突。

```text
dnf repolist enabled | grep -E 'alma|mysql'
```

## 九 安裝 MySQL 8.4 LTS

### 加入 MySQL Community Repository

```text
dnf -y install https://repo.mysql.com/mysql84-community-release-el10-3.noarch.rpm
dnf repolist all | grep mysql
```

本手冊固定使用 MySQL 8.4 的 EL10 Repository Setup Package。安裝後仍須核對 Enabled Repo 與 Candidate Version；如果主機曾加入 MySQL 9.7 或其他版本 Repository，必須先停用，再啟用 8.4。

#### 若曾加入 MySQL 9.7 Repository，才執行

```text
dnf config-manager --disable mysql-9.7-lts-community
dnf config-manager --disable mysql-tools-9.7-lts-community
```

#### 確認並啟用 MySQL 8.4 Repository

```text
dnf config-manager --enable mysql-8.4-lts-community
dnf config-manager --enable mysql-tools-8.4-lts-community
```

### 安裝前強制核對版本

```text
dnf repolist enabled | grep mysql
dnf info mysql-community-server
```

停止條件：如果 Candidate Version 或 Enabled Repository 不是 MySQL 8.4，先停止，不要安裝。Repo 名稱日後可能調整，應以 repolist 的實際輸出為準。

### 安裝並啟動 MySQL

```text
dnf -y install mysql-community-server
mountpoint -q /var/lib/mysql || { echo 'STOP  DB disk is not mounted'; exit 1; }
chown mysql:mysql /var/lib/mysql
chmod 750 /var/lib/mysql
restorecon -Rv /var/lib/mysql
systemctl enable --now mysqld
systemctl --no-pager --full status mysqld
mysqld --version
grep 'temporary password' /var/log/mysqld.log
```

mysqld --version 必須顯示 8.4.x。記下暫時 Root 密碼後執行安全初始化。

```text
mysql_secure_installation
```

依公司密碼政策設定新的 Root 密碼，移除匿名帳號、禁止 Root 遠端登入、移除 Test Database 並重新載入權限。

MySQL 管理密碼：請由部署者自行設定並安全保存；本文件不記錄密碼值、設定日期與帳密配對。

### MySQL 基本調校

使用 sudoedit 或 vi 建立 /etc/my.cnf.d/zabbix.cnf。下列設定適用於 16 GB RAM 且 Zabbix 與 MySQL 同機的起始配置。dnf -y install nano

```text
[mysqld]
bind-address=127.0.0.1
mysqlx-bind-address=127.0.0.1
character-set-server=utf8mb4
collation-server=utf8mb4_bin
default-storage-engine=InnoDB
innodb_buffer_pool_size=8G
innodb_redo_log_capacity=2G
max_connections=200
tmp_table_size=64M
max_heap_table_size=64M
slow_query_log=ON
long_query_time=2
binlog_expire_logs_seconds=604800
```

```text
mysqld --validate-config
systemctl restart mysqld
systemctl --no-pager --full status mysqld
mysql -u__DB_ADMIN_USER__ -p -e "SHOW VARIABLES LIKE 'version'; SHOW VARIABLES LIKE 'innodb_buffer_pool_size';"
```

### MySQL 設定檔防呆與套用驗證

設定檔只能放置選項與參數，mysqld、mysql、systemctl、sleep、ss 與 curl 等 Shell 指令只能在命令提示字元執行。若把 Shell 指令貼入 [mysqld] 區段，MySQL 會把整行當成未知選項，造成 mysqld 無法啟動。重新啟動前先執行下列檢查。

```text
grep -nE '^!include|^!includedir' /etc/my.cnf
grep -RInE '^[[:space:]]*(mysqld|mysql|systemctl|sleep|ss|curl)([[:space:]]|$)' \
  /etc/my.cnf /etc/my.cnf.d
my_print_defaults mysqld
```

成功判定：grep 不得在 MySQL 設定檔中找到 Shell 指令；my_print_defaults mysqld 應顯示預期的 MySQL 選項。如果輸出出現 --mysqld --validate-config、--systemctl 或其他完整命令，立即編輯來源檔刪除誤貼行，不得反覆重啟服務。

若第一個 grep 沒有顯示 /etc/my.cnf.d 的 includedir，使用編輯器在 /etc/my.cnf 最後加入下列一行。若已存在則不得重複加入。

```text
!includedir /etc/my.cnf.d
```

確認實際載入的本機綁定設定，再進行語法檢查。mysqld --validate-config 成功時通常不輸出訊息，echo $? 必須回傳 0。若錯誤訊息為 unknown option '--mysqld --validate-config'，代表設定檔仍有誤貼命令，不代表 MySQL 8.4 不支援驗證參數。

```text
my_print_defaults mysqld | grep -E 'bind-address|mysqlx-bind-address'
mysqld --validate-config
echo $?
```

若 my_print_defaults 未顯示預期值，先檢查 include 與重複設定，不要繼續。若驗證成功，再重新啟動並確認實際生效值。

```text
grep -RInE 'bind[-_]address|mysqlx[-_]bind[-_]address' /etc/my.cnf /etc/my.cnf.d
systemctl restart mysqld
sleep 5
systemctl is-active mysqld
mysql -u__DB_ADMIN_USER__ -p -e \
"SHOW VARIABLES WHERE Variable_name IN ('bind_address','mysqlx_bind_address','innodb_buffer_pool_size');"
ss -lntp | grep -E ':(3306|33060)\b'
```

成功判定：mysqld 為 active；bind_address 與 mysqlx_bind_address 均為 127.0.0.1；innodb_buffer_pool_size 為 8589934592，也就是 8 GiB；TCP 3306 與 33060 只監聽 127.0.0.1。若任何值仍為星號，表示設定未載入或被後載入的設定覆寫。

## 十 建立 Zabbix 資料庫

請自行設定 Zabbix DB 專用密碼並存入密碼管理工具。建議至少 16 碼並符合組織密碼政策；必須與 MySQL 管理密碼及備份帳號密碼不同。不要把密碼留在 Shell History、郵件、群組或安裝截圖中。

本手冊不使用 Shell 指令產生或顯示密碼。為降低 SQL 與設定檔解析錯誤，自訂密碼建議避免單引號、反斜線與空格。

以核准的 MySQL 管理帳號互動登入，先在受控工作副本替換 SQL 中帳號、資料庫及密碼占位符，再逐行執行。僅新建空白平台可執行；既有平台不可重跑 CREATE。

```text
mysql -u__DB_ADMIN_USER__ -p
```

```text
CREATE DATABASE __DB_NAME__ CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER '__DB_APP_USER__'@'127.0.0.1' IDENTIFIED BY '__DB_APP_PASSWORD__';
GRANT ALL PRIVILEGES ON __DB_NAME__.* TO '__DB_APP_USER__'@'127.0.0.1';
SET GLOBAL log_bin_trust_function_creators = 1;
SHOW DATABASES;
SHOW GRANTS FOR '__DB_APP_USER__'@'127.0.0.1';
EXIT;
```

填寫提示：`__DB_APP_USER__` 為自行提供的資料庫應用帳號，`__DB_APP_PASSWORD__` 為該帳號的獨立密碼；帳號與 Host 授權需同步核對。秘密只填入受控執行環境，不回填至此分享版。

## 十一 安裝 Zabbix 7.0 LTS

### 加入 Zabbix 官方 Repository

```text
dnf -y install https://repo.zabbix.com/zabbix/7.0/alma/10/x86_64/zabbix-release-7.0-7.el10.noarch.rpm
dnf clean all
dnf makecache
dnf repolist enabled | grep zabbix
```

若已啟用 EPEL，在 /etc/yum.repos.d/epel*.repo 的每個實際啟用 EPEL 區段加入 excludepkgs=zabbix*；若已有排除清單，追加而非覆蓋其他值。未安裝 EPEL 不需為此安裝。不要把排除值加到官方 zabbix repo。

```text
dnf repolist enabled
grep -RInE '^\[|^enabled=|^excludepkgs=' /etc/yum.repos.d/epel*.repo
dnf clean metadata
dnf info zabbix-server-mysql zabbix-agent2 zabbix-release
dnf install --assumeno zabbix-server-mysql zabbix-web-mysql zabbix-nginx-conf zabbix-agent2
```

固定 Repository RPM 只固定來源設定。正式安裝另記錄並保存核准 NEVRA／GPG／SHA256 與完整交易清單，套件須同屬 7.0 LTS；禁止忽略候選版本核對或混用 EPEL 套件。升級走第三十一章，不自動追最新版。

### 安裝 Server Web Nginx Agent 2 與 AlmaLinux 10 SELinux Policy

```text
rpm -q selinux-policy-targeted-extra
dnf info zabbix-selinux-policy-main
dnf -y install zabbix-server-mysql zabbix-web-mysql zabbix-nginx-conf \
  zabbix-sql-scripts zabbix-selinux-policy-main zabbix-agent2
rpm -qa | grep '^zabbix' | sort
zabbix_server -V
```

版本確認：zabbix_server -V 必須顯示 7.0.x。若顯示 7.4、8.0 或其他版本，表示 Repository 選錯，先停止。
AlmaLinux 10 使用 zabbix-selinux-policy-main；若已安裝 selinux-policy-targeted-extra，兩者會互相衝突。不要
直接移除既有 Policy 套件，應先確認其他服務相依性並納入變更評估。

## 十二 匯入 Zabbix Schema

匯入前先確認套件提供的實際路徑。AlmaLinux 10 套件路徑為 /usr/share/zabbix-sql-scripts/mysql/server.sql.gz。不要直接執行 .gz 檔、不要修改其執行權限，也不要把資料庫密碼附在命令列。匯入時間依 VM 與儲存效能而定；命令結束時沒有輸出通常代表成功，不要在執行中中斷。

```text
rpm -ql zabbix-sql-scripts | grep 'server.sql.gz'
ls -lh /usr/share/zabbix-sql-scripts/mysql/server.sql.gz
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --protocol=TCP -h127.0.0.1 -P3306 --default-character-set=utf8mb4 -u__DB_APP_USER__ -p __DB_NAME__
```

看到 Enter password 後才輸入 Zabbix DB 密碼；輸入時畫面不顯示字元屬正常現象。等待命令回到 Shell 提示字元後，再驗證資料表數量與字元集。

```text
mysql --protocol=TCP -h127.0.0.1 -P3306 -u__DB_APP_USER__ -p -e "SELECT COUNT(*) AS table_count FROM information_schema.tables WHERE table_schema='__DB_NAME__';"
mysql --protocol=TCP -h127.0.0.1 -P3306 -u__DB_APP_USER__ -p -e "SELECT DEFAULT_CHARACTER_SET_NAME, DEFAULT_COLLATION_NAME FROM information_schema.SCHEMATA WHERE SCHEMA_NAME='__DB_NAME__';"
```

7.0.32 建置參考 table_count 為 203；後續 7.0 LTS 更新可能不同，但不得為 0。資料庫字元集必須是 utf8mb4，Collation 必須是 utf8mb4_bin。確認 Schema 已成功建立後，關閉暫時權限。

```text
mysql -u__DB_ADMIN_USER__ -p -e "SET GLOBAL log_bin_trust_function_creators = 0;"
mysql -u__DB_ADMIN_USER__ -p -e "SHOW VARIABLES LIKE 'log_bin_trust_function_creators';"
```

回復點二：建立 Hyper V Production Checkpoint，名稱 Zabbix Schema Imported。若此後 Web 或設定出錯，可回到已完成資料庫初始化的狀態；驗收後依變更程序移除。

## 十三 設定 Zabbix Server

使用 sudoedit 或 vi 編輯 /etc/zabbix/zabbix_server.conf。確認下列參數存在且未被前方井字號停用。

```text
DBHost=127.0.0.1
DBPort=3306
DBName=__DB_NAME__
DBUser=__DB_APP_USER__
DBPassword=__DB_APP_PASSWORD__
CacheSize=256M
HistoryCacheSize=128M
HistoryIndexCacheSize=64M
TrendCacheSize=64M
ValueCacheSize=256M
StartPingers=5
Timeout=10
```

DBPassword 必須填入目前有效的 Zabbix DB 密碼，等號前後不留空格，也不要加單引號或雙引號。先以 mysql --protocol=TCP -h127.0.0.1 -P3306 -u`__DB_APP_USER__` -p -e "SELECT 1;" 驗證密碼，再進行服務啟動驗證。

```text
systemctl enable --now zabbix-server
sleep 3
systemctl is-active zabbix-server
systemctl --no-pager --full status zabbix-server
tail -n 50 /var/log/zabbix/zabbix_server.log
ss -lntp | grep ':10051'
```

成功判定：systemctl 顯示 active，TCP 10051 正在監聽，Log 出現 poller、trapper、preprocessing worker 與 history poller 等程序啟動訊息，而且不再出現 database is down、mysql.sock (13)、Access denied、Unsupported database 或 Cannot connect。僅顯示 active 不代表資料庫連線成功，仍必須核對 Log。

### 本機 TCP 連線基線與遷移

新建環境統一使用 Zabbix Server 與 Web → 127.0.0.1:3306；MySQL 仍只綁定 loopback，X Protocol 維持 mysqlx-bind-address=127.0.0.1 或經驗證後關閉。這避免應用服務為了 socket 穿越 /var/lib/mysql 而新增 other execute。root 的資料庫維護與備份帳號仍可使用 socket。

本章 DBHost=127.0.0.1、DBPort=3306；第十章建立的帳號改為 '`__DB_APP_USER__`'@'127.0.0.1'。帳號 Host 與連線 Host 是不同設定，不能假設原有 '`__DB_APP_USER__`'@'localhost' 一定匹配 TCP。以下輸出 CURRENT_USER() 必須符合預期帳號，SELECT 1 成功後才啟動服務。

```text
mysql --protocol=TCP -h127.0.0.1 -P3306 -u__DB_APP_USER__ -p -e "SELECT USER(), CURRENT_USER(), 1;"
ss -lntp | grep -E '127\.0\.0\.1:(3306|33060)\b'
```

Web Setup 同樣選 127.0.0.1、3306；既有 /etc/zabbix/web/zabbix.conf.php 的 SERVER 與 PORT 須同步，密碼維持既有值，勿整份覆蓋。

```text
$DB['SERVER'] = '127.0.0.1';
$DB['PORT'] = '3306';
```

既有平台在維護窗口備份 DB／設定後，先由 DBA 檢查帳號；若缺少 127.0.0.1 帳號，以第十章相同既有密碼建立並授權 `__DB_NAME__`.*。不直接刪除 localhost 帳號，不重跑已存在的 CREATE，不修改原有密碼。完成兩端設定、登入與新資料驗證後，才評估收回舊 socket 權限。

SELinux 維持 Enforcing。先確認政策、服務 domain 與 3306 port type，再以實際 AVC 判斷；不能把舊 RHEL 的 zabbix_can_network 說明直接視為 EL10 必要設定，也不能一鍵 audit2allow 安裝所有拒絕。

```text
getenforce
ps -eZ | grep -E 'zabbix_server|php-fpm'
semanage port -l | grep mysqld_port_t
getsebool httpd_can_network_connect_db
ausearch -m AVC,USER_AVC -ts recent
```

第十五章 Web 的 httpd_can_network_connect_db 依套件政策啟用。若 zabbix_t 的 TCP name_connect 仍被拒絕，停止上線，由政策管理員針對 mysqld_port_t 的必要規則審查、測試與核准；保留核准政策檔及重開機證據。loopback TCP 能避開 socket 目錄權限，但不保證免除 SELinux 或帳號驗證。

只有 Zabbix Server、Web 及其他 socket 使用者都確認不依賴原例外後，才依原始擁有者／權限基線恢復資料目錄；新建基線為 mysql:mysql、750。既有平台不可尚未測試便 chmod。若 TCP 遷移失敗，還原原設定與已核准的 socket 權限例外，維持原服務並記錄待改善，不能關閉 SELinux。

## 十四 設定 Nginx 與 PHP FPM

編輯 /etc/nginx/conf.d/zabbix.conf，只修改 listen 與 server_name，不要刪除套件提供的 location 與 PHP 設定。本章 HTTP 僅限本機無認證測試；依第二十章先完成 HTTPS，才可執行第十八章 Web Setup 與修改 Admin 密碼。

```text
listen 80;
server_name __ZABBIX_FQDN__;
```

編輯 /etc/php-fpm.d/zabbix.conf，確認時區設定。

```text
php_value[date.timezone] = __TIMEZONE__
```

```text
nginx -t
php-fpm -t
systemctl enable --now php-fpm nginx
systemctl restart php-fpm nginx
systemctl --no-pager --full status php-fpm nginx
```

#### PHP 8.4 CLI 版本一致性檢查

只有在 Shell 顯示 php command not found，或需要執行 php -l 驗證前端 PHP 檔案時，才安裝 CLI。此主機的 Web 套件使用 PHP 8.4，CLI 必須使用相同版本。

```text
rpm -q php8.4-common php8.4-cli
dnf -y install php8.4-cli
php -v
```

不要直接安裝通用套件 php-cli。AlmaLinux 10 AppStream 可能選到 PHP 8.3，與既有 php8.4-common 衝突。遇到版本衝突時不得使用 --allowerasing 或 --skip-broken 規避，應改裝 php8.4-cli 並確認 php -v 顯示 8.4.x。

## 十五 SELinux 與 Firewall

### SELinux 維持 Enforcing

```text
getenforce
rpm -q zabbix-selinux-policy-main
getsebool httpd_can_connect_zabbix httpd_can_network_connect_db
setsebool -P httpd_can_connect_zabbix on
setsebool -P httpd_can_network_connect_db on
restorecon -Rv /etc/zabbix /usr/share/zabbix /var/log/zabbix
```

httpd_can_connect_zabbix 專門允許 Zabbix Web 前端連線至 Zabbix Server TCP 10051。httpd_can_network_connect_db 用於 Web 前端透過 TCP 連線資料庫；若實際只使用 localhost 的 MySQL Unix Socket，仍應依公司最小權限政策確認是否需要保留。

不要用 setenforce 0 或修改 SELINUX=disabled 當成正式解法。若服務遭拒絕，先查看 AVC Log。

```text
ausearch -m AVC,USER_AVC -ts recent
```

### Firewall 起始規則

僅允許核准來源。先用 firewall-cmd --get-active-zones 確認套用的 zone，保留已驗證 SSH 管理路徑；下列 `__MGMT_CIDR__`、`__AGENT_CIDR__`、`__FIREWALL_ZONE__` 為占位符，替換後才執行。若既有環境已放寬 http／https／10051，先新增精確規則並從第二個管理終端驗證，再移除對應的廣泛規則，不一次改掉 SSH。

```text
firewall-cmd --get-active-zones
firewall-cmd --zone=__FIREWALL_ZONE__ --permanent --add-rich-rule='rule family="ipv4" source address="__MGMT_CIDR__" service name="https" accept'
firewall-cmd --zone=__FIREWALL_ZONE__ --permanent --add-rich-rule='rule family="ipv4" source address="__AGENT_CIDR__" port port="10051" protocol="tcp" accept'
firewall-cmd --reload
firewall-cmd --zone=__FIREWALL_ZONE__ --list-all
```

Web Setup 前必須 HTTPS；80 若開放也只准導向且限管理來源。本機 Agent 10050 只綁 loopback，MySQL 3306 與 33060 不開外部規則。有 IPv6 時另行建立核准來源規則，不假設 IPv4 規則涵蓋 IPv6。

### 外部看門狗的固定來源規則

啟用第三十章外部看門狗前，將其實際到達 Zabbix 的固定來源 IPv4 記為 `__WATCHDOG_IPV4__`。它必須可連 HTTPS 443 與 TCP 10051；原本的 `__MGMT_CIDR__`、`__AGENT_CIDR__` 若已涵蓋該 IP，先核對現有規則，不重複放寬。若經 NAT，依 Zabbix 端觀察到的來源設定；跨區網路、VPN 與中間 ACL 亦須同步核准。

以下占位符必須換成核准值。只加該來源 /32 與必要連接埠，不開放整個網段或公網；80 不在看門狗需求內。先保留管理連線與 Hyper-V Console，套用 runtime 規則後從看門狗主機測試，成功才持久化。

```text
WATCHDOG_IP='__WATCHDOG_IPV4__'
ZONE='__FIREWALL_ZONE__'
python3 -c 'import ipaddress,sys; ipaddress.IPv4Address(sys.argv[1])' "$WATCHDOG_IP" || exit 1
firewall-cmd --get-active-zones
# 人工核對 __FIREWALL_ZONE__ 對應 Zabbix 管理網卡，才執行下列兩行
firewall-cmd --zone="$ZONE" --add-rich-rule="rule family=\"ipv4\" source address=\"$WATCHDOG_IP/32\" port port=\"443\" protocol=\"tcp\" accept"
firewall-cmd --zone="$ZONE" --add-rich-rule="rule family=\"ipv4\" source address=\"$WATCHDOG_IP/32\" port port=\"10051\" protocol=\"tcp\" accept"
```

在看門狗主機驗證路由、DNS、公司 CA、443、10051 與實際 API。僅 curl -I 成功不等同 Token 可讀心跳，還要執行第三十章 zabbix-watch.service。不得加 -k 略過憑證驗證。

```text
curl -I https://__ZABBIX_FQDN__/
python3 -c 'import socket; socket.create_connection(("__ZABBIX_FQDN__",10051),5).close(); print("TCP 10051 OK")'
```

確認以上與 API 心跳均成功，再回 Zabbix 主機、使用同一組核准變數執行：

```text
firewall-cmd --zone="$ZONE" --permanent --add-rich-rule="rule family=\"ipv4\" source address=\"$WATCHDOG_IP/32\" port port=\"443\" protocol=\"tcp\" accept"
firewall-cmd --zone="$ZONE" --permanent --add-rich-rule="rule family=\"ipv4\" source address=\"$WATCHDOG_IP/32\" port port=\"10051\" protocol=\"tcp\" accept"
firewall-cmd --zone="$ZONE" --list-rich-rules
firewall-cmd --zone="$ZONE" --permanent --list-rich-rules
```

在測試窗口重開機或 reload 後複驗；非核准來源仍須被拒絕。若需回復，僅對本次新增的相同規則執行 --remove-rich-rule，runtime 與 permanent 分別回復，不清空既有規則。使用 IPv6 時另訂固定 /128 規則並測試，不直接放開全部 IPv6。

## 十六 設定本機 Zabbix Agent 2

編輯 /etc/zabbix/zabbix_agent2.conf。Hostname 必須與 Zabbix Frontend 建立的 Host Name 完全一致。

```text
Server=127.0.0.1
ServerActive=127.0.0.1
Hostname=__ZABBIX_FQDN__
ListenIP=127.0.0.1
```

```text
systemctl enable --now zabbix-agent2
systemctl --no-pager --full status zabbix-agent2
tail -n 100 /var/log/zabbix/zabbix_agent2.log
```

在第十八章登入後，到 Data collection → Hosts 找到安裝預設的 Zabbix server，將 Host name 改為 `__ZABBIX_FQDN__`，Agent interface 使用 127.0.0.1:10050。Visible name 可保留 Zabbix server。保留 Linux by Zabbix agent 與 Zabbix server health 範本，確認 Enabled；不要另建重複 Host。此本機 Host 採 loopback 被動檢查，與遠端 Windows Active-only 模式不同。

## 十七 啟動與整體服務驗證

```text
systemctl is-active mysqld zabbix-server zabbix-agent2 php-fpm nginx
ss -lntp | grep -E ':80|:10050|:10051|:3306'
curl -I http://__ZABBIX_FQDN__/
```

| 服務 | 應有狀態 | 主要檢查位置 |
| --- | --- | --- |
| mysqld | active running  僅本機 3306 | /var/log/mysqld.log |
| zabbix server | active running  TCP 10051 | /var/log/zabbix/zabbix_server.log |
| zabbix agent2 | active running  本機 TCP 10050 | /var/log/zabbix/zabbix_agent2.log |
| php fpm | active running | journalctl -u php-fpm |
| nginx | active running  TCP 80 或 443 | /var/log/nginx/error.log |

### 整體服務強化檢核

整體驗證時採分段重新啟動。先確認 MySQL 穩定，再啟動依賴資料庫的 Zabbix Server，避免一次重啟全部服務後無法判斷失敗層級。每一行都在 Shell 執行，不得貼入任何設定檔。

```text
systemctl restart mysqld
sleep 5
systemctl is-active mysqld
systemctl restart zabbix-server
sleep 5
systemctl restart zabbix-agent2 php-fpm nginx
systemctl is-active mysqld zabbix-server zabbix-agent2 php-fpm nginx
tail -n 50 /var/log/zabbix/zabbix_server.log
nginx -t
php-fpm -t
ss -lntp | grep -E ':(80|443|10050|10051|3306|33060)\b'
curl -I http://127.0.0.1/
firewall-cmd --list-all
```

成功判定：五項服務全部為 active；Zabbix Server Log 不再出現 database is down、mysql.sock (13) 或 Access denied；Nginx 與 PHP FPM 語法檢查成功；TCP 80、10050、10051 正常監聽；MySQL 3306 與 33060 只監聽 127.0.0.1；curl 的本機 HTTP 回傳 200／302，或在 HTTPS 導向啟用後回傳 301；正式登入另須通過第二十章 HTTPS 驗證。若其中一項失敗，只處理該服務，不要停用 SELinux 或開放 MySQL 對外連線作為臨時解法。

## 十八 完成 Web Setup

先完成第二十章 HTTPS 與信任鏈驗證，再使用管理端瀏覽器開啟 https://`__ZABBIX_FQDN__`。若 DNS 尚未完成，可暫時在管理端 Hosts 檔加入對應，但正式驗收前必須完成 DNS。

進入安裝精靈前，依附錄 A 的 A1 至 A4 確認中文與英文系統 locale、PHP gettext 及 Web 服務正常。語言下拉選單出現項目並不代表作業系統已具備對應 locale。

1.  Default Language 選简体中文（zh_CN）。保留 English（en_GB）與 English（en_US）供個別使用者切換。若語言存在但呈灰色，先依附錄 A 安裝系統 locale 並驗證；zh_TW 維持選用，正式主流程不設為預設語言。

2.  確認所有 PHP Pre requisites 顯示 OK。

3.  Database Type 選 MySQL。

4.  Database Host 填 127.0.0.1，Port 填 3306。

5.  Database Name 填 `__DB_NAME__`，User 填 `__DB_APP_USER__`，Password 由保管庫取用 `__DB_APP_PASSWORD__`。

6.  Zabbix Server Host 填 localhost，Port 填 10051。

7.  Name 填公司識別名稱，例如 Company Infrastructure Monitoring。

8.  Default Time Zone 選 `__TIMEZONE__`。

9.  首次登入帳號及初始密碼請依該版本官方安裝文件與部署方式確認，由管理員自行提供；登入後立即變更初始密碼並建立具名管理帳號，不在本 SOP 記錄任何預設或正式密碼值。

若無法寫入設定檔：依 Web Setup 畫面下載 zabbix.conf.php，安全傳送到 /etc/zabbix/web/zabbix.conf.php，限制檔案權限並重新整理頁面。該檔案含 DB 密碼，不得公開。

完成登入後開啟系統資訊，確認 Zabbix 伺服器端運行中顯示是，連線位置為 localhost:10051。若畫面上方同時殘留連線失敗訊息，先強制重新整理一次；若錯誤持續，依第二十五節的 Web 前端連線檢核處理。

### 安裝後語系設定與驗收

以 Super admin 登入 Administration → General → GUI，將 Default language 設為 Chinese（zh_CN）。個別使用者在 User settings → Profile → Language 選 English（en_GB）或 English（en_US）；選 System default 則跟隨全域預設。個人語言設定只影響該使用者，詳細檢核與回復依附錄 A。

## 十九 上線前安全強化

☐ 已變更預設 Admin 密碼並建立個人管理帳號

☐ 已停用或降低共用 Admin 帳號的日常使用

☐ 已使用公司 CA 或受信任憑證切換 HTTPS

☐ TCP 10051 僅允許 Agent Active 與 Proxy 來源網段

☐ MySQL bind address 為 127.0.0.1 且 Firewall 未開放 3306

☐ Zabbix DB 密碼、Root 密碼與備份帳號已存入密碼庫

☐ SELinux 為 Enforcing

☐ 已套用公司弱點修補與 OS 更新流程

☐ 已規劃 LDAP 或 AD 登入與最小權限 Role

☐ 已記錄 Emergency Access 與人員異動停權流程

### 強化檢核補充

☐ my_print_defaults mysqld 已確認載入 bind-address 與 mysqlx-bind-address，且兩者均為 127.0.0.1

☐ /etc/my.cnf 與 /etc/my.cnf.d 內沒有誤貼 mysqld、mysql、systemctl、sleep、ss 或 curl 等 Shell 指令

☐ ss 已確認 MySQL 3306 與 33060 未監聽 0.0.0.0、星號或非本機位址

☐ Nginx 只使用核准的 TCP 80 與 443；若使用非標準埠，已依變更程序建立對應 SELinux http_port_t 規則

☐ systemctl is-enabled 已確認 mysqld、zabbix-server、zabbix-agent2、php-fpm 與 nginx 均為 enabled

☐ Firewalld 未開放 TCP 3306 與 33060，TCP 10050 與 10051 僅允許必要來源

☐ 分享前已檢查正文、表格、頁首頁尾、作者資訊、批註、追蹤修訂與附件；沒有回填真實帳密、內部資產或秘密值。

### SSH 登入基線

先確認具名 wheel 帳號能以金鑰登入並 sudo，保留 Hyper-V Console 與現有管理連線。將以下內容寫入 /etc/ssh/sshd_config.d/10-`__ORG_TAG__`.conf；檢查既有更早載入的設定與 Match 區塊，不能只憑檔案內容認定生效。

```text
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
```

```text
sshd -t
sshd -T | grep -Ei 'permitrootlogin|pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication'
systemctl reload sshd
```

另開第二個管理終端，以具名帳號金鑰登入並驗證 sudo 後才關閉原連線。若公司 SSH 使用 PAM／MFA，先依核准設計調整 KbdInteractiveAuthentication，不直接套此無互動基線。依來源位址套用的 Match 規則另用 sshd -T -C 核對。

### Guest 與角色權限

Users → User groups 確認 Guests 群組停用或 Frontend access 為 Disabled，檢查 guest 沒有藉其他群組取得權限。不要刪除系統保留帳號來代替停權。建立 Monitoring-Viewers 與 Monitoring-Operators，分別授予必要 Host groups 的唯讀與讀寫權限；Users → User roles 另限制設定選單、API 與可執行動作。Host group 權限與 Role 必須同時核對。

日常使用具名帳號；Super admin 僅供平台管理。保留一個本機 Emergency Access，設獨立強密碼並存入保管庫，驗證登入與使用稽核；帳號及保管庫紀錄由部署單位自行提供。

### AD LDAP 登入

先確認 php-ldap 與 PHP-FPM 使用相同版本；本平台若為 PHP 8.4，先用 dnf info php8.4-ldap 核對，不混裝其他 PHP 套件。公司 CA 放入系統信任庫並 update-ca-trust；到 AD DC 的 LDAPS 636 或 StartTLS 389 限定來源。禁止明文 LDAP bind。

在 Users → Authentication → LDAP settings 啟用 LDAP，新增伺服器：Host 使用 ldaps://`__LDAP_FQDN__`、Port 636、Base DN 依 AD 實際 OU／DN、Search attribute 使用 sAMAccountName、Bind DN 使用最小權限專用查詢帳號。LDAPS 與 StartTLS 二選一，不同時啟用；必須驗證 CA 與主機名稱，不關閉憑證驗證。

先用 Test 驗證測試帳號。第一階段手動建立同名 Zabbix 使用者與專屬 LDAP 群組，Frontend access 設 LDAP；待登入與停權測試成功，再評估 JIT provisioning 及精確 AD 群組到 Zabbix Role／User group 的映射。不先把全部 LDAP 人員映射為 Admin。保留本機 Emergency Access 群組為 Internal，避免 AD 故障時所有管理員都被鎖住。

### TOTP 多因素驗證

Users → Authentication → MFA settings 勾選 Enable multi-factor authentication，新增 TOTP 方法；確認驗證器支援所選雜湊與碼長。先對測試 User group 指定該方法，登出再登入完成 QR code 註冊，驗證正確與錯誤碼，再逐步套用具名管理員。伺服器與驗證器時間須正確。

測試群組限制、LDAP 加 TOTP 登入與裝置遺失的管理員重設程序後才擴大。不要在沒有可用 Emergency Access 或第二管理員的情況下，一次對全部帳號強制 MFA。OTP 秘鑰與 QR code 不存一般工單或截圖。

### 1.11 強化檢核

- ☐ 第五與二十二章：獨立備份掛載、容量、外部保留與失敗通知已設定。

- ☐ 第十三章：Server／Web TCP、帳號 Host、SELinux 與回復路徑已驗證。

- ☐ 第二十三與二十四章：完整還原與 RTO／RPO 有實測證據，未達標不得簽署。

- ☐ 二十八章：代表性 EDR 試點、Cisco view 對應與共享 PSK 窗口完成。

- ☐ 第三十章：備份 Items／Trigger、獨立心跳與第二通道真正收件。

### 1.11 強化檢核補充

單份異常副本告警、容量估值、CA／防火牆設定備份、看門狗固定來源、核心 MSI 釘選與暫存驗收，須一併完成第二十四章新增證據；原本 1.10 已勾選的項目不能自動代表新增測試也已通過。

## 二十 HTTPS 設定原則

先完成本章，再回第十八章進行 Web Setup。取得符合 `__ZABBIX_FQDN__` 的 SAN 憑證與完整鏈，私鑰由 root 管制；先備份原 Nginx 設定。既有 Zabbix server block 保留原有 root、location、PHP 與 fastcgi 內容，刪除該 block 的 listen 80，只將監聽及憑證部分改成下列設定，不可用片段整份覆蓋套件設定。

```text
listen 443 ssl;
server_name __ZABBIX_FQDN__;
ssl_certificate /etc/pki/tls/certs/zabbix-fullchain.pem;
ssl_certificate_key /etc/pki/tls/private/zabbix.key;
ssl_protocols TLSv1.2 TLSv1.3;
```

另建立 /etc/nginx/conf.d/zabbix-redirect.conf；確認沒有其他相同 server_name 與相同 listen 的重複 block。

```text
server {
    listen 80;
    server_name __ZABBIX_FQDN__;
    return 301 https://__ZABBIX_FQDN__$request_uri;
}
```

HTTP 僅作重新導向，不提供登入與 Setup。正式防火牆允許來源依第十五章限縮；不需要 HTTP 導向時可關閉外部 80。公司根 CA 依核准流程匯入 OS 與管理端信任庫，主機名稱必須與憑證 SAN 一致。

```text
chmod 600 /etc/pki/tls/private/zabbix.key
restorecon -Rv /etc/pki/tls/certs /etc/pki/tls/private
nginx -t
systemctl reload nginx
curl -I http://__ZABBIX_FQDN__/
curl -I https://__ZABBIX_FQDN__/
```

第一個回應應為 301 且 Location 指向同一核准 HTTPS FQDN；第二個必須在不加 -k 時成功驗證憑證，回傳 200 或正常 302。初期本機 HTTP 僅可測套件是否運作，不傳送 Admin、DB 密碼或 PSK。若憑證尚未就緒，暫停 Web Setup，或由核准的 SSH 加密通道與僅本機監聽方案另行處理，不能直接讓管理端走明文 HTTP。

## 二十一 Housekeeping 與容量設定

進入 Administration  Housekeeping，第一階段建議採以下保留策略。正式值仍應在上線一個月後，依 NVPS、DB 成長率與稽核需求調整。

| 資料類型 | 起始保留值 | 說明 |
| --- | --- | --- |
| History | 30 天起算，90 天須容量簽核 | 詳細數值  對容量影響最大 |
| Trends | 365 天 | 長期容量與效能趨勢 |
| Events Problems | 365 天 | 事故追蹤與稽核 |
| Audit Log | 365 天或依公司政策 | 管理操作紀錄 |
| Network Discovery | 依實際需求 | 避免持續保留無效探索資料 |

Cisco Interface Discovery 必須排除 Administratively Down、Loopback、Null、未使用 Port 與大量短生命週期介面，否則監控項目數與 History 會迅速增加。

### 初始容量估算與保留決策

容量不能只按伺服器台數決定。先估 NVPS = 各類 Item 數量／各自採集秒數的總和，將依賴項、Discovery 與字串／Log 類型另外列入。數值 History 起算公式如下，B_history 須用實測資料與索引占用量校正，不把經驗值當成 MySQL 保證。

```text
History bytes = NVPS x 86400 x History_days x B_history
Trends bytes  = Numeric_items x 24 x Trends_days x B_trend
Required DB   = History + Trends + Events + Config + Index overhead
                + Binlog + Redo + Temp space + Growth reserve
```

試算假設 B_history=100 bytes、B_trend=100 bytes，皆含粗略規劃餘裕而非已量測值。100 NVPS、90 天 History 即約 77.8 GB；500 NVPS 同條件約 388.8 GB，尚未加 Trends、事件與其他開銷，因此 300 GB 不能直接保證足夠。針對每類資料計算，避免把已含索引的實測 bytes 再重複加索引。

初始上線先以 30 天 History、365 天 Trends 評估；若業務要求 90 天 History，先完成容量試算與簽核才調整。每日記錄 NVPS、information_schema.tables 的 data_length＋index_length、檔案系統用量、binlog 與 slow log，至少連續一週；以尖峰／成長率推估到達 80% 的日期，在前 30 天啟動擴容或保留調整。Housekeeping 後 InnoDB 檔案不保證立刻縮小，不把刪除保留期當成即時釋放磁碟工具。

### Binlog 與 Slow log 維護

先確認 SHOW VARIABLES LIKE 'log_bin'、SHOW BINARY LOGS 與 slow_query_log_file 的實際路徑。7 天 binlog 保留只代表清理設定，不等於已具備時間點還原；未外送 binlog 並記錄一致快照座標前，不承諾 PITR。不得用 rm 刪除正在使用的 binlog；經核准且確認複寫／備份需求後才使用 MySQL PURGE BINARY LOGS。

```text
mysql -u__DB_ADMIN_USER__ -p -e "SHOW VARIABLES WHERE Variable_name IN ('log_bin','binlog_expire_logs_seconds','slow_query_log_file','log_output'); SHOW BINARY LOGS;"
du -sh /var/lib/mysql
df -hT / /var/lib/mysql /backup
```

Slow log 建議每週或超過 100 MB 輪替、保留 8 份，優先使用套件現有 /etc/logrotate.d/mysql* 並核對是否已涵蓋實際 slow log。新增規則前確認沒有重複路徑。以 rename＋MySQL FLUSH SLOW LOGS 重新開檔，專用輪替帳號僅授予所需 RELOAD 並將認證檔設 600；先 logrotate -d，再於維護窗口測試。不要直接複製未核對路徑的規則或使用 copytruncate 宣稱無遺失。

### 時間來源

本平台以 chronyd 及公司 NTP 為持續校時來源。Hyper-V Time Synchronization 可參與開機／恢復及提供宿主機時鐘，不應一律認定它與 chrony 衝突，也不可未驗證就停用全部 Integration Services。先查 chronyc sources -v、chronyc tracking、timedatectl，以及是否配置 PHC／Hyper-V PTP refclock。不要同時額外啟用 ntpd 或 systemd-timesyncd。

若恢復／遷移後發生時間跳動，連同宿主機 NTP 與 Integration Service 設定一起調查；選定公司 NTP 或核准 PHC 的時間策略並實測重開機、暫停恢復與遷移。對正在運作的 MySQL／Zabbix 不進行未核准的大幅手動校時。

### Housekeeper 與分區評估條件

保留 Zabbix 7.0 原生 Housekeeping 作起始方案；History 建議先以核准的 30 天計算，延至 90 天前先重算空間與還原時間。少於 30 台主機不代表資料量小，NVPS、LLD 數量、採樣頻率、文字值與保留政策才是負載依據。

InnoDB 大量 DELETE 的主要風險是 I/O、undo／purge 壓力及併發影響，不能一概稱為表鎖。於 Latest data 連續觀察 Housekeeper busy、Queue、資料庫延遲與磁碟 IOPS，並記錄資料表成長及過期資料是否持續累積。

以下為本專案的初始評估門檻，需以基線修訂，不是原廠效能保證：Housekeeper busy 超過 75% 持續 15 分鐘且伴隨 DB 延遲／Queue 上升；過期資料連續兩個清理週期仍成長；或容量預測觸及 80%、還原超出核准預算。任一成立即安排 DBA 評估，不能自動執行分區 DDL。

MySQL History／Trends 分區屬獨立變更：先驗證精確 Zabbix／MySQL 版本、主鍵與唯一鍵限制、升級相容性、建分區排程、未來分區不足告警、DROP 舊分區、metadata lock 與失敗回復。只有外部分區管理已通過完整演練，才關閉對應 History／Trends 的內建清理；Events、Audit 等其他清理不隨之全部關閉。分區不是所有規模的必選項，亦不能代替備份。

### DB 連線與權限改善邊界

本版新建平台採本機 TCP，不需要為 Zabbix 新增 MySQL 資料目錄 other execute。既有 socket 例外依第十三章維護窗口驗證與回復；未驗證其他使用者前不直接收權限。

## 二十二 MySQL 備份

### 執行模型與復原目標

本版取代 1.9 單一備份腳本，使用隨附 Zabbix_SOP_v1.11_Operations.zip 的三個獨立 systemd 工作。本機備份每 6 小時執行，外送每 15 分鐘重試一份、優先最新快照，保留清理每小時執行。本機備份不等待外送，也不因外送失敗停止；本機與外送各自記錄結果及快照時間。

RTO 4 小時、RPO 8 小時均為待服務負責人核准與演練驗證的起始目標，不是已達成的保證。300 GB 是 DB 磁碟容量，不代表實際資料量。若完整還原超時，依第二十三章調整備份方式或核定目標，不能以加警語代替改善。

| 工作 | 主要行為 | 失敗影響 |
| --- | --- | --- |
| local | InnoDB 一致性邏輯備份、設定備份、校驗、原子發布 .local_verified | 不更新本機成功快照；保留最近完整副本並告警 |
| upload | 讀取完整副本，外送到 .incoming，遠端校驗後發布並標記 .offsite_verified | 不更新外部成功快照；下次重試，不阻止 local |
| retention | 份數／快照年齡／空間輪替，清理過期未運作中的 partial | 保護最新兩份及上傳鎖定副本；安全清理後仍不足就告警 |

### 備份儲存與容量

/backup 必須是獨立掛載的備份檔案系統，與 / 及 /var/lib/mysql 不同。新增專用 VHDX 或核准 NAS，不把備份寫入約 110 GB 根目錄，也不與 DB 同一檔案系統。第五章已將備份儲存列為正式必要資源；同一 Hyper-V 上的第三磁碟仍不能抵抗宿主機損壞，必須另有故障範圍外副本。

以 S 表示單份壓縮 DB 加設定檔的實測大小。起始本機容量至少規劃為 (28 × S + 1.5 × S + 10 GiB) × 1.3，並隨成長重估；每天四份、保留七天。S=10 GiB 的示例需約 397 GiB，可評估 500 GiB；這不是本環境已量測值。首次未知時以 DB 已用量及 SQL 輸出膨脹保守估算，首次估值須依 1.11 容量預檢明確設定。

新增磁碟前先依第七章辨識容量、序號、既有檔案系統與用途，再建立獨立 XFS 掛載 /backup；不得照抄 /dev/sdb，該裝置在原建置流程用作 DB。先使用測試磁碟演練，新增 fstab 後執行 findmnt --verify、mount -a、systemctl daemon-reload 與重開機驗證。任何磁碟身分疑問立即停止。

```text
lsblk -e7 -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL,SERIAL
findmnt /backup
findmnt /var/lib/mysql
df -hT / /var/lib/mysql /backup
```

### 帳號與備份內容

以自行提供的 MySQL 管理帳號建立獨立備份帳號 `__DB_BACKUP_USER__`；`__DB_BACKUP_PASSWORD__` 請自行設定且不得與應用或管理帳號共用。以下為待替換範例，不含可使用的認證值。

```text
CREATE USER '__DB_BACKUP_USER__'@'localhost' IDENTIFIED BY '__DB_BACKUP_PASSWORD__';
GRANT SELECT, SHOW VIEW, TRIGGER ON __DB_NAME__.* TO '__DB_BACKUP_USER__'@'localhost';
SHOW GRANTS FOR '__DB_BACKUP_USER__'@'localhost';
```

以 root 建立 /root/.zabbix-backup.cnf，owner root、權限 600。備份帳號仍由 root 執行的本機工作使用 Unix socket；應用服務改用 TCP 不要求備份帳號也改連線方式。

```text
[client]
user=__DB_BACKUP_USER__
password=__DB_BACKUP_PASSWORD__
host=localhost
```

備份包含 firewalld 規則、內部 CA 信任錨、zabbix 邏輯資料、Trigger、/etc/zabbix（包含 Web DB 設定）、MySQL／Nginx／PHP 設定、憑證與私鑰、自訂 alertscripts／externalscripts、Linux keys、systemd、Repo、fstab 及套件清單。自訂 routines／events 另列權限與備份清單；MySQL 系統帳號由恢復流程重建。Windows Agent 正式 keys 由端點備份或密碼庫覆蓋，不會自動出現在 Linux 備份。

DB 與設定檔不是同一原子快照，備份窗口禁止 DDL／Schema 升級及設定變更；變更完成後補一份備份。gzip、SHA256 與 tar 校驗只證明檔案可讀與傳輸完整，不等同資料庫已還原成功。所有副本含密碼與 PSK，需儲存加密、ACL 與取件稽核。

### 安裝腳本與遷移舊排程

將腳本包解壓至 /root/Zabbix-SOP-1.11-Operations，以下由平台管理員一次設定，與工程師端 Agent 部署無關。先備份舊腳本、設定與 systemd 單元。

```text
dnf -y install python3 rsync tar gzip util-linux zabbix-sender
cd /root/Zabbix-SOP-1.11-Operations
bash install-linux.sh platform
systemctl disable --now zabbix-backup.timer
systemctl is-active zabbix-backup.service
```

若舊 zabbix-backup.service 仍在執行，等待其完成並確認狀態，不與新版並行操作同一目錄。安裝器不格式化磁碟、不變更密碼、不啟用排程；既有 /etc/zabbix-backup.json 不覆蓋。

編輯 /etc/zabbix-backup.json。first_backup_estimate_bytes 改為必須明確設定的正整數 bytes，新範例為 null，不提供可能被誤用的固定 100 GiB。keep_local_count=28、protect_latest_count=2、keep_local_days=7、partial_max_age_hours=24 維持起始政策；minimum_free_bytes=10 GiB 是安全底線。啟動所需可用空間為 max（規劃估值，最近實測大小）×1.5＋底線，不自動降低規劃下限。依本章 1.11 更新與 preflight 步驟核對。

外送帳號、目錄、SSH 私鑰依公司環境填寫。私鑰及 JSON 為 root 600；known_hosts 指紋由可信管道確認。不得使用 StrictHostKeyChecking=no。遠端先建立專用目錄，核准 rsync、sha256sum 與原子 rename 所需權限；此 push 帳號不代表不可變保護，真正防勒索副本由獨立備份平台建立。

### 保留與空間壓力政策

清理先於本機空間檢查執行。符合份數或快照年齡的舊副本可清理；空間不足時，可再刪較舊副本以保留新備份空間。任何情況保護最新兩份已完成副本及正在上傳的副本，不刪最後可用副本換取表面成功。仍不足則停止本次備份並告警，由管理員擴容或調整已核准政策。

外送中斷時，未外送的舊副本也受本機保留政策限制，不無限累積。每次刪除這類副本會增加 unshipped_pruned_total、更新 unshipped_pruned_epoch 並寫入 Journal，必須列入告警與稽核。這代表外部副本保護已降級，不能顯示全部正常。此有限保留政策須經服務負責人同意。

過期 partial 只有在 local job 未執行時才清除；完整副本以檔案鎖避免與 upload／retention 互相刪除。1.9 舊格式目錄不會自動納入新版輪替，須在新版完整備份與隔離還原通過後，按變更單處理舊檔。不得以萬用 rm 清空備份目錄。

遠端平台保留 30 天及月備份 12 份，另清理逾 24 小時且未運作的 .incoming 殘留，確認它長於單次上傳 timeout；本套件不會替遠端平台啟用保留或不可變政策，必須由備份管理員設定及測試，未設定不得勾選完成。

### 測試通知及啟用排程

先由郵件管理員配置本機 MTA，驗證 /usr/sbin/sendmail 的核准 relay；編輯 /usr/local/lib/`__ORG_TAG__`-zabbix/zabbix-backup-notify 收件者。外部看門狗另使用不同故障範圍的通知路徑。

```text
python3 /usr/local/lib/__ORG_TAG__-zabbix/zabbix_backup.py preflight
python3 -m py_compile /usr/local/lib/__ORG_TAG__-zabbix/zabbix_backup.py
systemd-analyze verify /etc/systemd/system/zabbix-backup-*.service /etc/systemd/system/zabbix-backup-*.timer
systemctl start zabbix-backup-failed.service
systemctl start zabbix-backup-local.service
journalctl -u zabbix-backup-local -n 100 --no-pager
systemctl start zabbix-backup-upload.service
systemctl start zabbix-backup-retention.service
ls -l /var/lib/zabbix/backup-status
```

確認測試信到達、本機 .local_verified、manifest.json／SHA256SUMS 與遠端 .offsite_verified。再依第二十三章完成隔離還原；完成第三十章狀態 Items／Trigger 後才啟用：

```text
systemctl enable --now zabbix-backup-local.timer zabbix-backup-upload.timer zabbix-backup-retention.timer
systemctl list-timers 'zabbix-backup-*'
```

必測故障為外送連續失敗時 local 仍成功、空間壓力安全輪替、只剩兩份時拒絕再刪、partial 清理、上傳鎖保護、網路恢復後外送最新副本，以及通知失敗。只在測試儲存模擬容量壓力，不填滿正式磁碟。

Veeam 仍作整機恢復用途；未確認 MySQL quiesce 及還原結果前，不以 application-aware 勾選或 Hyper-V Production Checkpoint 宣稱 MySQL 一致性。資料庫以實際驗證的邏輯或經另案驗收的物理備份為恢復依據。

### 1.11 單份損壞隔離與狀態判讀

catalog 掃描會逐份檢查必要檔案、metadata 格式／快照時間／run 名稱，以及 metadata 與 manifest 的校驗一致性。單份缺失或損壞時不納入正常清單，保留原目錄並告警，不自動搬移或刪除；其他正常副本仍可備份、外送及輪替。完整 SQL／設定內容 SHA256 仍於新備份及外送時驗證，目錄掃描不等於每小時重讀所有大型備份。

invalid_backup_count 大於 0 必須通知維運人員，root-only invalid-backups.json 列出異常目錄及錯誤類別，invalid_backup_epoch 記錄異常集合變更時間。不得把已隔離副本當成最新兩份正常保護副本。若它持續占用空間，管理員先取證、嘗試復原或從外部副本取回，再依變更單處理；不能為了清告警刪光目錄。

隔離不掩蓋整體儲存故障：掛載遺失、整個目錄不可讀或檔案系統 I/O 錯誤仍停止工作。備份子目錄改為掛載及檔案系統確認後才建立，避免在未掛載的 OS 根磁碟留下同名目錄。

| 狀況 | 結果與維運處理 |
| --- | --- |
| 備份尚未開始，空間不足或無法安全輪替 | local_result=1，沒有新成功快照；維持最近可用副本並告警 |
| 備份已發布，後續清理失敗 | local_result=0，retention_result=1；成功快照保留，清理問題獨立告警 |
| 單份 metadata 缺失／無法解析／校驗不符 | invalid_backup_count 增加；異常副本保留，其他正常工作繼續 |
| 外送中斷 | upload_result=1，不標記外送完成；下次重試同一副本可利用暫存部分資料 |

### 外送續傳與逾時預算

rsync 在受限 .incoming-RUN_ID 內使用相對 .rsync-partial。正常中斷可保留部分資料，重試同一 RUN_ID 時利用它；所有檔案完整校驗通過後才發布正式遠端目錄。命令逾時先送中斷訊號、最多等 10 秒收尾，再必要時強制結束。斷電或直接強殺無法保證保存部分檔，仍可重新傳輸，不能把未完成內容標成可還原副本。

upload service 預設 50 分鐘，涵蓋約 40.5 分鐘的命令逾時總和並保留餘裕。本機大檔完整校驗另需時間，首次須量測整個工作耗時；若超出預算，由管理員按容量／鏈路調整並重新驗收，不無限等待，也不宣稱慢線一定能在一個窗口完成。遠端 .incoming 與 .rsync-partial 保留／清理須長於核准傳輸窗口，不能清掉正在重試的副本。

### 從 1.10 更新維運腳本

由平台管理員在核准窗口執行，工程師端 Agent 不受影響。先停用三個 timer，等待既有服務完成；不要強制中止正在進行的正常備份或上傳。

```text
systemctl disable --now zabbix-backup-local.timer zabbix-backup-upload.timer zabbix-backup-retention.timer
systemctl is-active zabbix-backup-local.service zabbix-backup-upload.service zabbix-backup-retention.service
```

若任一仍 active／activating，等待並檢查 Journal，尚未完成就停止更新。備份 /usr/local/lib/`__ORG_TAG__`-zabbix、/etc/zabbix-backup.json 及對應 systemd units 到 root 700 的受限變更目錄，再從新版解壓目錄執行 install-linux.sh platform。安裝器不覆蓋既有 JSON，也不自動啟用 timer。

新範例 first_backup_estimate_bytes=null；管理員須填入單份備份的保守規劃正整數 bytes，未設定就停止 local。例：10 GiB 為 10737418240 bytes；該示例與 minimum_free_bytes=10 GiB 合計要求至少 25 GiB 可用空間，並非本環境量測值。原 1.10 的 100 GiB 會要求 160 GiB，可用容量不足時必須重新規劃，不能直接拿掉安全底線。設定值持續作規劃下限，不因近期副本較小而自動降低。

既有 JSON 另須在 config_paths 加入 etc/firewalld、etc/pki/ca-trust/source/anchors。完成設定後執行 preflight；它只顯示所需／可用 bytes，不做輪替或外送，亦不代表 MySQL 一致性或實際還原已通過。

```text
cd /root/Zabbix-SOP-1.11-Operations
bash install-linux.sh platform
sudoedit /etc/zabbix-backup.json
python3 /usr/local/lib/__ORG_TAG__-zabbix/zabbix_backup.py preflight
systemd-analyze verify /etc/systemd/system/zabbix-backup-*.service /etc/systemd/system/zabbix-backup-*.timer
```

通過後依本章測試 local、upload、retention、實際通知與第三十章新增告警；再啟用三個 timer。驗收須包括異常目錄隔離、備份成功但清理失敗、容量預檢、中斷續傳及看門狗防火牆來源。1.10 格式的正常副本可繼續使用，1.9 舊格式仍按原政策人工處理。

## 二十三 還原演練

### 隔離還原演練

正式上線前及每季，在另一台隔離 VM 還原。與正式主機相同 MySQL 裡建立測試 DB 只算匯入測試，不能替代災難復原演練。演練使用獨立磁碟、相同核准版本套件及隔離 vSwitch／ACL，封鎖連往正式 Agent、Proxy、SMTP、Webhook、LDAP 與業務系統；受控管理端才可存取。不得使用正式 DNS 名稱或正式 IP。

演練不停止正式 zabbix-server，也不修改正式 DB 權限。先從外部備份平台取回一份有 .local_verified 且外部已校驗的備份，在隔離端確認 SHA256SUMS、gzip 與 tar；記錄取件耗時及快照下限時間。

```text
cd /restore/zabbix/RUN_ID
sha256sum -c SHA256SUMS
gzip -t zabbix.sql.gz
tar -tzf config.tar.gz > /dev/null
```

RUN_ID 必須替換成選定的備份目錄。以下命令僅在隔離 VM 執行，先確認 hostnamectl、IP、MySQL 版本與隔離 ACL。若不是隔離端立即停止。MySQL 的 `__DB_NAME__` 資料庫必須不存在；本節不提供可誤刪正式資料庫的 DROP 前置命令。

```text
hostnamectl
ip -brief address
mysqld --version
systemctl stop zabbix-server nginx php-fpm
mysql -u__DB_ADMIN_USER__ -p -e "SHOW DATABASES LIKE '__DB_NAME__';"
mysql -u__DB_ADMIN_USER__ -p -e "CREATE DATABASE __DB_NAME__ CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;" || { echo 'STOP: target DB exists or CREATE failed'; exit 1; }
set -o pipefail
gzip -dc zabbix.sql.gz | mysql -u__DB_ADMIN_USER__ -p __DB_NAME__ || { echo 'STOP: restore failed'; exit 1; }
mysql -u__DB_ADMIN_USER__ -p -e "SELECT * FROM __DB_NAME__.dbversion; SELECT COUNT(*) FROM information_schema.tables WHERE table_schema='__DB_NAME__';"
```

在隔離端重建 `__DB_APP_USER__` 專用帳號與 `__DB_NAME__`.* 權限，使用隔離端憑證；不要把正式帳號授權到額外測試 DB。將 config.tar.gz 解到 /restore/config-review 而不是直接覆蓋 /。比對後只還原必要設定，改成隔離主機名稱、隔離 DB 認證與測試憑證；先停用通知 Action、自動註冊 Action、外部腳本與備份排程。確認網路封鎖仍生效才啟動應用服務。

```text
install -d -m 700 /restore/config-review
tar --acls --xattrs --selinux -xzf config.tar.gz -C /restore/config-review
# 完成設定比對、隔離帳號建立與通知停用後才執行以下命令
nginx -t
php-fpm -t
systemctl start zabbix-server php-fpm nginx
systemctl is-active mysqld zabbix-server php-fpm nginx
tail -n 100 /var/log/zabbix/zabbix_server.log
```

驗收須包含 HTTPS 登入、Host／Template／巨集／Action 設定、歷史資料抽查及隔離測試 Agent 的新資料。比較備份前記錄的關鍵物件數量，不能只看 table_count。RTO 從宣告演練故障起算，包含建機、取件、安裝、匯入與功能確認；RPO 以模擬故障時間減去該備份 metadata.json 的 snapshot_lower_bound_epoch 的時間，這是保守窗口，不以檔案完成時間代替快照時間。

完成後封存結果、清除隔離 VM 與還原工作區，依公司安全刪除政策處理含密碼與 PSK 的副本。若沿用舊版做過同機匯入測試，先 SHOW GRANTS 確認，再由 DBA 僅針對 zabbix_restore_test.* 回收權限並 DROP 該測試 DB；必須人工核對名稱，不得對正式 zabbix 執行。

### CA 與防火牆設定恢復檢核

從設定備份取回 etc/pki/ca-trust/source/anchors 與 etc/firewalld，先在隔離工作區審查，不直接覆蓋或 reload 正式來源 IP 規則。還原核准 CA anchors 後執行 update-ca-trust extract、restorecon，驗證 HTTPS／LDAPS 信任；防火牆規則以恢復環境的管理 IP／watchdog IP 修改，透過 Console 測試後才持久化。證書檔存在不代表系統信任庫已更新。

### 真實災難恢復

在核准變更與已確認的正式恢復目標上，隔離寫入流量後停止 zabbix-server 與 Web，重建資料庫並還原核准備份。完成 DB／設定／密鑰與 SELinux 標籤檢查，再依序啟動 mysqld、zabbix-server、Agent 2、PHP-FPM、Nginx。確認 Latest data、事件、通知及備份排程後才恢復正式入口。恢復程序必須包含啟動步驟；停止服務後不得只留下匯入命令。

### RTO 容量驗證與升級路徑

RTO 4 小時是起始目標；未有完整還原紀錄，驗收狀態就是「未證實」，不能簽署已達標。以實際資料庫大小、索引、表數、CPU／RAM／磁碟延遲及取件頻寬測試，不用 300 GB VHDX 容量推定資料量，也不用主機台數推定負載。

RTO = 建機與安裝 + 取件／解密／校驗 + DB 還原及索引完成 + 設定與密鑰恢復 + 功能驗證。各階段記錄開始／結束時間、備份快照下限、資料量、版本、硬體與瓶頸，保留至少三次具代表性的完整演練。RPO = 模擬故障時間 − 選用備份 metadata.json 的 snapshot_lower_bound_epoch；外送完成時間不是資料的新鮮度。

同等結構與硬體下，可用「預估資料量 ÷ 完整演練的有效還原速率」做容量預警；小樣本、壓縮檔大小或單表吞吐不能證明整庫 RTO。容量成長與最慢一次演練均須納入餘裕；取件或還原已耗盡核准階段預算，立即進行改善評估。

| 路徑 | 適用條件與限制 | 導入驗收 |
| --- | --- | --- |
| 既有 mysqldump 邏輯備份 | 小至中等資料量且完整演練達標 | 保留目前易操作流程，每季及大幅成長後重測 |
| MySQL Shell 8.4 dump／load | 需要並行匯出與載入；須另裝核准版本並審查必要權限 | 在隔離環境測 util.dumpSchemas 與 util.loadDump、threads、consistent 與 local_infile；後者是不同格式，不能直接載入 sql.gz |
| MySQL 8.4 相容物理備份 | 邏輯還原仍超標且具 DBA 維運能力 | 先核對工具、授權、精確版本相容性、prepare 與完整還原；不能未查證就套用 XtraBackup 指令 |
| 設定與歷史分層恢復 | 業務明確同意歷史資料降級／遺失 | 另案設計 schema 相依、使用者與設定一致性及回補，不能任意挑數張表匯出就稱可恢復 |

本版不因理論估算宣稱 8–12 小時或保證 4 小時。實測若不能達成目標，先改善備份／恢復方式，再由服務負責人核准仍無法消除的 RTO／RPO 差距。正式演練結束必須確認 mysqld、zabbix-server、Agent、PHP-FPM、Nginx 與後續備份已恢復。

## 二十四 驗收測試

每項記錄執行人、日期、實際結果、證據連結及審核人；未測寫待驗收，不能預先勾選。高優先項不通過就停止正式交付，保留既有服務並修正。

| 檢核範圍 | 合格條件與證據 | 對應章節 |
| --- | --- | --- |
| OS／DB／Web | 重開機自啟、固定版本、loopback 綁定、TCP 帳號、Enforcing、HTTPS 信任及 Log 連續 30 分鐘正常 | 七至十八、二十 |
| 帳號與語系 | 具名操作、Guest／Role、核准 LDAP／TOTP；zh_CN 預設，en_GB／en_US Profile 可選並儲存 | 十九、附錄 A |
| Agent 工程師操作 | 一台代表性 Server 僅複製 EXE 及右鍵管理員執行；不用輸入 PSK；Running／Auto、Latest data、重開機與升級正常 | 二十八之五、二十至二十四 |
| EDR 與來源 | 實際政策下完成試點；外層 EXE、腳本與 MSI 雜湊／簽章及 C／D 路徑證據 | 二十八之二十 |
| Linux／Windows | 各至少一台新資料、PSK、停止／恢復事件與通知；共享窗口結束及來源清理有紀錄 | 二十八 |
| Cisco | authPriv／ACL；選用範本所有必要 OID、LLD、CPU、Memory、Inventory、介面及適用環境指標；N/A 附型號證據 | 二十八之七至十 |
| 通知 | Problem、升級、Recovery 送達實際值班人；抑制與手動關閉行為可預期 | 二十九 |
| 備份儲存 | /backup 獨立掛載並重開機保留；容量公式與遠端保留／安全 ACL 已核准 | 五、二十二 |
| 備份故障 | 連續外送失敗 local 仍成功、有限輪替、最後兩份保護、partial／上傳鎖、校驗失敗及網路恢復重試 | 二十二 |
| 完整恢復 | 獨立隔離 VM、設定／密鑰、物件與歷史資料、新資料、服務重啟；完整 RTO 及快照 RPO 達核准目標 | 二十三 |
| 平台自我監控 | Health 範本、磁碟／cache／queue、備份狀態與無資料告警，真正收件 | 三十 |
| 第二通道 | 不同故障範圍，斷電、舊心跳、API／SMTP 失敗與復原測試；實測通知時間 | 三十 |
| 維運交付 | CMDB、責任與聯絡、密碼庫位置、升級回復、審核欄；執行資料與密碼庫權限核准；分享前複查 | 十九、二十六、三十一 |

不能以 gzip／checksum 成功代替完整還原，也不能以本文件的模擬測試代替上述實機證據。

### 1.11 新增驗收證據

- ☐ 第十五與第三十章：外部看門狗實際固定來源可連 443／10051，API 可讀新心跳，其他來源仍被拒絕。

- ☐ 第二十二章：metadata 損壞只隔離單份、異常副本未自動刪除、剩餘正常工作成功且告警送達。

- ☐ 容量 preflight、未掛載不建立備份目錄、備份成功後清理失敗分項狀態均已驗證。

- ☐ 受控中斷後續傳、50 分鐘整體預算及實際完整校驗耗時已有證據。

- ☐ 設定還原包含 firewalld 與 CA anchors；更新信任庫後 HTTPS／LDAPS 可驗證，隔離 VM 不直接套正式來源規則。

- ☐ Windows 單獨 Deploy 缺 MSI／錯誤雜湊／簽章時不讀 PSK、不變更服務；IExpress 初次暫存與中斷後殘留 ACL 已實測。

## 二十五 常見故障排查

| 現象 | 優先檢查 | 處理方向 |
| --- | --- | --- |
| Zabbix Server 無法啟動 | zabbix server log  journalctl  DBPassword | 先確認 DBPassword、Schema、127.0.0.1:3306 與帳號 Host；檢查實際 AVC，舊 socket 平台另依既有設定排查 |
| Web 顯示 DB Connection Error | zabbix conf php  MySQL User  SELinux AVC | 確認 127.0.0.1:3306、帳號 Host、密碼與 httpd_can_network_connect_db |
| Nginx 顯示 502 | php fpm 狀態與 Socket | 核對 zabbix nginx conf 與 php fpm zabbix conf |
| Agent 顯示 Unavailable | Hostname  10050  Server  ServerActive | Host Name 必須完全相同  確認被動或主動模式 |
| SNMP 沒資料 | UDP 161  SNMPv3 Credential  ACL | 先用 snmpwalk 驗證  再檢查 Template 與 Discovery |
| DB 成長過快 | Items  NVPS  History Retention  LLD | 排除無效介面  延長間隔  縮短 History |
| CPU 不高但畫面很慢 | Disk Latency  MySQL Slow Log  Cache | 優先檢查儲存延遲與 DB  不只加 CPU |
| Web 無法連線 localhost 10051 | zabbix server 10051  httpd can connect zabbix  SELinux AVC | 先確認服務與監聽，再啟用 httpd_can_connect_zabbix；本機連線不以關閉 SELinux 或廣泛開放 Firewall 處理 |
| English 存在但灰色 | locale -a 及 PHP setlocale | 依附錄 A 的 A1 至 A5 檢查並安裝 glibc-langpack-en，驗證 PHP-FPM 環境後再判斷是否需要修改語系檔 |
| 修改語系檔後 HTTP 500 | php -l locales.inc.php 及 PHP Log | 依附錄 A 的 A10 與 A13 還原當次備份，語法檢查成功後才重啟 Web 服務 |

```text
journalctl -u zabbix-server -n 200 --no-pager
journalctl -u nginx -u php-fpm -n 200 --no-pager
tail -n 200 /var/log/zabbix/zabbix_server.log
tail -n 200 /var/log/nginx/error.log
ausearch -m AVC,USER_AVC -ts recent
mysqladmin -u__DB_ADMIN_USER__ -p status
```

### 標準化故障排查流程

排查原則是先確認設定與掛載，再確認服務與 Log，最後才處理 SELinux 與 Firewall。假設建置已失敗，最常見原因依序是設定檔混入 Shell 指令或未被載入、MySQL Socket 的目錄穿越權限或 SELinux 標籤錯誤，以及 Nginx 使用未被 SELinux 核准的非標準埠。

#### 第一步 確認 DB 磁碟與 systemd 掛載狀態

```text
findmnt --verify --verbose
systemctl daemon-reload
mount -a
findmnt /var/lib/mysql
mountpoint /var/lib/mysql
df -hT /var/lib/mysql
lsblk -f
```

成功判定：findmnt 沒有 parse error；/var/lib/mysql 是掛載點；來源為 300 GB DB 磁碟的 XFS 分割區。若 findmnt 提示 systemd 仍使用舊版 fstab，執行 systemctl daemon-reload 後重新驗證。掛載錯誤修正前不得啟動 MySQL。

#### 第二步 MySQL 無法啟動或設定未生效

```text
my_print_defaults mysqld
mysqld --validate-config
journalctl -u mysqld -n 100 --no-pager
grep -RInE '^[[:space:]]*(mysqld|mysql|systemctl|sleep|ss|curl)([[:space:]]|$)' \
  /etc/my.cnf /etc/my.cnf.d
grep -RInE 'bind[-_]address|mysqlx[-_]bind[-_]address' /etc/my.cnf /etc/my.cnf.d
```

若看到 unknown option '--mysqld --validate-config'，代表完整命令被放進選項檔；刪除誤貼命令後重新執行驗證。不要刪除原有調校參數，也不要使用 --no-defaults 啟動正式服務來規避設定錯誤。

#### 第三步 Zabbix Server 無法連線 MySQL

先核對 DBHost=127.0.0.1、DBPort=3306 與 Web 的 SERVER／PORT；依第十三章執行 TCP SELECT USER(), CURRENT_USER(), 1，再確認 ss、服務 Log 與 ausearch。Access denied 先檢查 MySQL 帳號 Host，Connection refused 先檢查 mysqld 與 loopback 綁定，AVC 按套件政策審查。不要用 chmod 資料目錄處理 TCP 問題。舊平台仍走 socket 時，先識別現行連線方式，依原變更回復，不照抄新版遷移片段。

#### 第四步 Nginx 無法啟動或顯示 SELinux bind denied

```text
nginx -t
grep -RInE '^[[:space:]]*(listen|server_name)' /etc/nginx/conf.d/zabbix.conf
journalctl -u nginx -n 100 --no-pager
ausearch -m AVC,USER_AVC -ts recent | tail -n 50
semanage port -l | grep http_port_t
```

本手冊正式介面使用 443，80 僅作 HTTPS 導向；server_name 統一為 `__ZABBIX_FQDN__`。若誤設為 8999 並看到 bind() failed (13 Permission denied)，先改回核准的 80 或 443。若業務確實要求非標準埠，須經變更核准後以 semanage port 將該埠加入 http_port_t；不得以 setenforce 0 當作解法。

#### 第五步 Web 前端無法連線 localhost 10051

若系統資訊上方顯示無法連線 Zabbix Server，但下方狀態同時顯示運行中為是，先重新整理瀏覽器一次，排除前一次檢查留下的暫時訊息。若錯誤持續，再依下列順序檢查。

```text
getenforce
getsebool httpd_can_connect_zabbix
systemctl is-active zabbix-server
ss -lntp | grep ':10051'
grep -E '^(ListenIP|ListenPort|NodeAddress)=' /etc/zabbix/zabbix_server.conf
grep -E 'ZBX_SERVER|ZBX_SERVER_PORT|ZBX_SERVER_NAME' /etc/zabbix/web/zabbix.conf.php
ausearch -m AVC,USER_AVC -ts recent | tail -n 50
journalctl -u zabbix-server -n 50 --no-pager
```

若 httpd_can_connect_zabbix 為 off，執行 setsebool -P httpd_can_connect_zabbix on，再重新啟動 php-fpm 與 nginx。若已為 on，則比對 Zabbix Server 的監聽位址與 /etc/zabbix/web/zabbix.conf.php 的 Host、Port，並保留 AVC 與 Journal 輸出。localhost 連線不需要因這個錯誤額外新增 Firewalld 對外規則；不得停用 SELinux，也不得以範圍較大的 httpd_can_network_connect 代替專用布林值。

```text
setsebool -P httpd_can_connect_zabbix on
systemctl restart php-fpm nginx
getsebool httpd_can_connect_zabbix
```

#### 第六步 復原後完整驗證

```text
systemctl is-enabled mysqld zabbix-server zabbix-agent2 php-fpm nginx
systemctl is-active mysqld zabbix-server zabbix-agent2 php-fpm nginx
mysql -u__DB_ADMIN_USER__ -p -e \
"SHOW VARIABLES WHERE Variable_name IN ('bind_address','mysqlx_bind_address','innodb_buffer_pool_size');"
ss -lntp | grep -E ':(80|443|10050|10051|3306|33060)\b'
curl -I http://127.0.0.1/
tail -n 50 /var/log/zabbix/zabbix_server.log
ausearch -m AVC,USER_AVC -ts recent
```

只有在五項服務均為 enabled 且 active、Web 回應正常、Zabbix Server Log 沒有資料庫錯誤、MySQL 僅本機監聽且沒有新增 AVC 拒絕時，才能繼續 Web Setup 或加入正式監控設備。任何驗證失敗都應保留原始 Log 與當次變更內容，再回到對應步驟處理。

## 二十六 變更與回復策略

| 階段 | 成功判定 | 失敗時處理 |
| --- | --- | --- |
| Base OS | 網路 DNS NTP 正常 | 回到 Base OS Ready Production Checkpoint |
| MySQL | 8.4.x  Service Active  Root 可登入 | 檢查 Repo 與 Log  不直接混裝 MariaDB |
| Schema | Table Count 正常  utf8mb4 bin | 清除僅限新建空白 DB  重新匯入前先確認目標 |
| Zabbix Server | Service Active  Log 無 DB Error | 回復設定檔或回到 Schema Imported Production Checkpoint |
| Web | FQDN 可登入  HTTPS 正常 | 回復 Nginx PHP 設定  不關閉 SELinux 逃避問題 |
| 正式上線 | Agent SNMP Alert Backup 全部驗收 | 停止新增 Host  先修正核心平台再擴展 |

## 二十七 建議的下一階段

1.  先加入 Zabbix Server 自身、兩台 Windows、兩台 Linux 與一台 Cisco Switch。

2.  觀察兩週 NVPS、Cache Utilization、DB 成長率、誤報與告警量。

3.  建立 Host Group、Template Naming、Severity、Maintenance 與責任人標準。

4.  Windows 與 Linux 優先採 Agent 2 Active Mode，敏感網段使用 TLS PSK。

5.  Cisco 改用 SNMPv3，從原廠 MIB 與實際業務需求挑選指標。

6.  再加入 AD、Oracle、Veeam、UPS、FortiGate 與 MES 服務層指標。

7.  跨廠區導入 Zabbix Proxy，避免 WAN 中斷期間失去資料。

## 二十八 Windows Linux 與 Cisco 監控納管流程

本章將 Windows Server、Linux Server 與 Cisco IOS 交換機納入正式監控。端點預設使用 Zabbix Agent 2 Active Mode，由端點主動連線 Zabbix Server TCP 10051，手動建檔流程使用每台主機獨立的 TLS PSK；本批 Windows 安全自動註冊改採共享 bootstrap PSK，依本章二十八之十四至二十七執行。Cisco 交換機使用 SNMPv3 authPriv、SHA 驗證與 AES 128 加密，並以 ACL 只允許 Zabbix Server 或 Proxy 查詢。密碼與 PSK 只存放在公司密碼保管庫，不寫入本 SOP、工單附件或群組訊息。

### 二十八之一 納管標準與責任分工

納管前先完成設備責任人、服務等級、管理 IP、DNS、NTP、維護時段與網路路徑確認。沒有責任人或無法確認告警接收人的設備，不得直接加入正式告警。

| 階段 | 執行人 | 必須完成 | 驗收證據 |
| --- | --- | --- | --- |
| 申請 | 系統或網路管理員 | 確認設備名稱、IP、用途、責任人、維護時段與敏感等級 | 納管申請單或變更單 |
| 連線準備 | 網路與資安管理員 | 確認 DNS、NTP、路由、ACL 與必要 Port | DNS 查詢、Port 測試與 ACL 紀錄 |
| 部署 | 系統或網路管理員 | 安裝 Agent 2 或設定 SNMPv3，不在文件留下密碼 | 設定備份、服務狀態與版本資訊 |
| Zabbix 建檔 | Zabbix 管理員 | 建立 Host、Host Group、Template、Encryption 或 SNMP 介面 | Host 設定截圖或變更紀錄 |
| 驗收 | 設備責任人與 Zabbix 管理員 | 確認 Latest data、可用性、告警測試與回復 | 驗收表、告警時間與回復時間 |
| 維運 | 設備責任人 | 每月檢查不可用設備、低階告警、憑證或密碼到期與版本 | 月報與改善工單 |

### 二十八之二 網路與命名基準

| 來源 | 目的 | 協定與 Port | 用途 | 正式原則 |
| --- | --- | --- | --- | --- |
| Windows 或 Linux Agent 2 | Zabbix Server 或 Proxy | TCP 10051 Outbound | Active checks 取得設定並送出資料 | 端點預設模式，只開放到核准的 Server 或 Proxy |
| Zabbix Server 或 Proxy | Windows 或 Linux Agent 2 | TCP 10050 Inbound | Passive checks | 預設不啟用；例外需求須限制來源 IP 並使用 PSK |
| Zabbix Server 或 Proxy | Cisco 管理 IP | UDP 161 | SNMP polling | 只允許 Zabbix Server 或 Proxy 管理 IP |
| Cisco 管理 IP | SNMP Trap Receiver | UDP 162 | SNMP trap | 第一階段不強制；啟用時另立 Trap 規範 |
| 全部設備 | 公司 DNS 與 NTP | UDP TCP 53 與 UDP 123 | 名稱解析與時間同步 | 時間誤差須在公司標準內 |

正式命名規則如下。Active Mode 的 Host name 必須與 Agent 設定檔內 Hostname 完全一致，包括大小寫；Visible name 可使用較易閱讀的名稱。

| 項目 | 標準 | 範例 |
| --- | --- | --- |
| Agent Host name | 使用 OS 的正式 FQDN，不使用臨時名稱 | `__HOST_FQDN__` |
| Windows Host Group | `__WINDOWS_GROUP__` 或 Servers/Windows/Test | `__WINDOWS_GROUP__` |
| Linux Host Group | `__LINUX_GROUP__` 或 Servers/Linux/Test | `__LINUX_GROUP__` |
| Cisco Host Group | `__NETWORK_GROUP__` | `__NETWORK_GROUP__` |
| PSK Identity | PSK_加上 Host name，每台主機唯一 | `__HOST_PSK_IDENTITY__` |
| SNMPv3 Security name | 使用公司統一唯讀帳號，不含人名 | `__SNMP_USER__` |

### 二十八之三 Linux Agent 2 下載與安裝

以下命令適用 AlmaLinux 或 RHEL 10 x86 64。其他 Linux 發行版先在 Zabbix 官方下載頁選擇 Zabbix 7.0 LTS、正確 OS 與 Agent 2，不得混用不同主版本的 Repository。

```text
dnf -y install https://repo.zabbix.com/zabbix/7.0/alma/10/x86_64/zabbix-release-7.0-7.el10.noarch.rpm
dnf clean all
dnf -y install zabbix-agent2
rpm -q zabbix-agent2 zabbix-release
zabbix_agent2 -V
```

安裝後先備份設定檔並產生每台主機唯一的 256 bit PSK。下列 HOST_FQDN 必須改為該主機正式 FQDN；產生的 PSK 值立即存入公司密碼保管庫。

```text
HOST_FQDN=$(hostname -f)
cp -a /etc/zabbix/zabbix_agent2.conf /etc/zabbix/zabbix_agent2.conf.pre-monitoring
install -d -m 750 -o root -g zabbix /etc/zabbix/keys
openssl rand -hex 32 > /etc/zabbix/keys/${HOST_FQDN}.psk
chown zabbix:zabbix /etc/zabbix/keys/${HOST_FQDN}.psk
chmod 640 /etc/zabbix/keys/${HOST_FQDN}.psk
stat -c '%U %G %a %n' /etc/zabbix/keys/${HOST_FQDN}.psk
```

### 二十八之四 Linux Agent 2 標準參數

編輯 /etc/zabbix/zabbix_agent2.conf。Active Mode 不設定 Server，避免開放 TCP 10050；同一參數只能保留一筆，不能同時散落在主設定檔與 Include 目錄。

```text
ServerActive=__ZABBIX_FQDN__:10051
Hostname=__HOST_FQDN__
HostMetadata=__LINUX_METADATA__
Timeout=10
TLSConnect=psk
TLSPSKIdentity=__HOST_PSK_IDENTITY__
TLSPSKFile=/etc/zabbix/keys/__HOST_FQDN__.psk
```

正式套用前執行下列檢查。Active Mode 正常時，Zabbix Agent 2 不需要監聽 TCP 10050；端點只需能解析並連出 `__ZABBIX_FQDN__` TCP 10051。

```text
grep -RInE '^(Server|ServerActive|Hostname|HostMetadata|TLSConnect|TLSAccept|TLSPSKIdentity|TLSPSKFile)=' \
  /etc/zabbix/zabbix_agent2.conf /etc/zabbix/zabbix_agent2.d
zabbix_agent2 -T -c /etc/zabbix/zabbix_agent2.conf
zabbix_agent2 -t agent.ping -c /etc/zabbix/zabbix_agent2.conf
getent hosts __ZABBIX_FQDN__
timeout 3 bash -c '>/dev/tcp/__ZABBIX_FQDN__/10051' && echo TCP_10051_OK
systemctl enable --now zabbix-agent2
systemctl --no-pager --full status zabbix-agent2
journalctl -u zabbix-agent2 -n 100 --no-pager
ss -lntp | grep ':10050' || echo ACTIVE_ONLY_NO_10050_LISTENER
```

1.11 快速部署：工程師只按二十八之五複製 EXE 並右鍵管理員執行；管理員按十四至十六準備註冊及二十打包。十七為進階手動診斷，日常部署不用輸入命令。

### 二十八之五 工程師快速部署 Agent 2

工程師操作維持簡單：從受控共用資料夾取得管理員產生的 EXE，複製到目標 Server，再右鍵選以系統管理員身分執行。不輸入 PSK、不貼 PowerShell 指令、不逐台選版本，也不需要 Server 從外網下載 MSI。UAC 提示仍需依公司授權確認。

| 操作人 | 操作 | 不需要做 |
| --- | --- | --- |
| 管理 NB 維護人 | 完整解壓新版 Builder，保留既有 NB PSK，右鍵 Build-EXE.cmd | 不重新產生 PSK、不逐台建立新密鑰 |
| 工程師 | 複製 EXE 到 C:\Temp 或 D:\Temp，右鍵管理員執行 | 不輸入 PSK、不編輯設定、不連外下載 MSI |
| 維運驗收人 | 確認正確 Host、IP、群組、模板與 Latest data | 不以安裝 SUCCESS 代替收數驗收 |

本單位核准的完整 ZabbixAgent2_Local_EXE_Builder_v1.11.zip 應內含固定 Agent 2 7.0.32 MSI，不含 PSK。只有管理 NB 打包時才加入既有密鑰。打包階段及 Server 執行階段都自動比對固定 SHA256 與有效 Zabbix Authenticode 簽章，不需工程師手動輸入雜湊。版本與核准雜湊由套件維護人統一管理，禁止自動追 latest 或把新版本改成舊檔名。

Server 端沿用 FQDN／IPv4 自動偵測、Active 模式、PSK、設定驗證與常駐服務啟動。較新 Agent 已存在時阻止自動降版。多網卡可能無法自動判定公司管理網段，少數例外由管理員預先準備 HostnameOverride／IPv4Override，不要求一般工程師自行判斷。

套件完整解壓後在管理 NB 右鍵 Build-EXE.cmd。成功後先在一台測試 Server 驗收，再分批派送。原來的線上 EXE 與 D 槽舊 ZIP 已淘汰，不混入新版 Payload。工程師看到錯誤只回報 ProgramData 下部署 Log，不重生密鑰、不停用安全防護。

驗收完成後必須刪除 Server 上含 PSK 的 EXE。來源 bootstrap.psk 與獨立受限部署子目錄會嘗試自動清理，異常中斷時仍須檢查殘留；Agent keys 的正式運作金鑰不得刪除。密鑰清理屬安裝後收尾，不增加安裝時的互動輸入。

離線指不下載 MSI；DNS、Zabbix TCP 10051 及原廠簽章所需信任鏈／撤銷驗證條件仍需具備。公司 EDR、GPO 或應用程式控制若要求公司簽章，交由套件管理員統一簽章或核准受控派送，不由工程師逐台繞過。

本版已修訂工具，但 Windows IExpress、MSI、ACL、服務及完整收數仍須首台實機驗收；舊版線上安裝成功不代表新版離線封裝已完成實機驗收。

### 二十八之六 Zabbix 前端建立 Agent 2 主機

先在 Data collection → Hosts 建立 Host，再啟動 Agent 2。Linux 與 Windows 的 Host name 必須與 Agent 設定完全一致。Active Template 不需要 Agent interface；若畫面要求介面，通常代表誤套 Passive Template。

| 欄位 | Linux | Windows |
| --- | --- | --- |
| Host name | 與 Hostname 完全相同，例如 `__HOST_FQDN__` | 與 Hostname 完全相同，例如 `__HOST_FQDN__` |
| Host groups | `__LINUX_GROUP__` | `__WINDOWS_GROUP__` |
| Templates | Linux by Zabbix agent active | Windows by Zabbix agent active |
| Interfaces | Active Mode 留空 | Active Mode 留空 |
| Encryption → Connections from host | PSK | PSK |
| PSK identity | 與 TLSPSKIdentity 相同 | 與 TLSPSKIdentity 相同 |
| PSK | 從密碼保管庫貼入，不存入 SOP | 從密碼保管庫貼入，不存入 SOP |

啟用後五分鐘內確認 Monitoring → Hosts 的 Active agent availability 正常，Latest data 出現 agent.ping、CPU、記憶體、檔案系統與網路資料。若 Log 顯示 host not found，先檢查 Host name；若顯示 TLS 錯誤，先比對 PSK identity、PSK 值與 Encryption 設定。

### 二十八之七 Cisco IOS SNMPv3 設定

以下範例適用 Cisco IOS 或 IOS XE，採唯讀 SNMPv3 authPriv。先確認實際機型與 IOS 版本支援 SHA 與 AES 128；`__SNMP_SOURCE_IPV4__` 改為 Zabbix Server 或 Proxy 管理 IP。AuthPass 與 PrivPass 必須不同，且只存放在密碼保管庫。

```text
configure terminal
ip access-list standard ZABBIX_SNMP_ONLY
 permit __SNMP_SOURCE_IPV4__
 deny any log
exit
snmp-server view ZABBIX_READ 1.3.6.1.2.1.1 included
snmp-server view ZABBIX_READ 1.3.6.1.2.1.2 included
snmp-server view ZABBIX_READ 1.3.6.1.2.1.31 included
snmp-server group ZABBIX_GROUP v3 priv read ZABBIX_READ access ZABBIX_SNMP_ONLY
snmp-server user __SNMP_USER__ ZABBIX_GROUP v3 auth sha __SNMP_AUTH_PASSWORD__ priv aes 128 __SNMP_PRIV_PASSWORD__
snmp-server location __DEVICE_LOCATION__
snmp-server contact __DEVICE_CONTACT__
end
show snmp user
show snmp group
show access-lists ZABBIX_SNMP_ONLY
show running-config | section snmp-server
```

舊型 Catalyst 若不支援 AES 或 SHA，不得直接降級為 SNMPv2c。先確認 IOS 升級可行性；確實無法升級時，才依例外程序限定管理 VLAN、來源 ACL 與唯讀權限，並登錄弱加密風險及改善期限。

上述 view 只先涵蓋 system、interfaces 與 ifMIB。Cisco 範本的 CPU、記憶體、環境、實體清單等項目可能需要其他 Cisco／標準 MIB；先列出實際模型與 Items 使用的 OID，再逐個增加最小 subtree 並驗收。未授權的項目會無資料，不能宣稱基礎 view 已涵蓋全部官方範本。不得為快速通過而恢復整個 iso。

### SNMP view 與官方 Cisco 範本對應

下列對應以 Zabbix release/7.0 的 Cisco IOS by SNMP 原始 YAML 核對；實際選用 Catalyst／Nexus 型號範本時，再匯出該範本含 Discovery 的 snmp_oid 清單比對。原三個 subtree 只足以驗證 system 與介面，不能作全項納管驗收。

| subtree | MIB／用途 | 適用判定 |
| --- | --- | --- |
| 1.3.6.1.2.1.1 | system，名稱與描述 | 基本必要 |
| 1.3.6.1.2.1.2 | interfaces，介面狀態與錯誤 | 基本必要 |
| 1.3.6.1.2.1.31 | ifMIB，64-bit 流量與介面屬性 | 依設備能力驗收 |
| 1.3.6.1.2.1.10.7 | EtherLike，雙工與 Ethernet 指標 | IOS 範本使用 |
| 1.3.6.1.2.1.47 | ENTITY，型號／序號／實體清單 | IOS 範本使用 |
| 1.3.6.1.2.1.25 | HOST-RESOURCES，裝置 uptime 等 | IOS 範本使用 |
| 1.3.6.1.4.1.9.9.109 | CISCO-PROCESS，CPU | 依實際 CPU discovery |
| 1.3.6.1.4.1.9.2.1.58 | 舊 Cisco CPU fallback | 僅使用舊 CPU 指標的設備 |
| 1.3.6.1.4.1.9.9.48 | CISCO-MEMORY-POOL，記憶體 | 依實際 memory discovery |
| 1.3.6.1.4.1.9.9.13 | CISCO-ENVMON，溫度／風扇／電源 | 設備有對應感測器時 |

保留前節 system、interfaces、ifMIB，對選用 IOS 範本再於核准變更中加入下列對應項；未使用舊 CPU fallback 時不加最後一項。不同機型若改用 ENTITY-SENSOR 或其他 Cisco MIB，必須依該型號 YAML 另增相應最小 subtree，不能沿用本表就宣稱完整。

```text
configure terminal
snmp-server view ZABBIX_READ 1.3.6.1.2.1.10.7 included
snmp-server view ZABBIX_READ 1.3.6.1.2.1.47 included
snmp-server view ZABBIX_READ 1.3.6.1.2.1.25 included
snmp-server view ZABBIX_READ 1.3.6.1.4.1.9.9.109 included
snmp-server view ZABBIX_READ 1.3.6.1.4.1.9.9.48 included
snmp-server view ZABBIX_READ 1.3.6.1.4.1.9.9.13 included
snmp-server view ZABBIX_READ 1.3.6.1.4.1.9.2.1.58 included
end
show snmp view
```

先不儲存 startup-config，逐個 subtree 以核准 SNMPv3 認證測試並核對實際 Instances。驗證 Zabbix Discovery、CPU、Memory、型號／序號與適用的環境感測器後才執行 copy running-config startup-config 儲存。noSuchObject／noSuchInstance 可能是未支援，也可能是 view 未授權；須與設備能力、view 及同來源實際查詢共同判定。未支援感測器可列 N/A，附型號／韌體及查詢證據；授權缺漏、Not supported 或空值不能勾選成功。

回復只移除該次變更新增的 view 行，例如 no snmp-server view ZABBIX_READ 1.3.6.1.2.1.47 included；按變更清單逐項處理所有新增 subtree。若整個 ZABBIX_READ、group、user 為本案獨建且無其他使用者，再依二十八之十二移除整組；不能只刪原三項而遺留擴充授權。

### 二十八之八 從 Zabbix Server 驗證 SNMPv3

在 Zabbix Server 安裝測試工具，先驗證 system OID，再進行前端建檔。範例密碼只能使用臨時測試值；正式密碼不要寫入 Shell Script、Ticket 或共用 Shell History。

```text
dnf -y install net-snmp-utils
snmpwalk --version
snmpwalk -v3 -l authPriv -u __SNMP_USER__ -a SHA -A '__SNMP_AUTH_PASSWORD__' \
  -x AES -X '__SNMP_PRIV_PASSWORD__' __SWITCH_IPV4__ 1.3.6.1.2.1.1
snmpget -v3 -l authPriv -u __SNMP_USER__ -a SHA -A '__SNMP_AUTH_PASSWORD__' \
  -x AES -X '__SNMP_PRIV_PASSWORD__' __SWITCH_IPV4__ 1.3.6.1.2.1.1.3.0
```

成功判定：snmpwalk 可取得 sysDescr、sysName、sysLocation 等資料，snmpget 可取得 sysUpTime。Timeout 優先檢查路由、UDP 161、ACL、Privacy password 與設備管理 VRF；Authentication failure 則比對 security name、SHA、Auth password 及設備 SNMPv3 user。

### 二十八之九 Zabbix 前端建立 Cisco Host

在 Data collection → Hosts 建立交換機，新增 SNMP interface。IP 使用交換機管理 IP，不使用使用者 VLAN IP。

| 欄位 | 設定值 |
| --- | --- |
| Host name | 使用網路設備正式名稱，例如 `__SWITCH_FQDN__` |
| Host groups | `__NETWORK_GROUP__` |
| Template | Cisco IOS by SNMP；若套件顯示 Cisco IOS SNMP，確認為 release 7.0 官方模板 |
| SNMP interface | 管理 IP，Port 161 |
| SNMP version | SNMPv3 |
| Security name | `__SNMP_USER__` |
| Security level | authPriv |
| Authentication protocol | SHA |
| Privacy protocol | AES |
| Authentication and privacy passphrases | 由密碼保管庫取用，不寫入 SOP |

建立後確認 SNMP availability 變綠，Latest data 出現 sysName、uptime、型號、序號、CPU、Memory 與介面流量；Temperature／風扇／電源按型號能力驗收，N/A 須附證據。介面 LLD 完成後，只保留 Uplink、Server、Firewall 與關鍵業務 Port 告警；Access Port 不應全部套用高嚴重度 Link Down 告警。

### 二十八之十 驗收測試與交付紀錄

| 驗收項目 | 合格標準 | 證據 |
| --- | --- | --- |
| Agent 2 版本與服務 | 版本為核准的 7.0 LTS，服務 Automatic 或 enabled 且 running | 版本輸出與服務狀態 |
| Active checks | 端點可連 TCP 10051，五分鐘內出現 Latest data | Test NetConnection 或 TCP 測試及 Latest data |
| TLS PSK | 前端與 Agent identity 相同，未開放 unencrypted | Encryption 設定與 Agent Log |
| Agent 可用性告警 | 測試主機停止服務後在門檻時間內告警，恢復後自動回復 | Problem Event 與 Recovery Event |
| SNMPv3 | authPriv 可取得 system OID，非核准來源被 ACL 拒絕 | snmpwalk 與 ACL 計數器 |
| Cisco Template | Inventory、CPU、Memory、關鍵介面及設備支援的環境指標有資料；N/A 附能力證據 | Latest data 與 Discovery 狀態 |
| 告警接收 | 測試告警送達核准 Email 或 Teams 對象 | 訊息時間與收件人 |
| 文件交付 | 設備責任人、群組、Template、PSK 或 SNMP 憑證位置與回復方式已登錄 | CMDB 或變更單 |

告警測試只能使用測試主機或經核准的維護時段。Cisco Link Down 測試不得操作核心 Uplink、Stack、Firewall、AP Controller 或正式伺服器 Port；優先使用預留測試 Port。

### 二十八之十一 常見故障與判斷順序

| 症狀 | 最可能原因 | 處理順序 |
| --- | --- | --- |
| Agent 無資料但服務正常 | Hostname 不一致、TCP 10051 不通、PSK 不一致或誤套 Passive Template | 比對 Hostname → 測試 DNS 與 10051 → 查 Agent Log → 比對 Encryption 與 Template |
| Agent 顯示 host not found | 前端 Host name 與 Agent Hostname 不完全一致 | 直接複製雙方值比對大小寫與 FQDN，不以 Visible name 判斷 |
| Windows MSI 失敗 | MSI 版本不支援參數、路徑或舊 Agent 殘留、參數引號錯誤 | 查安裝 Log → 確認 MSI 版本 → 查既有服務與安裝目錄 → 修正後只重試一次 |
| SNMP Timeout | UDP 161、ACL、VRF、Privacy password 或設備未啟用 SNMP | 路由與 ACL → show snmp user → snmpwalk system OID → 比對 PrivPass |
| SNMP Authentication failure | Security name、SHA、Auth password 或安全層級不一致 | 比對 authPriv → user → SHA → AuthPass；不要先放寬 ACL |
| Cisco 介面告警過多 | LLD 將 Access Port 與虛擬介面全部納入 | 先確認關鍵 Port 清單，再調整 Discovery Filter 或 Template Macro，不直接停用整個 Template |

### 二十八之十二 回復與移除流程

Linux 回復前先將 Host 設為 Disabled 或進入 Maintenance，再停止 Agent、還原備份設定或移除套件。不要刪除 PSK 保管庫紀錄，直到稽核保留期限結束。

```text
systemctl disable --now zabbix-agent2
cp -a /etc/zabbix/zabbix_agent2.conf.pre-monitoring /etc/zabbix/zabbix_agent2.conf
zabbix_agent2 -T -c /etc/zabbix/zabbix_agent2.conf
dnf remove zabbix-agent2
```

Windows 回復先匯出安裝 Log 與設定備份，再從 Apps and Features 或核准的軟體派送工具移除 Zabbix Agent 2。Cisco 回復則依序移除 user、group、view 與 ACL；確認沒有其他 NMS 共用後才能刪除。

```text
configure terminal
no snmp-server user __SNMP_USER__ ZABBIX_GROUP v3
no snmp-server group ZABBIX_GROUP v3 priv
no snmp-server view ZABBIX_READ 1.3.6.1.2.1.1 included
no snmp-server view ZABBIX_READ 1.3.6.1.2.1.2 included
no snmp-server view ZABBIX_READ 1.3.6.1.2.1.31 included
no ip access-list standard ZABBIX_SNMP_ONLY
end
copy running-config startup-config
```

### 二十八之十三 每台設備納管紀錄

| 欄位 | 填寫內容 |
| --- | --- |
| 設備名稱與管理 IP |  |
| 系統或設備型號與版本 |  |
| 環境與服務等級 | Production Test  DR 及 SLA |
| 設備責任人與代理人 |  |
| Zabbix Host Group 與 Template |  |
| Agent 模式或 SNMPv3 Security name |  |
| PSK 或密碼保管庫紀錄編號 | 只填保管庫編號，不填秘密值 |
| 網路 ACL 與變更單號 |  |
| 驗收日期與執行人 |  |
| 告警測試與回復結果 |  |
| 例外風險與改善期限 |  |

### 二十八之十四 目前採用的自動納管流程

本節規範核准窗口內的批次納管。Windows 本批部署採 Agent 2 主動模式與 Zabbix 原生安全自動註冊，以共享 bootstrap PSK 自動加入群組及連結範本。前述每台主機獨立 PSK 與手動建檔流程仍保留作為另一種部署模式；同一台主機一次只選一種流程，不要把兩種 PSK 身分或密鑰混用。

作業順序為：NB 準備既有密鑰 → Zabbix 設定全域安全自動註冊 → 建立自動註冊 Action → NB 打包含固定 MSI 的離線 EXE → 單台 Server 部署 → Hosts 與 Latest data 驗收 → 分批推廣 → 停用批次 Action 並清理來源密鑰套件。安裝程式顯示 SUCCESS 不等同資料已進入 Zabbix。

| 項目 | 本批設定 |
| --- | --- |
| Zabbix Server | `__ZABBIX_FQDN__`；IP 以第二章核准值及 DNS 為準 |
| 模式與連線 | Active checks；端點連至 Server TCP 10051 |
| Metadata | `__WINDOWS_METADATA__` |
| 共享 PSK identity | `__BOOTSTRAP_PSK_IDENTITY__`；這是身分名稱，不是密鑰值 |
| 密鑰格式 | 現有密鑰為 64 個十六進位字元；不得寫入本文件 |
| 範本 | Windows by Zabbix agent active |
| 群組 | Action 選取核准的 `__WINDOWS_GROUP__` 群組；以既有群組物件名稱為準 |
| 主機名稱與 IP | 自動偵測 FQDN 與主要 IPv4，可用部署參數明確指定 |
| 被動監聽 | 不設定 Server，停用 Agent 被動檢查；不建立 TCP 10050 防火牆例外 |

### 二十八之十五 共享 PSK 的使用邊界

原生安全自動註冊使用全域 PSK。自動建立的主機會使用該共享身分與密鑰，系統不會在納管後自行轉換為每台主機不同的 PSK。Windows 與 Linux 若都使用同一個全域 PSK 自動註冊流程，就必須配合相同的全域設定；目前交付的自動部署腳本僅適用 Windows。Cisco 的 SNMPv3 驗證與加密密碼不使用這組 Agent PSK。

共享密鑰外洩會影響使用該密鑰的整批端點。應保存於公司密碼保管庫，限制部署套件與來源密鑰的存取。若改採每台主機獨立密鑰，需另行安排前端與端點同步切換；單獨更換全域自動註冊 PSK，不會自動更新既有 Agent 密鑰檔或 Host 加密設定。

初始化僅在可信任的管理 NB 執行一次。現有 InitializePsk 流程會重用既有來源檔，不應在每台 Server 各自初始化不同密鑰。初始化只準備密鑰，不安裝或啟動 Agent。

```text
.\Deploy-ZabbixAgent2-AutoRegistration.ps1 -InitializePsk
```

NB 的來源密鑰預設位置如下。不要將 `__BOOTSTRAP_PSK_IDENTITY__`、提示訊息、檔案路徑或帶引號的說明當成 PSK 值。

```text
C:\ProgramData\__ORG_TAG__\ZabbixAutoregistration\bootstrap.psk
```

Zabbix 接受偶數長度的 32 至 512 個十六進位字元；本批初始化與 EXE 包裝固定使用 64 碼。不要為了排除格式錯誤而重新產生密鑰，先確認複製的內容與全域設定一致。

### 二十八之十六 Zabbix 自動註冊設定

先到 Administration → General → Autoregistration，將允許的加密模式設為 PSK，填入 identity `__BOOTSTRAP_PSK_IDENTITY__` 與 NB 已產生的同一組 PSK，儲存後才部署未知端點。不要把密鑰值放進截圖或工單。

接著先建立目標 Host group，再到 Alerts → Actions → Autoregistration actions 建立並啟用 Windows Server 自動納管。Action 頁設定條件；Operations 頁按 Add，逐項加入下表操作。空白 Operations 不能儲存。

| 類型 | 設定 |
| --- | --- |
| Condition | Host metadata contains `__WINDOWS_METADATA__` |
| Operation | Add host |
| Operation | Add to host group，選取既有正式 Windows 群組 |
| Operation | Link templates，選取 Windows by Zabbix agent active |
| Status | Enabled |

儲存後在清單確認條件與三項操作均存在，狀態為 Enabled。此流程不必事先逐台手動建立 Host。若主機早已建檔，先核對現有 Hostname、加密設定與範本，不要直接建立同名或不同拼法的重複主機。

### 二十八之十七 進階手動診斷與 PowerShell 操作

在目標端以系統管理員開啟 Windows PowerShell。原部署程式要求 x64 Windows、Windows 10 或 Server 2016 以上，以及 PowerShell 5.1。Process 執行政策只對目前程序有效；MachinePolicy 或 UserPolicy 若由 GPO 限制，交由管理者核准，不以此指令規避網域政策。

```text
cd C:\Temp
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
Unblock-File .\Deploy-ZabbixAgent2-AutoRegistration.ps1
.\Deploy-ZabbixAgent2-AutoRegistration.ps1 -ZabbixServer __ZABBIX_FQDN__ -HostMetadata __WINDOWS_METADATA__ -PskIdentity __BOOTSTRAP_PSK_IDENTITY__ -MsiPath 'C:\Temp\Agent2.msi'
```

指令必須按順序執行。跨行範例的反引號表示續行，整段是一個命令；進階手動診斷優先使用單行版本，避免尾端空白或貼上順序顛倒。看到 >> 而未執行時，先按 Ctrl+C 回到正常提示，再重新輸入。

提示 Enter the Zabbix global autoregistration PSK 後才貼上密鑰一次並按 Enter。Read-Host 隱藏輸入不顯示字元屬正常現象；右鍵是否貼上取決於終端設定。不要因看不到內容而重複貼上。優先使用受保護的密鑰來源檔以免貼錯。

```text
& 'C:\Temp\Deploy-ZabbixAgent2-AutoRegistration.ps1' -PskSourceFile 'C:\Temp\bootstrap.psk' -MsiPath 'C:\Temp\Agent2.msi'
```

只有在初始化 NB，才會自然存在下列原始路徑；一般 Server 必須由部署套件提供 C:\Temp\bootstrap.psk，不得假設它已存在。

```text
& 'C:\Temp\Deploy-ZabbixAgent2-AutoRegistration.ps1' -PskSourceFile 'C:\ProgramData\__ORG_TAG__\ZabbixAutoregistration\bootstrap.psk' -MsiPath 'C:\Temp\Agent2.msi'
```

若需檢查剪貼簿，以下只顯示長度與格式結果，不輸出密鑰。檢查完成不代表密鑰一定與 Server 相同，仍須確認保管庫來源。

```text
$checkPsk = (Get-Clipboard -Raw).Trim()
[pscustomobject]@{
    Length = $checkPsk.Length
    Valid64Hex = ($checkPsk -match '^[0-9A-Fa-f]{64}$')
}
Remove-Variable checkPsk
```

### 二十八之十八 自動部署的設定與保護

部署腳本偵測主機 FQDN、主要 IPv4、DNS 與 TCP 10051，取得同一組共享 PSK，由新版 Run 入口讀取封裝內固定 MSI、比對核准 SHA256，再驗證 Authenticode 簽章與 Zabbix 簽署者。安裝後才寫入 PSK 與管理設定，驗證設定檔與 agent.ping，最後啟動服務並設為延遲自動啟動。

來源設定不提供 NetIPInterface 屬性；已改用 Get-NetIPInterface -InterfaceIndex 配合 IPv4 查詢連線狀態與 metric，避免 StrictMode 發生屬性不存在錯誤。多網卡選取只按候選介面 metric 排序，不能保證選到公司指定的管理網段；Hyper-V、VPN、多個預設閘道或備援網卡必須核對結果，必要時指定 IPv4Override 與 HostnameOverride。

```text
.\Deploy-ZabbixAgent2-AutoRegistration.ps1 -PskSourceFile 'C:\Temp\bootstrap.psk' -HostnameOverride '__HOST_FQDN__' -IPv4Override '__HOST_IPV4__' -MsiPath 'C:\Temp\Agent2.msi'
```

管理設定主要路徑如下。Hostname 與 HostInterface 每台不同；其餘依本批 Server、metadata 與 PSK identity 帶入。

| 類型 | 路徑 |
| --- | --- |
| Agent 主設定檔 | C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf |
| 管理用設定檔 | C:\Program Files\Zabbix Agent 2\zabbix_agent2.d\90-`__ORG_TAG__`-autoregistration.conf |
| 正式密鑰 | C:\ProgramData\`__ORG_TAG__`\ZabbixAgent2\keys\autoregistration.psk |
| Agent 與部署日誌 | C:\ProgramData\`__ORG_TAG__`\ZabbixAgent2\logs |

```text
ServerActive=__ZABBIX_FQDN__:10051
Hostname=__HOST_FQDN__
HostMetadata=__WINDOWS_METADATA__
HostInterface=__HOST_IPV4__
TLSConnect=psk
TLSAccept=psk
TLSPSKIdentity=__BOOTSTRAP_PSK_IDENTITY__
TLSPSKFile=C:\ProgramData\__ORG_TAG__\ZabbixAgent2\keys\autoregistration.psk
HeartbeatFrequency=60
ForceActiveChecksOnStart=1
```

這是主要參數摘錄，不可整段覆蓋主設定檔或重複加入已存在的 managed drop-in。端點省略 Server 才停用 Agent 2 被動檢查；STARTAGENTS=0 是 MSI 的安裝參數，不要把它當成 Agent 2 設定檔參數。

正式密鑰限制 SYSTEM 與 Administrators 存取。設定變更前保留備份；安裝或寫檔失敗時嘗試還原。前置檢查尚未修改 Agent 前就失敗時，不應停用既有 Agent。升級套件本身不保證可自動降版，已有 Agent 的 Server 要另外確認維護窗口及回復方式。

### 二十八之十九 舊 D 槽套件淘汰

舊 ZabbixAgent2_D_Drive_Deploy.zip 已淘汰，禁止作為新部署來源。使用二十八之五與二十的新離線 EXE，工程師仍只需複製與右鍵管理員執行；既有正常 Agent 不為工具改版而重裝。

### 二十八之二十 管理 NB 一次產生離線 EXE

完整解壓 ZabbixAgent2_Local_EXE_Builder_v1.11.zip 到新資料夾，保持下列結構。管理員應先確認 MSI 已包含；本文件不附安裝檔；PSK 從原管理 NB 的 C:\ProgramData\`__ORG_TAG__`\ZabbixAutoregistration\bootstrap.psk 自動讀取，不放到 ZIP，也不重新初始化。

| 檔案或目錄 | 用途 |
| --- | --- |
| Build-EXE.cmd 與 Build-EXE.ps1 | 右鍵管理員打包入口與檢查 |
| Installer\zabbix_agent2-7.0.32-windows-amd64-openssl.msi | 固定版本官方 MSI，Build 自動比對 SHA256 與簽章 |
| Payload | 成套啟動、解壓與 Deploy 腳本，不混入舊檔 |
| README.md | 一次準備、工程師操作與驗收邊界 |

右鍵 Build-EXE.cmd → 以系統管理員身分執行。工具在受限目錄複製既有 PSK、MSI 與腳本，再呼叫 IExpress；成功輸出帶日期時間的 EXE 並顯示 EXE SHA256。失敗不發送中間檔。EXE 含共享 PSK 與 MSI，只存於限制部署人員存取的共用資料夾；不公開、不上傳聊天。

工程師端不再下載 MSI。若需更新版本，由管理員更換 Installer MSI，核對可信原廠來源與簽章，同步核准 Build／Run／Deploy 三層的固定版本、SHA256 與完整發行者，再測試並重新打包。單純記錄未知檔案的 hash 不構成來源驗證。

IExpress 產生的 EXE 尚無公司程式碼簽章，不能把 MSI 原廠簽章當成整個 EXE 已簽章。正式派送採受控軟體庫及 ACL；有公司簽章規範時由管理員統一處理。NB 保留 PSK 是一次性準備，工程師不接觸明文密鑰。

### 大量派送前的 EDR 試點

工程師操作維持「複製已核准 EXE → 右鍵以系統管理員身分執行」，不增加現場輸入 PSK 或命令。離線 MSI、既定 SHA256、服務常駐與自動納管流程保留。沒有修改安裝核心時，不要求重新建 EXE；只要 payload／簽章／PSK 有變就必須重建並重驗。

IExpress 自解壓、Temp 寫入與 PowerShell 執行可能被 EDR／App Control 阻擋，是否攔截取決於產品、規則與簽章，不能宣稱 100%。正式推廣前，在實際 EDR 政策的代表性 Windows Server 2016／2019／2022 等環境試裝；只測 NB 或關閉 EDR 不算通過。

交付前記錄外層 EXE 雜湊／簽章、內層腳本／MSI 雜湊與簽章、ACL、解壓路徑、安裝與升級 Log、Running／Auto、10051、Latest data、失敗後重試及清理結果。官方 MSI 簽章不等於外層 EXE 已簽章。公司核准可採簽署者或精確檔案雜湊，禁止廣泛放行 Temp／PowerShell 或停用防護。

若現有封裝被阻擋，由管理端改採核准簽章包裝或 GPO／SCCM／Intune 派送固定 MSI 及設定；工程師仍只取受控套件執行，不自行加參數排錯。共享 PSK 及 EXE 的控管、批次窗口與後續清除仍依二十八之十五及二十八之二十八。

### 1.11 固定來源與暫存驗收

管理 NB 完整解壓新版 ZabbixAgent2_Local_EXE_Builder_v1.11.zip，沿用原 bootstrap.psk，再右鍵管理員執行 Build-EXE.cmd。新 ZIP 不含 PSK，不需重新初始化。因 payload 已更新，後續派送須重新建 EXE，不能繼續用舊 EXE 代表修正已生效。既有正常收數的 Agent 無需因此重裝。

Build、Run 與 Deploy 核心三層均核對固定 7.0.32 MSI SHA256、Valid Authenticode 與完整發行者 Subject。Subject 以內附固定 MSI 的憑證解析取得，忽略逗號周邊空白後為 CN=Zabbix SIA,O=Zabbix SIA,L=Riga,C=LV；不再只比對含 Zabbix 字串。換版或原廠換證由管理員核准對應檔案、雜湊及發行者，不能由工程師臨時略過驗證。

Deploy 不再接受 MsiUrl、不自行下載 MSI。手動診斷若缺 -MsiPath、檔案不存在、雜湊或簽章不符，在讀取 PSK、安裝或變更服務前停止；-InitializePsk 維持獨立，不要求 MSI。此設計不增加日常操作：EXE wrapper 已自動帶入內附 MSI。

IExpress 仍先解壓到執行帳號的 TEMP／IXP*.TMP，之後才複製到有 ACL 的 C:\Temp 或 D:\Temp 子目錄。初次暫存沿用實際父目錄 ACL，不能宣稱已受到後續腳本 ACL 保護，也不能推定必然全員可讀。修改 destinationRoot 不會改變 IExpress 第一次解壓位置。

首台與 EDR 試點新增正常成功、受控失敗、測試 VM 強制中止／重開機後的殘留驗收。由管理員使用新版 Inspect-ZabbixResidue.cmd（與同名 ps1 放一起）右鍵管理員執行；工具只列出目前帳號 TEMP 與 Windows TEMP 中符合本套件的 IXP 目錄、bootstrap.psk 是否存在及 ACL SDDL，不讀金鑰內容、不刪檔。其他部署帳號的 TEMP 需另行檢查。

確認無安裝程序仍運作後，管理員逐個確認本套件標記與檔案，再清除來源密鑰及暫存資料夾。禁止用萬用指令刪除全部 IXP*.TMP，禁止刪除 ProgramData\`__ORG_TAG__`\ZabbixAgent2\keys 正式運作金鑰。`__ORG_TAG__`-Package.json 是辨識標記，不是加密或存取控制。暫存檢查由管理端驗收處理，工程師日常仍只需複製 EXE、右鍵管理員執行。

### 二十八之二十一 Server 右鍵執行與 C D 槽路徑

工程師把新 EXE 放 C:\Temp 或 D:\Temp，右鍵管理員執行即可。解壓器先建立獨立 `__ORG_TAG__`-ZabbixDeploy-隨機識別碼子目錄並限制為 SYSTEM／Administrators，才複製含密鑰內容；安裝結束後嘗試清理，不放寬整個 C:\Temp 權限。

若公司規定部署子目錄必須位於 D:\Temp，只需管理員在 NB 編輯 Payload\Extract-And-Install.ps1 的以下一行，重新打包一次；工程師不逐台改參數。只把 EXE 搬到 D 槽不會修改這個設定。

```text
$destinationRoot = 'D:\Temp'
```

IExpress 初始暫存及正式 Program Files／ProgramData 仍可能使用 C 槽；改部署目錄不等於完全不使用 C 槽。若 C 槽權限政策禁止正式安裝位置，須先由管理員核對，不能只改 Temp 規避。

完成後查 ProgramData\`__ORG_TAG__`\ZabbixAgent2\logs 與前端 Latest data，刪除 Server 上原始含密鑰 EXE，保留正式 Agent keys。若部署失敗，重新執行原 EXE 會重新解壓來源；不要直接重跑已清除 bootstrap.psk 的暫存子目錄。

### 二十八之二十二 已發生的部署故障與修正

以下依失敗階段分類。先讀第一個明確錯誤，不要因後續警告顯示服務可能停止就直接重啟整台 Server。

| 階段與症狀 | 已知判斷 | 處理 |
| --- | --- | --- |
| PS 腳本被禁止載入 | 執行政策或下載封鎖 | 管理員視窗設定 Process Bypass、Unblock-File；GPO 限制另行核准 |
| 找不到 NetIPInterface 屬性 | 腳本讀取不適用的網卡屬性 | 使用修正版 Get-NetIPInterface，重新核對主機 IP |
| PSK 至少 32 碼或格式錯誤 | 輸入不是符合規範的十六進位字串 | 使用現有來源檔；不把 identity 當 PSK、不重新產生密鑰 |
| MSI 1603 | 通用失敗碼，不足以定位 | 讀日誌 Return value 3 前文 |
| The SERVERACTIVE value is missing | MSI Log 顯示 LaunchConditions 阻擋時 | MSI 安裝命令帶 SERVERACTIVE，不能只在安裝後寫設定 |
| IExpress exit 1 | 通用打包失敗 | 顯示錯誤對話框並讀 Build-EXE-last-error.txt |
| Error opening the IExpress Self Extraction Directive file | SED 開啟或讀取階段失敗 | 使用標準 Options／Strings SED；指定工作目錄，呼叫 /N package.sed |
| 無法建立 SSL/TLS 安全通道 | Server 2016 下載 MSI 時交握失敗 | 歷史線上流程；1.11 使用固定離線 MSI，不再下載 |

MSI 安裝修正後的必要參數包含下列內容。MSI 先保持不啟動，寫入與驗證 PSK 設定後才啟用 Agent 服務；密鑰值不放到 msiexec 命令列。

```text
STARTAGENTS=0
SERVERACTIVE="__ZABBIX_FQDN__:10051"
DONOTSTART=1
SKIP=fw
STARTUPTYPE=disabled
```

MSI 日誌擷取範例。更換檔名為該次實際日誌；不要只回報最後一行 1603。

```text
$logPath = 'C:\ProgramData\__ORG_TAG__\ZabbixAgent2\logs\msi-日期時間.log'
Select-String -LiteralPath $logPath -SimpleMatch 'Return value 3' -Context 35,5 |
    Select-Object -First 2 |
    Out-String -Width 240
```

若搜尋無結果，再讀取尾端 100 行。IExpress 診斷報告只應包含參數、版本與檔名，不含密鑰值；禁止上傳含 bootstrap.psk 的整包或產生的 EXE。

IExpress 故障排查使用完整受控版本，不混搭 SED 格式與啟動路徑。歷史打包成功範圍見附錄 B。

### 二十八之二十三 Server 2016 下載錯誤與離線診斷

舊版線上下載曾在 Windows Server 2016 PowerShell 5.1 因 TLS 協定失敗。此為歷史故障分類；1.11 Deploy 已移除線上下載分支，不再以設定 TLS 1.2 作為重試安裝方式。正式工程師使用內含核准 MSI 的新版 EXE。

手動診斷由管理員指定本機 MSI，雜湊與完整發行者驗證和 EXE 相同；下列 Agent2.msi 必須是 Builder 內核准的 7.0.32 檔案。簽章無效時檢查時間、根憑證、撤銷資訊及代理，不停用驗證。

```text
& 'C:\Temp\Deploy-ZabbixAgent2-AutoRegistration.ps1' -PskSourceFile 'C:\Temp\bootstrap.psk' -MsiPath 'C:\Temp\Agent2.msi'
```

### 二十八之二十四 安裝成功與納管完成的驗收

依本節分層驗收每台目標 Server；歷史安裝與封裝證據移至附錄 B，不以歷史成功取代本次版本的收數與通知驗收。

| 檢查層次 | 合格證據 |
| --- | --- |
| 安裝與本機設定 | MSI 0 或 3010；設定驗證 successful；agent.ping 回傳 1 |
| 常駐服務 | Zabbix Agent 2 為 Running；依核准標準設為延遲自動啟動 |
| 連線 | 端點可連 Server TCP 10051；主動模式端點未監聽 10050 |
| 自動註冊 | 等待約 2 至 5 分鐘後，Hosts 出現正確 FQDN；實際依探索與設定同步時間而異 |
| 群組範本與加密 | 加入指定群組、套 Windows by Zabbix agent active、PSK 身分與密鑰來源一致 |
| 持續收數 | Monitoring → Latest data 中 CPU、記憶體等有新數值，時間持續更新 |
| 告警與復原 | 測試環境或核准窗口停止並恢復 Agent，記錄告警與復原時間 |
| 推廣門檻 | 新 EXE 至少一台目標 Server 全流程通過，再分批部署 |

主動模式不以 TCP 10050 是否可連判定成功，也不要只憑 ZBX 可用性圖示下結論。Host 若為 Disabled，先確認是否刻意停用；在啟用前不作完整正式收數驗收。已有正常收數的主機不需要只為打包工具更新而重裝。

### 二十八之二十五 Google 服務告警與降噪

Windows 範本會探索服務，針對納入範圍且非 Running 的服務產生停止告警。例如版本化的 GoogleUpdaterService 與 GoogleUpdaterInternalService；服務名稱與啟動類型不足以證明更新功能正常。

若這兩類服務不屬於必須持續執行的服務，先從主機層級排除，避免為消除告警而改變 Windows 更新行為。到 Data collection → Hosts，直接點左側主機名稱，不是 Items；選 Macros → Inherited and host macros，找到以下巨集，使用 Change 或新增同名主機巨集覆寫：

```text
{$SERVICE.NAME.NOT_MATCHES}
```

保留繼承的完整原值，在末尾加上以下片段。若先前已加舊片段，替換該片段而不是清空整個原值。

```text
|^GoogleUpdater(?:Internal)?Service.*$
```

這段只涵蓋新版 GoogleUpdaterService 與 GoogleUpdaterInternalService 名稱系列，不會順便排除所有名稱含 Google 的服務，也不涵蓋舊 gupdate／gupdatem。若需納入其他例外，另行確認服務用途再調整。

按 Update 後等待下一次 Windows services discovery；既有資源會受探索規則的遺失資源停用／刪除設定影響，告警不保證立即消失。Active 型探索的立即執行支援受限，依實際介面與排程確認，不把手動 Execute now 當成必要步驟。不要移除並清空整個範本來消除單一服務告警，以免丟失歷史資料。

若決定直接停用 Google 更新服務，先確認 Chrome 或 Google 軟體已由其他機制更新並登錄責任人。停用更新服務可能影響安全更新；停用服務本身不保證既有停止告警即刻復原。

### 二十八之二十六 無法手動關閉問題的判斷

更新問題視窗中的 Close problem 灰色，不表示平台故障。先檢查該 Trigger 是否開啟 Allow manual close，以及使用者角色與 Host 的讀寫權限；已復原、正在關閉或選取多筆事件時，也要確認事件本身是否可操作。

Google 服務的停止告警來自低階探索的 Trigger prototype。若確有人工關閉需求，從 Data collection → Templates → Windows by Zabbix agent active → Discovery → Windows services discovery → Trigger prototypes，開啟 is not running 原型，檢查 Allow manual close。主機上已發現的繼承觸發器通常不能任意直接改這項屬性，應回到來源原型處理。

啟用來源原型後，等待探索同步，再檢查實際已發現的 Trigger 是否更新，才回到 Problems 操作。修改官方範本會影響全部繼承該原型的主機與服務；正式共用政策應評估複製為公司管理範本，不要因一筆 Google 告警就全面開放所有關鍵服務人工結案。

| 操作 | 維運意義 |
| --- | --- |
| Acknowledge | 記錄已接手，不代表故障恢復或事件關閉 |
| Suppress | 暫時抑制通知或展示，仍保留問題狀態 |
| Close problem | 在允許條件下結束該事件，不修復 Windows 服務 |
| 巨集排除 | 長期改變服務探索範圍；既有項目需等待遺失資源處理 |

若故障仍存在，手動關閉不等於根因已解決；是否再次產生事件仍取決於 Trigger 狀態、事件產生模式與後續恢復／再觸發，不把手動結案當成恢復驗收。長期忽略 Google 更新服務用巨集排除；需要持續運作的 AD、DB、備份、Hyper-V 等服務則保留監控與處理責任。

### 二十八之二十七 交付紀錄與版本驗證

維運交付須記錄管理 NB、打包日期、Payload 版本、EXE SHA256、Server／metadata／identity、密碼庫紀錄編號、目標主機名與 IP、安裝結果、Latest data 時間、告警測試與回復證據。文件和截圖不記錄 PSK 本文。

工具版本、歷史證據與尚待執行的驗收統一登錄附錄 B。新版離線 EXE 必須在一台目標 Server 完成真實打包、解壓、安裝、服務、註冊與 Latest data 驗收後，才分批推廣。

每次產出先核對下載、MSI 必填參數、主機辨識、PSK 來源與 ACL、設定檔驗證、常駐服務、失敗回復、自動註冊和 Latest data；檔案存在與 ZIP 可解壓是交付檢查，不等於使用者瀏覽器下載已被實測。不得把語法檢查、模擬測試或打包成功宣稱為 Windows 全流程實測。

### 二十八之二十八 批次結束與共享 PSK 退出

批次部署須先登錄目標資產清單、允許來源網段、操作人、啟用與關閉時間。只有部署窗口啟用對應 Autoregistration Action，metadata 條件不是安全認證；持有共享密鑰者可能註冊未核准主機或冒用使用相同金鑰的 Hostname。來源 ACL、資產比對與密鑰保管必須同時實施。

部署完成後先確認 Hosts／群組／模板／Latest data 與資產清單一致，再停用本批 Autoregistration Action，確認沒有其他寬鬆 Action 仍可接納未知端點。Zabbix 全域加密選項不是一般的自動註冊總開關；不要為關閉 PSK 註冊而勾選 No encryption，這會放寬未知主機連線。若介面無法清空所有允許模式，維持 PSK-only 配合 Action 停用與網路控制，不切回明文。

停用 Action 不會撤銷既有 Host 的共享 PSK，也不會阻止持鑰者冒用仍共用同一金鑰的既有 Host。若已有金鑰外洩疑慮，須處理整批已納管端點；只更換全域 bootstrap 值不夠。

目標 Server 驗收後必須刪除部署目錄的 bootstrap.psk、含金鑰 EXE、解壓副本及失敗殘留；不得刪除 Agent keys 目錄中的正式運作金鑰。管理 NB 的原始密鑰移入核准密碼保管位置並按保留政策管制。備份、EDR 隔離區及檔案版本也可能留有副本，按公司資料銷毀政策處理，刪除檔案不宣稱為安全抹除。

長期退出共享方式：先在測試 Host 產生唯一 identity 與 PSK，預備端點新密鑰檔並限制 ACL；於維護窗口停止 Agent、同步切換前端 Host Encryption 與 Agent 設定，驗證後啟動並確認資料。成功後逐批處理其餘主機，失敗則以前端與端點成對回復。全部完成後才退役舊共享密鑰與含密鑰套件。維運紀錄只填保管庫編號，不填秘密值。

## 二十八之二十九 逐台獨立 PSK 的進階納管

此為管理員手動納管及後續獨立密鑰路線，日常工程師仍使用二十八之五的共享 bootstrap EXE，不增加現場步驟。每台獨立 PSK 不能塞進所有主機共用的 EXE；如改用受控自動派送，應由管理端安全注入各主機金鑰。

在管理 NB 的管理員 Windows PowerShell 5.1，先建立該主機的專用目錄，再產生一次 32 bytes（64 hex）金鑰。不使用 -InitializePsk 覆寫共享 bootstrap.psk。下列 FQDN 是占位範例，須改成目標電腦實際名稱；PSK 值不輸出到畫面。

```text
$TargetFqdn = '__HOST_FQDN__'
$KeyDir = Join-Path $env:ProgramData ('__ORG_TAG__\PerHostKeys\' + $TargetFqdn)
New-Item -ItemType Directory -Path $KeyDir -Force | Out-Null
& icacls.exe $KeyDir /inheritance:r /grant:r '*S-1-5-18:(OI)(CI)F' '*S-1-5-32-544:(OI)(CI)F'
if ($LASTEXITCODE -ne 0) { throw 'ACL failed' }
$KeyFile = Join-Path $KeyDir 'agent.psk'
if (Test-Path $KeyFile) { throw 'Key exists; do not overwrite' }
$Bytes = New-Object byte[] 32
$Rng = [Security.Cryptography.RandomNumberGenerator]::Create()
try { $Rng.GetBytes($Bytes) } finally { $Rng.Dispose() }
$Hex = -join ($Bytes | ForEach-Object { $_.ToString('x2') })
[IO.File]::WriteAllText($KeyFile, $Hex, [Text.Encoding]::ASCII)
Set-Clipboard -Value $Hex
Remove-Variable Hex, Bytes
```

將剪貼簿值存入密碼庫，並在 Data collection → Hosts 預建精確 FQDN Host，連結 Windows by Zabbix agent active、核准群組。Encryption 的 Connections from host 僅 PSK，Identity 設 `__HOST_PSK_IDENTITY__`，PSK 填該值。Connections to host 依是否採被動檢查設定；本流程仍為 Active、無 10050。確認沒有其他自動註冊 Action 會匹配手動主機。

經受控方式將專屬金鑰與原部署腳本、固定 MSI 放到該目標的 C:\Temp，限制 ACL。管理員使用既有部署腳本的參數，不修改部署核心；HostMetadata 改 `__MANUAL_METADATA__`，避免包含 `__WINDOWS_METADATA__` 而誤觸批次 Action。先驗證 MSI 的核准雜湊與簽章。

```text
$Msi = 'C:\Temp\zabbix_agent2-7.0.32-windows-amd64-openssl.msi'
$Expected = 'B91069F7372F05F704CC31C021561F4E9558EBE2C45D7956A9C841221C357ED6'
if ((Get-FileHash $Msi -Algorithm SHA256).Hash -ne $Expected) { throw 'MSI hash mismatch' }
Get-AuthenticodeSignature $Msi | Format-List Status,SignerCertificate
# 必須確認 Status=Valid 且簽署者為核准 Zabbix 後才繼續
& 'C:\Temp\Deploy-ZabbixAgent2-AutoRegistration.ps1' `
  -ZabbixServer '__ZABBIX_FQDN__' `
  -HostnameOverride '__HOST_FQDN__' `
  -HostMetadata '__MANUAL_METADATA__' `
  -PskIdentity '__HOST_PSK_IDENTITY__' `
  -PskSourceFile 'C:\Temp\agent.psk' `
  -MsiPath $Msi
```

參數請整段執行，反引號後不可留空格。既有腳本版本須支援這些參數；先在管理端 Get-Help 或檢查 param 區，不要拿舊版猜測。完成 Running／Auto、agent.ping、Latest data、TLS 與重開機測試，再清除 C:\Temp 的來源 PSK 與安裝包；保留正式 Agent 使用的 PSK 路徑及密碼庫紀錄。換鑰須兩端協調、保留回復窗口，不直接刪正式密鑰。

## 二十九 告警通知與升級流程

告警鏈必須依序驗證 Media type、User media、Trigger Action、實際收件與 Recovery。Media type 的 Test 成功不代表完整通知鏈已通過。使用公司核准 SMTP relay，第一階段不假設 Microsoft 365 帳密式 SMTP AUTH 可用；若公司政策不允許，改用核准 relay 或另立整合方案。

### SMTP 與 User media

Alerts → Media types → Email，選 Generic SMTP。依郵件管理員提供的 relay 填 SMTP server、Port、Sender／Email、HELO 與認證。例：`__SMTP_HOST__`、587、STARTTLS；這只是欄位範例，不代表現有 Exchange 已設定。開啟 SSL verify peer 與 SSL verify host，信任公司 CA，不停用驗證。若採來源 IP 授權 relay，限制只允許監控主機與核准寄件者，禁止開放式 relay。

Message templates 必須包含 Problem、Problem recovery 與 Problem update。使用 Media type 的 Test 寄到測試維運信箱，保留時間與 Message ID。再到 Users → Users → 指定使用者 → Media 新增 Email，填 Send to、When active 為 1-7,00:00-24:00、Enabled，以及核准的 Severity。使用者必須具備目標 Host 的至少唯讀權限；測試不能只靠 Super admin 帳號。

### Trigger Action 與分級

Alerts → Actions → Trigger actions → Create action，名稱 Production monitoring notifications。條件限定正式 Host groups 或已核准的 env=production 標籤，嚴重度至少 Warning；另排除測試與維護設備。Action 啟用前確認不會回頭大量通知既有無人處理事件。

| 通知層級 | 範例策略 | 設定 |
| --- | --- | --- |
| 初次通知 | Warning 以上立即通知值班 | Operations 步驟 1，Send message 至值班群組，媒介限定 Email |
| 持續未處理 | 15 分鐘後仍未確認通知第二線 | Default operation step duration 15m，步驟 2，條件 Event acknowledged = No |
| 重大故障 | High／Disaster 另通知系統負責人 | 分開建立精確嚴重度 Action，避免與初次規則重複收件 |
| 恢復 | 通知已收到問題訊息的人員 | Recovery operations，Notify all involved 或指定相同責任群組 |
| 更新 | 記錄確認與處置訊息 | Update operations 按責任需求設定，避免每次更新群發 |

Action 先只選一台測試 Host 演練，驗證步驟 1、15 分鐘升級、確認後不再升級，以及恢復通知。不需要的重複寄送不設無限步驟。維護與 Suppress 的暫停行為按實際 Action 設定驗證，不假設全部情境自動停止發信。

### 安全的端到端通知測試

在專用測試 Host 建立 Zabbix trapper item，key 為 test.notify，Numeric unsigned；建立 Trigger 表達式 last(/TEST-NOTIFY/test.notify)=1，Severity Warning。若該版本提供 Allowed hosts，限制為 127.0.0.1 或指定測試來源。不要在正式業務主機停止關鍵服務來測通知。

```text
dnf -y install zabbix-sender
zabbix_sender -z 127.0.0.1 -s TEST-NOTIFY -k test.notify -o 0
zabbix_sender -z 127.0.0.1 -s TEST-NOTIFY -k test.notify -o 1
# 確認問題、收件與升級後送出恢復值
zabbix_sender -z 127.0.0.1 -s TEST-NOTIFY -k test.notify -o 0
```

保留 Event ID、Action log、實際收件、升級時間與 Recovery 時間。故障優先查 Alerts → Action log，以及 User media 時段／Severity、Host 權限、Action 條件與 SMTP relay log。完成後停用測試 Action 或移除測試標籤，保留測試紀錄。

## 三十 平台自我監控與第二通道

先在 `__ZABBIX_FQDN__` Host 確認 Linux by Zabbix agent 與 Zabbix server health 連結、Enabled 且 Latest data 有新值。監看官方 internal items 的 Queue、NVPS、Housekeeper／poller busy、preprocessing、cache，以及 OS 的 /、/var/lib/mysql、/backup。磁碟起始 80% 警告、90% 高嚴重度，另依日增量設定剩餘天數與絕對容量，不能等寫滿才通知。

### 備份 Items 與 Trigger

安裝第二十二章腳本後，在平台 Host 建立下列 Items：Type=Zabbix agent（沿用本機被動介面 127.0.0.1），Information type=Numeric unsigned，Update interval=1m，History=7d，Trends=90d。檔案須 root:zabbix 640，目錄 750；不要為讀指標而放寬 PSK／備份目錄。

| Item key | 意義 |
| --- | --- |
| vfs.file.contents[/var/lib/zabbix/backup-status/local_snapshot_epoch] | 本機可用副本的快照下限時間 |
| vfs.file.contents[/var/lib/zabbix/backup-status/offsite_snapshot_epoch] | 外部已校驗副本的最新快照下限 |
| vfs.file.contents[/var/lib/zabbix/backup-status/local_result] | 本機工作結果，0 成功、1 失敗 |
| vfs.file.contents[/var/lib/zabbix/backup-status/upload_result] | 外送工作結果，0 成功、1 失敗 |
| vfs.file.contents[/var/lib/zabbix/backup-status/retention_result] | 保留清理結果，0 成功、1 失敗 |
| vfs.file.contents[/var/lib/zabbix/backup-status/pending_count] | 等待外送的副本數 |
| vfs.file.contents[/var/lib/zabbix/backup-status/unshipped_pruned_total] | 因本機政策刪除未外送舊副本的累計 |
| vfs.file.contents[/var/lib/zabbix/backup-status/unshipped_pruned_epoch] | 最近一次上述降級的時間 |

以 Expression constructor 選對 Host 與 Items，建立下列條件。所有 Items 另設 nodata(...,10m)=1 並連到第二十九章通知 Action。初始化 0 表示尚無成功備份，不是正常。

```text
now()-last(/__ZABBIX_FQDN__/vfs.file.contents[/var/lib/zabbix/backup-status/local_snapshot_epoch])>28800
now()-last(/__ZABBIX_FQDN__/vfs.file.contents[/var/lib/zabbix/backup-status/offsite_snapshot_epoch])>28800
last(/__ZABBIX_FQDN__/vfs.file.contents[/var/lib/zabbix/backup-status/local_result])<>0
last(/__ZABBIX_FQDN__/vfs.file.contents[/var/lib/zabbix/backup-status/upload_result])<>0
last(/__ZABBIX_FQDN__/vfs.file.contents[/var/lib/zabbix/backup-status/retention_result])<>0
nodata(/__ZABBIX_FQDN__/vfs.file.contents[/var/lib/zabbix/backup-status/local_snapshot_epoch],10m)=1
now()-last(/__ZABBIX_FQDN__/vfs.file.contents[/var/lib/zabbix/backup-status/unshipped_pruned_epoch])<86400
```

最後一條在最近 24 小時內刪除未外送副本時告警，須保留累計與變更紀錄；不是只清告警就算處理。pending_count 超過日常基線時通知備份管理員，快照逾 8 小時提高嚴重度。恢復外送後看 offsite_snapshot_epoch，不能看完成上傳時間冒充 RPO 恢復。

### 平台處理心跳

在平台 Host 建立 Type=Zabbix trapper、Key=monitor.heartbeat、Numeric unsigned、History=1d、Trends=Do not keep；Allowed hosts 僅 127.0.0.1。本機 trapper 測試使用 loopback 明文且來源受限，不修改 Windows PSK。若平台 Host 另設只收 TLS，應改用核准的本機 TLS sender 設定並同步腳本，不能關掉其他端點加密。

平台安裝器已放置 heartbeat service／timer，但未啟用。先測一次，確認 processed: 1、failed: 0，Latest data 時間與值均更新。

```text
systemctl start zabbix-heartbeat.service
journalctl -u zabbix-heartbeat -n 30 --no-pager
systemctl enable --now zabbix-heartbeat.timer
systemctl list-timers zabbix-heartbeat.timer
```

### 獨立主機的第二通道

在另一主機、電源與網路故障範圍執行看門狗；不能裝在同一 Zabbix VM 再稱外部監控。此主機須能驗證 HTTPS 公司 CA、連線 TCP 10051，並有獨立 SMTP／通知路徑。本套件以 sendmail 介面提交本機 MTA，須由郵件管理員設定 relay 並驗證實際收件。

在 Zabbix 建唯讀監控使用者，僅可讀平台 Host；角色 API 方法只允許 host.get、item.get、history.get，建立有期限的 API Token。於外部主機用 root 編輯 /root/.zabbix-watch.token（不要把 Token 寫在命令列），owner root、600。API Token 不授權改設定或操作主機。

```text
cd /root/Zabbix-SOP-1.11-Operations
bash install-linux.sh watchdog
install -m 600 /dev/null /root/.zabbix-watch.token
# 用 sudoedit 寫入 API Token；編輯 JSON 的 CA、收件者與實際 URL
sudoedit /root/.zabbix-watch.token /etc/zabbix-watch.json
python3 /usr/local/lib/__ORG_TAG__-zabbix/monitor_watch.py --test-notification
systemctl start zabbix-watch.service
journalctl -u zabbix-watch -n 30 --no-pager
systemctl enable --now zabbix-watch.timer
```

JSON 預設 host_name=`__ZABBIX_FQDN__`、API=https://`__ZABBIX_FQDN__`/api_jsonrpc.php、心跳最大年齡 180 秒、連續失敗 3 次通知。CA 未受系統信任時設定 ca_file，不可略過憑證驗證。心跳同時檢查最後 history clock 及 epoch 值，未取得資料、舊資料、時間超前、API／TCP 失敗都算失敗；通知失敗不標記已通知，下次重試。持續故障每小時重送，復原發一封恢復訊息。

告警時間包含 180 秒心跳過期門檻及三次檢查，約數分鐘，不能保證停機三分鐘內一定收信；記錄實測最大時間。/var/lib/zabbix-watch/state.json 與 Journal 是外部證據。測試 Zabbix VM 斷電、停用 heartbeat、API Token 失效、10051 拒絕及 SMTP 失敗，確認真正收件與復原；測試 SMTP 失敗時須有郵件平台自身告警，不能由同一壞掉的信道自我通報。

### 1.11 新增的監控項目與看門狗前置條件

外部看門狗啟用前先完成第十五章固定來源規則與中間網路 ACL。從看門狗主機驗證公司 CA、443、10051，再 start zabbix-watch.service 驗證真正 API 讀回；firewalld reload／重開機後複驗。若現有 `__MGMT_CIDR__`／`__AGENT_CIDR__` 已允許該來源，保留查核證據即可，不開放額外全網規則。

在原備份狀態 Items 之外新增兩個相同 Type／Numeric unsigned／1m 的 Items，History=7d、Trends=90d；各自另設 nodata 10m。invalid_backup_count>0 連到第二十九章通知 Action，不因 local_result=0 而忽略。

```text
vfs.file.contents[/var/lib/zabbix/backup-status/invalid_backup_count]
vfs.file.contents[/var/lib/zabbix/backup-status/invalid_backup_epoch]
```

```text
last(/__ZABBIX_FQDN__/vfs.file.contents[/var/lib/zabbix/backup-status/invalid_backup_count])>0
nodata(/__ZABBIX_FQDN__/vfs.file.contents[/var/lib/zabbix/backup-status/invalid_backup_count],10m)=1
```

invalid_backup_epoch 是異常集合變更的稽核時間，0 表示尚未偵測到；是否仍有異常以 count 判定。詳細名單只由 root 讀 invalid-backups.json，不將含密備份內容開放給 Agent。retention_result 非 0 即使 local_result=0 仍需告警。

## 三十一 修補升級與回復

### 升級前停止點

每次分開變更 OS、MySQL 8.4 小版與 Zabbix 7.0 小版，先閱讀各版本 Release notes 與相容矩陣。不得把 OS major、MySQL major、Zabbix major 放同一張一般修補單。保存 rpm -qa、dnf history、Repo 設定、設定檔與已驗證外部 DB 備份；先完成隔離複製環境升級與回復演練。

升級次序依目標版本相容矩陣決定；本版預設拆成三個窗口：OS 修補並重開機驗證、MySQL 8.4 小版、Zabbix 7.0 小版。若 Release notes 要求不同順序，以經審核的依賴關係為準，不宣稱固定順序適用所有版本。每個窗口驗收通過後再進下一個。

### OS 與 MySQL 小版

先 dnf check-update 與 dnf upgrade --assumeno 檢查交易；排除尚未核准的應用套件，按公司變更單更新。保留可開機舊核心與 Console，重開機後確認磁碟掛載、chrony、SELinux、SSH、DB 與 Web。正式主機不無條件重跑第八章的 dnf -y update。

MySQL 變更先暫停 Zabbix 寫入與 Web 管理，確認沒有正在備份或 DDL；套用已核准的 8.4 目標套件，啟動 mysqld 並完成版本、表格、錯誤 Log、綁定與登入檢查，再啟動 Zabbix 與 Web。必須把停止與重新啟動服務列入變更單，不留下未恢復的狀態。

### Zabbix 7.0 小版

進入 Maintenance 並由第二通道接手通知，備份後停止 zabbix-server 與 Web，核對 Server、Frontend、SQL scripts 的目標版本與套件交易，避免 EPEL 混裝。更新後啟動 Server，持續讀取 DB 升級 Log，不中斷正在執行的 Schema patch；完成才恢復 Web 與 Agent。核對 .rpmnew／.rpmsave，不直接覆蓋自訂參數。最後測 HTTPS、Latest data、通知、語系、備份與自我監控。

若 Schema 已升級，不以降版 RPM 或 dnf history undo 當成資料庫回復。回復需使用成套的舊版本套件、升級前 DB 與對應設定；MySQL 資料目錄也不能直接交給舊版 binary 啟動。Hyper-V Checkpoint 只能在確認 OS 與 DB 磁碟一致、支援且短期的條件下使用，不能取代已驗證外部備份。

## 附錄 A 中英文語系設定與安全排錯

正式環境預設使用简体中文（zh_CN），並保留 English（en_GB）與 English（en_US）供個別使用者選擇。zh_TW 維持選用，不作為正式環境預設語言。以下流程依序區分語言呈灰色、介面未翻譯、項目消失與 PHP 語法錯誤。

語言存在但呈灰色時，先處理作業系統 locale。只有 locale 與 PHP 測試正常、且確認 display 被停用時，才精準修改既有語言項目。修改前必須備份，php -l 成功後才能重啟服務；套件更新後重新檢查，不得用舊版備份直接覆蓋新版檔案。English 是前端基礎語言，通常不需要英文 frontend.mo 翻譯檔。

### A1 確認系統 locale、PHP gettext 與中文翻譯檔

```text
locale -a | grep -Ei '^(en_(GB|US)|zh_CN)(\.|$)'

php -r 'echo extension_loaded("gettext") ? "gettext=yes\n" : "gettext=no\n";'

rpm -ql zabbix-web | \
  grep '/locale/zh_CN/LC_MESSAGES/frontend.mo'
```

成功標準：

- locale -a 至少出現 zh_CN.utf8，並包含 en_US.utf8 與 en_GB.utf8。

- PHP 輸出 gettext=yes。

- /usr/share/zabbix/locale/zh_CN/LC_MESSAGES/frontend.mo 存在。

- English 是前端基礎語言，沒有 English frontend.mo 不代表英文語系缺失。

如果英文或中文 locale 缺少，再安裝對應語系套件：

```text
dnf -y install glibc-langpack-en glibc-langpack-zh
locale -a | grep -Ei '^(en_(GB|US)|zh_CN)(\.|$)'
```

### A2 判讀語言選項狀態

| 畫面狀態 | 主要原因 | 正確第一步 |
| --- | --- | --- |
| 語言存在但呈灰色 | Web Server 缺少該語言的系統 locale | 安裝對應的 glibc-langpack-*，不要修改 PHP 檔 |
| 語言可選但介面未翻譯 | gettext、翻譯檔或服務快取問題 | 檢查 gettext、frontend.mo，重啟 PHP-FPM/Nginx |
| 語言項目完全不存在 | 套件版本、display 設定或檔案受損 | 檢查套件與 locales.inc.php，但先不要直接修改 |

若 English 項目存在但灰色，且 locale -a 只有中文語系，先補英文系統 locale，不修改 PHP 語系清單。

### A3 安裝英文 locale 並驗證

先確認現況：

```text
rpm -q glibc-langpack-en || true
locale -a | grep -Ei '^en_(GB|US)(\.|$)' || true
```

若 en_GB 或 en_US 任一系統 locale 缺少，安裝英文語系包：

```text
dnf -y install glibc-langpack-en

rpm -q glibc-langpack-en
locale -a | grep -Ei '^en_(GB|US)(\.|$)'
```

再由 PHP 驗證是否能設定英文 locale：

```text
php -r '
$ok = true;
foreach (["en_GB", "en_US"] as $lang) {
    $result = setlocale(LC_MESSAGES, $lang . ".UTF-8", $lang . ".utf8");
    echo $lang, "=";
    var_dump($result);
    $ok = $ok && ($result !== false);
}
exit($ok ? 0 : 1);
'
```

成功標準：en_GB 與 en_US 各自輸出有效 locale 字串；兩者均不得為 bool(false)，命令結束碼應為 0。

### A4 重新啟動並驗證 Web 服務

```text
systemctl restart php-fpm nginx

systemctl is-active php-fpm nginx
nginx -t
php-fpm -t

curl -sS -o /dev/null -w 'HTTP_STATUS=%{http_code}\n' \
  http://127.0.0.1/
```

瀏覽器登出 Zabbix 後重新登入，並使用 Ctrl+F5 強制重新整理，再到 User settings → Profile → Language 分別測試 English（en_GB）與 English（en_US）。

成功標準：

- php-fpm 與 nginx 均為 active。

- Nginx 與 PHP-FPM 設定測試成功。

- HTTP 回應為 200 或正常導向的 301/302。

- English (en_GB) 與 English (en_US) 均不再呈灰色且可儲存。

### A5 英文仍呈灰色時的進階檢查

先確認 PHP-FPM 實際服務帳號，再用該帳號測試 locale：

```text
grep -RHE '^[[:space:]]*user[[:space:]]*=' /etc/php-fpm.d/*.conf
ps -eo user,comm | awk '$2 ~ /^php-fpm/ {print}'
```

假設上一步顯示服務帳號為 apache，執行：

```text
runuser -u apache -- php -r '
$ok = true;
foreach (["en_GB", "en_US"] as $lang) {
    $result = setlocale(LC_MESSAGES, $lang . ".UTF-8", $lang . ".utf8");
    echo $lang, "=";
    var_dump($result);
    $ok = $ok && ($result !== false);
}
exit($ok ? 0 : 1);
'
```

如果實際帳號不是 apache，必須以查到的帳號取代，不可直接照抄。

若服務帳號測試成功但前端仍為灰色，再執行：

```text
LOCALE_FILE=$(rpm -ql zabbix-web | \
  awk '/\/include\/locales\.inc\.php$/ {print; exit}')

test -n "$LOCALE_FILE" && test -f "$LOCALE_FILE" || {
  echo 'ERROR: locales.inc.php not found'
  exit 1
}

echo "LOCALE_FILE=$LOCALE_FILE"
grep -nE "'(en_GB|en_US|zh_CN|zh_TW)'" "$LOCALE_FILE"
php -l "$LOCALE_FILE"
journalctl -u php-fpm -u nginx -n 100 --no-pager
```

此階段先蒐集結果，不得直接執行 sed。若 English 列存在且 display => true，問題仍在系統 locale 或 PHP-FPM 執行環境；修改 PHP 檔不會解決。

### A6 語言項目缺失或停用時的處理

必須先確認以下條件全部成立：

- locale -a 已有要保留的 en_GB 與 en_US。

- PHP 與 PHP-FPM 服務帳號的 setlocale() 均成功。

- English 項目完全不存在，或明確顯示 display => false。

符合以上條件後才建立備份：

```text
BACKUP="${LOCALE_FILE}.pre-v1.7.$(date +%Y%m%d-%H%M%S)"
cp -a "$LOCALE_FILE" "$BACKUP"
echo "BACKUP=$BACKUP"
```

只針對已存在但為 false 的項目進行精準修改；若項目完全不存在，不得自行新增，應先重裝與目前 Zabbix 版本相符的 zabbix-web 套件或送交變更審查。

```text
sed -i -E \
  "/'(en_GB|en_US)'/ s/'display'[[:space:]]*=>[[:space:]]*false/'display' => true/" \
  "$LOCALE_FILE"

restorecon -v "$LOCALE_FILE"
diff -u "$BACKUP" "$LOCALE_FILE" || true
```

差異只允許 en_GB 或 en_US 的 display 從 false 變為 true。接著執行 PHP 語法驗證與自動回復：

```text
if ! php -l "$LOCALE_FILE"; then
  echo 'ERROR: PHP syntax check failed; restoring backup'
  cp -a "$BACKUP" "$LOCALE_FILE"
  restorecon -v "$LOCALE_FILE"
  php -l "$LOCALE_FILE"
  exit 1
fi
```

只有看到以下訊息才能繼續：

```text
No syntax errors detected
```

### A7 修改 PHP 檔後重新啟動並驗證 Web 服務

```text
systemctl restart php-fpm nginx

systemctl is-active php-fpm nginx
nginx -t
php-fpm -t

curl -sS -o /dev/null -w 'HTTP_STATUS=%{http_code}\n' \
  http://127.0.0.1/

journalctl -u php-fpm -u nginx -n 100 --no-pager
```

成功標準：

- php-fpm 與 nginx 均為 active。

- nginx -t 與 php-fpm -t 均成功。

- 本機 HTTP 回應為 200 或正常導向的 301/302。

- Log 沒有 PHP Parse error 或 Nginx upstream error。

### A8 Zabbix 前端設定

使用具有 Super admin 權限的帳號登入：

1. 進入 Administration → General → GUI。

2. 將 Default language 設為 Chinese (zh_CN)。

3. 儲存設定。

4. 個別使用者若需要英文，進入 User settings → Profile，將 Language 改為：

- English (en_GB)；或

- English (en_US)。

5. 若個別使用者要跟隨全域中文設定，Language 選 System default。

注意：選擇 English (en_US) 會在前端啟用美式日期與時間格式。請依企業日期與時間顯示標準選擇 en_US 或 en_GB。

### A9 驗收項目

| 驗收項目 | 合格標準 |
| --- | --- |
| 系統預設語言 | Administration → General → GUI 顯示 Chinese (zh_CN) |
| 英文保留 | User profile 可選 English (en_GB) 及 English (en_US) |
| 英文系統 locale | locale -a 同時有 en_GB 與 en_US，兩者的 PHP setlocale() 均不為 false |
| 中文可用 | zh_CN 可選且主要頁面顯示中文 |
| PHP 語法 | php -l locales.inc.php 無錯誤 |
| Web 服務 | PHP-FPM、Nginx active，HTTP 200/301/302 |
| 使用者隔離 | 一名使用者切換英文不影響其他使用者的中文設定 |
| 更新後複查 | Zabbix Web 套件更新後重新執行 grep、php -l 與前端語系檢查 |

### A10 故障回復

若修改後出現 HTTP 500、PHP Parse error 或語系下拉選單異常：

```text
test -n "${BACKUP:-}" && test -f "$BACKUP" || {
  echo 'STOP: select the correct pre-change backup first'
  exit 1
}
cp -a "$BACKUP" "$LOCALE_FILE"
restorecon -v "$LOCALE_FILE"
if ! php -l "$LOCALE_FILE"; then
  echo 'STOP: restored file failed PHP syntax check'
  exit 1
fi
systemctl restart php-fpm nginx
systemctl is-active php-fpm nginx
```

如果原 Shell 已結束而 BACKUP 變數不存在，先列出備份：

```text
ls -1t /usr/share/zabbix/include/locales.inc.php.pre-v1.7.*
```

選擇本次變更前的正確備份後再還原，不得直接使用很久以前版本的 locales.inc.php 覆蓋目前套件版本。

### A11 最可能失敗的三個原因

1. AlmaLinux 最小化安裝只有中文 locale，缺少 glibc-langpack-en，因此 English 存在但呈灰色。

2. 安裝語系包後未重新啟動 PHP-FPM/Nginx，或瀏覽器仍使用舊快取。

3. 在 locale 問題尚未排除前修改 locales.inc.php，不但無法解決灰色選項，還可能誤刪 PHPDoc、逗號或括號而造成 HTTP 500。

### A12 選用繁體中文與翻譯完整度驗證

正式建置主流程使用简体中文（zh_CN）。Zabbix 7.0 套件雖包含 zh_TW 的 frontend.mo 與系統語系檔，但 locales.inc.php 預設將 zh_TW 設為不顯示，且部分訊息沒有繁體翻譯，缺少的字串會回退為英文。以下步驟只供選用與故障排除，不是正式安裝的必要條件。

#### 確認系統語系與翻譯檔

```text
locale -a | grep -Ei 'zh_(TW|CN)'
rpm -ql zabbix-web | grep -E '/locale/zh_(TW|CN)/LC_MESSAGES/frontend.mo'
```

成功判定：locale -a 至少包含 zh_CN.utf8；若要測試繁體，還需包含 zh_TW.utf8，且對應 frontend.mo 必須存在。檔案存在只代表套件提供翻譯目錄，不代表所有介面字串都已完成翻譯。

如需選用 zh_TW，先完成 A1 的 glibc-langpack-zh 與 gettext 檢查，再依 A5 取得 LOCALE_FILE。執行以下單一變更流程，不要手動刪除註解或其他語言項目。

```text
BACKUP="${LOCALE_FILE}.pre-v1.7.$(date +%Y%m%d-%H%M%S)"
cp -a "$LOCALE_FILE" "$BACKUP"
grep -n "'zh_TW'" "$LOCALE_FILE"
sed -i -E "/'zh_TW'/ s/'display'[[:space:]]*=>[[:space:]]*false/'display' => true/" "$LOCALE_FILE"
restorecon -v "$LOCALE_FILE"
diff -u "$BACKUP" "$LOCALE_FILE" || true
if ! php -l "$LOCALE_FILE"; then
  cp -a "$BACKUP" "$LOCALE_FILE"
  restorecon -v "$LOCALE_FILE"
  php -l "$LOCALE_FILE"
  exit 1
fi
```

只有 php -l 顯示 No syntax errors detected 才能重新啟動服務。啟用後若部分畫面仍是英文，通常是 zh_TW 翻譯項目未完成，不代表 PHP、locale 或 frontend.mo 安裝失敗。套件更新可能還原 locales.inc.php，升級後應重新檢查，但不得直接覆蓋新版套件檔。

語法檢查成功後，依 A7 重啟並驗證服務，再執行以下 gettext 測試。

```text
php -r '
echo "gettext=", extension_loaded("gettext") ? "yes\n" : "no\n";
var_dump(setlocale(LC_MESSAGES, "zh_TW.UTF-8", "zh_TW.utf8", "zh_TW"));
bindtextdomain("frontend", "/usr/share/zabbix/locale");
bind_textdomain_codeset("frontend", "UTF-8");
textdomain("frontend");
echo gettext("Welcome"), PHP_EOL;
'
```

若輸出 gettext=yes、setlocale 回傳 zh_TW.UTF-8，但 Welcome 仍維持英文，代表 PHP gettext 與系統語系均正常，而該字串沒有可用的繁體翻譯。此時不應重裝 PHP 或關閉 SELinux；應接受部分英文、改用 zh_CN，或等待官方補齊翻譯。

### A13 HTTP 500 與 PHP 註解誤刪案例

若修改 locales.inc.php 後出現 HTTP 500，先處理 PHP 語法，不要先修改 Nginx、Firewall 或 SELinux。php-fpm -t 只驗證 PHP-FPM 設定，不會替所有 Zabbix PHP 檔案做語法檢查，因此必須另外執行 php -l。

```text
systemctl is-active php-fpm nginx
nginx -t
php-fpm -t
php -l "$LOCALE_FILE"
nl -ba "$LOCALE_FILE" | sed -n '58,72p'
tail -n 100 /var/log/nginx/error.log
journalctl -u php-fpm -n 100 --no-pager
```

若出現 unexpected token "*" expecting end of file，可能原因是語系清單後方 PHPDoc 註解的開頭 /** 被誤刪，使下一行的星號被當成 PHP 程式碼。不要只依錯誤行號猜測並刪除內容；應優先使用修改前備份還原，再精準重新套用 zh_TW 的 display 變更。

先依 A10 還原當次變更前備份並執行 php -l；確認網站恢復後，才視需求重新套用 A6 或 A12 的單一語系變更。如果只是要保留英文，無須重新啟用 zh_TW。

### A14 還原語系檔與套件更新後複查

需要取消本次語系檔修改時，使用 A10 的當次備份還原。若備份早於 Zabbix 套件更新，先比對版本與內容，不得用舊檔覆蓋新版。完成後執行以下檢查，並重新驗證 zh_CN、en_GB、en_US 的前端選項。

```text
php -l "$LOCALE_FILE"
rpm -V zabbix-web
grep -nE "'(en_GB|en_US|zh_CN|zh_TW)'" "$LOCALE_FILE"
locale -a | grep -Ei '^(en_(GB|US)|zh_(CN|TW))(\.|$)'
```

完成還原後改選简体中文（zh_CN）。rpm -V zabbix-web 若沒有輸出，代表套件管理的檔案已恢復原始內容；若仍有輸出，先比對檔案，不要直接重裝或覆寫其他已驗證設定。

## 附錄 B 審查處理與實機驗收紀錄

| 審查項目 | 本版處置 | 實機證據或責任 |
| --- | --- | --- |
| 真實密碼 | 本分享版已移除；執行環境另由部署者安全提供 | 管理、應用與備份帳號使用獨立密碼並驗收 |
| ALTER 在 CREATE 前 | 刪除錯序重複 ALTER；CREATE 改用自行提供的獨立帳號與密碼 | 既有 DB 不重跑 CREATE |
| MySQL X Protocol | 補 mysqlx-bind-address=127.0.0.1，與驗證一致 | 實機 SHOW VARIABLES 與 ss |
| 備份 | 獨立掛載、6 小時 timer、校驗、外送、輪替與失敗通知 | 單次／排程／外送中斷演練 |
| 還原 | 改獨立隔離 VM；不停止正式 Server；補啟動與清理 | 完整 RTO／RPO 紀錄 |
| HTTPS | Web Setup 前建 TLS，補 80 轉 443 與 CA 驗證 | 首次登入不得走明文網路 |
| 告警 | SMTP、User media、Action、升級、Recovery | 完整收件鏈測試 |
| PSK | 部署窗口、清理來源、逐台換鑰路線 | Action 停用不等於共享風險消除 |
| 安裝來源 | 固定 Repo RPM；離線 MSI 雜湊比對流程 | 既有 ZIP／EXE 未自動更新 |
| 帳號與維運 | SSH、Guest／Role、LDAP／TOTP、容量、自我監控與修補 | 分項變更單驗收 |
| Oracle／AD 深度監控 | 保留下一階段範圍，不宣稱已完成 | 另行確認版本、授權、指標與驗收 |

### 歷史驗證範圍

部署單位須自行建立代表性 Windows Server 的安裝、重開機、EDR、TLS、服務狀態與自動納管驗收紀錄。其他環境的歷史成功結果不能代替本環境驗收；原始 Log 應保存於本單位受控變更單，分享前再次去識別。

歷史結果不能代表所有 Windows 版本相容，也不能證明新版離線流程或新增備份已通過。含 TLS 修正的 EXE、D:\Temp 路徑、離線 MSI 與 SHA256 阻擋、備份 timer／外送／通知、隔離還原、LDAP／TOTP 及外部心跳，均須個別登錄測試日期、版本、執行人、預期與實際結果、證據位置及審核人。

### 1.10 驗證範圍

2026-11-07 已執行維運腳本隔離模擬測試，結果記錄於腳本包 VALIDATION.md。測試不連正式 MySQL／SSH，不寄信、不安裝 Agent。語法與故障模擬已驗證；AlmaLinux 10.2 的 systemd／SELinux、實際 MySQL 一致性與完整還原、SMTP、遠端權限、EDR 及 Windows 首台仍待對應實機驗收。原 1.9 Windows 歷史成功紀錄保留，不能擴張為新版平台腳本已通過。

| 審查議題 | 1.10 定案 | 實機停止點 |
| --- | --- | --- |
| 外送連鎖失效 | local／upload／retention 分離、有限保留、最新兩份與 pin 保護 | 三個 timer、容量壓力、外送恢復及通知 |
| RTO | 4 小時為待驗證目標；分階段測量與並行／物理備份評估 | 完整隔離還原達標才簽署 |
| Housekeeper | 保留起始方案，按 busy／I/O／過期資料與容量啟動評估 | 不未經試驗導入分區或關閉清理 |
| IExpress | 保留快速部署，管理端先完成實際 EDR 試點 | 禁止廣泛放行或關防護 |
| DB socket 權限 | 新建採 loopback TCP，既有平台窗口遷移 | 帳號、Web、SELinux 通過後才收舊權限 |
| Cisco | 依官方 YAML 補 MIB 對應、完整授權與 N/A 證據 | 選用型號／韌體逐項驗收 |
| 手動逐台 PSK | 補完整進階流程，工程師共享 EXE 操作不變 | 實機獨立密鑰試點 |
| VM／自我監控 | 加備份儲存、Items／Trigger、heartbeat timer、外部 watchdog | 獨立故障範圍與實際通知 |

### 1.11 驗證範圍與發布配套

2026-11-08 已執行 Linux 30 項隔離模擬測試與 Windows 15 項隔離模擬／靜態檢查，全部通過；Linux 原始輸出見腳本包 TEST-RESULTS.txt，Windows 見 TEST-RESULTS.json。另完成 Python 編譯、Bash 語法、PowerShell 7.4 parser、內附固定 MSI 雜湊、封裝完整性及 Word／Markdown 全文一致性檢查。

新增測試包括 metadata 損壞、有效 JSON 被改動、必要檔案缺失、隔離目錄不刪除、備份成功與清理失敗分離、首次估值、preflight 不刪副本、掛載遺失、續傳參數與相同遠端目錄、逾時收尾、MSI 核心與 wrapper 提前阻擋、錯誤完整發行者及只讀暫存檢查。續傳使用 mock 驗證參數與流程，尚未在真實 SSH／慢線做中斷傳輸；PowerShell 測試以 Linux 7.4 模擬 Windows-only API，不能代替 Windows PowerShell 5.1、Authenticode、IExpress、EDR、MSI、ACL 或服務實機驗收。

本分享版交付完整 Word 與 Markdown 文件。部署用 Linux 腳本、Windows Builder、固定 MSI、憑證與密鑰須由接收單位另行準備及核准，不隨本文件附送。管理 NB 預先完成參數與密鑰設定後，工程師仍可只複製核准 EXE、右鍵管理員執行；正常運作中的 Agent 不因文件去識別而需重裝。

| 審查問題 | 1.11 處理 | 尚待實機證據 |
| --- | --- | --- |
| metadata 單份損壞 | 邏輯隔離、保留原副本、count／epoch 告警 | 真實磁碟與通知 |
| 首次估值與掛載 | 必填估值、preflight、先檢查再建目錄 | 容量與掛載恢復 |
| 外送中斷／逾時 | partial-dir、先收尾再強殺、service 50 分鐘 | 真實 SSH 中斷與大檔總時間 |
| 備份成功與清理失敗 | local／retention 分開結果 | 告警與權限故障 |
| 還原設定範圍 | firewalld 與 CA anchors | 實際 CA／LDAPS／ACL 恢復 |
| 看門狗來源 | 固定 /32，僅 443／10051 | NAT／中間 ACL／重開機 |
| 部署核心 latest | 移除下載、核心固定 SHA256 與完整 Subject | Windows 5.1 首台 |
| IExpress 暫存 PSK | 唯讀檢查工具與首台中斷驗收 | 真正暫存 ACL 與殘留清理 |

### 官方資料與套件核對紀錄

核對基準日 2026-11-07。MySQL 官方索引提供 EL10 Repository RPM，但修訂號會更新；固定已核准檔案後記錄 URL、RPM NEVRA、GPG 驗證與 SHA256。Zabbix 官方 Alma 10 索引可確認 zabbix-release-7.0-7.el10.noarch.rpm 與 7.0.32 套件；Repo RPM 固定不代表全部安裝套件已固定，還需核對 dnf 交易及保存核准版本。

Zabbix 7.0 官方安裝文件確認 EL10 的 zabbix-selinux-policy-main 與 selinux-policy-targeted-extra 不可共存。既有主機不可直接移除已用的 Policy；先檢查相依並審查交易。MSI 文件可確認 SERVERACTIVE 與 STARTAGENTS 參數，STARTAGENTS=0 自 7.0.22 支援；7.0.32 的特定 LaunchConditions 仍須以該 MSI 的實際安裝 Log 判斷，不能單憑一般 1603 推論。

核對補充：MySQL 官方下載頁列出 mysql84-community-release-el10-3.noarch.rpm，部分索引或鏡像仍顯示 -2；此版本差異以原廠下載頁與實機 dnf／RPM 簽章核對為準，失敗停止，不改裝 9.7。MSI 本版固定 SHA256 與官方 HTTPS 下載、既有安裝紀錄相符；原廠簽章在 Windows NB 與 Server 仍須驗證。

## 三十二 官方參考來源

- MySQL 8.4 X Plugin 綁定  https://dev.mysql.com/doc/refman/8.4/en/x-plugin-options-system-variables.html

- MySQL 8.4 mysqldump  https://dev.mysql.com/doc/refman/8.4/en/mysqldump.html

- MySQL Repository RPM 下載  https://dev.mysql.com/downloads/repo/yum/

- Zabbix 7.0 EPEL 與已知問題  https://www.zabbix.com/documentation/7.0/en/manual/installation/known_issues

- Zabbix Email  https://www.zabbix.com/documentation/7.0/en/manual/config/notifications/media/email

- Zabbix LDAP  https://www.zabbix.com/documentation/7.0/en/manual/web_interface/frontend_sections/users/authentication/ldap

- Zabbix MFA  https://www.zabbix.com/documentation/7.0/en/manual/web_interface/frontend_sections/users/authentication/mfa

- Zabbix 升級  https://www.zabbix.com/documentation/7.0/en/manual/installation/upgrade/packages/rhel

- Alma 10 官方套件索引  https://repo.zabbix.com/zabbix/7.0/alma/10/x86_64/

- Zabbix 7.0 安全自動註冊  https://www.zabbix.com/documentation/7.0/en/manual/discovery/auto_registration

- Zabbix 7.0 Windows MSI 安裝參數  https://www.zabbix.com/documentation/7.0/en/manual/installation/install_from_packages/win_msi

- Zabbix 7.0 Windows Agent 2 設定  https://www.zabbix.com/documentation/7.0/en/manual/appendix/config/zabbix_agent2_win

- Zabbix 7.0 手動關閉問題  https://www.zabbix.com/documentation/7.0/en/manual/config/events/manual_close

- Zabbix Windows 7.0 主動範本原始設定  https://github.com/zabbix/zabbix/blob/release/7.0/templates/os/windows_agent_active/template_os_windows_agent_active.yaml

- Microsoft PowerShell 執行政策  https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies?view=powershell-5.1

- Microsoft PowerShell TLS 1.2 處理  https://devblogs.microsoft.com/powershell/powershell-gallery-tls-support/

- Microsoft IExpress 打包說明  https://learn.microsoft.com/en-us/cpp/windows/deploying-visual-cpp-application-by-using-the-vcpp-redistributable-package

- Zabbix 7.0 官方下載頁  https://www.zabbix.com/download?zabbix=7.0&os_distribution=alma_linux&os_version=10&components=server_frontend_agent&db=mysql&ws=nginx

- Zabbix 7.0 Installation from Packages  https://www.zabbix.com/documentation/7.0/en/manual/installation/install_from_packages

- Zabbix 7.0 Requirements  https://www.zabbix.com/documentation/7.0/en/manual/installation/requirements

- Zabbix Database Creation  https://www.zabbix.com/documentation/7.0/en/manual/appendix/install/db_scripts

- MySQL 8.4 Yum Repository Installation  https://dev.mysql.com/doc/refman/8.4/en/linux-installation-yum-repo.html

- MySQL Yum Repository Quick Guide  https://dev.mysql.com/doc/mysql-yum-repo-quick-guide/en/

- AlmaLinux 10.2 Release Notes  https://wiki.almalinux.org/release-notes/10.2

- AlmaLinux Installation Guide  https://wiki.almalinux.org/documentation/installation-guide.html

- RHEL 10 Interactively Installing from Installation Media  https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/interactively_installing_rhel_from_installation_media/interactively_installing_rhel_from_installation_media

- Microsoft Hyper V Generation 2 VM Security Settings  https://learn.microsoft.com/windows-server/virtualization/hyper-v/learn-more/generation-2-virtual-machine-security-settings-for-hyper-v

- Zabbix 7.0 Additional Frontend Languages  https://www.zabbix.com/documentation/7.0/en/manual/appendix/install/locales

- Zabbix Translating UI Guidelines  https://www.zabbix.com/documentation/guidelines/en/translator_guidelines/ui

- Zabbix 7.0 Agent 2  https://www.zabbix.com/documentation/7.0/en/manual/concepts/agent2

- Zabbix 7.0 Agent 2 UNIX Parameters  https://www.zabbix.com/documentation/7.0/en/manual/appendix/config/zabbix_agent2

- Zabbix 7.0 Agent 2 Windows MSI  https://www.zabbix.com/documentation/7.0/en/manual/installation/install_from_packages/win_msi

- Zabbix 7.0 Agent 2 Windows Parameters  https://www.zabbix.com/documentation/7.0/en/manual/appendix/config/zabbix_agent2_win

- Zabbix 7.0 SNMP Agent  https://www.zabbix.com/documentation/7.0/en/manual/config/items/itemtypes/snmp

- Zabbix Cisco IOS by SNMP  https://www.zabbix.com/integrations/cisco_snmp

- Cisco Secure SNMPv3 Guidance  https://www.cisco.com/c/en/us/support/docs/ip/simple-network-management-protocol-snmp/20370-snmpsecurity-20370.html

- Cisco Catalyst 9000 SNMPv3 Configuration  https://www.cisco.com/c/en/us/support/docs/switches/catalyst-9500-series-switches/224891-configure-snmpv2c-v3-on-catalyst-9000.html

- Zabbix 7.0 User Settings  https://www.zabbix.com/documentation/7.0/en/manual/web_interface/user_profile

最終建議  先依本手冊完成單機正式平台與備份還原驗收，再導入 Agent、SNMP、Oracle 與跨廠區 Proxy。不要在核心平台尚未穩定前一次匯入所有設備。

https://dev.mysql.com/doc/dev/mysqlsh-api-javascript/8.4/group__util.html

https://www.zabbix.com/documentation/7.0/en/manual/api/reference/history/get

https://github.com/zabbix/zabbix/blob/release/7.0/templates/net/cisco/cisco_snmp/template_net_cisco_snmp.yaml

https://download.samba.org/pub/rsync/rsync.1

https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/get-authenticodesignature?view=powershell-5.1
