# ESLAB FINAL — 太鼓達人 × EMG 打擊

STM32L475（B-L475E-IOT01）WiFi 打擊感測 + PC 太鼓節奏遊戲 + WS2812 判定燈條。

GitHub: [jopplkm/ESLAB-FINAL-PROJECT](https://github.com/jopplkm/ESLAB-FINAL-PROJECT)

## 專案結構

```
ESLAB-FINAL-PROJECT/     ← 本 repo 根目錄（clone 後的資料夾名稱可自訂）
├── README.md
├── ESLAB_FINAL.ioc      # CubeMX 設定
├── ESLAB_FINAL.launch   # Debug 設定
├── Core/                # 韌體原始碼
└── tools/               # PC 端 Python
    ├── taiko_pygame.py
    ├── taiko_chart_editor.py
    ├── ESLAB_FINAL_chart.py
    ├── pc_adc_tcp_server.py
    ├── charts/
    └── music/
```

## 硬體接線

| 功能 | 開發板腳位 | 說明 |
|------|-----------|------|
| Don 打擊（紅） | Arduino **A0** (PC5) | EMG / 類比感測 CH0 |
| Ka 打擊（藍） | Arduino **A1** (PC4) | EMG / 類比感測 CH1 |
| WS2812 資料 | Arduino **D5** (PB4) | DIN；**不要接 D4**（TIM2） |
| WS2812 電源 | 5V + GND | 建議外接 5V，GND 與板子共地 |
| LED 顆數 | 19 顆 | 見下方 `APP_WS2812_NUM_LEDS` |

開機約 **10 秒校正**：兩路感測器保持放鬆。

## 通訊架構

```
PC (Python, TCP server :8002)
  ↑↓ 同一條 WiFi TCP
STM32 (WiFi client)
  → hit:a0 / hit:a1
  ← jdg:perfect / good / miss  → WS2812 綠 / 黃 / 紅
```

**順序：先開 PC 程式，再開發板。**

---

## 網路與 IP 設定（重要）

### 在哪裡改？

| 要改什麼 | 檔案位置 | 巨集名稱 |
|----------|----------|----------|
| WiFi 名稱 | `Core/Inc/main.h` | `APP_WIFI_SSID` |
| WiFi 密碼 | 同上 | `APP_WIFI_PASSWORD` |
| **PC 的 IP**（開發板連過來的目標） | 同上 | `APP_WIFI_REMOTE_IP0`～`IP3` |
| TCP 埠（須與 Python 一致） | 同上 | `APP_WIFI_REMOTE_PORT`（預設 `8002`） |

在 CubeIDE 或 Cursor 開啟 **`Core/Inc/main.h`**，找到 `#if APP_USE_WIFI` 區塊，例如：

```c
#define APP_WIFI_SSID "Zenfone10"
#define APP_WIFI_PASSWORD "你的密碼"
#define APP_WIFI_REMOTE_IP0 10U
#define APP_WIFI_REMOTE_IP1 110U
#define APP_WIFI_REMOTE_IP2 115U
#define APP_WIFI_REMOTE_IP3 229U
#define APP_WIFI_REMOTE_PORT 8002U
```

**改完一定要重新 Build 並燒錄開發板。**

### 怎麼查 PC 的 IP？

在 PC 開 PowerShell：

```powershell
ipconfig
```

找目前連線的 **Wi-Fi** 介面下的 **IPv4**，例如 `10.110.115.229`，對應填法：

| IPv4 每一段 | 巨集 |
|-------------|------|
| `10` | `APP_WIFI_REMOTE_IP0` |
| `110` | `APP_WIFI_REMOTE_IP1` |
| `115` | `APP_WIFI_REMOTE_IP2` |
| `229` | `APP_WIFI_REMOTE_IP3` |

或一行指令（介面名稱常為 `Wi-Fi`）：

```powershell
(Get-NetIPAddress -AddressFamily IPv4 -InterfaceAlias "Wi-Fi").IPAddress
```

### PC 端 Python 要改什麼？

通常 **不用改 IP**（PC 是伺服器，監聽本機所有介面）。預設：

```powershell
py -3 taiko_pygame.py    # 監聽 0.0.0.0:8002
```

只有 port 與韌體不同時才加 `--port 8002`，此時 `main.h` 的 `APP_WIFI_REMOTE_PORT` 也要改成相同數字。

### 注意事項

1. **PC 與開發板同一個 WiFi**（手機熱點請開 **2.4GHz**，板子不支援 5GHz only）。
2. **先跑** `taiko_pygame.py`，**再**開發板或按 RESET。
3. 換 WiFi 或重開路由器後，PC IP 可能變，要再查 `ipconfig` 並更新 `main.h`。
4. Windows 防火牆若擋連線，允許 Python 的 **私人網路 TCP 入站**（port 8002）。

### 連線除錯（UART1, 115200）

| 序列埠訊息 | 意義 |
|------------|------|
| `WIFI_Connect FAILED` | SSID/密碼或 2.4GHz 問題 |
| `STA IP x.x.x.x` | 板子已連上 WiFi |
| `TCP target a.b.c.d:8002` | 板子要連的 PC 位址（確認與 `main.h` 一致） |
| `TCP connected` | 成功 |
| `TCP FAILED` | PC 程式沒開、IP 錯、或防火牆 |

---

## STM32 韌體

### 編譯與燒錄

1. STM32CubeIDE → **File → Import → Existing Projects into Workspace**
2. Root directory 選本 repo 根目錄（含 `ESLAB_FINAL.ioc` 的資料夾）
3. Project Explorer 選 **ESLAB_FINAL** → Build → Run/Debug

### 其他韌體參數（`Core/Inc/main.h`）

| 巨集 | 說明 |
|------|------|
| `APP_WS2812_NUM_LEDS` | 燈條顆數（預設 19） |
| `APP_WS2812_USE_PWM_DMA` | `1` = TIM3+DMA；`0` = GPIO bit-bang |
| `APP_WS2812_HOLD_MS` | 判定燈亮多久（ms） |

---

## PC 端 Python

### 安裝

```powershell
cd tools
py -3 -m pip install -r requirements_taiko.txt
```

### 玩遊戲

```powershell
cd tools
py -3 taiko_pygame.py --chart charts\music2.json --audio music\music2.mp3
```

| 參數 | 說明 |
|------|------|
| `--tcp-debug` | 印 TCP 收發 |
| `--no-tcp` | 僅鍵盤 |
| `--skip-help` | 跳過標題 |
| `--no-jdg-out` | 不送判定給燈條 |

**鍵盤：** `Space` = don · `X` = ka · `C` = both · `R` = 重玩

### 編譜面

```powershell
py -3 taiko_chart_editor.py
```

`Ctrl+O` 開 MP3 · `Ctrl+E` 開 JSON · `Z`/`X`/`C` 放音符 · `Ctrl+S` 儲存

### TCP 測試

```powershell
py -3 pc_adc_tcp_server.py
```

與 `taiko_pygame.py` 不可同時佔用 8002。

## 判定與燈色

| PC 送出 | WS2812 |
|---------|--------|
| `jdg:perfect` | 綠 |
| `jdg:good` | 黃 |
| `jdg:miss` | 紅 |

亮 450 ms 後自動熄滅。

## 疑難排解

| 問題 | 檢查 |
|------|------|
| 連不上 PC | Python 先跑？`main.h` IP？防火牆？ |
| WiFi 失敗 | SSID/密碼、2.4GHz |
| 燈條異常 | DIN 接 D5；試 `APP_WS2812_USE_PWM_DMA 0` |
| CubeIDE 無法 Run | 選 **ESLAB_FINAL 專案**，不要只選 `main.c` |
| 沒音樂 | `--audio` 手動指定 MP3 |
