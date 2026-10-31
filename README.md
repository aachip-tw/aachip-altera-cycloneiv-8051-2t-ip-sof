# aachip-altera-cycloneiv-8051-2t-ip-sof
AACHIP 8051 2T FPGA SOF 韌體，適用於 Altera Cyclone IV EP4CE6E22C8N CPU Emulator 開發板，支援 GPIO Push-Pull、Timer2、UART 與外接 SPI ADC。

本專案提供 AACHIP 8051 2T 核心的 FPGA SOF 檔案，適用於 AACHIP ALTERA CYCLONE IV CPU_1 EMULATOR 開發板。

## 硬體需求

- FPGA：Altera Cyclone IV EP4CE6E22C8N
- 板上振盪器：40MHz
- 相容的 JTAG 下載器
- Quartus II Programmer

本版本使用 40MHz 輸入、20MHz 核心時脈。SOF 的 FPGA 型號及腳位配置須與開發板一致。

## 主要功能

- AACHIP 8051 2T 核心
- P0～P3 GPIO，支援 Push-Pull 等輸出模式
- Timer0、Timer1、Timer2
- Timer2 計數、擷取、自動重載與中斷
- UART0、UART1
- 雙 DPTR
- 外部中斷及擴充中斷控制
- 外接 SPI ADC 控制與資料讀取

ADC 功能需搭配相容的外部 ADC 晶片。

## SFR 位址

| 暫存器 | 位址 | 說明 |
|---|---|---|
| P3M1 | 0xF1 | P3 模式設定 |
| P3M0 | 0xF2 | P3 模式設定 |
| EIE | 0xE8 | 擴充中斷控制，支援位元操作 |

EIE 支援 SETB、CLR。以下範例啟用與關閉 EX2：

    SETB 0ECH
    CLR  0ECH

P3M1、P3M0 請使用位元組指令設定。本版本未使用 SFR 分頁。

## 下載與載入

1. 從本專案 Releases 下載 `aachip_8051_2t.sof`。
2. 接妥開發板電源與 JTAG 下載器。
3. 開啟 Quartus II Programmer，選擇對應的硬體。
4. 加入 SOF，勾選 Program/Configure。
5. 按下 Start，完成 FPGA 配置。

SOF 透過 JTAG 載入 FPGA，斷電後配置會消失；永久開機配置需要另行製作適用的配置記憶體檔案。

## 驗證狀態

- Quartus II 13.1 編譯成功
- Setup／Hold 時序檢查通過
- I/O max delay：5ns；min delay：0ns
- 已完成 RTL 模擬，涵蓋 GPIO、Timer2、ADC SPI 與 SFR 操作
## SFR Registers 說明

本版本使用直接位址 `0x80～0xFF` 存取 SFR，未使用 SFR 分頁。

- RW：軟體可讀寫。
- HW：硬體會更新暫存器或旗標。
- R：唯讀。
- 未列出的位址為保留位址，請勿使用。

### CPU 與記憶體控制

| 位址 | 名稱 | 功能 |
|---|---|---|
| 0x81 | SP | 堆疊指標，重置為 0x07 |
| 0x82 | DPL0 | DPTR0 低位元組 |
| 0x83 | DPH0 | DPTR0 高位元組 |
| 0x84 | DPL1 | DPTR1 低位元組 |
| 0x85 | DPH1 | DPTR1 高位元組 |
| 0x86 | DPS | bit0：0 選 DPTR0，1 選 DPTR1 |
| 0x87 | PCON | UART0 SMOD 倍率及軟體控制旗標 |
| 0x8E | CKCON | Timer 時脈與 MOVX 額外等待週期 |
| 0x8F | SPC_FNC | bit0 WRS：MOVX 寫入程式記憶體／XRAM 選擇 |
| 0x92 | MPAGE | MOVX @R0／@R1 的高位址 |
| 0xD0 | PSW | CPU 狀態字 |
| 0xE0 | ACC | 累加器 |
| 0xF0 | B | 乘除法及一般資料暫存器 |

PSW 位元：

