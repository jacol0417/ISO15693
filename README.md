前言：<br/>
      為何要使用ESP32開發版加上PN5180模組實現讀寫ISO 15693卡片<br/>
      直接購買PM3等裝置來讀寫ISO 15693卡片不是更省事<br/>
      剛開始確實有這樣想過但是價格有點偏高便打消念頭<br/>
      畢竟最初只是為了研究ISO 14443A 也就是門禁IC卡讀寫和模擬<br/>
      前後陸續買了PN522、PN532模組測試後效果不太理想<br/>
      後來又購買PN5180模組測試模擬14443A卡片<br/>
      結果Arduino IDE針對PN5180的函示庫缺少模擬這部分<br/>
      也有想過改用樹莓派等開發版繼續測試模擬功能<br/>
      以經濟效益來看直接買14443A Gen2 CUID卡來重複使用更划算<br/>
      於是轉而研究將舊手機ROOT測試模擬14443A ID皆失敗告終<br/>
      只有將小米手機解鎖刷成eu ROM再安裝錢包功能成功模擬門禁卡<br/>
      原本工作場合只有使用14443A IC卡作為管理用途<br/>
      前陣子突然發現有少數幾張卡片是ISO 15693<br/>
      正好之前購買的PN5180模組有支援ISO 15693<br/>
      便繼續研究該如何利用ESP32和PN5180讀寫ISO 15693<br/>
      同時也完成撰寫手機程式管理卡號及寫入ISO 15693 Gen 2卡<br/>
      到這裡其實不就已經可以透過手機程式實現讀寫ISO 15693<br/>
      為何後續還要花費不少時間去研究讓PN5180寫入ISO 15693 Gen2卡<br/>
      問題就出在完成手機程式前的測試過程中陸續寫錯幾張卡號<br/>
      手機程式測試過程中ID碼寫錯順序導致E0不在第一碼<br/>
      也就是後面有提到第一碼必須為E0的規範<br/>
      除了使用PM3等裝置之外手機完全無法讀取到卡片<br/>
      要救回卡片就得寫入正確卡號也就是第一碼為E0開頭<br/>
      目前手頭上能夠有機會做到的只有PN5180模組<br/>
      所以才有這一篇完成ESP32和PN5180組合成功寫入正確卡號救回卡片<br/>      


<img width="284" height="511" alt="image" src="https://github.com/user-attachments/assets/6ea9c397-7583-4269-a91b-93b3786b61d4" />

<img width="500" height="500" alt="image" src="https://github.com/user-attachments/assets/867bc39c-1998-459a-b8d8-723a73ee9f88" />


# ESP32 + PN5180 ISO 15693 Gen2 魔術卡 (Magic Card) 複製與修復系統

本專案使用 **ESP32** 搭配 **PN5180** NFC/RFID 模組，實現對 **ISO 15693 (NFC-V / I-CODE)** 協定標籤及 **Gen2 魔術卡** 的底層讀寫、UID 竄改與急救修復功能。

透過解析 NXP 晶片底層私有指令與暫存器寫入規律，成功突破一般市售讀卡機無法竄改 ISO 15693 UID 的限制，並提供 Web API 介面進行遠端操作。

---

## 💡 專案亮點與技術突破

* **底層指令放行**：解除原廠 PN5180 ISO 15693 函式庫存取限制，直接發送 NXP 私有寫入指令 (`0x40` / `0x41`)。
* **解決鏡像對調 (Byte Swapping)**：破解 Gen2 魔術卡暫存器寫入時的前後位元組鏡像反轉特性，實現精準 UID 覆寫。
* **單次直通零報錯**：簡化傳統寫入流程，剔除冗餘的區塊寫入 (`Flag 42`) 指令，改用 **RF 磁場重置 (Reset RF)** 觸發 EEPROM 永久寫入。
* **卡片急救修復**：可修復因先前寫入錯誤、位元組紊亂或結構損壞而無法被普通讀卡機辨識的磚化魔術卡。

---

## 📌 ISO 15693 協定與 Gen2 魔術卡特性

### 1. ISO 15693 標準規範
* **UID 長度**：固定為 **8 Bytes (64 Bits)**。
* **製造商開頭 (E0)**：UID 最高位元組（MSB）固定為 **`0xE0`**（標準前綴）。
* **製造商代碼**：緊接在 `0xE0` 後的 1 Byte 為 IC 製造商代碼（如 NXP 通常為 `0x04`）。
* **傳輸順序**：空中介面（RF Interface）傳輸預設採用 **LSB (最低位元組優先)**。

