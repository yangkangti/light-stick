# light-stick
// --- 腳位定義 (R->11, G->10, B->9, 共陽極) ---
const int buttonPin = 2; // 按鈕接在 D2 (使用 INPUT_PULLUP: 按下=LOW, 釋放=HIGH)
const int redPin = 11;   // R 的 PWM 輸出使用 D11 腳位
const int bluePin = 9;   // B 的 PWM 輸出使用 D9 腳位
const int greenPin = 10; // G 的 PWM 輸出使用 D10 腳位

// --- 模式和狀態變數 ---
enum DisplayMode {
  MODE_GRADIENT_NORMAL, // 模式 1: 正常速度 RGB 漸變
  MODE_GRADIENT_FLASH,  // 模式 2: 閃爍 RGB 漸變
  MODE_GRADIENT_FAST,   // 模式 3: 快速 RGB 漸變
  MODE_OFF              // 按鈕按下時：熄滅 (過渡狀態)
};

// --- 控制變數 ---
DisplayMode currentMode = MODE_GRADIENT_NORMAL; 
int currentModeIndex = 0; // 0, 1, 2 對應三個主要模式 (M1, M2, M3)
bool isLEDOnForBlink = true; // 閃爍狀態
int currentR = 0, currentG = 0, currentB = 0; // 當前顏色

// --- 計時變數 ---
unsigned long previousMillis = 0;     
unsigned long pressStartTime = 0;       

// 💡 模式速度參數 (ms/全循環)
const float GRADIENT_SPEED_NORMAL = 8000.0; // 模式 1 速度: 8 秒完成一次全循環
const float GRADIENT_SPEED_FAST = 2000.0;   // 模式 3 速度: 2 秒完成一次全循環
const long BLINK_INTERVAL = 250;           // 模式 2 閃爍間隔: 250ms 亮 / 250ms 滅


// --- 函數原型 ---
void turnOffLED();
void setLEDColor(int r, int g, int b);
void setCorrectedColor(int dr, int dg, int db);
void colorWheel(int pos, int &r, int &g, int &b); 
void updateMode1Gradient(); 
void updateMode2FlashGradient();
void updateMode3FastGradient();
void handleModeSwitch(int rawButtonState);


void setup() {
  pinMode(redPin, OUTPUT);
  pinMode(bluePin, OUTPUT);
  pinMode(greenPin, OUTPUT);
  
  pinMode(buttonPin, INPUT_PULLUP); 
  
  turnOffLED(); 
}

void loop() {
  int rawButtonState = digitalRead(buttonPin);
  
  handleModeSwitch(rawButtonState);
  
  // 根據當前模式執行對應的更新函數
  switch (currentMode) {
    case MODE_GRADIENT_NORMAL:
      updateMode1Gradient(); 
      break;
      
    case MODE_GRADIENT_FLASH:
      updateMode2FlashGradient(); 
      break;
      
    case MODE_GRADIENT_FAST:
      updateMode3FastGradient(); 
      break;
      
    case MODE_OFF:
      break; // 保持熄滅狀態
  }
}

// ===================================================================
//              核心功能函數
// ===================================================================

/**
 * @brief 將所需的顏色 (Desired R, G, B) 轉換為實際輸出 (處理R/G腳位交換)。
 * @param dr 期望的紅色亮度 (0-255)
 * @param dg 期望的綠色亮度 (0-255)
 * @param db 期望的藍色亮度 (0-255)
 */
void setCorrectedColor(int dr, int dg, int db) {
  // 根據先前測試，要顯示綠色需要輸出到紅色的腳位，要顯示紅色需要輸出到綠色的腳位。
  // 我們呼叫底層 setLEDColor 時，將 Desired G (dg) 視為 R 的輸入，將 Desired R (dr) 視為 G 的輸入。
  // (dr, dg, db) -> (dg, dr, db)
  setLEDColor(dg, dr, db); 
  
  // 紀錄當前顏色，供 Mode 2 (閃爍) 使用
  currentR = dr;
  currentG = dg;
  currentB = db;
}


/**
 * @brief 根據位置 (0-765) 計算標準 RGB 顏色。
 * 順序為 R -> Y -> G -> C -> B -> M -> R
 */
void colorWheel(int pos, int &r, int &g, int &b) {
  pos = pos % 765; // 確保在 0-764 範圍內
  r = 0; g = 0; b = 0;
  
  if (pos < 255) {        // 0-254: R 漸弱, G 漸強
    r = 255 - pos;
    g = pos;
  } else if (pos < 510) { // 255-509: G 漸弱, B 漸強
    pos -= 255;
    g = 255 - pos;
    b = pos;
  } else {                // 510-764: B 漸弱, R 漸強
    pos -= 510;
    b = 255 - pos;
    r = pos;
  }
}


