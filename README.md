<img width="284" height="511" alt="image" src="https://github.com/user-attachments/assets/6ea9c397-7583-4269-a91b-93b3786b61d4" />

<img width="500" height="500" alt="image" src="https://github.com/user-attachments/assets/867bc39c-1998-459a-b8d8-723a73ee9f88" />


# ESP32 + PN5180 ISO 15693 Gen2 魔術卡開發與除錯紀錄

## 一、 核心問題與技術瓶頸

1. **函式庫封鎖底層傳輸 (Library Access Control)**
   * **現象**：原版 PN5180 函式庫將 `issueISO15693Command` 設為 `private`，導致主程式無法直接發送 NXP 特有的 UID 修改指令。若使用通用的 `transceiveCommand` 則會因為 CRC 與封包格式錯誤導致卡片無回應。

2. **魔術卡寫入位元組反轉 (Byte Mirroring Issue)**
   * **現象**：寫入 UID 後，讀回來的卡號發生位元組排列混亂，寫入的結果與預期的目標卡號顛倒（例如傳送 `07 42 14 AF` 卻變成了 `AF 14 42 07`）。

3. **後續指令回報錯誤 (False Error Warnings)**
   * **現象**：在發送 UID 修改指令後，執行 `Flag 42` (寫入區塊資料) 會不斷跳出 `No card detected!` 的失敗訊息，但實測卡片 UID 卻已經成功被變更。

---

## 二、 關鍵解決方案與技術重點

1. **修改函式庫開通底層通道**
   * **解決方式**：修改 `PN5180ISO15693.h`，將 `issueISO15693Command` 宣告改為 `public`，允許主程式直接調用 ISO 15693 專屬的傳輸封包。
   * **重點**：確保 `.cpp` 中對自定義指令碼 `0x40` 與 `0x41` 放行，避免被函式庫前端誤判為無效指令而攔截。

2. **掌握 Gen2 魔術卡「前後對調」的寫入規律**
   * **解決方式**：經過測試，發現不需要逐字元迴圈反轉，直接將 8-byte 原始陣列 (`rawBytes[0..7]`) 的**前半段與後半段大對調**，即可完美抵銷晶片內部的鏡像特性：
     ```cpp
     uint8_t part1[4] = { rawBytes[4], rawBytes[5], rawBytes[6], rawBytes[7] }; // 後半段送到前面 (0x40)
     uint8_t part2[4] = { rawBytes[0], rawBytes[1], rawBytes[2], rawBytes[3] }; // 前半段送到後面 (0x41)
     ```

3. **簡化寫入流程，剔除冗餘指令 (Eliminate Flag 42)**
   * **解決方式**：`0x40` 與 `0x41` 為直接覆寫 UID 暫存的核心指令。當晶片接收到這兩組指令後會進入寫入 busy 狀態，導致後續的 `Flag 42` 存取失敗。
   * **重點**：直接**刪除/註解掉 Flag 42 區塊寫入**，發送完 `0x40`/`0x41` 後直接進行 **RF 磁場重置（Reset RF）**，強制卡片將暫存寫入 EEPROM，成功達到單次直通、零報錯的寫入邏輯。

---

## 三、 系統硬體腳位配置 (以 .NET nanoFramework 為例)

| PN5180 訊號 | ESP32 GPIO 腳位 | 說明 |
| :--- | :--- | :--- |
| **MOSI** | `GPIO 13` | SPI 資料輸出 |
| **MISO** | `GPIO 12` | SPI 資料輸入 |
| **SCK** | `GPIO 14` | SPI 時脈訊號 |
| **NSS / CS** | `GPIO 15` | SPI 晶片選擇 |
| **BUSY** | `GPIO 5` | 晶片忙碌狀態指示 |
| **RST / RESET** | `GPIO 17` | 硬體重置腳位 |

---

## 四、 最終最佳化寫入流程 (4 步驟)

1. **尋卡 (Inventory)**：讀取原始 ISO 15693 卡片 UID。
2. **陣列調補**：將目標 16 碼 Hex 轉為 8-byte 陣列，並執行 `part1` (後半段) 與 `part2` (前半段) 切分。
3. **發送自定義指令**：
   * 發送 `0x40` 寫入 `part1`
   * 發送 `0x41` 寫入 `part2`
4. **重置 RF 磁場**：呼叫 `setRF_off()` 延遲 150ms 後呼叫 `setRF_on()`，卡片重啟即永久生效。

---

# ISO 15693 協定與 Gen2 魔術卡核心特性彙整

### 1. ISO 15693 標準規範特性

* **UID 格式與固定長度**：UID 固定為 **8 Bytes (64 Bits)** 長度。
* **製造商開頭 (E0)**：UID 的最高位元組（MSB）必須固定為 **`0xE0`**，此為 ISO 15693 規範的標準前綴識別碼。
* **IC 製造商代碼**：緊接在 `0xE0` 後的 1 Byte 為 IC 製造商代碼（例如 NXP 晶片通常為 `0x04`）。
* **位元組傳輸順序**：空中無線介面（RF Interface）傳輸預設採用 **LSB (最低位元組優先)** 順序傳輸。

---

### 2. Gen2 魔術卡專屬寫入特性

* **專屬自定義指令**：不使用標準的區塊寫入指令，而是透過 NXP 特有的底層自定義指令 **`0x40`**（寫入前半段 UID）與 **`0x41`**（寫入後半段 UID）進行覆寫。
* **暫存區鏡像對調**：晶片內部的寫入暫存器具有位元組反轉特性，寫入時需將原始 8-byte UID 的**前半段 (Bytes 0~3) 與後半段 (Bytes 4~7) 互換**發送。
* **免 Flag 42 區塊寫入**：不需要發送通常用於改寫 Block 的 `Flag 42` 指令，發送該指令反而會因晶片 Busy 導致系統跳出 `No card detected` 錯誤。
* **RF 斷電生效 (Reset RF)**：發送完 `0x40`/`0x41` 後，必須關閉並重新啟動 PN5180 的 **RF 磁場 (RF Field Off/On)**，卡片重啟後才會將暫存區資料寫入 EEPROM 永久保存。
