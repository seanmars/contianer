# Observability Stack — 完整技術文件

> 適用對象：完全不熟悉可觀測性工具的新進人員
> 版本：OTel Collector 0.145.0 / Prometheus v3.9.1 / Loki 3.6 / Tempo 2.10.0 / Grafana 12.3

---

## 目錄

1. [什麼是可觀測性（Observability）](#1-什麼是可觀測性observability)
2. [可觀測性的三大支柱](#2-可觀測性的三大支柱)
3. [整體架構總覽](#3-整體架構總覽)
4. [各服務角色說明](#4-各服務角色說明)
5. [資料完整流向](#5-資料完整流向)
6. [目錄結構](#6-目錄結構)
7. [設定檔逐行解析](#7-設定檔逐行解析)
   - 7.1 [compose.yml](#71-composeyml)
   - 7.2 [otelcol-config.yaml（OTel Collector）](#72-otelcol-configyamlotlp-collector)
   - 7.3 [prometheus.yaml（Prometheus）](#73-prometheusyamlprometheus)
   - 7.4 [loki-config.yaml（Loki）](#74-loki-configyamlloki)
   - 7.5 [tempo-config.yaml（Tempo）](#75-tempo-configyamltempo)
   - 7.6 [datasources.yaml（Grafana 資料來源）](#76-datasourcesyamlgrafana-資料來源)
8. [快速啟動](#8-快速啟動)
9. [驗證各服務健康狀態](#9-驗證各服務健康狀態)
10. [應用程式整合指南](#10-應用程式整合指南)
11. [Grafana 使用說明](#11-grafana-使用說明)
12. [常見問題排查](#12-常見問題排查)
13. [生產環境升級建議](#13-生產環境升級建議)
14. [官方文件參考](#14-官方文件參考)

---

## 1. 什麼是可觀測性（Observability）

**可觀測性**指的是「透過系統對外輸出的資料，推斷系統內部狀態的能力」。

在軟體系統中，當線上服務發生異常時，工程師需要快速回答幾個核心問題：

- **系統現在有沒有問題？** → 靠 **Metrics（指標）** 回答
- **哪裡出了問題？訊息是什麼？** → 靠 **Logs（日誌）** 回答
- **一個請求是如何在多個服務間流動的？** → 靠 **Traces（追蹤）** 回答

可觀測性與傳統「監控（Monitoring）」的差異在於：

| 監控（Monitoring） | 可觀測性（Observability） |
|---|---|
| 事先定義好問哪些問題 | 可以隨時提出新的問題 |
| 只知道「發生了什麼」 | 還能知道「為什麼發生」 |
| 適合穩定的單體架構 | 適合複雜的微服務架構 |

> **官方參考**：[CNCF Observability Whitepaper](https://github.com/cncf/tag-observability/blob/main/whitepaper.md)

---

## 2. 可觀測性的三大支柱

### 2.1 Metrics（指標）

Metrics 是**數值型的時序資料**，代表某個時間點系統的量化狀態。

```
# 範例：HTTP 請求總數
http_requests_total{method="GET", status="200"} 1234  @ timestamp
http_requests_total{method="POST", status="500"} 5    @ timestamp

# 範例：CPU 使用率
cpu_usage_percent{host="web-01"} 75.3  @ timestamp
```

常見 Metrics 類型：

| 類型 | 說明 | 範例 |
|------|------|------|
| **Counter** | 只增不減的累計數 | 請求總數、錯誤次數 |
| **Gauge** | 可增可減的當前值 | 記憶體用量、連線數 |
| **Histogram** | 數值分佈（含桶狀分組） | 請求延遲分佈 |
| **Summary** | 百分位數（p50/p95/p99） | 回應時間 p99 |

> **官方參考**：[Prometheus Metric Types](https://prometheus.io/docs/concepts/metric_types/)

### 2.2 Logs（日誌）

Logs 是**帶有時間戳記的文字或結構化事件**，記錄系統在某一時刻發生的事情。

```
# 非結構化日誌（傳統）
2025-01-15T10:30:00Z ERROR Failed to connect to database: timeout

# 結構化日誌（現代，推薦）
{
  "timestamp": "2025-01-15T10:30:00Z",
  "level": "error",
  "service": "order-service",
  "message": "Failed to connect to database",
  "error": "timeout",
  "traceID": "abc123def456"
}
```

結構化日誌的優點是可以精確篩選，例如：找出所有 `level=error` 且 `service=order-service` 的日誌。

> **官方參考**：[Grafana Loki Documentation](https://grafana.com/docs/loki/latest/)

### 2.3 Traces（追蹤）

Traces 記錄**一個請求在分散式系統中的完整旅程**。在微服務架構中，一個使用者請求可能會經過多個服務，Traces 讓你看清楚每個步驟花了多少時間、在哪裡失敗。

```
Trace ID: abc123
│
├─ Span: API Gateway (0ms → 150ms)
│  ├─ Span: auth-service.validate (5ms → 20ms)
│  ├─ Span: order-service.create (22ms → 120ms)
│  │  ├─ Span: inventory-service.check (25ms → 60ms)
│  │  └─ Span: database.insert (62ms → 115ms)
│  └─ Span: notification-service.send (121ms → 148ms)
```

核心概念：

| 術語 | 說明 |
|------|------|
| **Trace** | 一個請求的完整追蹤記錄，由多個 Span 組成 |
| **Span** | 追蹤中的一個操作單元，有開始/結束時間與狀態 |
| **Trace ID** | 全域唯一識別碼，串聯整個請求路徑 |
| **Span ID** | 每個 Span 的唯一識別碼 |
| **Parent Span** | 呼叫其他服務的那個 Span（父節點） |

> **官方參考**：[OpenTelemetry Traces Concepts](https://opentelemetry.io/docs/concepts/signals/traces/)

---

## 3. 整體架構總覽

### 3.1 架構圖

```
┌─────────────────────────────────────────────────────────────────┐
│                        你的應用程式                              │
│              (透過 OTLP 協定送出 Traces/Metrics/Logs)            │
└──────────────────────────┬──────────────────────────────────────┘
                           │  OTLP gRPC (port 4317)
                           │  OTLP HTTP (port 4318)
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                  OpenTelemetry Collector                         │
│                                                                  │
│  ┌──────────┐   ┌───────────────┐   ┌─────────────────────┐    │
│  │ Receiver │──▶│  Processors   │──▶│      Exporters      │    │
│  │  (OTLP)  │   │ memory_limiter│   │ otlp/tempo          │    │
│  │ port 4317│   │ batch         │   │ prometheusremotewrite│    │
│  │ port 4318│   │ resource      │   │ loki                │    │
│  └──────────┘   └───────────────┘   └──────┬──────────────┘    │
│                                             │                    │
└─────────────────────────────────────────────┼────────────────────┘
                                              │
              ┌───────────────────────────────┼──────────────────┐
              │               │               │                  │
              ▼               ▼               ▼                  │
   ┌─────────────────┐ ┌────────────┐ ┌────────────┐           │
   │     Tempo       │ │ Prometheus │ │    Loki    │           │
   │  (port 3200)    │ │ (port 9090)│ │ (port 3100)│           │
   │  儲存 Traces    │ │ 儲存Metrics│ │ 儲存 Logs  │           │
   └────────┬────────┘ └─────┬──────┘ └─────┬──────┘           │
            │                │               │                  │
            └────────────────┼───────────────┘                  │
                             │                                  │
                             ▼                                  │
                  ┌──────────────────────┐                      │
                  │       Grafana        │◀─────────────────────┘
                  │     (port 3000)      │  Prometheus 也 scrape
                  │   統一視覺化介面     │  OTel Collector metrics
                  └──────────────────────┘
```

### 3.2 為什麼需要 OpenTelemetry Collector？

你可能會問：「應用程式為什麼不直接把資料送到 Prometheus/Loki/Tempo？」

直接送的問題：

1. **每個服務都要設定多個目的地** — 應用程式程式碼要同時知道 Prometheus、Loki、Tempo 的位址
2. **格式不統一** — 各工具的接收格式不同，需要各自的 SDK
3. **維運彈性差** — 更換後端工具（例如從 Jaeger 換到 Tempo）需要修改所有應用程式
4. **缺乏緩衝與批次處理** — 沒有流量保護機制

**OpenTelemetry Collector 解決了這些問題**：

```
應用程式 → 只需知道 OTel Collector 的位址（一個目的地）
OTel Collector → 負責路由到各後端（可隨時更換後端，應用程式零修改）
```

> **官方參考**：[Why use a Collector?](https://opentelemetry.io/docs/collector/)

---

## 4. 各服務角色說明

### 4.1 OpenTelemetry Collector（資料收集與路由中心）

- **Image**：`otel/opentelemetry-collector-contrib:0.145.0`
- **角色**：遙測資料的「中央交換機」
- **開放端口**：
  - `4317` — OTLP gRPC（應用程式送資料）
  - `4318` — OTLP HTTP（支援 HTTP 的應用程式）
  - `8888` — Collector 自身的 Prometheus metrics 端點
  - `13133` — Health Check 端點

使用 `otelcol-contrib`（contrib 版本）而非 `otelcol`（core 版本）的原因是，contrib 版本包含了更多社群貢獻的 receiver/exporter，包含本專案需要的 `loki` exporter 和 `prometheusremotewrite` exporter。

> **官方參考**：[OpenTelemetry Collector Contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib)

### 4.2 Prometheus（指標儲存與查詢）

- **Image**：`prom/prometheus:v3.9.1`
- **角色**：時序資料庫，專門儲存 Metrics
- **開放端口**：`9090`
- **重要特性**：
  - 本專案啟用了 `--web.enable-remote-write-receiver`，讓 Prometheus 可以**接收**資料（被動接收）
  - 同時也透過 `scrape_configs` **主動抓取**各服務的 metrics

Prometheus 有兩種接收資料的方式：

| 方式 | 說明 | 本專案使用 |
|------|------|----------|
| **Pull（拉取）** | Prometheus 主動去各服務抓 `/metrics` | 抓取 OTel Collector、Tempo、Loki 自身的 metrics |
| **Push（推送）** | 服務主動把資料推到 Prometheus | OTel Collector 透過 Remote Write 推送應用程式 metrics |

> **官方參考**：[Prometheus Documentation](https://prometheus.io/docs/introduction/overview/)

### 4.3 Loki（日誌儲存與查詢）

- **Image**：`grafana/loki:3.6.5`
- **角色**：專為日誌設計的時序資料庫
- **開放端口**：`3100`

Loki 與傳統日誌系統（如 Elasticsearch）的核心差異：

| 特性 | Loki | Elasticsearch |
|------|------|---------------|
| **索引方式** | 只索引 Labels（標籤），不索引日誌內容 | 索引全文 |
| **儲存成本** | 低（日誌內容直接壓縮儲存） | 高（索引佔用大量空間） |
| **查詢語言** | LogQL | Lucene Query |
| **定位** | 配合 Grafana 使用 | 獨立完整的搜尋引擎 |

Loki 的資料模型：

```
日誌流（Log Stream）= 一組 Labels 的組合
例如：{service_name="order-service", level="error"}

每個日誌流包含：
  - Labels：識別這是哪個服務、哪個環境的日誌
  - Log Lines：實際的日誌內容（未被索引，查詢時用 grep）
```

> **官方參考**：[Grafana Loki Documentation](https://grafana.com/docs/loki/latest/)

### 4.4 Tempo（追蹤儲存與查詢）

- **Image**：`grafana/tempo:2.10.0`
- **角色**：分散式追蹤後端儲存
- **開放端口**：
  - `3200` — HTTP API（Grafana 查詢用）
  - `4317`（容器內部）— 接收 OTel Collector 送來的 traces

Tempo 的特色是**只索引 Trace ID**，其他 metadata 不建立索引，因此儲存成本極低。查詢時透過 TraceQL 或 Service Map 進行探索。

Tempo 的重要功能 — **Metrics Generator**：

Tempo 可以從 Traces 自動推導出 Metrics，這個功能叫做 Metrics Generator。它能產生：
- **service-graphs**：服務間呼叫關係圖（呈現在 Grafana 的 Service Map）
- **span-metrics**：每個 Span 的 RED metrics（Request rate、Error rate、Duration）

這些 Metrics 會透過 Remote Write 自動寫入 Prometheus，讓你不需要在應用程式裡額外實作 metrics，只要有 Traces 就能自動產生！

> **官方參考**：[Grafana Tempo Documentation](https://grafana.com/docs/tempo/latest/)

### 4.5 Grafana（統一視覺化平台）

- **Image**：`grafana/grafana:12.3.2`
- **角色**：讀取 Prometheus/Loki/Tempo 的資料，提供統一的視覺化介面
- **開放端口**：`3000`
- **預設帳號**：`admin` / `admin`

Grafana 在本專案的三個核心功能：

1. **Dashboard**：繪製 Metrics 圖表（從 Prometheus 查詢）
2. **Logs 探索**：搜尋與篩選日誌（從 Loki 查詢）
3. **Trace 探索**：查看請求追蹤與 Service Map（從 Tempo 查詢）

更強大的是三者之間的**跨服務連結**：
- 在 Prometheus 的 metric 圖表上，可以點擊一個資料點直接跳轉到對應時間的 Trace
- 在 Tempo 的 Trace 詳情中，可以點擊直接跳轉到對應時間的 Loki 日誌
- 在 Loki 的日誌行中，如果包含 `traceID`，可以直接跳轉到 Tempo 查看 Trace

> **官方參考**：[Grafana Documentation](https://grafana.com/docs/grafana/latest/)

---

## 5. 資料完整流向

### 5.1 Traces 流向

```
應用程式
  │  以 OTLP 格式送出 Trace Spans
  │  (包含：TraceID、SpanID、服務名稱、操作名稱、開始/結束時間、狀態碼、Attributes)
  ▼
OTel Collector (4317 gRPC)
  │  Receiver: otlp
  │  Processors: memory_limiter → batch
  │  Exporter: otlp/tempo
  ▼
Tempo (容器內部 4317)
  │  Distributor 接收
  │  Ingester 暫存在記憶體（WAL）
  │  定期 Compactor 壓縮到本地磁碟 /var/tempo/blocks
  │
  ├──▶ Metrics Generator（自動從 Traces 產生 RED metrics）
  │         │
  │         └──▶ Prometheus (Remote Write 9090)
  │
  ▼
Grafana (查詢 Tempo 3200)
  └─ 使用 TraceQL 語言查詢追蹤資料
```

### 5.2 Metrics 流向

```
應用程式
  │  以 OTLP 格式送出 Metrics
  │  (包含：metric name、value、labels、timestamp)
  ▼
OTel Collector (4317)
  │  Receiver: otlp
  │  Processors: memory_limiter → batch
  │  Exporter: prometheusremotewrite
  ▼
Prometheus (9090 /api/v1/write)
  │  接收 Remote Write 資料
  │  以 TSDB 格式儲存（壓縮時序資料）
  │  保留 15 天（設定值）
  │
  └─ 同時主動 scrape：
      ├─ OTel Collector:8888 (Collector 自身 metrics)
      ├─ Tempo:3200/metrics  (Tempo 自身 metrics)
      └─ Loki:3100/metrics   (Loki 自身 metrics)
  ▼
Grafana (查詢 Prometheus 9090)
  └─ 使用 PromQL 語言查詢 metrics 資料
```

### 5.3 Logs 流向

```
應用程式
  │  以 OTLP 格式送出 Log Records
  │  (包含：時間戳記、嚴重程度、訊息內容、resource attributes)
  ▼
OTel Collector (4317)
  │  Receiver: otlp
  │  Processors: memory_limiter → resource (設定 loki.resource.labels) → batch
  │  Exporter: loki
  ▼
Loki (3100 /loki/api/v1/push)
  │  根據 loki.resource.labels 設定的 attribute（service.name 等）建立 Labels
  │  日誌內容以壓縮格式儲存在本地磁碟 /loki/chunks
  │  TSDB 索引儲存在 /loki/index
  ▼
Grafana (查詢 Loki 3100)
  └─ 使用 LogQL 語言查詢日誌
```

---

## 6. 目錄結構

```
observability/
│
├── compose.yml                          # Docker Compose 主設定檔
│
└── config/                              # 各服務設定檔目錄
    ├── otelcol-config.yaml              # OpenTelemetry Collector 設定
    ├── prometheus.yaml                  # Prometheus 設定
    ├── loki-config.yaml                 # Loki 設定
    ├── tempo-config.yaml                # Tempo 設定
    │
    └── grafana/
        └── provisioning/                # Grafana 自動佈建設定
            ├── datasources/
            │   └── datasources.yaml     # 資料來源設定（Prometheus/Loki/Tempo）
            └── dashboards/
                └── dashboards.yaml      # Dashboard 載入設定
```

**Grafana Provisioning** 是指在 Grafana 啟動時自動套用設定，不需要手動在 UI 介面點選設定。這讓整個 stack 做到「基礎設施即程式碼（Infrastructure as Code）」，重建環境時設定不會消失。

> **官方參考**：[Grafana Provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/)

---

## 7. 設定檔逐行解析

### 7.1 `compose.yml`

Docker Compose 是用來定義和執行多容器 Docker 應用的工具。整個 `compose.yml` 宣告了 5 個服務、4 個持久化 Volume、1 個網路。

#### 服務版本資訊

```yaml
# 服務名稱：image 版本
otel-collector: otel/opentelemetry-collector-contrib:0.145.0
prometheus:      prom/prometheus:v3.9.1
loki:            grafana/loki:3.6.5
tempo:           grafana/tempo:2.10.0
grafana:         grafana/grafana:12.3.2
```

#### 啟動順序控制（depends_on + healthcheck）

```yaml
# otel-collector 要等 tempo、loki、prometheus 都健康才啟動
otel-collector:
  depends_on:
    tempo:
      condition: service_healthy    # ← 必須通過 healthcheck 才算「健康」
    loki:
      condition: service_healthy
    prometheus:
      condition: service_healthy
```

這樣設計的原因是：如果 OTel Collector 比後端服務先啟動，它嘗試送資料時後端還沒準備好，會導致資料遺失或連線錯誤。

每個後端服務都定義了健康檢查：

```yaml
# Prometheus 健康檢查
healthcheck:
  test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:9090/-/healthy"]
  interval: 10s    # 每 10 秒檢查一次
  timeout: 5s      # 超過 5 秒沒回應視為失敗
  retries: 5       # 失敗 5 次才宣告不健康
```

#### Prometheus 啟動參數

```yaml
prometheus:
  command:
    - --config.file=/etc/prometheus/prometheus.yaml
    - --storage.tsdb.path=/prometheus              # 資料儲存路徑
    - --storage.tsdb.retention.time=15d            # 資料保留 15 天
    - --web.enable-remote-write-receiver           # 開啟 Remote Write 端點
    - --enable-feature=exemplar-storage            # 開啟 Exemplars 功能
    - --web.enable-lifecycle                       # 允許透過 HTTP 觸發 reload
```

`--web.enable-remote-write-receiver` 這個參數非常重要，沒有它，OTel Collector 就無法把 metrics 推送到 Prometheus。

> **官方參考**：[Prometheus Remote Write](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#remote_write)

#### 持久化儲存（Volumes）

```yaml
volumes:
  prometheus-data:    # Prometheus TSDB 資料
  loki-data:          # Loki 日誌資料
  tempo-data:         # Tempo Trace 資料
  grafana-data:       # Grafana 設定、Dashboard
```

使用 Named Volume（具名卷）而非 Bind Mount 的原因：Named Volume 由 Docker 管理，跨平台相容性更好，且不會因為宿主機目錄權限問題而失敗。

#### 網路設計

```yaml
networks:
  observability:
    driver: bridge
```

所有服務都在同一個 `observability` bridge network 中，服務間可以用**容器名稱**（container_name）直接溝通，例如 `tempo:4317`、`loki:3100`。這些名稱不需要 IP 位址，Docker 內建 DNS 解析。

> **官方參考**：[Docker Compose Networking](https://docs.docker.com/compose/networking/)

---

### 7.2 `otelcol-config.yaml`（OTel Collector）

OTel Collector 的設定分成四個區塊：`extensions`、`receivers`、`processors`、`exporters`，最後由 `service.pipelines` 組裝成流水線。

#### extensions（擴充功能）

```yaml
extensions:
  health_check:
    endpoint: 0.0.0.0:13133
```

`health_check` extension 讓 Collector 在 port 13133 提供健康檢查 HTTP 端點（`/`），Docker 的 healthcheck 就是打這個端點。

> **官方參考**：[Health Check Extension](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/extension/healthcheckextension)

#### receivers（資料接收器）

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317    # 監聽所有網路介面的 4317 port
      http:
        endpoint: 0.0.0.0:4318    # 監聽所有網路介面的 4318 port
```

`otlp` receiver 支援 OTLP（OpenTelemetry Protocol），這是 OpenTelemetry 定義的標準傳輸協定。

- **gRPC (4317)**：高效能，適合高流量場景，使用 Protocol Buffers 序列化
- **HTTP (4318)**：相容性好，適合無法使用 gRPC 的環境（如瀏覽器、某些語言 SDK）

> **官方參考**：[OTLP Receiver](https://github.com/open-telemetry/opentelemetry-collector/tree/main/receiver/otlpreceiver)

#### processors（資料處理器）

```yaml
processors:
  memory_limiter:
    check_interval: 1s          # 每 1 秒檢查記憶體用量
    limit_percentage: 75        # 用到 75% 開始拒絕新資料
    spike_limit_percentage: 15  # 突增 15% 時觸發 GC

  batch:
    timeout: 5s                 # 最多等 5 秒，把資料批次送出
    send_batch_size: 10000      # 或累積到 10000 筆就送出
    send_batch_max_size: 11000  # 單批次上限

  resource:
    attributes:
      - action: insert
        key: loki.resource.labels
        value: service.name, service.namespace, service.version
```

**memory_limiter**：防止記憶體爆炸。當流量突增時，不讓 Collector 無限制地佔用記憶體，超過閾值就回壓（backpressure）給上游。

**batch**：批次處理可以大幅減少網路連線次數，提升整體吞吐量。個別資料到達後先累積，達到閾值才一次送出。

**resource**：這裡設定了一個特殊的 attribute `loki.resource.labels`，其值是一組以逗號分隔的 resource attribute 名稱。這是 OTel Loki Exporter 的特殊 hint，告訴 Loki Exporter 把哪些 resource attributes 當作 Loki 的 Labels（而非 log 內容的一部分）。

> **官方參考**：
> - [Memory Limiter Processor](https://github.com/open-telemetry/opentelemetry-collector/tree/main/processor/memorylimiterprocessor)
> - [Batch Processor](https://github.com/open-telemetry/opentelemetry-collector/tree/main/processor/batchprocessor)

#### exporters（資料輸出器）

```yaml
exporters:
  # Traces → Tempo
  otlp/tempo:
    endpoint: tempo:4317          # Docker 網路內部，用容器名稱 + port
    tls:
      insecure: true              # 內部網路不需要 TLS

  # Metrics → Prometheus
  prometheusremotewrite:
    endpoint: http://prometheus:9090/api/v1/write
    tls:
      insecure_skip_verify: true
    resource_to_telemetry_conversion:
      enabled: true               # 將 OTLP resource attributes 轉為 Prometheus labels

  # Logs → Loki
  loki:
    endpoint: http://loki:3100/loki/api/v1/push
    default_labels_enabled:
      exporter: false             # 不加 exporter label（減少基數）
      job: true                   # 加 job label
      level: true                 # 加 level label（嚴重程度）
      service_name: true          # 加 service_name label
```

`otlp/tempo` 中的 `/tempo` 是 exporter 的**命名識別碼**（component ID），當同一類型的 exporter 需要多個實例時，用 `類型/名稱` 的格式區分。

> **官方參考**：
> - [OTLP Exporter](https://github.com/open-telemetry/opentelemetry-collector/tree/main/exporter/otlpexporter)
> - [Prometheus Remote Write Exporter](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/prometheusremotewriteexporter)
> - [Loki Exporter](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/lokiexporter)

#### service pipelines（流水線組裝）

```yaml
service:
  extensions: [health_check]
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp/tempo]

    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheusremotewrite]

    logs:
      receivers: [otlp]
      processors: [memory_limiter, resource, batch]  # logs 多了 resource processor
      exporters: [loki]
```

Pipeline 是 OTel Collector 的核心概念：每種遙測資料（traces/metrics/logs）都有獨立的流水線，資料在流水線中按順序經過 Receiver → Processors → Exporters。

注意 `logs` pipeline 的 processors 順序：`memory_limiter → resource → batch`。Processor 的順序很重要，`resource` 必須在 `batch` 之前，確保 attributes 在批次送出前就已經設定好。

> **官方參考**：[OTel Collector Pipelines](https://opentelemetry.io/docs/collector/configuration/)

---

### 7.3 `prometheus.yaml`（Prometheus）

```yaml
global:
  scrape_interval: 15s      # 每 15 秒抓一次所有 targets 的 metrics
  evaluation_interval: 15s  # 每 15 秒評估一次告警規則
  scrape_timeout: 10s       # 單次抓取超過 10 秒視為失敗

scrape_configs:
  - job_name: "prometheus"       # Prometheus 監控自己
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "otel-collector"   # 監控 OTel Collector 的健康狀態
    static_configs:
      - targets: ["otel-collector:8888"]

  - job_name: "tempo"            # 監控 Tempo 的健康狀態
    static_configs:
      - targets: ["tempo:3200"]
    metrics_path: /metrics

  - job_name: "loki"             # 監控 Loki 的健康狀態
    static_configs:
      - targets: ["loki:3100"]
    metrics_path: /metrics
```

`job_name` 是 Prometheus 為這組 targets 加上的 `job` label，可以在 Grafana 中用這個 label 篩選是哪個服務的 metrics。

透過 scraping 這些基礎設施服務的 metrics，可以在 Grafana 中觀察：
- OTel Collector 每秒處理多少 spans/metrics/logs
- Tempo 的查詢延遲
- Loki 的 ingestion 速率
- Prometheus 的 query 效能

> **官方參考**：[Prometheus Scrape Configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config)

---

### 7.4 `loki-config.yaml`（Loki）

#### 認證設定

```yaml
auth_enabled: false    # 關閉多租戶認證（開發/單機環境）
```

在生產多租戶環境需設為 `true`，並在請求 header 中帶 `X-Scope-OrgID`。

#### 伺服器設定

```yaml
server:
  http_listen_port: 3100    # HTTP API 端口（Grafana 連線用）
  grpc_listen_port: 9096    # gRPC 端口（內部元件用）
  log_level: info
```

#### common 區塊（Loki 3.x 簡化設定）

```yaml
common:
  instance_addr: 127.0.0.1
  path_prefix: /loki              # 所有路徑前綴
  storage:
    filesystem:
      chunks_directory: /loki/chunks    # 日誌資料儲存位置
      rules_directory: /loki/rules      # 告警規則儲存位置
  replication_factor: 1           # 單節點模式（不做複製）
  ring:
    kvstore:
      store: inmemory             # 使用記憶體做叢集協調（單節點用）
```

`replication_factor: 1` 配合 `kvstore: inmemory` 是單節點模式的標準設定。如果是多節點叢集，需要改用 `etcd` 或 `consul` 作為 KV store，並提高 replication_factor。

#### Schema 設定

```yaml
schema_config:
  configs:
    - from: 2024-01-01            # 從這個日期起使用以下 schema
      store: tsdb                 # 使用 TSDB 作為索引引擎（Loki 3.x 推薦）
      object_store: filesystem    # 物件儲存用本地檔案系統
      schema: v13                 # v13 是目前最新的 schema 版本
      index:
        prefix: index_
        period: 24h               # 每 24 小時建一個索引分片
```

Loki 的 Schema 代表索引的格式版本。`v13` 是 Loki 3.x 中推薦的版本，配合 `tsdb` 存儲引擎，比舊的 `boltdb-shipper` 更高效。

> **官方參考**：[Loki Schema Configuration](https://grafana.com/docs/loki/latest/configure/#schema_config)

#### 限制設定

```yaml
limits_config:
  reject_old_samples: true            # 拒絕太舊的日誌（防止亂序導入）
  reject_old_samples_max_age: 168h    # 超過 7 天的日誌不接受
  allow_structured_metadata: true     # 支援 OTLP structured metadata（OTel 必須開啟）
  volume_enabled: true                # 開啟 log volume 查詢功能
```

`allow_structured_metadata: true` 是接收 OTLP logs 的必要設定，沒有開啟這個選項，OTLP 格式的日誌可能無法正確被 Loki 接受。

> **官方參考**：[Loki Limits Configuration](https://grafana.com/docs/loki/latest/configure/#limits_config)

#### 查詢快取

```yaml
query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100    # 查詢結果快取最多使用 100MB 記憶體
```

啟用查詢結果快取可以大幅提升重複查詢的速度，特別是在 Grafana Dashboard 頻繁刷新時。

---

### 7.5 `tempo-config.yaml`（Tempo）

#### 伺服器設定

```yaml
server:
  http_listen_port: 3200    # Tempo API 端口（Grafana 查詢用）
  log_level: info
```

#### Distributor（接收 traces）

```yaml
distributor:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317    # 接收 OTel Collector 送來的 traces
        http:
          endpoint: 0.0.0.0:4318
```

Tempo 的 Distributor 是負責接收 traces 的元件，支援多種接收格式（OTLP、Jaeger、Zipkin 等）。本專案只啟用 OTLP。

#### Ingester（暫存）

```yaml
ingester:
  max_block_duration: 5m    # 每 5 分鐘將記憶體中的資料 flush 到磁碟
```

Ingester 負責把剛收到的 traces 暫存在記憶體（Write-Ahead Log）中，並定期寫入磁碟 block。縮短 `max_block_duration` 可以減少記憶體用量，但會增加磁碟 I/O。

#### 儲存設定

```yaml
storage:
  trace:
    backend: local              # 本地儲存（生產建議用 S3 或 GCS）
    wal:
      path: /var/tempo/wal      # Write-Ahead Log 路徑（容錯機制）
    local:
      path: /var/tempo/blocks   # 正式 block 儲存路徑
```

**WAL（Write-Ahead Log）** 是容錯機制：資料先寫到 WAL，再異步寫到正式儲存。如果服務崩潰，重啟後可以從 WAL 恢復尚未寫入的資料。

#### Compactor（壓縮與清理）

```yaml
compactor:
  compaction:
    block_retention: 48h    # Trace 資料保留 48 小時後自動刪除
```

Compactor 負責定期把多個小 block 合併成大 block（減少檔案數），並清理超過保留期限的資料。

#### Metrics Generator（最重要的功能之一）

```yaml
metrics_generator:
  registry:
    external_labels:
      source: tempo
      cluster: docker-compose
  storage:
    path: /var/tempo/generator/wal
    remote_write:
      - url: http://prometheus:9090/api/v1/write    # 把產生的 metrics 寫入 Prometheus
        send_exemplars: true                          # 附上 Exemplars（trace 連結）
  traces_storage:
    path: /var/tempo/generator/traces

overrides:
  defaults:
    metrics_generator:
      processors: [service-graphs, span-metrics, local-blocks]
      generate_native_histograms: both
```

**Metrics Generator 原理**：

1. Tempo 在儲存 Traces 時，同時分析 Span 之間的呼叫關係
2. 根據分析結果，自動生成以下 Prometheus metrics：
   - `traces_service_graph_request_total` — 服務間請求計數
   - `traces_service_graph_request_failed_total` — 服務間失敗請求計數
   - `traces_service_graph_request_server_seconds_*` — 服務處理時間分佈
   - `traces_spanmetrics_calls_total` — Span 呼叫計數（依服務/操作）
   - `traces_spanmetrics_duration_milliseconds_*` — Span 延遲分佈
3. 這些 metrics 透過 Remote Write 送到 Prometheus 儲存

**Exemplars** 是 Prometheus 的特殊功能：在 metric 的資料點上附加一個 `trace_id`，讓你可以從 metrics 圖表直接點擊跳轉到對應的 Trace。這需要 Prometheus 開啟 `--enable-feature=exemplar-storage`。

> **官方參考**：
> - [Tempo Metrics Generator](https://grafana.com/docs/tempo/latest/metrics-generator/)
> - [Prometheus Exemplars](https://prometheus.io/docs/prometheus/latest/exemplars/)

---

### 7.6 `datasources.yaml`（Grafana 資料來源）

#### Prometheus 資料來源

```yaml
- name: Prometheus
  type: prometheus
  uid: prometheus              # 唯一識別碼（其他設定用 uid 引用）
  url: http://prometheus:9090
  isDefault: true              # 設為預設資料來源
  jsonData:
    httpMethod: POST           # 使用 POST 方法（效能較好）
    exemplarTraceIdDestinations:
      - name: trace_id         # metrics 資料點上的 trace_id 欄位
        datasourceUid: tempo   # 點擊後跳轉到 Tempo
        urlDisplayLabel: "View Trace in Tempo"
```

`exemplarTraceIdDestinations` 設定了 Prometheus metrics 上的 Exemplar 連結目標：當 Prometheus 的 metric 資料點帶有 `trace_id`（由 Tempo Metrics Generator 的 `send_exemplars: true` 寫入），Grafana 會在圖表上顯示一個按鈕，點擊後直接跳到 Tempo 查看對應的 Trace。

#### Loki 資料來源

```yaml
- name: Loki
  type: loki
  uid: loki
  url: http://loki:3100
  jsonData:
    derivedFields:
      - datasourceUid: tempo
        matcherRegex: "traceID=(\\w+)"    # 在日誌內容中尋找 traceID
        name: TraceID
        url: "${__value.raw}"             # 點擊後跳轉到 Tempo 查看該 TraceID
        urlDisplayLabel: "View Trace in Tempo"
```

`derivedFields` 是 Loki 的「衍生欄位」功能：Grafana 會用正規表達式掃描每一條日誌行，如果匹配到 `traceID=...` 的格式，就在日誌旁邊顯示一個「View Trace in Tempo」的連結按鈕。

這個功能讓日誌和追蹤無縫連結。你的應用程式只需要在日誌中輸出 `traceID=<value>`，Grafana 就會自動解析並建立連結。

#### Tempo 資料來源

```yaml
- name: Tempo
  type: tempo
  uid: tempo
  url: http://tempo:3200
  jsonData:
    # Traces → Logs 連結
    tracesToLogsV2:
      datasourceUid: loki
      customQuery: true
      query: '{service_name="${__span.tags["service.name"]}"} |= "${__span.traceId}"'

    # Traces → Metrics 連結
    tracesToMetrics:
      datasourceUid: prometheus
      queries:
        - name: "Request Rate"
          query: 'rate(traces_spanmetrics_calls_total{$$__tags}[5m])'

    # Service Map 資料來源
    serviceMap:
      datasourceUid: prometheus

    # 在 Loki 中搜尋 Traces
    lokiSearch:
      datasourceUid: loki

    # 節點關係圖
    nodeGraph:
      enabled: true
```

**tracesToLogsV2**：在查看一個 Trace 的 Span 詳情時，Grafana 會自動生成並執行一個 LogQL 查詢，找出相同服務名稱和相同 traceId 的日誌。

**tracesToMetrics**：在 Trace 詳情中顯示相關 Metrics 圖表，讓你可以同時看到「這個請求的追蹤」和「這段時間這個服務的 RED metrics」。

**serviceMap**：Grafana 從 Prometheus 讀取 Tempo Metrics Generator 產生的 `traces_service_graph_*` metrics，繪製成互動式的服務關係圖。

> **官方參考**：[Grafana Tempo Datasource](https://grafana.com/docs/grafana/latest/datasources/tempo/)

---

## 8. 快速啟動

### 前置需求

- Docker Engine 24.0+
- Docker Compose v2.20+（`docker compose` 而非 `docker-compose`）

確認版本：
```bash
docker --version
docker compose version
```

### 啟動指令

```bash
# 進入專案目錄
cd D:\workspace\docker\observability

# 在背景啟動所有服務
docker compose up -d

# 查看啟動進度（等待所有服務 healthy）
docker compose ps
```

首次啟動需要 Pull Images，根據網路速度約需 2-5 分鐘。

### 等待健康狀態

```bash
# 持續監看服務狀態，直到所有服務都是 healthy
watch docker compose ps

# Windows 沒有 watch 指令，可以用：
docker compose ps
# 重複執行直到所有服務顯示 (healthy)
```

正常啟動後，`docker compose ps` 應顯示：

```
NAME             IMAGE                                     STATUS
grafana          grafana/grafana:12.3.2                    Up (healthy)
loki             grafana/loki:3.6.5                        Up (healthy)
otel-collector   otel/opentelemetry-collector-contrib:...  Up
prometheus       prom/prometheus:v3.9.1                    Up (healthy)
tempo            grafana/tempo:2.10.0                      Up (healthy)
```

### 停止服務

```bash
# 停止但保留資料（Volume）
docker compose down

# 停止並刪除所有資料（完全清除）
docker compose down -v
```

---

## 9. 驗證各服務健康狀態

### 9.1 Prometheus

```bash
# 確認 Prometheus 健康
curl http://localhost:9090/-/healthy
# 預期回應：Prometheus Server is Healthy.

# 在瀏覽器開啟 Prometheus UI
# http://localhost:9090
# 點選 Status → Targets，確認所有 targets 狀態為 UP
```

### 9.2 Loki

```bash
# 確認 Loki 就緒
curl http://localhost:3100/ready
# 預期回應：ready

# 查詢 Loki 中有哪些 labels
curl http://localhost:3100/loki/api/v1/labels
```

### 9.3 Tempo

```bash
# 確認 Tempo 就緒
curl http://localhost:3200/ready
# 預期回應：ready

# 查詢 Tempo 版本資訊
curl http://localhost:3200/version
```

### 9.4 OTel Collector

```bash
# 確認 Collector 健康
curl http://localhost:13133
# 預期回應：{"status":"Server available","upSince":"...","uptime":"..."}

# 查看 Collector 自身的 metrics
curl http://localhost:8888/metrics
```

### 9.5 Grafana

開啟瀏覽器至 `http://localhost:3000`，使用帳號 `admin` / 密碼 `admin` 登入。

登入後前往 **Connections → Data sources** 確認三個資料來源都顯示 **Data source connected and labels found.**

---

## 10. 應用程式整合指南

### 10.1 整合端點

你的應用程式需要設定 OTLP Exporter，指向 OTel Collector：

| 協定 | 端點 | 適用場景 |
|------|------|---------|
| gRPC | `localhost:4317` | 高效能、低延遲 |
| HTTP | `http://localhost:4318` | 簡單相容、防火牆友好 |

### 10.2 各語言整合範例

**Go（使用 `go.opentelemetry.io/otel`）**

```go
import (
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/sdk/trace"
)

exporter, _ := otlptracegrpc.New(ctx,
    otlptracegrpc.WithEndpoint("localhost:4317"),
    otlptracegrpc.WithInsecure(),
)
provider := trace.NewTracerProvider(trace.WithBatcher(exporter))
```

> **官方參考**：[OpenTelemetry Go SDK](https://opentelemetry.io/docs/languages/go/)

**Python（使用 `opentelemetry-sdk`）**

```python
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace.export import BatchSpanProcessor

exporter = OTLPSpanExporter(endpoint="localhost:4317", insecure=True)
provider.add_span_processor(BatchSpanProcessor(exporter))
```

> **官方參考**：[OpenTelemetry Python SDK](https://opentelemetry.io/docs/languages/python/)

**Java（Spring Boot，使用 `opentelemetry-spring-boot-starter`）**

```yaml
# application.yaml
otel:
  exporter:
    otlp:
      endpoint: http://localhost:4317
  service:
    name: my-spring-service
```

> **官方參考**：[OpenTelemetry Java Agent](https://opentelemetry.io/docs/languages/java/automatic/)

**Node.js**

```javascript
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-grpc')

const exporter = new OTLPTraceExporter({
  url: 'http://localhost:4317',
})
```

> **官方參考**：[OpenTelemetry JavaScript SDK](https://opentelemetry.io/docs/languages/js/)

### 10.3 日誌整合（讓日誌包含 TraceID）

為了讓 Grafana 的「Logs → Traces」連結功能運作，你的日誌中需要包含 `traceID`。

**建議的日誌格式**：

```json
{
  "timestamp": "2025-01-15T10:30:00Z",
  "level": "info",
  "message": "Processing order",
  "service.name": "order-service",
  "traceID": "4bf92f3577b34da6a3ce929d0e0e4736",
  "spanID": "00f067aa0ba902b7"
}
```

在 OpenTelemetry SDK 中，可以使用 `trace.SpanFromContext(ctx).SpanContext().TraceID()` 取得當前的 TraceID，並加入到日誌中。

### 10.4 設定必要的 Resource Attributes

OTLP 資料中需要包含以下 resource attributes，才能讓本 stack 的標籤和連結正確運作：

```
service.name       → 必填，服務名稱（如 "order-service"）
service.namespace  → 建議，命名空間（如 "production"）
service.version    → 建議，服務版本（如 "1.2.3"）
```

這些 attributes 會被 OTel Collector 的 `resource` processor 提取，作為 Loki 的 Labels，讓你可以在 Grafana 中用 `{service_name="order-service"}` 篩選日誌。

---

## 11. Grafana 使用說明

### 11.1 查看 Metrics

1. 點選左側選單 **Explore**
2. 選擇資料來源 **Prometheus**
3. 在查詢框輸入 PromQL，例如：
   ```promql
   # 查看 order-service 的每秒請求數
   rate(traces_spanmetrics_calls_total{service_name="order-service"}[5m])
   ```

### 11.2 查看 Logs

1. 點選左側選單 **Explore**
2. 選擇資料來源 **Loki**
3. 使用 LogQL 查詢，例如：
   ```logql
   # 查看 order-service 的所有 error 日誌
   {service_name="order-service", level="error"}

   # 搜尋包含特定關鍵字的日誌
   {service_name="order-service"} |= "timeout"
   ```

### 11.3 查看 Traces

1. 點選左側選單 **Explore**
2. 選擇資料來源 **Tempo**
3. 在 Search 模式下，可以篩選：
   - **Service Name**：`order-service`
   - **Span Name**：操作名稱
   - **Duration**：篩選超過特定時間的 traces
4. 或使用 TraceQL 語言：
   ```traceql
   # 找出 duration 超過 200ms 的 order-service traces
   { resource.service.name = "order-service" && duration > 200ms }
   ```

### 11.4 Service Map（服務關係圖）

1. 點選左側選單 **Explore**
2. 選擇資料來源 **Tempo**
3. 切換到 **Service Graph** tab
4. 會看到所有服務的互動關係圖，節點大小代表流量，顏色代表錯誤率

### 11.5 跨信號導航（最強大的功能）

**從 Metrics 找 Trace**：
- 在 Prometheus 圖表上，有 Exemplar 的資料點會顯示小鑽石圖示
- 點擊鑽石圖示 → 選擇 "View Trace in Tempo"

**從 Trace 找 Logs**：
- 在 Tempo Trace 詳情中，點擊任一 Span
- 在右側 panel 點擊 "Logs for this span"
- 自動開啟對應時間範圍與服務的 Loki 查詢

**從 Log 找 Trace**：
- 在 Loki 的日誌列表中，包含 `traceID=xxx` 的行
- 點擊行最右側的 "View Trace in Tempo" 連結

---

## 12. 常見問題排查

### Q1: `otel-collector` 啟動失敗，提示 "connection refused"

**原因**：OTel Collector 啟動時，後端服務（Tempo/Loki/Prometheus）尚未就緒。

**解決**：`depends_on` 已設定 `service_healthy`，通常等待幾秒後 Collector 會自動重試。如果持續失敗，檢查後端服務的 healthcheck 是否通過：
```bash
docker compose ps    # 確認 STATUS 欄顯示 healthy
docker compose logs loki    # 查看 Loki 的錯誤日誌
```

### Q2: Grafana 資料來源顯示 "Data source connection failed"

**排查步驟**：
```bash
# 確認服務名稱解析正常（在 Grafana 容器內）
docker compose exec grafana wget -q -O- http://prometheus:9090/-/healthy
docker compose exec grafana wget -q -O- http://loki:3100/ready
docker compose exec grafana wget -q -O- http://tempo:3200/ready
```

如果某個服務回應失敗，查看該服務的日誌：
```bash
docker compose logs --tail=50 tempo
```

### Q3: 送到 OTel Collector 的資料沒有出現在 Grafana

**步驟 1**：確認 OTel Collector 有收到資料
```bash
docker compose logs otel-collector | grep -i "traces\|metrics\|logs"
```

**步驟 2**：確認 Tempo/Loki/Prometheus 有收到資料
```bash
# 查看 Tempo 最近 trace
curl http://localhost:3200/api/search

# 查看 Loki 中有哪些 label values
curl http://localhost:3100/loki/api/v1/label/service_name/values
```

**步驟 3**：確認應用程式的 `service.name` resource attribute 有正確設定

### Q4: Tempo 的 Service Map 是空的

**原因**：Metrics Generator 尚未產生資料，或 Prometheus 中沒有 `traces_service_graph_*` metrics。

**確認方式**：
```bash
# 在 Prometheus 中搜尋 service graph metrics
curl "http://localhost:9090/api/v1/query?query=traces_service_graph_request_total"
```

如果沒有資料，確認：
1. Tempo config 中 `metrics_generator` 區塊設定正確
2. Prometheus 的 `--web.enable-remote-write-receiver` 有啟用
3. 確實有 traces 資料進入 Tempo（Service Map 需要有跨服務呼叫的 traces）

### Q5: 日誌出現但 Loki 的「View Trace in Tempo」連結不顯示

**原因**：日誌內容中沒有匹配 `traceID=(\w+)` 的正規表達式。

**解決**：確認應用程式的日誌格式，日誌內容中必須包含類似 `traceID=4bf92f3577b34da6a3ce929d0e0e4736` 的字串（注意大小寫）。

---

## 13. 生產環境升級建議

本 stack 的設定針對開發與測試環境最佳化。投入生產前，建議進行以下調整：

### 13.1 更換儲存後端

| 服務 | 開發設定 | 生產建議 |
|------|---------|---------|
| Tempo | `backend: local` | AWS S3 / GCS / Azure Blob |
| Loki | `filesystem` | AWS S3 / GCS + 對應 chunk store |
| Prometheus | 本地 TSDB | Thanos / Cortex / VictoriaMetrics（長期儲存）|

### 13.2 啟用 TLS

所有服務間的通訊都應使用 TLS 加密，並配置適當的憑證。

### 13.3 資料保留策略

```yaml
# Prometheus（目前設定）
--storage.tsdb.retention.time=15d    # 可依需求延長

# Tempo（目前設定）
block_retention: 48h                  # 生產建議至少 7-30 天

# Loki（透過 compactor 的 retention 設定）
limits_config:
  retention_period: 720h             # 保留 30 天
```

### 13.4 資源限制

在 `compose.yml` 中為每個服務加入資源限制，避免某個服務耗盡所有資源：

```yaml
services:
  otel-collector:
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          memory: 128M
```

### 13.5 安全性

- 修改 Grafana 預設密碼（`GF_SECURITY_ADMIN_PASSWORD`）
- 啟用 Loki 多租戶認證（`auth_enabled: true`）
- 將敏感設定（密碼、Token）移到 `.env` 檔案或 Docker Secrets
- 不要將 `4317`/`4318` 端口暴露在公開網路（應透過內部網路或 VPN 存取）

> **官方參考**：
> - [Loki Production Best Practices](https://grafana.com/docs/loki/latest/operations/storage/)
> - [Tempo Production Guidelines](https://grafana.com/docs/tempo/latest/operations/best-practices/)
> - [Prometheus Storage](https://prometheus.io/docs/prometheus/latest/storage/)

---

## 14. 官方文件參考

### OpenTelemetry

| 資源 | 連結 |
|------|------|
| OpenTelemetry 官方網站 | https://opentelemetry.io/ |
| OTel 核心概念 | https://opentelemetry.io/docs/concepts/ |
| OTel Collector 文件 | https://opentelemetry.io/docs/collector/ |
| OTel Collector 設定參考 | https://opentelemetry.io/docs/collector/configuration/ |
| OTel Collector Contrib GitHub | https://github.com/open-telemetry/opentelemetry-collector-contrib |
| OTLP 協定規格 | https://opentelemetry.io/docs/specs/otlp/ |
| Loki Exporter 文件 | https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/lokiexporter |
| Prometheus Remote Write Exporter | https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/prometheusremotewriteexporter |

### Prometheus

| 資源 | 連結 |
|------|------|
| Prometheus 官方文件 | https://prometheus.io/docs/introduction/overview/ |
| PromQL 查詢語言 | https://prometheus.io/docs/prometheus/latest/querying/basics/ |
| Metric Types | https://prometheus.io/docs/concepts/metric_types/ |
| Remote Write 設定 | https://prometheus.io/docs/prometheus/latest/configuration/configuration/#remote_write |
| Exemplars | https://prometheus.io/docs/prometheus/latest/exemplars/ |
| Scrape 設定參考 | https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config |

### Grafana Loki

| 資源 | 連結 |
|------|------|
| Loki 官方文件 | https://grafana.com/docs/loki/latest/ |
| LogQL 查詢語言 | https://grafana.com/docs/loki/latest/query/ |
| Loki 設定參考 | https://grafana.com/docs/loki/latest/configure/ |
| Schema 設定 | https://grafana.com/docs/loki/latest/configure/#schema_config |
| 接收 OTLP logs | https://grafana.com/docs/loki/latest/send-data/otel/ |

### Grafana Tempo

| 資源 | 連結 |
|------|------|
| Tempo 官方文件 | https://grafana.com/docs/tempo/latest/ |
| TraceQL 查詢語言 | https://grafana.com/docs/tempo/latest/traceql/ |
| Tempo 設定參考 | https://grafana.com/docs/tempo/latest/configuration/ |
| Metrics Generator | https://grafana.com/docs/tempo/latest/metrics-generator/ |
| Tempo 生產最佳實踐 | https://grafana.com/docs/tempo/latest/operations/best-practices/ |

### Grafana

| 資源 | 連結 |
|------|------|
| Grafana 官方文件 | https://grafana.com/docs/grafana/latest/ |
| Provisioning 說明 | https://grafana.com/docs/grafana/latest/administration/provisioning/ |
| Tempo 資料來源設定 | https://grafana.com/docs/grafana/latest/datasources/tempo/ |
| Loki 資料來源設定 | https://grafana.com/docs/grafana/latest/datasources/loki/ |
| Prometheus 資料來源設定 | https://grafana.com/docs/grafana/latest/datasources/prometheus/ |

### 延伸閱讀

| 資源 | 連結 |
|------|------|
| CNCF 可觀測性白皮書 | https://github.com/cncf/tag-observability/blob/main/whitepaper.md |
| Google SRE Book（可靠性工程） | https://sre.google/sre-book/monitoring-distributed-systems/ |
| OpenTelemetry Demo（完整範例） | https://opentelemetry.io/docs/demo/ |
| Grafana 官方 Blog | https://grafana.com/blog/ |

---

*文件版本：2026-02-12*
*維護者：請參考各服務官方文件獲取最新資訊*