// ===================================================================
//              模式更新函數
// ===================================================================

/**
 * @brief 模式 1: 正常速度 RGB 漸變。
 */
void updateMode1Gradient() {
  unsigned long currentTime = millis();
  
  // 計算當前顏色位置 (pos)
  // (millis() / 速度) * 765, 確保 pos 隨時間在 0-764 循環
  int pos = (int)((currentTime / GRADIENT_SPEED_NORMAL) * 765.0); 
  
  int r, g, b;
  colorWheel(pos, r, g, b); // 計算期望的 RGB 顏色
  
  setCorrectedColor(r, g, b); // 輸出校準後的顏色
}

/**
 * @brief 模式 2: 閃爍 RGB 漸變。
 */
void updateMode2FlashGradient() {
  unsigned long currentTime = millis();
  
  // 1. 執行漸變邏輯 (使用正常速度的參數)
  int pos = (int)((currentTime / GRADIENT_SPEED_NORMAL) * 765.0); 
  int r, g, b;
  colorWheel(pos, r, g, b);
  
  // 2. 執行閃爍邏輯 (獨立於漸變速度)
  if (currentTime - previousMillis >= BLINK_INTERVAL) {
    previousMillis = currentTime;
    isLEDOnForBlink = !isLEDOnForBlink;

    if (isLEDOnForBlink) {
      setCorrectedColor(r, g, b); // 亮起時使用漸變色
    } else {
      turnOffLED(); // 熄滅
    }
  } else if (isLEDOnForBlink) {
     // 在亮燈期間，持續更新顏色以保持平滑過渡
     setCorrectedColor(r, g, b);
  }
}

/**
 * @brief 模式 3: 快速 RGB 漸變。
 */
void updateMode3FastGradient() {
  unsigned long currentTime = millis();
  
  // 計算當前顏色位置 (pos)，使用更快的速度
  int pos = (int)((currentTime / GRADIENT_SPEED_FAST) * 765.0); 
  
  int r, g, b;
  colorWheel(pos, r, g, b); // 計算期望的 RGB 顏色
  
  setCorrectedColor(r, g, b); // 輸出校準後的顏色
}

// ===================================================================
//              模式切換與底層函數
// ===================================================================

/**
 * @brief 處理按鈕的邏輯：短按切換 M1/M2/M3。
 */
void handleModeSwitch(int rawButtonState) {
  unsigned long currentTime = millis();
  
  if (rawButtonState == LOW) { // 按鈕被按住 (LOW)
    if (pressStartTime == 0) { // 剛按下
        pressStartTime = currentTime;
        currentMode = MODE_OFF; // 進入熄滅過渡狀態
        turnOffLED();
    }
  } else { // 按鈕被釋放 (HIGH)
    if (pressStartTime > 0) { // 剛從按下狀態釋放
      
      // 執行模式切換
      currentModeIndex = (currentModeIndex + 1) % 3;
      
      // 重設計時，確保新模式立即開始
      previousMillis = currentTime;
      isLEDOnForBlink = true; 

      // 模式啟動邏輯
      switch (currentModeIndex) {
          case 0: 
              currentMode = MODE_GRADIENT_NORMAL;
              break;
          case 1: 
              currentMode = MODE_GRADIENT_FLASH;
              break;
          case 2: 
              currentMode = MODE_GRADIENT_FAST;
              break;
      }
        
      // 重設按鈕狀態變數
      pressStartTime = 0;
    }
  }
}

/**
 * @brief 將所有 LED 輸出設為 0 (熄滅)。
 */
void turnOffLED() {
  // 共陽極邏輯：輸出 255 = 熄滅
  analogWrite(redPin, 255);
  analogWrite(greenPin, 255);
  analogWrite(bluePin, 255);
}

/**
 * @brief 設定 RGB LED 的顏色 (共陽極邏輯)。
 * 此函數與硬體腳位對應，不做顏色修正。
 */
void setLEDColor(int r, int g, int b) {
  // 共陽極邏輯：將期望的亮度 (0-255) 反轉為實際的 PWM 輸出值 (255-0)
  analogWrite(redPin, 255 - r);
  analogWrite(greenPin, 255 - g);
  analogWrite(bluePin, 255 - b);
}