| bit | 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
|---|---|---|---|---|---|---|---|---|
| 名稱 | CY | AC | F0 | RS1 | RS0 | OV | F1 | P |

RS1／RS0 選擇 R0～R7 的四組 register bank；P 由硬體依 ACC 奇偶同位更新。

CKCON 的 bit5／4／3 分別為 T2M／T1M／T0M，選擇對應 Timer 時脈；bit2:0 設定 MOVX 額外等待週期。

SPC_FNC.WRS＝0 時，MOVX 寫入 XRAM；WRS＝1 時，MOVX 寫入程式記憶體。寫入範圍受實際記憶體容量限制。

本 1T 版本保留 20MHz 核心時脈，未實作以 SPC_FNC 切換 10／20／40MHz 的功能。

### GPIO 資料與模式

| GPIO | 資料暫存器 | M1 暫存器 | M0 暫存器 |
|---|---|---|---|
| P0 | 0x80 | P0M1：0xE1 | P0M0：0xE2 |
| P1 | 0x90 | P1M1：0xE3 | P1M0：0xE4 |
| P2 | 0xA0 | P2M1：0xE5 | P2M0：0xE6 |
| P3 | 0xB0 | P3M1：0xF1 | P3M0：0xF2 |

GPIO 資料鎖存器重置為 0xFF，模式暫存器重置為 0x00。每個腳位由同位置的 M1、M0 位元設定：

| M1 | M0 | 模式 |
|---|---|---|
| 0 | 0 | 準雙向：輸出低／釋放高 |
| 0 | 1 | Push-Pull：主動輸出高、低 |
| 1 | 0 | 輸入：關閉主動輸出 |
| 1 | 1 | Open-Drain：輸出低／釋放高 |

一般 GPIO 讀取取得腳位值；讀改寫指令使用鎖存值。

FPGA 的準雙向模式沒有傳統 8051 的暫態強上拉電路。UART、Timer2 等替代輸出也會受到 GPIO 模式設定影響。

範例：將 P3 全部設定為 Push-Pull：

```asm
MOV 0F1H,#00H       ; P3M1
MOV 0F2H,#0FFH      ; P3M0
```

GPIO 模式暫存器請使用位元組指令。`SETB 0F1H`、`SETB 0F2H` 操作的是 B.1、B.2，不是 P3M1、P3M0。

### Timer0／Timer1

| 位址 | 名稱 | 功能 |
|---|---|---|
| 0x88 | TCON | 啟動、溢位及外部中斷控制／旗標 |
| 0x89 | TMOD | Timer0／Timer1 模式設定 |
| 0x8A | TL0 | Timer0 低位元組 |
| 0x8B | TL1 | Timer1 低位元組 |
| 0x8C | TH0 | Timer0 高位元組 |
| 0x8D | TH1 | Timer1 高位元組 |

TCON 位元：

| bit | 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
|---|---|---|---|---|---|---|---|---|
| 名稱 | TF1 | TR1 | TF0 | TR0 | IE1 | IT1 | IE0 | IT0 |

- TR0／TR1：啟動對應 Timer。
- TF0／TF1：溢位旗標。
- IT0／IT1：1 為下降緣中斷，0 為低電位中斷。
- IE0／IE1：外部中斷旗標。

TMOD 高半位元組控制 Timer1，低半位元組控制 Timer0，各自為 `GATE、C/T、M1、M0`。

| M1:M0 | 模式 | 功能 |
|---|---|---|
| 00 | Mode 0 | 13-bit 計數 |
| 01 | Mode 1 | 16-bit 計數 |
| 10 | Mode 2 | 8-bit 自動重載 |
| 11 | Mode 3 | Timer0 分拆模式；Timer1 自身 Mode 3 停止 |

C/T＝0 使用內部 Timer 時脈，C/T＝1 計數外部 T 腳位下降緣。GATE＝1 時，還要求對應 INT 腳位為高才計數。

### Timer2

