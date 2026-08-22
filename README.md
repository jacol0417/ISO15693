# ISO15693
ESP32+PN5180

函式庫修正方式 (PN5180ISO15693 或對應的標頭檔)
原版的 PN5180 函式庫通常會把底層指令發送函式（如 issueISO15693Command）宣告為 private 或 protected，這會導致你在主程式寫入 0x40 和 0x41 時發生編譯錯誤。

1. 打開標頭檔（例如 PN5180ISO15693.h 或主類別標頭檔）
找到宣告 ISO 15693 專屬指令發送的方法，將其存取權限從 private 改為 public：

class PN5180ISO15693 {
public:
    // 🔍 必須確保這個函式宣告在 public 區塊底下！
    ISO15693ErrorCode issueISO15693Command(uint8_t *cmd, uint8_t cmdLen, uint8_t **responsePtr);

    // 其他原本的公開函式...
    void setupRF();
    void setRF_off();
    void setRF_on();
    // ...
};

避免 CRC 與 Framing 錯誤：如果使用一般通用的 transceiveCommand 去送 0x40 / 0x41，PN5180 的硬體層常會自動附加或搞錯 ISO 15693 特有的 EOF/SOF 與 CRC 格式，導致卡片直接無回應或回傳錯誤。

精準控制自定義指令：透過公開的 issueISO15693Command，函式庫才能正確處理晶片底層的傳輸協議時序，讓帶有 0x02, 0xE0, 0x09, 0x40 的自定義 payload 完美送達魔術卡晶片中。


<img width="805" height="694" alt="image" src="https://github.com/user-attachments/assets/9a9c20e7-3692-4907-9de4-e3e6c8815adf" />

ISO 15693 Gen2 魔術卡讀寫邏輯架構
1. 讀取邏輯 (Inventory & Read)
尋卡 (Inventory)：送出標準 ISO 15693 尋卡指令。

取得 UID：接收卡片回傳的 8-byte LSB 原始 UID。

格式化：將讀回來的 8 bytes 轉為 16 碼 Hex 字串（例如：E00781B8AF144207）。

2. 寫入邏輯 (Gen2 魔術卡專屬單次直通)
當接收到標準 16 碼 MSB 目標卡號（例如：074214AFB88107E0）時，底層的轉換與執行步驟如下：

步驟 A：位元組切割與通道對應（解決晶片反轉特性）

// 假設透過 hexToBytes 取得 8 bytes 原始陣列
byte[] rawBytes = HexToBytes(targetUidHex);

// 實測對應：將後半段與前半段大對調，完美抵銷晶片內部的鏡像特性
byte[] part1 = new byte[] { rawBytes[4], rawBytes[5], rawBytes[6], rawBytes[7] }; 
byte[] part2 = new byte[] { rawBytes[0], rawBytes[1], rawBytes[2], rawBytes[3] };

步驟 B：發送 NXP 自定義 UID 覆寫指令 (0x40 與 0x41)
透過 SPI 與 PN5180 依序送出指令：

發送 0x40：封包格式為 [0x02, 0xE0, 0x09, 0x40, part1... ]

發送 0x41：封包格式為 [0x02, 0xE0, 0x09, 0x41, part2... ]
(註：此時晶片已接收暫存修改，不需執行會報錯的 Flag 42 區塊寫入)

步驟 C：重置 RF 磁場使 EEPROM 永久保存
指令發送完成後，透過開關 RF 磁場觸發晶片內部將暫存寫入 EEPROM：

nfc15693.SetRF_off();
Thread.Sleep(150);
nfc15693.SetRF_on();
Thread.Sleep(100);






