# STM32 FSM-Based LED Control System with EXTI Interrupts (Lab 3)

本專案基於 **NUCLEO-G474RE** 開發板，實作一個具備非阻塞式（Non-blocking）計時與外部中斷（EXTI）機制的有限狀態機（Finite State Machine, FSM）LED 控制系統。

---

## 📌 功能特點 (Features)

* **有限狀態機 (FSM) 架構**：清晰劃分 `NORMAL`、`WARNING` 與 `EMERGENCY` 三種系統運行狀態。
* **非阻塞式時間管理**：完全擺脫 `HAL_Delay()`，使用 `HAL_GetTick()` 實現精準且零卡頓的 LED 閃爍與逾時控制。
* **即時中斷響應 (EXTI)**：緊急停止（`PC13`）採用硬體外部中斷驅動，確保微秒級的極速安全響應。
* **自動逾時返回**：`WARNING` 狀態下若無任何操作，達 5 秒後自動返回 `NORMAL` 狀態。
* **狀態切換零延遲**：於狀態轉移瞬間重置時間戳記與強制輸出，實現按鍵即時點亮反應。

---

## ⚙️ 狀態轉移與燈號邏輯 (State Transition Logic)

| 狀態 (State) | LED 輸出狀態 | 閃爍頻率 / 動作 | 狀態切換條件 |
| :--- | :--- | :--- | :--- |
| **STATE_NORMAL** | 綠燈 (PC8) | 1 Hz 閃爍 (500 ms 切換) | 按下 **MODE** 按鈕 (PC9) $\rightarrow$ 進入 `WARNING` |
| **STATE_WARNING** | 黃燈 (PC5) | 2 Hz 閃爍 (250 ms 切換) | 按下 **MODE** 或 **5 秒逾時** $\rightarrow$ 返回 `NORMAL` |
| **STATE_EMERGENCY** | 紅燈 (PC6) | 常亮 (Solid ON) | 按下 **STOP** 按鈕 (PC13 中斷觸發) 進入<br>*（僅能透過硬體 Reset 鍵復位）* |

---

## 🛠️ 硬體腳位配置 (Hardware Configuration)

### 1. GPIO 與中斷 (Peripherals)
* **STOP / RESET 按鈕**：`PC13`（EXTI 外部中斷，上升緣觸發，內部下拉 `PULLDOWN`）
* **MODE 按鈕**：`PC9`（GPIO 輸入，內部上拉 `PULLUP`，輪詢 Polling + 150ms 非阻塞防彈跳）
* **Green LED**：`PC8`（GPIO 推挽輸出）
* **Yellow LED**：`PC5`（GPIO 推挽輸出）
* **Red LED**：`PC6`（GPIO 推挽輸出）

### 2. NVIC 中斷優先權 (Interrupt Priority)
* **`EXTI15_10_IRQn`**：Preemption Priority = 1, SubPriority = 0

---

## 📁 專案架構與關鍵程式碼說明 (Software Architecture)

### 1. EXTI 中斷服務機制 (Interrupt Service Routine)
為了確保系統安全與即時性，`PC13` 按鈕在中斷服務函式中僅執行「設定 Flag」動作，避免在中斷內發生阻塞：

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == GPIO_PIN_13)
    {
        STOP_event = 1; // 幾微秒內拉高旗標，交由 main 迴圈處理
    }
}