| 位址 | 名稱 | 功能 |
|---|---|---|
| 0xC8 | T2CON | Timer2 控制與旗標 |
| 0xC9 | T2MOD | 時脈輸出及上下數控制 |
| 0xCA | RCAP2L | 重載／擷取值低位元組 |
| 0xCB | RCAP2H | 重載／擷取值高位元組 |
| 0xCC | TL2 | 計數器低位元組 |
| 0xCD | TH2 | 計數器高位元組 |

以上暫存器重置均為 0x00。

| T2CON bit | 名稱 | 功能 |
|---|---|---|
| 7 | TF2 | 溢位旗標 |
| 6 | EXF2 | 外部事件旗標 |
| 5 | RCLK | UART0 接收使用 Timer2 baud 來源 |
| 4 | TCLK | UART0 傳送使用 Timer2 baud 來源 |
| 3 | EXEN2 | 允許 T2EX 下降緣觸發 |
| 2 | TR2 | 啟動 Timer2 |
| 1 | C/T2 | 0：內部計時；1：外部下降緣計數 |
| 0 | CP/RL2 | 1：擷取；0：自動重載 |

T2MOD：

- bit1 T2OE：允許 P1.0 時脈輸出。
- bit0 DCEN：允許上下數，由 P1.1 控制方向。

P1.0 為 T2，P1.1 為 T2EX。Timer2 中斷使用 IE.ET2，ISR 須由軟體清除 TF2／EXF2。

Timer2 用於 baud 或時脈輸出時，不產生一般 TF2 溢位中斷。讀取運行中的 16-bit 計數值時，兩次 byte 讀取不是原子快照。

### UART0／UART1

| 位址 | 名稱 | 功能 |
|---|---|---|
| 0x98 | SCON0 | UART0 控制與旗標 |
| 0x99 | SBUF0 | UART0 接收／傳送資料 |
| 0xC0 | SCON1 | UART1 控制與旗標 |
| 0xC1 | SBUF1 | UART1 接收／傳送資料 |

SCON0／SCON1 的位元格式相同：

| bit | 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
|---|---|---|---|---|---|---|---|---|
| 名稱 | SM0 | SM1 | SM2 | REN | TB8 | RB8 | TI | RI |

- SM0:SM1：00 同步移位、01 為 8-bit UART、10／11 為 9-bit UART。
- REN：接收允許。
- TI／RI：傳送／接收完成，需由軟體清除。
- 寫 SBUF 啟動傳送；讀 SBUF 取得接收資料，讀取不會自動清 RI。

UART0 使用 P3.0 RXD0／P3.1 TXD0。
UART1 使用 P1.2 RXD1／P1.3 TXD1。

### 中斷控制

| 位址 | 名稱 | 功能 |
|---|---|---|
| 0x91 | EXIF | 外部中斷 2～5 旗標 |
| 0xA8 | IE | 中斷使能 |
| 0xB8 | IP | 中斷優先權 |
| 0xD8 | EICON | UART1 倍率及擴充控制 |
| 0xE8 | EIE | 外部中斷 2～5 使能與優先權 |

IE／IP：

| bit | IE 使能 | IP 優先權 |
|---|---|---|
| 7 | EA：總使能 | 固定為 1 |
| 6 | ES1：UART1 | PS1 |
| 5 | ET2：Timer2 | PT2 |
| 4 | ES0：UART0 | PS0 |
| 3 | ET1：Timer1 | PT1 |
| 2 | EX1：INT1 | PX1 |
| 1 | ET0：Timer0 | PT0 |
| 0 | EX0：INT0 | PX0 |

優先權為 1 表示高層級，0 表示低層級。中斷來源使能與 EA 必須同時開啟。

EIE 支援 MOV、ANL、ORL，以及 SETB／CLR／CPL／JB／JNB：

| EIE bit | 名稱 | 功能 | 位元位址 |
|---|---|---|---|
| 0 | PX2 | INT2 優先權 | 0xE8 |
| 1 | PX3 | INT3 優先權 | 0xE9 |
| 2 | PX4 | INT4 優先權 | 0xEA |
| 3 | PX5 | INT5 優先權 | 0xEB |
| 4 | EX2 | INT2 使能 | 0xEC |
| 5 | EX3 | INT3 使能 | 0xED |
| 6 | EX4 | INT4 使能 | 0xEE |
| 7 | EX5 | INT5 使能 | 0xEF |

