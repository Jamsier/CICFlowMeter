# CICFlowMeter

CICFlowMeter 是一款開源的網路流量分析工具，能夠從 PCAP 封包擷取檔讀取封包、將其組合成雙向流（Bidirectional Flow），並提取統計特徵，輸出為 CSV 格式，可直接用於機器學習相關研究（如入侵偵測、惡意流量識別等）。

---

## 目錄

1. [專案架構](#專案架構)
2. [使用腳本與入口點](#使用腳本與入口點)
3. [環境需求與安裝](#環境需求與安裝)
4. [執行方式](#執行方式)
5. [PCAP 轉換特徵流程](#pcap-轉換特徵流程)
6. [特徵一覽表（含 PCAP 原始欄位對應）](#特徵一覽表含-pcap-原始欄位對應)
7. [依賴套件](#依賴套件)
8. [參考文獻](#參考文獻)

---

## 專案架構

```
CICFlowMeter/
├── src/main/java/cic/cs/unb/ca/
│   ├── ifm/
│   │   ├── App.java               # GUI 應用程式入口（JavaFX / Swing）
│   │   └── Cmd.java               # 命令列介面（CLI）入口
│   └── jnetpcap/
│       ├── PacketReader.java      # 讀取 PCAP 檔，解析每個封包
│       ├── BasicPacketInfo.java   # 封包基本資訊資料結構
│       ├── FlowGenerator.java     # 將封包組合成流，管理流的生命週期
│       ├── BasicFlow.java         # 單一流的特徵計算核心
│       └── FlowFeature.java       # 特徵名稱列舉定義（85 個欄位）
├── jnetpcap/
│   ├── linux/jnetpcap-1.4.r1425/ # Linux 原生函式庫（.so）
│   └── win/jnetpcap-1.4.r1425/   # Windows 原生函式庫（.dll）
├── build.gradle                   # Gradle 建置設定
├── pom.xml                        # Maven 建置設定（備用）
├── ReadMe.txt                     # 原始安裝說明
└── README.md                      # 本文件
```

---

## 使用腳本與入口點

### 1. `cic.cs.unb.ca.ifm.App`（GUI 模式）

圖形化介面，讓使用者透過視窗操作選取 PCAP 檔、設定參數並輸出 CSV。

**對應 Gradle 任務：**
```bash
./gradlew execute        # Linux / macOS（需要 sudo）
gradlew.bat execute      # Windows
```

### 2. `cic.cs.unb.ca.ifm.Cmd`（CLI 命令列模式）

批次處理模式，從命令列直接指定輸入 PCAP 檔（或資料夾）及輸出資料夾，適合自動化流程。

**對應 Gradle 任務：**
```bash
./gradlew exeCMD --args="<pcap_路徑> <輸出資料夾>"
```

**直接以 Java 執行：**
```bash
# Linux / macOS
java -Djava.library.path=jnetpcap/linux/jnetpcap-1.4.r1425 \
     -cp "build/libs/*:libs/*" \
     cic.cs.unb.ca.ifm.Cmd <pcap_檔或資料夾> <輸出資料夾>

# Windows
java -Djava.library.path=jnetpcap\win\jnetpcap-1.4.r1425 ^
     -cp "build\libs\*;libs\*" ^
     cic.cs.unb.ca.ifm.Cmd <pcap_檔或資料夾> <輸出資料夾>
```

**參數說明：**

| 參數 | 說明 |
|------|------|
| `<pcap_檔或資料夾>` | 單一 `.pcap` 檔，或包含多個 `.pcap` 檔的資料夾 |
| `<輸出資料夾>` | CSV 輸出目錄，每個 PCAP 對應一個 `<檔名>.pcap_Flow.csv` |

**硬編碼超時設定（`Cmd.java`）：**

| 設定 | 預設值 | 說明 |
|------|--------|------|
| `flowTimeout` | 120,000,000 µs（120 秒） | 流閒置超過此時間則強制結束 |
| `activityTimeout` | 5,000,000 µs（5 秒） | 用於區分 Active / Idle 時段 |

---

## 環境需求與安裝

- **Java 8**（JDK 1.8，建置設定以 Java 8 為目標版本；實際是否支援更新的 JDK 版本未經驗證）
- **Gradle** 或 **Maven**
- **Linux 需要 root 權限**（因 jnetpcap 使用 libpcap 原生函式庫）

### 安裝 jnetpcap 到本地 Maven Repository

```bash
# Linux
cd jnetpcap/linux/jnetpcap-1.4.r1425
mvn install:install-file -Dfile=jnetpcap.jar \
    -DgroupId=org.jnetpcap -DartifactId=jnetpcap \
    -Dversion=1.4.1 -Dpackaging=jar

# Windows
cd jnetpcap\win\jnetpcap-1.4.r1425
mvn install:install-file -Dfile=jnetpcap.jar ^
    -DgroupId=org.jnetpcap -DartifactId=jnetpcap ^
    -Dversion=1.4.1 -Dpackaging=jar
```

### 建置專案

```bash
# Gradle（建置 zip 發行套件）
./gradlew distZip
# 輸出：build/distributions/

# Maven
mvn package
# 輸出：target/
```

---

## 執行方式

### GUI 模式（IntelliJ IDEA）

```bash
sudo bash          # Linux 需要 root
./gradlew execute
```

### GUI 模式（Eclipse）

1. 以 root 身分啟動 Eclipse
2. 右鍵 `App.java` → **Run As** → **Run Configurations** → **Arguments** → **VM arguments**：
   ```
   -Djava.library.path="<專案路徑>/jnetpcap/linux/jnetpcap-1.4.r1425"
   ```
3. 點選 **Run**

---

## PCAP 轉換特徵流程

下圖說明 CICFlowMeter 將 PCAP 封包轉換成特徵 CSV 的完整資料流：

```
┌─────────────────────────────────────────────────────────────────────┐
│  PCAP 檔案 (.pcap)                                                  │
│  - 每個封包包含：Ethernet 標頭 + IP 標頭 + TCP/UDP 標頭 + Payload   │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                    PacketReader.java
                    使用 jnetpcap 函式庫讀取封包
                               │
                    ┌──────────▼──────────────┐
                    │  BasicPacketInfo        │
                    │  從每個封包解析：       │
                    │  - src IP / dst IP      │
                    │  - src port / dst port  │
                    │  - protocol (TCP=6,     │
                    │              UDP=17)    │
                    │  - timestamp (µs)       │
                    │  - payload bytes        │
                    │  - header bytes         │
                    │  - TCP flags (8 種)     │
                    │  - TCP window size      │
                    └──────────┬──────────────┘
                               │
                    FlowGenerator.java
                    以 5-tuple 為鍵分組封包：
                    (src IP, dst IP, src port, dst port, protocol)
                    - 第一個封包決定「正向」方向
                    - TCP FIN flag 或超時觸發流結束
                               │
                    ┌──────────▼──────────────┐
                    │  BasicFlow              │
                    │  計算每條流的 84 個     │
                    │  統計特徵               │
                    └──────────┬──────────────┘
                               │
                    ┌──────────▼──────────────┐
                    │  CSV 輸出               │
                    │  每行 = 一條流          │
                    │  85 欄（84 特徵 + Label)│
                    └─────────────────────────┘
```

### 封包解析細節（`PacketReader.java`）

| 解析層 | 使用 jnetpcap 類別 | 提取內容 |
|--------|-------------------|---------|
| 資料鏈路層（Layer 2） | `Ethernet` | 掃描封包以識別上層協定 |
| 網路層（Layer 3） | `Ip4`（IPv4）/ `Ip6`（IPv6） | 來源 IP、目的 IP |
| 傳輸層（Layer 4） | `Tcp` / `Udp` | 埠號、Payload 長度、標頭長度 |
| TCP 專屬 | `Tcp` | 8 個旗標（FIN/SYN/RST/PSH/ACK/URG/CWR/ECE）、視窗大小 |
| VPN 支援 | `L2TP` | 解封裝 L2TP 隧道後，再以上述方式解析內層封包 |
| 時間戳記 | `PcapHeader` | `timestampInMicros()`（微秒精度） |

### 流分組規則（`FlowGenerator.java` / `BasicFlow.java`）

- **5-tuple 鍵值**：`src_ip-dst_ip-src_port-dst_port-protocol`（已正規化方向）
- **正向（Forward）**：與流建立的第一個封包來源 IP 相同的封包
- **反向（Backward）**：來源 IP 不同（對方回應）的封包
- **流結束條件**：
  - TCP 封包帶有 FIN 旗標
  - 封包間隔超過 `flowTimeout`（預設 120 秒）

---

## 特徵一覽表（含 PCAP 原始欄位對應）

輸出 CSV 共 **85 欄**，前 84 欄為特徵，最後 1 欄為標籤。

**資料型態說明：**

| 型態 | 說明 |
|------|------|
| `String` | 字串，輸出為文字（如 IP 位址、識別碼、日期時間） |
| `int` | 32 位元整數（Java `int`） |
| `long` | 64 位元整數（Java `long`） |
| `double` | 64 位元浮點數（Java `double`），用於統計量、速率、平均值等 |
| `String（類別）` | 分類標籤字串 |

### 流身份識別特徵（欄 1–7）

| # | 特徵名稱 | 縮寫 | 資料型態 | PCAP 原始來源 / 計算方式 |
|---|---------|------|---------|--------------------------|
| 1 | Flow ID | FID | `String` | `src_ip-dst_ip-src_port-dst_port-protocol`（正規化後的 5-tuple 字串） |
| 2 | Src IP | SIP | `String` | IPv4 `source()` 或 IPv6 `source()`（`Ip4` / `Ip6` 標頭） |
| 3 | Src Port | SPT | `int` | TCP `source()` 或 UDP `source()`（`Tcp` / `Udp` 標頭） |
| 4 | Dst IP | DIP | `String` | IPv4 `destination()` 或 IPv6 `destination()`（`Ip4` / `Ip6` 標頭） |
| 5 | Dst Port | DPT | `int` | TCP `destination()` 或 UDP `destination()`（`Tcp` / `Udp` 標頭） |
| 6 | Protocol | PROT | `int` | IP 協定號碼（TCP = 6，UDP = 17） |
| 7 | Timestamp | TSTP | `String` | 流第一個封包的 `PcapHeader.timestampInMicros()` 轉換為日期時間字串 |

### 流時間特徵（欄 8）

| # | 特徵名稱 | 縮寫 | 資料型態 | PCAP 原始來源 / 計算方式 |
|---|---------|------|---------|--------------------------|
| 8 | Flow Duration | DUR | `long` | `flowLastSeen − flowStartTime`（單位：微秒，使用封包時間戳記） |

### 封包數量特徵（欄 9–10）

| # | 特徵名稱 | 縮寫 | 資料型態 | PCAP 原始來源 / 計算方式 |
|---|---------|------|---------|--------------------------|
| 9 | Total Fwd Packet | TFwP | `long` | 正向封包數量（`src == flow.src` 的封包數） |
| 10 | Total Bwd packets | TBwP | `long` | 反向封包數量（`src != flow.src` 的封包數） |

### 位元組總量特徵（欄 11–12）

| # | 特徵名稱 | 縮寫 | 資料型態 | PCAP 原始來源 / 計算方式 |
|---|---------|------|---------|--------------------------|
| 11 | Total Length of Fwd Packet | TLFwP | `double` | 所有正向封包的 `tcp.getPayloadLength()` 或 `udp.getPayloadLength()` 累加 |
| 12 | Total Length of Bwd Packet | TLBwP | `double` | 所有反向封包的 Payload 長度累加 |

### 正向封包長度統計特徵（欄 13–16）

以 `SummaryStatistics`（Apache Commons Math）對正向封包 Payload 位元組計算：

| # | 特徵名稱 | 縮寫 | 資料型態 | PCAP 原始來源 / 計算方式 |
|---|---------|------|---------|--------------------------|
| 13 | Fwd Packet Length Max | FwPLMA | `double` | `fwdPktStats.getMax()`（正向 Payload 最大值） |
| 14 | Fwd Packet Length Min | FwPLMI | `double` | `fwdPktStats.getMin()`（正向 Payload 最小值） |
| 15 | Fwd Packet Length Mean | FwPLAG | `double` | `fwdPktStats.getMean()`（正向 Payload 平均值） |
| 16 | Fwd Packet Length Std | FwPLSD | `double` | `fwdPktStats.getStandardDeviation()`（正向 Payload 標準差） |

### 反向封包長度統計特徵（欄 17–20）

| # | 特徵名稱 | 縮寫 | 資料型態 | PCAP 原始來源 / 計算方式 |
|---|---------|------|---------|--------------------------|
| 17 | Bwd Packet Length Max | BwPLMA | `double` | `bwdPktStats.getMax()`（反向 Payload 最大值） |
| 18 | Bwd Packet Length Min | BwPLMI | `double` | `bwdPktStats.getMin()`（反向 Payload 最小值） |
| 19 | Bwd Packet Length Mean | BwPLAG | `double` | `bwdPktStats.getMean()`（反向 Payload 平均值） |
| 20 | Bwd Packet Length Std | BwPLSD | `double` | `bwdPktStats.getStandardDeviation()`（反向 Payload 標準差） |

### 流速率特徵（欄 21–22）

| # | 特徵名稱 | 縮寫 | 資料型態 | PCAP 原始來源 / 計算方式 |
|---|---------|------|---------|--------------------------|
| 21 | Flow Bytes/s | FB/s | `double` | `(forwardBytes + backwardBytes) / (flowDuration / 1,000,000)` |
| 22 | Flow Packets/s | FP/s | `double` | `totalPacketCount / (flowDuration / 1,000,000)` |

### 流 IAT（Inter-Arrival Time）特徵（欄 23–26）

IAT = 同一流中相鄰兩個封包的時間戳記差值（微秒）：

| # | 特徵名稱 | 縮寫 | 資料型態 | PCAP 原始來源 / 計算方式 |
|---|---------|------|---------|--------------------------|
| 23 | Flow IAT Mean | FLIATAG | `double` | `flowIAT.getMean()`（所有封包間隔的平均值） |
| 24 | Flow IAT Std | FLIATSD | `double` | `flowIAT.getStandardDeviation()`（標準差） |
| 25 | Flow IAT Max | FLIATMA | `double` | `flowIAT.getMax()`（最大封包間隔） |
| 26 | Flow IAT Min | FLIATMI | `double` | `flowIAT.getMin()`（最小封包間隔） |

### 正向 IAT 特徵（欄 27–31）

| # | 特徵名稱 | 縮寫 | 資料型態 | PCAP 原始來源 / 計算方式 |
|---|---------|------|---------|--------------------------|
| 27 | Fwd IAT Total | FwIATTO | `double` | `forwardIAT.getSum()`（正向封包間隔總和） |
| 28 | Fwd IAT Mean | FwIATAG | `double` | `forwardIAT.getMean()` |
| 29 | Fwd IAT Std | FwIATSD | `double` | `forwardIAT.getStandardDeviation()` |
| 30 | Fwd IAT Max | FwIATMA | `double` | `forwardIAT.getMax()` |
| 31 | Fwd IAT Min | FwIATMI | `double` | `forwardIAT.getMin()` |

### 反向 IAT 特徵（欄 32–36）

| # | 特徵名稱 | 縮寫 | 資料型態 | PCAP 原始來源 / 計算方式 |
|---|---------|------|---------|--------------------------|
| 32 | Bwd IAT Total | BwIATTO | `double` | `backwardIAT.getSum()`（反向封包間隔總和） |
| 33 | Bwd IAT Mean | BwIATAG | `double` | `backwardIAT.getMean()` |
| 34 | Bwd IAT Std | BwIATSD | `double` | `backwardIAT.getStandardDeviation()` |
| 35 | Bwd IAT Max | BwIATMA | `double` | `backwardIAT.getMax()` |
| 36 | Bwd IAT Min | BwIATMI | `double` | `backwardIAT.getMin()` |

### 方向性 TCP 旗標計數特徵（欄 37–40）

> UDP 封包此類特徵恆為 0

| # | 特徵名稱 | 縮寫 | 資料型態 | PCAP 原始來源 / 計算方式 |
|---|---------|------|---------|--------------------------|
| 37 | Fwd PSH Flags | FwPSH | `int` | 正向封包中 `tcp.flags_PSH() == true` 的封包數量 |
| 38 | Bwd PSH Flags | BwPSH | `int` | 反向封包中 `tcp.flags_PSH() == true` 的封包數量 |
| 39 | Fwd URG Flags | FwURG | `int` | 正向封包中 `tcp.flags_URG() == true` 的封包數量 |
| 40 | Bwd URG Flags | BwURG | `int` | 反向封包中 `tcp.flags_URG() == true` 的封包數量 |

### 標頭長度特徵（欄 41–42）

| # | 特徵名稱 | 縮寫 | 資料型態 | PCAP 原始來源 / 計算方式 |
|---|---------|------|---------|--------------------------|
| 41 | Fwd Header Length | FwHL | `long` | 所有正向封包的 `tcp.getHeaderLength()` 或 `udp.getHeaderLength()` 累加（位元組） |
| 42 | Bwd Header Length | BwHL | `long` | 所有反向封包的標頭長度累加（位元組） |

### 每秒封包數特徵（欄 43–44）

| # | 特徵名稱 | 縮寫 | 資料型態 | PCAP 原始來源 / 計算方式 |
|---|---------|------|---------|--------------------------|
| 43 | Fwd Packets/s | FwP/s | `double` | `forward.size() / (flowDuration / 1,000,000)` |
| 44 | Bwd Packets/s | Bwp/s | `double` | `backward.size() / (flowDuration / 1,000,000)` |

### 全流封包長度統計特徵（欄 45–49）

以 `flowLengthStats` 對所有封包（正向 + 反向）的 Payload 位元組計算：

| # | 特徵名稱 | 縮寫 | 資料型態 | PCAP 原始來源 / 計算方式 |
|---|---------|------|---------|--------------------------|
| 45 | Packet Length Min | PLMI | `double` | `flowLengthStats.getMin()` |
| 46 | Packet Length Max | PLMA | `double` | `flowLengthStats.getMax()` |
| 47 | Packet Length Mean | PLAG | `double` | `flowLengthStats.getMean()` |
| 48 | Packet Length Std | PLSD | `double` | `flowLengthStats.getStandardDeviation()` |
| 49 | Packet Length Variance | PLVA | `double` | `flowLengthStats.getVariance()` |

### TCP 旗標總計數特徵（欄 50–57）

對整條流（正向 + 反向）中各 TCP 旗標出現次數計算：

| # | 特徵名稱 | 縮寫 | 資料型態 | PCAP 原始來源 / 計算方式 |
|---|---------|------|---------|--------------------------|
| 50 | FIN Flag Count | FINCT | `int` | `tcp.flags_FIN() == true` 的封包總數 |
| 51 | SYN Flag Count | SYNCT | `int` | `tcp.flags_SYN() == true` 的封包總數 |
| 52 | RST Flag Count | RSTCT | `int` | `tcp.flags_RST() == true` 的封包總數 |
| 53 | PSH Flag Count | PSHCT | `int` | `tcp.flags_PSH() == true` 的封包總數 |
| 54 | ACK Flag Count | ACKCT | `int` | `tcp.flags_ACK() == true` 的封包總數 |
| 55 | URG Flag Count | URGCT | `int` | `tcp.flags_URG() == true` 的封包總數 |
| 56 | CWR Flag Count | CWRCT | `int` | `tcp.flags_CWR() == true` 的封包總數 |
| 57 | ECE Flag Count | ECECT | `int` | `tcp.flags_ECE() == true` 的封包總數 |

### 比率與平均大小特徵（欄 58–61）

| # | 特徵名稱 | 縮寫 | 資料型態 | PCAP 原始來源 / 計算方式 |
|---|---------|------|---------|--------------------------|
| 58 | Down/Up Ratio | D/URO | `double` | `backward.size() / forward.size()`（反向封包數 ÷ 正向封包數） |
| 59 | Average Packet Size | PSAG | `double` | `flowLengthStats.getSum() / totalPacketCount`（所有封包 Payload 總和 ÷ 封包總數） |
| 60 | Fwd Segment Size Avg | FwSgAG | `double` | `fwdPktStats.getSum() / forward.size()`（正向 Payload 總和 ÷ 正向封包數） |
| 61 | Bwd Segment Size Avg | BwSgAG | `double` | `bwdPktStats.getSum() / backward.size()`（反向 Payload 總和 ÷ 反向封包數） |

> **注意：** 原始程式碼中欄位 62 原為 `Fwd Header Length` 的重複欄位（與欄位 41 相同），已在 `FlowFeature.java` 中刪除，因此輸出 CSV 中不存在欄位 62，後續欄號維持原始程式碼中的編號（63 起）。

| 62 | *(已移除)* | *(N/A)* | *(N/A)* | 原為 Fwd Header Length 的重複值，與欄位 41 相同，已刪除 |

### 批量傳輸（Bulk）特徵（欄 63–68）

Bulk 定義：同方向連續 ≥ 4 個 Payload > 0 的封包，且相鄰封包間隔 < 1 秒（1,000,000 µs）則視為一次 Bulk：

| # | 特徵名稱 | 縮寫 | 資料型態 | PCAP 原始來源 / 計算方式 |
|---|---------|------|---------|--------------------------|
| 63 | Fwd Bytes/Bulk Avg | FwB/BAG | `long` | `fbulkSizeTotal / fbulkStateCount`（正向每次 Bulk 的平均位元組數） |
| 64 | Fwd Packet/Bulk Avg | FwP/BAG | `long` | `fbulkPacketCount / fbulkStateCount`（正向每次 Bulk 的平均封包數） |
| 65 | Fwd Bulk Rate Avg | FwBRAG | `long` | `fbulkSizeTotal / fbulkDuration(秒)`（正向 Bulk 平均傳輸速率 bytes/s） |
| 66 | Bwd Bytes/Bulk Avg | BwB/BAG | `long` | `bbulkSizeTotal / bbulkStateCount`（反向每次 Bulk 的平均位元組數） |
| 67 | Bwd Packet/Bulk Avg | BwP/BAG | `long` | `bbulkPacketCount / bbulkStateCount`（反向每次 Bulk 的平均封包數） |
| 68 | Bwd Bulk Rate Avg | BwBRAG | `long` | `bbulkSizeTotal / bbulkDuration(秒)`（反向 Bulk 平均傳輸速率 bytes/s） |

### 子流（Subflow）特徵（欄 69–72）

Subflow 定義：流中相鄰封包間隔 > 1 秒時，計為一個新的子流（`sfCount++`）：

| # | 特徵名稱 | 縮寫 | 資料型態 | PCAP 原始來源 / 計算方式 |
|---|---------|------|---------|--------------------------|
| 69 | Subflow Fwd Packets | SFFwP | `long` | `forward.size() / sfCount`（每個子流的平均正向封包數） |
| 70 | Subflow Fwd Bytes | SFFwB | `long` | `forwardBytes / sfCount`（每個子流的平均正向位元組數） |
| 71 | Subflow Bwd Packets | SFBwP | `long` | `backward.size() / sfCount`（每個子流的平均反向封包數） |
| 72 | Subflow Bwd Bytes | SFBwB | `long` | `backwardBytes / sfCount`（每個子流的平均反向位元組數） |

### TCP 視窗與資料封包特徵（欄 73–76）

| # | 特徵名稱 | 縮寫 | 資料型態 | PCAP 原始來源 / 計算方式 |
|---|---------|------|---------|--------------------------|
| 73 | FWD Init Win Bytes | FwWB | `int` | 正向第一個封包的 `tcp.window()`（TCP 標頭中的接收視窗大小，位元組） |
| 74 | Bwd Init Win Bytes | BwWB | `int` | 反向第一個封包的 `tcp.window()`（TCP 標頭中的接收視窗大小，位元組） |
| 75 | Fwd Act Data Pkts | FwAP | `long` | 正向封包中 `payloadBytes >= 1` 的封包數量（含有 TCP Payload 的封包數） |
| 76 | Fwd Seg Size Min | FwSgMI | `long` | 正向封包中 `headerBytes` 的最小值（`tcp.getHeaderLength()` 最小值） |

### 活躍時段（Active）統計特徵（欄 77–80）

Active 時段：流持續有封包傳輸的連續時間段（前後封包間隔 < `activityTimeout`）：

| # | 特徵名稱 | 縮寫 | 資料型態 | PCAP 原始來源 / 計算方式 |
|---|---------|------|---------|--------------------------|
| 77 | Active Mean | AcAG | `double` | `flowActive.getMean()`（各活躍時段持續時間的平均值，微秒） |
| 78 | Active Std | AcSD | `double` | `flowActive.getStandardDeviation()`（標準差） |
| 79 | Active Max | AcMA | `double` | `flowActive.getMax()`（最長活躍時段） |
| 80 | Active Min | AcMI | `double` | `flowActive.getMin()`（最短活躍時段） |

### 閒置時段（Idle）統計特徵（欄 81–84）

Idle 時段：流中相鄰封包間隔超過 `activityTimeout`（預設 5 秒）的時間段：

| # | 特徵名稱 | 縮寫 | 資料型態 | PCAP 原始來源 / 計算方式 |
|---|---------|------|---------|--------------------------|
| 81 | Idle Mean | IlAG | `double` | `flowIdle.getMean()`（各閒置時段持續時間的平均值，微秒） |
| 82 | Idle Std | IlSD | `double` | `flowIdle.getStandardDeviation()`（標準差） |
| 83 | Idle Max | IlMA | `double` | `flowIdle.getMax()`（最長閒置時段） |
| 84 | Idle Min | IlMI | `double` | `flowIdle.getMin()`（最短閒置時段） |

### 標籤（欄 85）

| # | 特徵名稱 | 縮寫 | 資料型態 | 說明 |
|---|---------|------|---------|------|
| 85 | Label | LBL | `String（類別）` | 流的分類標籤，預設為 `NeedManualLabel`，需手動標記 |

---

## 依賴套件

| 套件 | 版本 | 用途 |
|------|------|------|
| `org.jnetpcap:jnetpcap` | 1.4.1 | 讀取 PCAP 檔案，解析各層封包標頭 |
| `org.apache.commons:commons-math3` | 3.5 | `SummaryStatistics`（計算 Max/Min/Mean/Std/Variance） |
| `org.apache.commons:commons-lang3` | 3.6 | 字串及數字工具函式 |
| `commons-io:commons-io` | 2.5 | 檔案讀寫工具 |
| `org.apache.logging.log4j:log4j-core` | 2.11.0 | 日誌記錄框架（**注意：此版本受 CVE-2021-44228 Log4Shell 漏洞影響，建議升級至 2.17.1 以上**） |
| `org.slf4j:slf4j-log4j12` | 1.7.25 | SLF4J 介面綁定 |
| `nz.ac.waikato.cms.weka:weka-stable` | 3.6.14 | 機器學習叢集分析（選用） |
| `org.jfree:jfreechart` | 1.5.0 | 圖表繪製（GUI 模式） |
| `com.google.guava:guava` | 23.6-jre | 事件匯流排（EventBus）等工具 |
| `org.apache.tika:tika-core` | 1.17 | 檔案類型偵測 |

---

## 參考文獻

- Arash Habibi Lashkari, Gerard Draper-Gil, Mohammad Saiful Islam Mamun and Ali A. Ghorbani, "Characterization of Tor Traffic Using Time Based Features", In the proceeding of the 3rd International Conference on Information System Security and Privacy, SCITEPRESS, Porto, Portugal, 2017

- Gerard Drapper Gil, Arash Habibi Lashkari, Mohammad Mamun, Ali A. Ghorbani, "Characterization of Encrypted and VPN Traffic Using Time-Related Features", In Proceedings of the 2nd International Conference on Information Systems Security and Privacy (ICISSP 2016), pages 407-414, Rome, Italy
