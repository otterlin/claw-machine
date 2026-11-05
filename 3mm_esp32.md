# 3mm 椴木板夾娃娃機製作與 ESP32 控制指南 (40×60 雷切版)

本指南對應 **`3mm椴木板夾娃娃機雷切圖生成器`** 所輸出的兩張 **$600\text{ mm} \times 400\text{ mm}$** 標準雷射切割圖檔，提供五金清單、組裝邏輯、線驅動繞線方式與 ESP32 完整控制程式。

---

## 一、 材料與五金零件清單 (BOM)

| 類別 | 零件名稱 | 規格與數量 | 用途說明 |
| :--- | :--- | :--- | :--- |
| **雷切板材** | **$3\text{ mm}$ 椴木合板 (Basswood)** | $600\text{ mm} \times 400\text{ mm} \times 2\text{ 張}$ | 整機外箱、天車滑塊、捲線輪與四爪連桿。 |
| **線性光軸** | **$8\text{ mm}$ 不鏽鋼圓棒 (光軸)** | 長度 $194\text{ mm} \times 2\text{ 支}$、$156\text{ mm} \times 2\text{ 支}$ | Y 軸前後軌道 ($194\text{ mm}$) 與 X 軸左右橫樑 ($156\text{ mm}$)。 |
| **驅動馬達** | **TT 雙軸減速馬達** | 減速比 $1:48$ 或 $1:120 \times 3\text{ 顆}$ | 分別負責 **X 軸左右**、**Y 軸前後**、**Z 軸升降** 捲線驅動。 |
| **夾爪馬達** | **N20 微型金屬減速馬達** | $6\text{ V}$，轉速 $60\sim 100\text{ RPM} \times 1\text{ 顆}$ | 安裝於**十字四連桿四爪機構**中心，驅動四向滑塊控制四爪同步張合。 |
| **控制核心** | **ESP32 開發板 + 雙馬達驅動模組** | ESP32 $\times 1$、TB6612FNG (或 L298N) $\times 2$ | 控制 4 顆直流減速馬達正反轉與 PWM 強弱爪力道。 |
| **傳動線材** | **高強度編織釣魚線 (PE 線)** | 線徑 $0.4\sim 0.6\text{ mm}$，約 $5\text{ m}$ | X/Y 閉迴路牽引線、Z 軸升降吊線與 N20 夾爪拉線。 |
| **緊固五金** | **M3 螺絲與防鬆螺母組** | $\text{M3}\times 12\text{ mm}$、$\text{M3}\times 20\text{ mm}$、$\text{M3}\times 30\text{ mm}$ | 疊合三層滑塊、固定 TT 馬達與四連桿轉軸關節。 |

---

## 二、 核心機構組裝重點

### 1. Y 軸三明治滑塊與 8mm 光軸 (`Y-SLD#1~6`)
* **三層疊合設計**：單片 $3\text{ mm}$ 木板與光軸接觸面太窄容易斜向卡死，請將 3 片 `Y-SLD` 用 $\text{M3}\times 16\text{ mm}$ 螺絲疊合鎖緊成 **$9\text{ mm}$ 厚滑塊**（左右各一組）。
* **潤滑技巧**：組裝時在 $8\text{ mm}$ 不鏽鋼光軸表面塗抹少許凡士林或石蠟，木質自潤滑孔即可順暢滑動。

### 2. N20 十字四爪四連桿夾爪 (`4JAW-X`、`4W-SLD` 與 `JAW1~4`)
* **十字正交骨架組裝**：取兩片 `4JAW-X#1` 與 `4JAW-X#2` 主體板，利用中央預留的 $3\text{ mm}$ 對插槽（Slot-in-Slot）呈 $90^\circ$ **十字正交卡合**，再將 `N20-LCK` 鎖緊環從上方套入，將 **N20 減速馬達** 穩固夾在十字中心。
* **四爪同步滑塊與連桿**：將圓形十字開槽的 **四向中心滑塊盤 (`4W-SLD`)** 套入十字骨架下方滑軌，使用 M3 螺絲與防鬆螺母將 **8 支連桿 (`LNK#1~8`)** 與 **4 支弧形爪指 (`JAW1~4`)** 分別安裝於前、後、左、右四個支臂上。當 N20 馬達捲動拉線拉動中心滑塊上下移動時，四支爪指即可 $360^\circ$ 同步張開與包覆合攏。

---

## 三、 線驅動（String Drive）繞線原理圖解

1. **X 軸與 Y 軸閉迴路拉線**：
   * 將釣魚線中段在 **TT 馬達三明治捲線輪 (`SPL-F` + `SPL-C`)** 上繞 $4\sim 5$ 圈並穿孔打結固定，確保不打滑。
   * 線的一端直接拉向滑塊正前方綁緊；另一端繞過對側的 **惰輪滑輪 (`PLY`)** 後再拉回滑塊後方綁緊，形成「一端收線、另一端放線」的雙向牽引迴路。