```asm
SETB 0AFH           ; EA = 1
SETB 0ECH           ; EX2 = 1
CLR  0ECH           ; EX2 = 0
```

`SETB 0E8H` 設定的是 PX2 優先權。EX2 的使能位元位址為 0xEC。

EXIF bit4～7 分別為 IE2～IE5，對應 P1.4～P1.7 的下降緣事件；由 ISR 使用 byte 指令清除旗標。

EICON bit7 為 SMOD1，控制 UART1 倍率。其餘擴充控制位包含 EPFI、PFI、WDTI、PWDI、EWDI；目前板級未提供實體電源失效偵測與 Watchdog 計時來源。

中斷向量：

| 來源 | 向量 |
|---|---|
| INT0 | 0x0003 |
| Timer0 | 0x000B |
| INT1 | 0x0013 |
| Timer1 | 0x001B |
| UART0 | 0x0023 |
| Timer2 | 0x002B |
| Power fail | 0x0033 |
| UART1 | 0x003B |
| INT2 | 0x0043 |
| INT3 | 0x004B |
| INT4 | 0x0053 |
| INT5 | 0x005B |
| Watchdog | 0x0063 |

### 外接 SPI ADC

| 位址 | 名稱 | 存取 | 功能 |
|---|---|---|---|
| 0xC2 | ADCRL | R | 結果低位元組 |
| 0xC3 | ADCRH | R | 結果高位元組 |
| 0xF8 | ADCCON0 | RW/HW | ADC 控制及完成旗標 |

ADCCON0：

| bit | 名稱 | 功能 |
|---|---|---|
| 7 | ADCF | 完成旗標 |
| 6 | ADCS | 啟動轉換 |
| 5:4 | 保留 | ADC 引擎未使用 |
| 3:0 | ADCHS | 通道碼 |

ADCS 由 0 寫為 1 啟動一次操作並清 ADCF；完成後硬體設定 ADCF，清除 ADCS。請使用 byte 指令操作 ADCCON0。

```asm
MOV 0F8H,#040H      ; 通道 0，開始操作
ADC_WAIT:
MOV A,0F8H
JNB ACC.7,ADC_WAIT
MOV A,0C2H
MOV R6,A            ; 結果低 byte
MOV A,0C3H
MOV R7,A            ; 結果高 byte
```

SPI 使用 Mode 3，傳輸順序為：

1. 傳送 0xA0 OR 通道碼。
2. 傳送 0x41。
3. 等待 MISO 為低，表示 Ready。
4. 傳送 0x00，接收高 byte。
5. 傳送 0x00，接收低 byte。

| ADC 訊號 | FPGA 腳位 |
|---|---|
| CS | PIN_32 |
| MOSI | PIN_31 |
| MISO | PIN_30 |
| SCLK | PIN_28 |

需使用符合此通訊流程的外部 ADC。有效解析度、通道定義與電壓換算依外接晶片規格決定；兩個結果 byte 不代表一定有 16-bit 有效解析度。

### 擴充匯出介面

| 位址 | 名稱 | 功能 |
|---|---|---|
| 0xF9 | EXPORTA | 擴充資料介面 A |
| 0xFA | EXDDRB | 介面 B 方向 |
| 0xFB | EXPORTB | 介面 B 資料 |
| 0xFC | EXDDRC | 介面 C 方向 |
| 0xFD | EXPORTC | 介面 C 資料 |
| 0xFE | EXDDRD | 介面 D 方向 |
| 0xFF | EXPORTD | 介面 D 資料 |

方向 bit 為 1 表示輸出，0 表示輸入。0xF8 已由 ADC 使用，不提供 DDRA。

這些暫存器提供 RTL 擴充介面，目前 SOF 未配置額外的板級 GSTC 腳位，不能當作已引出的實體 P4～P7 使用。板上 GPIO 請使用 P0～P3。

本 Repository 提供已編譯的 SOF；Verilog 原始碼是否提供，以實際發布內容為準。
