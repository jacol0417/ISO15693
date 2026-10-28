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

本專案使用 **ESP32** 搭配 **PN5180** NFC/RFID 模組，實現對 **ISO 15693 (NFC-V，包含 ICODE 系列標籤)** 及特定 **Gen2 魔術卡** 的底層讀寫、UID 改寫與異常 UID 修復功能。

透過 PN5180 底層 ISO 15693 傳輸介面發送特定 Gen2 Magic 卡支援的自訂 UID 寫入封包，成功在實測卡片上改寫 UID，並透過 RF 重啟與重新讀取救回先前 UID 寫錯、一般手機無法辨識的卡片。本專案亦描述以 Web API 介面進行操作的應用方向；公開檔案尚未包含完整 API 端點與可直接部署的實作。

---

## 💡 專案亮點與技術突破

* **底層指令放行**：調整 PN5180 ISO 15693 函式庫的存取權限，透過完整的自訂封包 `0x02 0xE0 0x09 0x40` / `0x02 0xE0 0x09 0x41` 傳送 UID 資料；`0x40` 與 `0x41` 是此類 Magic 卡自訂命令中的操作參數，並非 PN5180 本身的指令。
* **解決位元組排列 (Byte Swapping)**：區分 UID 顯示順序與 ISO 15693 封包中的 UID 資料順序，正確將 8-byte UID 反序後分成兩組 4-byte 寫入資料，避免 `E0` 前綴寫到錯誤位置。
* **精簡 UID 寫入流程**：在已驗證的 Gen2 測試卡上，不額外執行帶有 Option Flag（例如 Flags `0x42`）的一般區塊寫入；兩段自訂 UID 封包須維持 RF 連續供電，完成後再以 **RF 磁場重置 (Reset RF)** 與重新讀取確認新 UID 已生效。
* **卡片急救修復**：本專案曾在特定 Gen2 Magic 卡上，修復因 UID 位元組順序錯誤、`E0` 前綴不正確而無法被一般手機辨識的卡片；不代表能修復所有晶片損壞、鎖定或硬體故障。

---

## 📌 ISO 15693 協定與 Gen2 魔術卡特性

### 1. ISO 15693 標準規範
* **UID 長度**：固定為 **8 Bytes (64 Bits)**。
* **製造商開頭 (E0)**：UID 最高位元組（MSB）固定為 **`0xE0`**（標準前綴）。
* **製造商代碼**：緊接在 `0xE0` 後的 1 Byte 為 IC 製造商代碼（如 NXP 通常為 `0x04`）。
* **傳輸順序**：ISO 15693 的 UID 在傳輸封包中採 **低位元組先傳（Least Significant Byte First）**。顯示給使用者的 UID（例如 `E011223344556677`）與 NFC 讀取器返回的原始 UID 陣列順序可能相反；必須分別定義與處理。

### 2. Gen2 魔術卡專屬寫入機制
* **專屬自定義指令**：此 Gen2 Magic 卡使用自訂命令碼 **`0xE0`**，其參數 `0x09 0x40`、`0x09 0x41` 對應 UID 的兩組 4-byte 寫入資料。完整封包形式為 `02 E0 09 40 [4 Bytes]` 與 `02 E0 09 41 [4 Bytes]`，並非一般 ISO 15693 `0x21` Write Single Block 命令。
* **UID 位元組重排**：以顯示順序的 UID（`E0` 在第一個位元組）為基準，`0x40` 的資料為 `UID[7], UID[6], UID[5], UID[4]`；`0x41` 的資料為 `UID[3], UID[2], UID[1], UID[0]`。因此不只是交換前後兩組 4 Bytes，每一組內部也須反序。若來源已是低位元組在前的陣列，不能再次盲目反轉。
* **不額外執行 Flags `0x42` 的區塊寫入**：`0x42` 是請求旗標值（包含 Option Flag），不是獨立的「Flag 42 指令」；一般 Write Single Block 的命令碼為 `0x21`。本專案的 Gen2 DirectWrite 實測不需要額外進行這項區塊寫入。函式庫回報 `No card detected` 也可能只是特殊命令沒有產生函式庫預期的回應，不能直接認定標籤進入 Busy 狀態或寫入失敗。
* **RF 重新啟動與驗證**：兩段 UID 封包之間保持 RF 磁場不中斷；完成後才執行 **RF Field Off/On**，再重新 Inventory 讀取 UID，確認結果。這是本專案實測的有效流程，但目前沒有晶片內部證據能證實 EEPROM 恰好在 RF Reset 當下才完成寫入。特殊命令可能不回傳標準成功回應，須以重新讀取與比對為準。

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
| **MOSI** | `GPIO 6` | 自訂 SPI MOSI（GPIO Matrix） |
| **MISO** | `GPIO 5` | 自訂 SPI MISO（GPIO Matrix） |
| **SCK** | `GPIO 4` | 自訂 SPI SCK（GPIO Matrix） |
| **NSS / CS** | `GPIO 7` | 自訂 SPI CS（GPIO Matrix） |
| **BUSY** | `GPIO 3` | 普通 GPIO (Input) |
| **RST / RESET** | `GPIO 2` | 普通 GPIO (Output) |

> **ESP32-C3 備註**：以上沿用原專案的自訂 GPIO 配置，並非晶片的原生 FSPI 腳位功能對應；程式必須明確設定 SPI 的 MOSI/MISO/SCK/CS。`GPIO 2` 屬啟動設定（strapping）腳位，作為 PN5180 Reset 使用時須注意上電電位與啟動行為。