2. **Z 軸升降與 N20 合爪拉線**：
   * Z 軸 TT 馬達固定於 `X-CARRIAGE` 天車頂部，釣魚線穿過天車中心 $\varnothing 5\text{ mm}$ 圓孔向下懸吊十字四爪主體。

---

## 四、 ESP32 完整控制程式碼 (含 PWM 強弱爪模擬)

```cpp
#include <Arduino.h>

// ================= 腳位定義 (ESP32 + 2x TB6612FNG / L298N) =================
//搖桿與下爪按鈕
#define PIN_JOY_X   34
#define PIN_JOY_Y   35
#define PIN_BTN_GO  32

// 馬達1：X軸左右 (TT馬達)
#define M_X_IN1     25
#define M_X_IN2     26
#define M_X_PWM     27

// 馬達2：Y軸前後 (TT馬達)
#define M_Y_IN1     14
#define M_Y_IN2     12
#define M_Y_PWM     13

// 馬達3：Z軸升降 (TT馬達)
#define M_Z_IN1     16
#define M_Z_IN2     17
#define M_Z_PWM     15

// 馬達4：N20 十字四爪開合
#define M_CLAW_IN1  18
#define M_CLAW_IN2  19
#define M_CLAW_PWM  21

void driveMotor(int in1, int in2, int pwmPin, int speed) {
    if (speed > 0) {
        digitalWrite(in1, HIGH);
        digitalWrite(in2, LOW);
        analogWrite(pwmPin, constrain(speed, 0, 255));
    } else if (speed < 0) {
        digitalWrite(in1, LOW);
        digitalWrite(in2, HIGH);
        analogWrite(pwmPin, constrain(-speed, 0, 255));
    } else {
        digitalWrite(in1, LOW);
        digitalWrite(in2, LOW);
        analogWrite(pwmPin, 0);
    }
}

void executeClawSequence() {
    // 1. 張開四爪並下放 Z 軸吊線
    driveMotor(M_CLAW_IN1, M_CLAW_IN2, M_CLAW_PWM, -180);
    delay(400);
    driveMotor(M_CLAW_IN1, M_CLAW_IN2, M_CLAW_PWM, 0);

    driveMotor(M_Z_IN1, M_Z_IN2, M_Z_PWM, 200); // 下爪
    delay(1800);
    driveMotor(M_Z_IN1, M_Z_IN2, M_Z_PWM, 0);

    // 2. N20 強爪合攏 (PWM = 255 全力抓緊)
    driveMotor(M_CLAW_IN1, M_CLAW_IN2, M_CLAW_PWM, 255);
    delay(800);

    // 3. Z 軸收線上升，上升至頂部時轉為「弱爪力道 (PWM = 135)」
    driveMotor(M_Z_IN1, M_Z_IN2, M_Z_PWM, -220);
    delay(1200);
    driveMotor(M_CLAW_IN1, M_CLAW_IN2, M_CLAW_PWM, 135); // 轉弱爪維持夾持
    delay(700);
    driveMotor(M_Z_IN1, M_Z_IN2, M_Z_PWM, 0);

    // 4. 自動歸位至左前角出物口
    driveMotor(M_X_IN1, M_X_IN2, M_X_PWM, -200);
    driveMotor(M_Y_IN1, M_Y_IN2, M_Y_PWM, 200);
    delay(2000);
    driveMotor(M_X_IN1, M_X_IN2, M_X_PWM, 0);
    driveMotor(M_Y_IN1, M_Y_IN2, M_Y_PWM, 0);

    // 5. 鬆開四爪放下獎品
    driveMotor(M_CLAW_IN1, M_CLAW_IN2, M_CLAW_PWM, -180);
    delay(450);
    driveMotor(M_CLAW_IN1, M_CLAW_IN2, M_CLAW_PWM, 0);
}

void setup() {
    pinMode(PIN_BTN_GO, INPUT_PULLUP);
    int pins[] = {
        M_X_IN1, M_X_IN2, M_X_PWM,
        M_Y_IN1, M_Y_IN2, M_Y_PWM,
        M_Z_IN1, M_Z_IN2, M_Z_PWM,
        M_CLAW_IN1, M_CLAW_IN2, M_CLAW_PWM
    };
    for (int p : pins) pinMode(p, OUTPUT);
}

void loop() {
    int jx = analogRead(PIN_JOY_X) - 2048;
    int jy = analogRead(PIN_JOY_Y) - 2048;

    // 搖桿控制 X / Y 天車移動
    driveMotor(M_X_IN1, M_X_IN2, M_X_PWM, abs(jx) > 400 ? (jx > 0 ? 190 : -190) : 0);
    driveMotor(M_Y_IN1, M_Y_IN2, M_Y_PWM, abs(jy) > 400 ? (jy > 0 ? 190 : -190) : 0);

    // 按下取物按鈕觸發自動下爪流程
    if (digitalRead(PIN_BTN_GO) == LOW) {
        driveMotor(M_X_IN1, M_X_IN2, M_X_PWM, 0);
        driveMotor(M_Y_IN1, M_Y_IN2, M_Y_PWM, 0);
        executeClawSequence();
    }
    delay(20);
}
```