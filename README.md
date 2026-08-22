<img width="284" height="511" alt="image" src="https://github.com/user-attachments/assets/6ea9c397-7583-4269-a91b-93b3786b61d4" />

<img width="500" height="500" alt="image" src="https://github.com/user-attachments/assets/867bc39c-1998-459a-b8d8-723a73ee9f88" />


第一階段：硬體介面與基礎環境建立

硬體架構：以 ESP32 為核心，透過 SPI 介面與 PN5180 高效能 NFC 前端晶片通訊。

腳位配置：

SPI 訊號：MOSI (GPIO 13)、MISO (GPIO 12)、SCK (GPIO 14)、NSS/CS (GPIO 15)。

控制訊號：BUSY (GPIO 5)、RST (GPIO 17)。

初始挑戰：原版函式庫將 ISO 15693 專屬的底層傳輸函式宣告為 private，導致自定義指令無法呼叫。

解法：修改 PN5180ISO15693.h，將 issueISO15693Command 提升為 public

並確保 .cpp 中對 0x40 與 0x41 等非標準自定義指令碼放行，打通底層通道。


第二階段：ISO 15693 讀取與目標卡號釐清

讀取機制：發送標準 ISO 15693 Inventory 指令

成功取得卡片回傳的 8-byte LSB 原始 UID

並在前端/序列埠正確解析呈現為 16 碼 Hex 格式（例如 E00781B8AF144207）

核心盲點：人類與標準系統習慣輸入正向高位元組在前（MSB）的目標卡號（例如 074214AFB88107E0）

但 NXP 晶片與魔術卡在暫存區的記憶體佈局帶有特殊的位元組對應與鏡像特性。


第三階段：魔術卡寫入規律的突破（Gen2 專屬特性）

經過多次序列埠除錯與對調測試

這張 ISO 15693 Gen2 魔術卡具備以下硬體特性：

讀取出來的 ID 第一碼一定要是 E0 

否則除非使用特殊裝置都無法讀取到卡片

而 PN5180 的特性正好就是其中一種特殊模組

可以無視第一碼非 E0 不可的限制進行讀取和寫入

略過區塊寫入：原本嘗試用 Flag 42（Write Single Block）寫入實體區塊

但卡片在接收 UID 更改時會進入忙碌狀態並回傳 No card detected!。實測證明，根本不需要 Flag 42。

NXP 自定義通道：真正能直接修改 UID 的是晶片特有的 0x40 與 0x41 底層指令。

位元組大對調：

輸入的標準 16 碼 UID 經 hexToBytes 切成 rawBytes[0..7] 後

必須將前半段與後半段直接大對調，才能完美抵銷晶片內部的鏡像特性：

C++

uint8_t part1[4] = { rawBytes[4], rawBytes[5], rawBytes[6], rawBytes[7] }; // 後半段送到前半

uint8_t part2[4] = { rawBytes[0], rawBytes[1], rawBytes[2], rawBytes[3] }; // 前半段送到後半


第四階段：最終完美寫入流程（大功告成）

當接收到使用者的目標卡號後，整個自動化執行的精準流程如下：

位元組智慧對應：自動將標準 MSB 卡號切換成魔術卡硬體唯一認得的排列順序（part1 與 part2）。

發送自定義指令：

透過 0x40 寫入 part1

透過 0x41 寫入 part2

EEPROM 永久保存：

執行 RF 磁場重置（setRF_off() 搭配短暫延遲後再 setRF_on()）

強制讓晶片將暫存區的 UID 永久寫入內部 EEPROM。

驗證成功：重新讀卡，完美呈現目標卡號 074214AFB88107E0，單次直通、乾淨俐落！