---

## 🔍 問題排除與經驗總結 (Troubleshooting)

1. **`issueISO15693Command` 無法呼叫**：
   * **原因**：C++ 標頭檔 (`PN5180ISO15693.h`) 將該函式設為 `private`。
   * **解法**：修改標頭檔將該函式移至 `public` 區塊，並確認實際 `.cpp` 可傳送完整 `0xE0` 自訂封包。已公開標頭檔的介面為 `issueISO15693Command(const uint8_t *cmd, uint8_t cmdLen, uint8_t **resultPtr)`；不能以 `(0x40, part1, sizeof(part1))` 直接呼叫。
2. **寫入後 UID 顛倒**：
   * **原因**：混淆 UID 顯示順序與寫入封包的低位元組先傳順序；目前無晶片內部證據證明是「NXP 暫存器鏡像」。
   * **解法**：明確定義來源 UID 的排列方式；若為 `E0` 在前的顯示順序，分別以 `UID[7..4]` 反序組成 `0x40` 資料，及以 `UID[3..0]` 反序組成 `0x41` 資料；寫入前先檢查封包位元組，再依實測流程執行。
3. **寫入後出現 `No card detected` 警告**：
   * **原因**：原測試流程額外進行帶有 Flags `0x42` 的區塊寫入後出現此警告；此外，PN5180 函式庫也可能因特殊命令沒有預期回應而回傳 `EC_NO_CARD`，不一定是卡片離開感應區。
   * **解法**：在已驗證的 DirectWrite 流程中省略不必要的額外區塊寫入；兩段自訂 UID 命令期間維持 RF 不中斷，之後執行 RF 重置並重新掃描、比對 UID。

---

## 📄 授權條款 (License)

本專案自行撰寫的程式與文件部分採用 [MIT License](LICENSE) 釋出，供技術研究與合法用途使用。第三方 `PN5180.h`、`PN5180ISO15693.h` 源自 Andreas Trappmann 的 PN5180 函式庫，原始檔案標示 **LGPL-2.1-or-later**；其授權不因本專案的 MIT 聲明而改變，應保留原始版權與授權文字。

---

## ⚙️ 核心演算法與寫入流程

> **注意**：下方是 **UID 位元組排列與封包建構範例**，不是可直接編譯執行的完整 PN5180 寫卡程式。公開的 `PN5180ISO15693.h` 雖已開放底層函式，但並未同時提供對應的完整 `.cpp` 傳送流程；必須以實際已驗證的版本處理回應、時序及 RF 控制。請勿將本範例直接用在未確認相容的卡片上。

```cpp
// Gen2 Magic ISO15693 UID 封包組合示例 (C++)
// uidDisplay 明確使用人類可讀的顯示順序（E0 在第一個位元組）
const uint8_t uidDisplay[8] = {
    0xE0, 0x11, 0x22, 0x33,
    0x44, 0x55, 0x66, 0x77
};

// 1. 組合完整自訂封包：Flags(02) + Command(E0) + 09 + 操作參數 + UID 資料
// 0x40：UID 顯示順序的後四個位元組，依傳輸順序反轉
const uint8_t frame40[8] = {
    0x02, 0xE0, 0x09, 0x40,
    uidDisplay[7], uidDisplay[6], uidDisplay[5], uidDisplay[4]
};

// 0x41：UID 顯示順序的前四個位元組，依傳輸順序反轉
const uint8_t frame41[8] = {
    0x02, 0xE0, 0x09, 0x41,
    uidDisplay[3], uidDisplay[2], uidDisplay[1], uidDisplay[0]
};

// 2. 上述範例形成的封包：
// frame40 = 02 E0 09 40 77 66 55 44
// frame41 = 02 E0 09 41 33 22 11 E0
//
// 需透過與 PN5180 函式庫簽章相容的底層方法發送完整封包。
// 公開標頭檔的函式宣告為：
// ISO15693ErrorCode issueISO15693Command(
//     const uint8_t *cmd, uint8_t cmdLen, uint8_t **resultPtr);
//
// 本 README 原示意流程為 0x40 -> 0x41；不同公開實作也有
// 0x41 -> 0x40 的案例。實際發送順序須以已驗證的卡片及程式為準，
// 兩段封包之間不得任意關閉 RF，也不應只靠回應碼判斷成功。

// 3. 完成兩段寫入後才進行 RF 重新啟動（原測試延遲 150 ms）
// 下列程式行為示意須放在實際可控制 PN5180 的程式環境中：
// nfc.setRF_off();
// delay(150);
// nfc.setRF_on();

// 4. 重新執行 Inventory，讀出 UID 並與目標 UID 比對；
//    寫入前亦應保存原始 UID，以便發生異常時進行修復。
```

> **驗證與限制**：UID 特殊寫入命令可能沒有一般函式庫預期的標準回應；沒有回應不等於寫入失敗。完成兩段寫入、RF 重新啟動後，必須重新尋卡並比對完整 8-byte UID。此流程只適用於已確認相容的 Gen2 Magic 卡；不要將其他專案的 `0x47` 等設定命令或不同世代的寫法直接混用。原始 README 也未附可直接編譯的完整 PN5180 `.cpp`，請以實際成功的程式版本為準。