### 2. Gen2 魔術卡專屬寫入機制
* **專屬自定義指令**：透過底層指令 **`0x40`**（寫入前半段 UID）與 **`0x41`**（寫入後半段 UID）進行覆寫。
* **暫存區前後對調**：寫入時必須將原始 8-byte UID 的**前半段 (Bytes 0~3) 與後半段 (Bytes 4~7) 互換**發送。
* **免 Flag 42 區塊寫入**：發送 `0x40`/`0x41` 後晶片會進入 Busy 狀態，此時若強制執行 `Flag 42` 會導致 `No card detected` 錯誤，應直接跳過。
* **RF 斷電生效**：發送完寫入指令後，必須關閉並重啟 PN5180 的 **RF 磁場 (RF Field Off/On)**，卡片重啟後才會將暫存區資料寫入 EEPROM 永久保存。

---

## 🛠️ 硬體腳位配置 (Pinout)

### 預設開發板配置 (ESP32-WROOM)

| PN5180 訊號 | ESP32 GPIO 腳位 | 說明 |
| :--- | :--- | :--- |
| **MOSI** | `GPIO 13` | SPI 資料輸出 |
| **MISO** | `GPIO 12` | SPI 資料輸入 |
| **SCK** | `GPIO 14` | SPI 時脈訊號 |
| **NSS / CS** | `GPIO 15` | SPI 晶片選擇 |
| **BUSY** | `GPIO 5` | 晶片忙碌狀態指示 (Input) |
| **RST / RESET** | `GPIO 17` | 硬體重置腳位 (Output) |

### 微型化開發板配置 (ESP32-C3 建議腳位)

| PN5180 訊號 | ESP32-C3 GPIO | 說明 |
| :--- | :--- | :--- |
| **MOSI** | `GPIO 6` | SPI FSPIHD |
| **MISO** | `GPIO 5` | SPI FSPID |
| **SCK** | `GPIO 4` | SPI FSPICLK |
| **NSS / CS** | `GPIO 7` | SPI FSPICS0 |
| **BUSY** | `GPIO 3` | 普通 GPIO (Input) |
| **RST / RESET** | `GPIO 2` | 普通 GPIO (Output) |

---

## 🔍 問題排除與經驗總結 (Troubleshooting)

1. **`issueISO15693Command` 無法呼叫**：
   * **原因**：C++ 標頭檔 (`PN5180ISO15693.h`) 將該函式設為 `private`。
   * **解法**：修改標頭檔將其移至 `public` 區塊，並確認 `.cpp` 未攔截 `0x40`/`0x41` 指令。
2. **寫入後 UID 顛倒**：
   * **原因**：NXP 寫入暫存器存在前後段鏡像特性。
   * **解法**：將 8-byte 原始陣列的 `Bytes[0..3]` 與 `Bytes[4..7]` 交換後再發送。
3. **寫入後出現 `No card detected` 警告**：
   * **原因**：發送 `0x40`/`0x41` 後執行冗餘的 `Flag 42` 區塊寫入。
   * **解法**：移除 `Flag 42` 指令，發送寫入指令後直接重置 RF 磁場。

---

## 📄 授權條款 (License)

本專案採用 [MIT License](LICENSE) 釋出，僅供學術研究與 RFID 技術探討使用。

---

## ⚙️ 核心演算法與寫入流程

```cpp
// Gen2 魔術卡 UID 寫入關鍵邏輯範例 (C++)

// 1. 陣列對調處理 (切分前後 4 位元組並互換)
uint8_t part1[4] = { rawBytes[4], rawBytes[5], rawBytes[6], rawBytes[7] }; // 後半段送到前面 (0x40)
uint8_t part2[4] = { rawBytes[0], rawBytes[1], rawBytes[2], rawBytes[3] }; // 前半段送到後面 (0x41)

// 2. 發送自定義寫入指令
nfc.issueISO15693Command(0x40, part1, sizeof(part1));
nfc.issueISO15693Command(0x41, part2, sizeof(part2));

// 3. 重置 RF 磁場以促使 EEPROM 寫入生效
nfc.setRF_off();
delay(150);
nfc.setRF_on();
