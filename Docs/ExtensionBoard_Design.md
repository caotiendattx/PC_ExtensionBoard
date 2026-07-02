# Tài liệu Thiết kế — PC Extension Board

**Nền tảng:** SKLH-E61-IO_CB / KBLH-E61 — COM Express **Type 6** Carrier Board
**Module:** COM Express Basic/Compact Type 6 (Intel Skylake-H / Kaby Lake-H class), CPU 4+4
**PCB:** 125 × 95 mm · nguồn vào DC 11–23 V · Power status: No Deepsleep / No S0ix / No M3
**Tham chiếu:** reference schematic `kblh-e61_cb_v20` (55 sheet)

> Tài liệu này mô tả **cách thiết kế** carrier board: kiến trúc hệ thống, connector trung tâm,
> kiến trúc nguồn/clock/reset, chi tiết từng bus & giao tiếp, và các **design pattern** (mẫu mạch)
> lặp lại xuyên suốt thiết kế. Phần cuối mô tả cách tổ chức project KiCad và quy ước thư viện để
> tái tạo lại thiết kế.

---

## Mục lục

1. [Giới thiệu & phạm vi](#1-giới-thiệu--phạm-vi)
2. [Khái niệm nền tảng: COM Express là gì](#2-khái-niệm-nền-tảng-com-express-là-gì)
3. [Kiến trúc tổng thể](#3-kiến-trúc-tổng-thể)
4. [Connector trung tâm — COMe Type 6 (CN3)](#4-connector-trung-tâm--come-type-6-cn3)
5. [Kiến trúc nguồn (Power architecture)](#5-kiến-trúc-nguồn-power-architecture)
6. [Kiến trúc Clock](#6-kiến-trúc-clock)
7. [Kiến trúc Reset & Power sequencing](#7-kiến-trúc-reset--power-sequencing)
8. [Các bus & giao tiếp — chi tiết](#8-các-bus--giao-tiếp--chi-tiết)
9. [Design patterns (mẫu mạch lặp lại)](#9-design-patterns-mẫu-mạch-lặp-lại)
10. [Bản đồ sheet chức năng (PDF → KiCad)](#10-bản-đồ-sheet-chức-năng-pdf--kicad)
11. [Tổ chức project KiCad & quy ước thư viện](#11-tổ-chức-project-kicad--quy-ước-thư-viện)
12. [Bảng tra cứu](#12-bảng-tra-cứu)
13. [Checklist thiết kế](#13-checklist-thiết-kế)

---

## 1. Giới thiệu & phạm vi

**Extension Board** ở đây là một **carrier board** (bo đế) cho module tính toán chuẩn **COM Express
Type 6**. Toàn bộ CPU/chipset/RAM/BIOS nằm trên **module** cắm rời; carrier board chịu trách nhiệm:

- Cấp **nguồn** cho module và toàn bộ ngoại vi.
- **Fan-out** (chia/định tuyến) các giao tiếp tốc độ cao từ module ra các connector thực tế
  (PCIe → mini-PCIe/M.2/MXM, USB, SATA, DisplayPort/HDMI/VGA, LVDS/eDP, LAN…).
- Tích hợp các thiết bị "board-level": **SuperIO** (COM/UART, WDT, GPIO), **codec audio**,
  **TPM**, **BIOS SPI flash**, **debug port 80**, front-panel, quạt, RTC…

Triết lý thiết kế: **module giữ phần "logic số phức tạp", carrier giữ phần "I/O + nguồn + cơ khí"**.
Điều này cho phép cùng một carrier hoạt động với nhiều đời module khác nhau, và ngược lại.

---

## 2. Khái niệm nền tảng: COM Express là gì

COM Express (PICMG COM.0) là chuẩn **Computer-on-Module**. Module giao tiếp với carrier qua **2
connector 220 chân** (tổng 440 chân), chia thành **4 hàng A, B, C, D**:

| Connector | Hàng | Nhóm tín hiệu chính |
|-----------|------|---------------------|
| **AB** (Row A, B) | 220 chân | USB 2.0, PCIe x1 (lanes 0–5), SATA, LPC, HDA, LVDS/VGA, SMBus/I2C, GPIO, nguồn 12V/5VSB/RTC, control/power-mgmt |
| **CD** (Row C, D) | 220 chân | USB 3.0 SuperSpeed, PCIe x1 (lanes 6–7), **DDI** (3 kênh Digital Display: DP/HDMI), **PEG** (PCIe Graphics x16), nguồn 12V |

**Type 6** là "pin-out type" phổ biến nhất hiện nay: nhấn mạnh **DDI (DisplayPort/HDMI)** thay cho
các giao tiếp video cũ, giữ **PEG x16** cho GPU rời, và **USB 3.0**. Đây chính là lý do carrier này
có nhiều ngõ ra hiển thị số (DP/HDMI) + khe MXM cho GPU.

> Trong project, connector này được mô hình hoá bằng symbol `KBLH-E61_COMe_Type6_16U`
> (gộp các chân GND/nguồn để rút gọn còn 198 chân logic thay vì 440 chân vật lý).

---

## 3. Kiến trúc tổng thể

```
                         ┌───────────────────────────────────────────────┐
   DC IN 11–23V ─▶ [Bảo vệ đầu vào] ─▶ [Charger/Path] ─▶ DCOUT ─┐          │
                     fuse/TVS/PMOS       BQ24725A               │          │
                                                                ▼          │
                                            ┌────────── Bộ chuyển nguồn ────┴─────┐
                                            │  RT6228A → +V5P0A_DC (5V, 8A)       │
                                            │  RT6228A → +V3P3S_DC (3.3V, 8A)     │
                                            │  AXP8020 → +V3A_DC_ATX (3.3V, 2A)   │
                                            └───────┬───────────────┬────────────┘
                                                    │ +5VRUN        │ +V3P3S
                                                    ▼               ▼
   ┌────────────────────────────────────────────────────────────────────────────┐
   │                        COM Express Type 6 MODULE                             │
   │        (CPU + PCH + RAM + BIOS)  —  cắm vào CN3 (Row A/B/C/D)                 │
   └───┬───────┬───────┬───────┬───────┬───────┬───────┬───────┬───────┬──────────┘
       │PCIe   │USB2/3 │SATA   │LPC    │HDA    │DDI    │PEG    │LVDS   │SMBus/I2C
       ▼       ▼       ▼       ▼       ▼       ▼       ▼       ▼       ▼
  mini-PCIe  4×USB3  4×SATA  SuperIO  ALC662  DP/HDMI  MXM   LVDS/eDP EEPROM
  M.2 M-key  (+USB2) HDD PWR  IT8768E  Codec   /VGA     GPU   panel    /LAN i210
  LAN i210            mSATA   TPM      + AMP           slot            /Charger
  WiFi/BT            (mux)    Debug80  jack
```

**Các khối chức năng chính** (functional blocks):

1. **Power** — bảo vệ đầu vào, charger, buck 5V/3.3V, phân phối rail có/không gating theo sleep state.
2. **Clock** — PI6C20800 fan-out 100 MHz cho toàn bộ endpoint PCIe; ICS9112 cho 33 MHz legacy.
3. **PCIe fabric** — mini-PCIe ×2 (WiFi/BT + 4G), M.2 M-key (PCIe/SATA), LAN i210, MXM (PEG x16).
4. **USB** — 4× USB 3.0 (kèm USB 2.0), USB 2.0 cho mini-PCIe/PS2.
5. **Storage** — 3× SATA + nguồn HDD, mSATA (mux với mini-PCIe).
6. **Display** — LVDS/eDP (6 kênh panel), DP×n, HDMI, VGA (qua bridge IT6516).
7. **Audio** — codec ALC662 + jack + 2× ampli class-D NS4150.
8. **Legacy/Board I/O** — SuperIO IT8768E (4× COM/RS-232/485), TPM, BIOS SPI, debug port 80, PS2, quạt, front-panel, RTC.

---

## 4. Connector trung tâm — COMe Type 6 (CN3)

CN3 là "trục xương sống" của toàn board. Mọi bus tốc độ cao đều **xuất phát hoặc kết thúc** ở đây.
Bảng dưới nhóm chân theo chức năng (tên chân theo chuẩn COM.0 Type 6):

### 4.1 Row A / B (connector AB) — sheet 3

| Nhóm | Chân tiêu biểu | Ghi chú |
|------|----------------|---------|
| **Gigabit LAN** | `GBE0_MDI0..3±`, `GBE0_LINK/ACT#`, `GBE0_CTREF` | LAN tích hợp trên module (LAN1) |
| **SATA** | `SATA0_TX/RX±`, `SATA1..3` | SATA0–3 |
| **USB 2.0** | `USB0..7±`, `USB_x_y_OC#` | 8 cổng USB2, chia sẻ over-current |
| **PCIe x1** | `PCIE_TX/RX 0..5±` | lanes 0–5 (dùng cho mini-PCIe, LAN, mSATA) |
| **LPC** | `LAD0..3`, `LFRAME#`, `LDRQ0/1#`, `SERIRQ`, `LPC_CLK`, `LRESET#` | bus cho SuperIO/TPM/debug/BIOS |
| **HDA** | `AC/HDA_SYNC/RST#/BITCLK/SDOUT/SDIN0..2` | audio codec |
| **LVDS** | `LVDS_A/B[0..3]±`, `LVDS_A/B_CK±`, `LVDS_I2C`, `LVDS_BKLT_EN/CTRL`, `LVDS_VDD_EN` | panel nội bộ |
| **VGA** | `VGA_RED/GRN/BLU`, `VGA_HSYNC/VSYNC`, `VGA_I2C` | DAC analog trên module |
| **SMBus / I2C** | `SMB_CK/DAT/ALERT#`, `I2C_CK/DAT` | quản lý board |
| **Control / PM** | `PWRBTN#`, `SUS_S3/S4/S5#`, `PWR_OK`, `SYS_RESET#`, `CB_RESET#`, `WDT`, `THRMTRIP#`, `THRM#`, `LID#`, `SLEEP#`, `BATLOW#`, `WAKE0/1#`, `SPKR` | bắt tay nguồn & trạng thái |
| **Fan** | `FAN_PWMOUT`, `FAN_TACHIN` | điều khiển quạt |
| **SPI (BIOS)** | `SPI_MOSI/MISO/CLK/CS#`, `SPI_POWER` | BIOS flash ngoài |
| **Straps** | `BIOS_DIS0/1#`, `TYPE0/1/2/10#`, `PP_TPM` | cấu hình cứng |
| **GPIO** | `GPI0..3`, `GPO0..3` | I/O đa dụng |
| **Serial** | `SER0_TX/RX`, `SER1_TX/RX` | 2 UART từ module |
| **ExpressCard** | `EXCD0/1_PERST#/CPPE#` | (tuỳ chọn) |
| **Nguồn** | `VCC_12V` ×nhiều, `VCC_5V_SBY` ×4, `VCC_RTC` | cấp nguồn cho module |

### 4.2 Row C / D (connector CD) — sheet 4

| Nhóm | Chân tiêu biểu | Ghi chú |
|------|----------------|---------|
| **USB 3.0 SuperSpeed** | `USB_SSRX/SSTX 0..3±` | 4 cổng USB3 |
| **PCIe x1** | `PCIE_TX/RX 6..7±` | lanes 6–7 (LAN i210, WiFi thứ 2) |
| **DDI (Digital Display)** | `DDI1/2/3_PAIR0..3±`, `DDIx_HPD`, `DDIx_CTRLCLK/DATA_AUX`, `DDIx_DDC_AUX_SEL` | 3 kênh DP/HDMI |
| **PEG (PCIe Graphics)** | `PEG_TX/RX 0..15±`, `PEG_LANE_RV#` | x16 cho GPU/MXM |
| **Nguồn** | `VCC_12V` ×nhiều | cấp cho PEG/GPU |

> **Nguyên tắc:** carrier **không** sinh ra các bus này — nó chỉ **định tuyến, ghép nối AC, bảo vệ
> ESD, và ghép clock/reset**. Toàn bộ intelligence nằm ở module.

---

## 5. Kiến trúc nguồn (Power architecture)

### 5.1 Dải điện áp đầu vào & bảo vệ (sheet 29)

```
DC IN (11–23V) → [Fuse 65V/15A] → [Ferrite HFZ3216 ×3] → [PMOS PDC3903] → V_ADP_CONN_LL (7.5–22V)
                                       │
                              [TVS SMBJ24A]  [Reverse/OVP: BZX84-B22, RB520S30]
```
- **Bảo vệ quá áp:** nếu `DCIN > 23V` → PMOS **không bật** (mạch so sánh ngưỡng dùng L2N7002 + zener).
- **Delay 30 ms:** `DCIN → +VBATA` có trễ để tránh in-rush.
- Ngưỡng adapter hợp lệ: **11V < VIN < 24V** (mô phỏng bằng Proteus, sheet 37).

### 5.2 Battery charger & power-path (sheet 37, 38)

- **BQ24725A (U10):** SMBus battery charger, cuộn 4.7 µH, sense 0.01R.
  Phát hiện adapter qua `ACIN_DET` (UVLO 2.4V / OVP 3.15V), báo `DC1_ACOK`.
- **Power-path:** `DCOUT` chọn giữa adapter và battery (PMOS PDC3903/PDC3906), dải **9–19.5V**.
- Giao tiếp EC: `EC_SMDAT1 / EC_SMCLK1`, `BATT1_ECIN#`.

### 5.3 Các bộ chuyển nguồn (sheet 30, 32)

| IC | Ngõ ra | Dòng | Ghi chú |
|----|--------|------|---------|
| **RT6228A** (U37) | `+V5P0A_DC` = **5.0V** | 8A limit | Vout=0.6/4.99K×(37.4K+4.99K); FB=0.6V |
| **RT6228A** (U39) | `+V3P3S_DC` = **3.3V** | 8A limit | gated bởi `PM_SLP_S3#`; Vout=0.6/11K×(49.9K+11K) |
| **AXP8020** (U31) | `+V3A_DC_ATX` = **3.3V** | 2A | rail "always/standby-ish"; Vout=0.6/10×(10+45.3) |
| **FA7603** (U42) | power switch | — | đóng/cắt rail theo sleep |

Các nhánh phân phối: `+V5P0A_DC → +V5A_DC_ATX` (tới CB), `ATX_5V_CB`, `+5VRUN`;
`+V3P3S_DC → +V3P3S` (rail 3.3V chính); `+V3A_DC_ATX` cho phần standby/always-on.

### 5.4 Cây nguồn & chế độ (sheet 53)

| Chế độ | Rail cấp |
|--------|----------|
| **ATX** | 5VSB + 12VRUN + 5VRUN + 3.3VRUN |
| **AT** | 12VRUN |
| **Adapter** | 12–19V |
| **Battery** | qua charger |

`PM_SLP_S3#` là tín hiệu gating trung tâm: rail "S" (suspend-to-RAM off) bị cắt khi vào S3;
rail "RUN"/"A" vẫn sống. `COME_PWROK` (sheet 36) được tổng hợp từ `+VCC12 / +5VRUN / +V3A_DC_ATX`.

### 5.5 Ngân sách công suất (sheet 52)

| Rail | Tổng | Phân bổ |
|------|------|---------|
| **3.3V** | **10A** | 4× mini-PCIe 4A · M.2 SSD 2.5A · LAN1&2 1A · LVDS 2A · MXM 1A · DP×6 3A |
| **5V** | **6.5A** | USB3×4 4A · SATA HDD 1.5A · SPK 1A · MXM 1A |

---

## 6. Kiến trúc Clock

**Nguyên tắc:** một clock 100 MHz duy nhất từ module (`PCIECLK_100M_P0/N0`) → **buffer fan-out**
→ nhiều diff-pair 100 MHz cho từng endpoint. (sheet 5)

```
COM-E CLKIN ─▶ PI6C20800SAE (U45, 48-pin) ─┬─▶ CLK_PCIE0  → USB3.0 controller trên module
                                            ├─▶ CLK_PCIE1  → M.2 M-key
                                            ├─▶ CLK_PCIE2  → mini-PCIe (4G)
                                            ├─▶ CLK_PCIE3  → mini-PCIe (WiFi/BT)
                                            ├─▶ CLK_PCIE4  → MXM (PEG)
                                            ├─▶ CLK_PCIE5  → LAN i210
                                            ├─▶ CLK_PCIE6  → mSATA
                                            └─▶ CLK_PCIE7  → dự phòng
```
- Mỗi ngõ ra qua **33R series** + tải đầu cuối **49.9R** (matching + kiểm soát EMI).
- `OE_INV=0` (mặc định), có PLL cho SSC/EMI nhưng **No SSC** (fan-out thuần); chân `OE` = H để bật.
- **ICS9112-16 (U9):** sinh clock 33 MHz legacy (`LPC_CLK_OUT1`) cho debug/port 80.
- **WGI210AT** có thạch anh **25 MHz** riêng (LAN có clock nội bộ).

---

## 7. Kiến trúc Reset & Power sequencing

### 7.1 Cây reset

`COME_PLT_RST#` (Platform Reset từ module) là **reset gốc**, được phân phối tới **mọi endpoint PCIe**
và thiết bị board-level. Nó xuất hiện trên hầu hết các sheet (3,6,7,8,9,10,12,23,25,42,43,47…).

```
COME_PLT_RST# ─┬─▶ mini-PCIe PERST#      (mỗi khe, có RC delay riêng)
               ├─▶ M.2 PERST#
               ├─▶ LAN i210 PE_RST_N
               ├─▶ MXM / PEG reset
               ├─▶ SuperIO LRESET#, TPM LRESET#, Debug PCIRST#
               └─▶ SYS_RST_N (front-panel reset)
```

### 7.2 Chuỗi power-good

- EN của các buck được lái bởi **PG của tầng trước** (sequencing tuần tự).
- `PWRGD` daisy-chain: `+V5A_PWRGD`, `+V3P3A_PWRGD` → tổng hợp `COME_PWROK`.
- Các tín hiệu sleep: `PM_SLP_S3#` (S3), `PM_SLP_S4#` (S4), `PM_SLP_S5#` (S5) — gate nguồn ngoại vi:
  - USB VBUS load switch bật khi **`PM_SLP_S4#` = active** (sheet 13,14).
  - Rail 3.3V "S" bật theo **`PM_SLP_S3#`** (sheet 32).

### 7.3 Các tín hiệu control quan trọng

| Tín hiệu | Vai trò |
|----------|---------|
| `PWRSW#` / `PWRSW#_EC` | nút nguồn → EC/module |
| `SYS_RST_N` | reset hệ thống (front-panel) |
| `CPU_SLEEP#`, `SLEEP#` | trạng thái ngủ |
| `WDT_OUT` | watchdog từ SuperIO |
| `LID#` (`SYS_LID#`) | cảm biến nắp (Hall APX8132) |
| `THRM#`, `THRMTRIP#` | cảnh báo/ngắt nhiệt |
| `BATLOW#` | pin yếu |
| `PCH_RI_WAKE#`, `WAKE0/1#` | wake events |

---

## 8. Các bus & giao tiếp — chi tiết

### 8.1 PCI Express (PCIe)

**Dùng ở:** mini-PCIe ×2, M.2 M-key, LAN i210, mSATA, MXM (PEG x16).

Nguyên tắc thiết kế lặp lại:
- **AC coupling trên cặp TX:** tụ nối tiếp trên `PCIE_TXP/N` phía **carrier** ("AC caps on COME").
  Giá trị theo bảng (sheet 10):

  | Điều kiện | PCIe Gen2 | PCIe Gen3 | SATA |
  |-----------|-----------|-----------|------|
  | Processor **TX** | 100 nF | 220 nF | 10 nF |
  | Processor **RX** | none | none | 10 nF² |

- **Clock:** mỗi khe nhận 1 cặp `CLK_PCIEx±` từ PI6C20800.
- **Reset:** `PERST#` = `COME_PLT_RST#` (có thể qua RC delay/level-shift).
- **W_DISABLE#/RF-kill:** `BT_EN`, `WAN_EN`, `3G_DIS#` điều khiển bật/tắt radio.
- **Polarity inversion:** với M.2 chạy PCIe cần đảo cực lane ("Lane Polarity Inversion should be made").

### 8.2 USB

- **USB 2.0** (`USB2_Nx/Px`): 8 cổng từ Row A/B; dùng cho USB3 combo, mini-PCIe (modem/WiFi), PS2.
- **USB 3.0** (`USB3_RX/TX x`): 4 cổng SuperSpeed từ Row C/D.
- **Cấp VBUS:** load switch **JW7115-2** (U5/U6/U18/U35), bật theo `PM_SLP_S4#`, cờ `USB30_PWR`.
- **Bảo vệ tín hiệu:** common-mode + ESD **PCMF112L900MFR**, common-mode filter **PT10A054U**,
  TVS **PESD0402V05U** trên từng line data.
- **Connector combo:** USB3.0 + RJ45 tích hợp (Lotes AUSB0001-K018C) dùng cho LAN.

### 8.3 SATA

- 3× data connector **G84160-001** (`SATA0/2/3`) + wafer nguồn HDD 4-pin.
- Rail `+5V_RUN_SATA`; **0R series** trên cặp data (tuỳ chọn stuffing).
- **mSATA** (sheet 9): dùng form-factor mini-PCIe nhưng mang tín hiệu SATA; **mux** SATA↔PCIe qua
  strap `SATA0-1_SATA_PCIE#` (jumper Head/1x3).

### 8.4 LPC (Low Pin Count) — bus "board-level"

Đây là bus giao tiếp chính cho thiết bị legacy: `LAD0..3`, `LFRAME#`, `LPC_CLK`, `LRESET#`, `SERIRQ`.

**Các thiết bị treo trên LPC:**
| Thiết bị | IC | Chức năng |
|----------|-----|-----------|
| **SuperIO** | IT8768E (sheet 23) | 4× UART/COM, WDT, GPIO |
| **TPM** | SLB9665 (sheet 43) | bảo mật TPM 2.0 |
| **Debug** | AK2001 + 7-seg (sheet 47) | Port-80 POST code |
| **Header** | LPC_TPM / CPU_GPIO | mở rộng |

- `LPC_CLK` từ ICS9112 (33 MHz).
- SuperIO cấu hình địa chỉ base qua **strap** `DTR1#` (2E/2F ↔ 4E/4F) và bật/tắt WDT qua `RTS1#`.

### 8.5 Display

Ba lối ra hiển thị song song:

**(a) LVDS / eDP (sheet 15, 28)**
- Panel nội bộ: `LVDS_A/B[0..3]±` + clock, hoặc **eDP** (muxed) `eDP_LANE0..3`, `eDP_AUX_CH`.
- Backlight: `LVDS_BKLT_EN`, `LVDS_BKLT_CTRL` (PWM), `LCD_BKL_VCC`; nguồn panel `LCD_VCC` chọn được.
- LCD connector FPC 30-pin (`LCD_CONN`, `GFX_EDP`), 6 kênh cho GPU output.
- Level-shift EN/PWM qua **LBSS138/2N7002** (open-drain).

**(b) DDI → DisplayPort / HDMI (sheet 17,18,26,27)**
- 3 kênh DDI từ module. Mỗi kênh có strap **`DDC_AUX_SEL`**: `0 = DP`, `1 = HDMI` (sheet 46).
- Connector N3160 (BHDM39B1N1289) dùng chung layout DP & HDMI.
- ESD **PT10A054U** + fuse 16V/0.75A trên nguồn +5V connector; TVS trên AUX/DDC.
- HPD level-shift qua LBSS138.

**(c) VGA (analog) qua bridge (sheet 16)**
- **IT6516BFN (U13):** bridge **DP → VGA**. DAC analog RGB (75R), `VGA_HSYNC/VSYNC`, DDC I2C.
- Rail riêng cho DAC: `DAC_VDDAC`, `RX_AVCC`, tách bằng ferrite bead 120Ω.

### 8.6 HD Audio (HDA)

- Codec **ALC662-VD0-GR (U32)** treo trên bus HDA: `BITCLK/SYNC/SDOUT/SDIN/RST#`.
- Ngõ ra: LINE-OUT, MIC-IN, internal MIC; jack combo **ABA-JAK-038-K44** (RED=MIC, GREEN=HP).
- 2× ampli class-D **NS4150** (U19/U20) 5V/4Ω/3W cho loa; `SPK_CTL#` shutdown.
- **Layout note quan trọng:** điện trở jack-detect (JD) phải đặt **sát chân sense** của codec.
- Tách nguồn analog/digital bằng ferrite bead + `LDO_OUT`/`VREF` riêng.

### 8.7 SMBus / I2C

- `SMB_CK/DAT/ALERT#` (system management) → EEPROM, LAN i210, charger, MXM EEPROM.
- **EEPROM AT24C02** (sheet 25, 45): lưu cấu hình board/MXM.
- Pull-up 10K trên SDA/SCL; tách rời từng segment nếu cần.

### 8.8 UART / RS-232 / RS-485 (sheet 23, 24, 35)

- SuperIO IT8768E cấp **4 COM** (COM1–4) + 2 UART từ module (SER0/1) → **5 COM tổng** (COM1–5).
- **RS-232/485 transceiver SP485ECN** (U22–U25): mỗi con phục vụ 1 COM, ra **DB9**.
- Buffer/đệm UART TXD/RXD qua cổng AND đơn **AiP74LV1T08** (sheet 35) — cách ly & định mức.

### 8.9 SPI (BIOS flash) — sheet 40

- **W25Q64 (U53):** BIOS flash ngoài trên bus SPI của module (`SPI_MOSI/MISO/CLK/CS#`).
- **SPI socket** song song để nạp/khôi phục.
- Strap chọn BIOS: `BIOS_DIS0/1#` (jumper `BIOS_SEL`); nguồn `VCC_ROM`/`VCC_SPI_COME`.

### 8.10 GPIO & Front-panel (sheet 41, 42, 48)

- `GPI0..3` / `GPO0..3` ra header + LED chỉ thị (transistor drive).
- Front-panel: nút Power/Reset/Sleep, LED HDD/Power, buzzer `PCH_SPKR`.
- **PS2** KB/MS (Foxconn MH11061); LED SATA/M.2/WDT.
- **RTC**: pin CR2032 (BS-08-B2AA002), rail `+V3P3A_RTC`/`VCC_RTC`.
- **Quạt**: `FAN_CONN` 4-pin PWM (`FAN_CTL1`/`FAN_TAC1`).

---

## 9. Design patterns (mẫu mạch lặp lại)

Đây là các "idiom" thiết kế xuất hiện lặp lại — nắm được chúng là hiểu 80% board.

| # | Pattern | Mô tả | Xuất hiện ở |
|---|---------|-------|-------------|
| 1 | **AC coupling** | Tụ nối tiếp trên cặp TX của PCIe/USB3/SATA. Giá trị theo Gen (bảng §8.1). | PCIe, USB3, SATA |
| 2 | **ESD/TVS array** | Diode TVS trên mọi line ra connector: `PESD0402V05U` (đơn), `PCMF112L900` (CM+ESD), `PT10A054U` (CM filter). | tất cả connector ngoài |
| 3 | **Series termination** | 33R trên clock; 0R (stuffing option) trên data; 75R Bob-Smith cho RJ45/VGA. | clock, LAN, VGA |
| 4 | **Load switch / power gate** | `JW7115` cho USB VBUS; PMOS + `2N7002` gate cho rail, gated theo `PM_SLP_Sx#`. | USB, rail switching |
| 5 | **Level shifter** | `LBSS138`/`2N7002` open-drain dịch mức tín hiệu control giữa các miền áp. | backlight, LED, HPD, GPIO |
| 6 | **Reset distribution** | `COME_PLT_RST#` fan-out → `PERST#` từng endpoint, RC delay riêng. | mọi PCIe endpoint |
| 7 | **Power-good chain** | EN tầng sau lái bởi PG tầng trước; delay RC; tổng hợp `COME_PWROK`. | power tree |
| 8 | **Clock fan-out** | 1 ref 100 MHz → buffer → n diff-pair + 33R series. | sheet 5 |
| 9 | **Config strap / mux** | Jumper hoặc pull R để chọn: SATA↔PCIe, DP↔HDMI, BIOS sel, SuperIO addr. | mSATA, M.2, DDC, BIOS, SIO |
| 10 | **Decoupling** | Bulk (10/22µF) + 0.1µF/chân nguồn; ferrite bead cách ly rail analog. | toàn board |
| 11 | **Indicator LED** | LED trạng thái lái bằng transistor/mosfet từ GPIO/tín hiệu hệ thống. | front-panel, GPIO |
| 12 | **Debug provision** | Port-80 (AK2001+7seg), header LPC_TPM/SMB/CPU_GPIO, test point. | debug/test |
| 13 | **DNP / stuffing option** | Linh kiện `Dummy` để tuỳ biến board theo biến thể (không hàn mặc định). | toàn board |
| 14 | **Analog isolation** | Ferrite bead + tụ tách nguồn cho codec audio, DAC VGA, PHY LAN. | audio, VGA, LAN |

---

## 10. Bản đồ sheet chức năng (PDF → KiCad)

Reference PDF có 55 sheet. Đã gom thành **8 hierarchical sheet** trong KiCad (theo khối chức năng
lớn, tương đương độ chi tiết của project gốc) — dễ điều hướng, mỗi sheet là một hệ con hoàn chỉnh.

| Sheet KiCad | Sheet PDF | Nội dung |
|-------------|-----------|----------|
| `Root` | — | 8 sheet symbol (bản đồ điều hướng) |
| `1_Module_Clock` | 3,4,5 | CN3 COMe connector (Row A/B/C/D) + clock buffer (PI6C20800, ICS9112) |
| `2_Power` | 29,30,32,33,36,37,38 | đầu vào, charger, buck 5V/3.3V, type/GPU ctrl, power-OK |
| `3_PCIe_LAN` | 6–12, 25 | mini-PCIe ×2 (4G+WiFi), SIM, mSATA, M.2 M-key, LAN i210, MXM/PEG |
| `4_USB_SATA` | 13,14,19 | USB3 ×4 (+load switch), SATA ×4, nguồn HDD |
| `5_Display` | 15,16,17,18,26,27,28 | LVDS/eDP, DisplayPort, HDMI, VGA (IT6516) |
| `6_Audio` | 20,21,22 | ALC662 codec + jack + 2× NS4150 amp |
| `7_Serial_SuperIO` | 23,24,35 | **IT8768E** + **SP485ECN ×4** + **AiP74LV1T08 ×4** ✓ đã place & gán label |
| `8_Misc_IO` | 40–48, 51 | TPM, debug port-80, BIOS SPI, LID, front-panel, PS2, GPIO, quạt, RTC, SMBus, EEPROM, lỗ vít |

**Đã place & gán label trên `7_Serial_SuperIO`** (global label = tên net theo reference, nối trực
tiếp vào chân — đã verify bằng netlist):
- `U21` IT8768E — 48 chân đều có label: bus LPC (`LPC_AD0..3`, `LPC_FRAME#`, `LPC_SERIRQ#`,
  `LPC_CLK`), `COME_PLT_RST#`, `IT8768_48MHZ`, `UART_WDT`, `VCORE`, nguồn `+V3P3S_IT8768`, `GND`,
  và các tín hiệu COM1–4 (`SOUT1..4`, `SIN1..4`, `DTR/RTS/CTS/DSR/DCD/RI x#`).
- `U22–U25` SP485ECN — label nguồn `+5VRUN` / `GND` (tín hiệu COM/DB9 gán khi đặt connector).
- `U2/U34/U36/U38` AiP74LV1T08 — buffer UART: `UART_TXD0/RXD0/TXD1/RXD1` → `..._CC`, `+V3P3S`, `GND`.

> Trạng thái: 8 sheet đã tạo & validate (netlist 9 linh kiện / **159 net** / ERC / PDF 9 trang).
> 7 sheet còn lại hợp lệ-nhưng-trống, chờ symbol batch 2–8.

---

## 11. Tổ chức project KiCad & quy ước thư viện

### 11.1 Cấu trúc thư mục
```
PC_ExtensionBoard/
├── Docs/                         # tài liệu, spec, hình
├── Libs/
│   ├── symbol/
│   │   └── A_PC_ExtensionBoard.kicad_sym    # 1 thư viện symbol dùng chung
│   └── footprints/
│       └── PC_Components_1.pretty/          # 1 thư viện footprint dùng chung
└── PC_ExtensionBoard/            # project chính
    ├── PC_ExtensionBoard.kicad_pro
    ├── PC_ExtensionBoard.kicad_sch          # root schematic (hierarchical)
    ├── <các sheet con>.kicad_sch
    ├── PC_ExtensionBoard.kicad_pcb
    ├── sym-lib-table                        # trỏ tới Libs/symbol (URI tuyệt đối)
    └── fp-lib-table                         # trỏ tới Libs/footprints
```

### 11.2 Quy ước thư viện symbol
- **Một thư viện duy nhất** `A_PC_ExtensionBoard.kicad_sym` cho toàn project (dễ quản lý version).
- **Đặt tên symbol:**
  - Linh kiện rời **mã hoá giá trị**: `R_0R_0402`, `R_100k_0402`, `C_100n_0402`, `C_22u_16V_0603`,
    `L_1u8_14A_Coilcraft-XAL6030` → chọn là ra ngay giá trị + footprint.
  - Linh kiện generic: `R`, `C`, `Fuse`, `D_Zener`, `Conn_01x04_Pin`.
  - IC/connector đặc thù đặt theo **MPN/chức năng**: `IT8768E`, `SP485ECN`, `ALC662`,
    `KBLH-E61_COMe_Type6_16U`, `HDMI_A`, `Bus_M.2_2199230-2`.
- **Property chuẩn** trên mỗi symbol: `Reference`, `Value`, `Footprint`, `Datasheet`,
  `Description`, `ki_keywords` (+ `MPN`/`Manufacturer`/`LCSC` nếu có, phục vụ BOM).
- **Kiểu chân (pin electrical type):** dùng đúng loại — `input`/`output`/`bidirectional`/`passive`/
  `power_in`/`power_out`/`open_collector` — để ERC bắt lỗi kết nối.
- **Layout symbol:** thân là `rectangle` fill `background`; pin length 2.54, font 1.27; đặt chân
  theo **nhóm chức năng** (không theo số chân) cho dễ đọc — trừ khi số chân quá lớn thì xếp tuần tự.

### 11.3 Quy ước footprint
- **Một thư viện** `PC_Components_1.pretty`; footprint đặt theo package: `R_0402_1005Metric`,
  `C_0603_1608Metric`, `SOT-23-5`, `QFN-24-1EP_4x4mm...`, hoặc theo connector cụ thể (`HDMI-019S`,
  `RJ45-TH_HR911130A`, `Bus_M.2_Socket_Key_M...`).
- `fp-lib-table` / `sym-lib-table` dùng **URI tuyệt đối** trỏ về `Libs/`.

### 11.4 Quy ước net & nguồn
- **Rail nguồn** dùng power symbol/global label rõ ràng: `+VCC12`, `+5VRUN`, `+V3P3S`,
  `+V3A_DC_ATX`, `+V5A_DC_ATX`, `+V3P3A_RTC`, `DCOUT`, `V_ADP_CONN_LL`…
- **Tín hiệu cross-sheet** dùng **hierarchical/global label** với hậu tố chuẩn:
  `#` = active-low, `_P/_N` = cặp vi sai, `_TX/_RX` = hướng.
- Ghi chú `[n]` cạnh label = số sheet liên quan (giữ như reference để dễ tra chéo).

### 11.5 Quy trình build (Windows)
- Python & CLI dùng bản đi kèm KiCad: `"C:/Program Files/KiCad/10.0/bin/python.exe"`,
  `kicad-cli.exe` (không dùng python hệ thống).
- **Validate thư viện symbol:** `kicad-cli sym upgrade --force <copy>.kicad_sym` trên **bản sao**
  (chạy in-place sẽ reformat toàn bộ file).
- **Kiểm tra render:** `kicad-cli sym export svg --symbol <NAME> ...`.

---

## 12. Bảng tra cứu

### 12.1 Rail nguồn chính
| Rail | Điện áp | Nguồn | Gating | Dùng cho |
|------|---------|-------|--------|----------|
| `V_ADP_CONN_LL` | 7.5–22V | đầu vào sau bảo vệ | — | charger, buck |
| `DCOUT` | 9–19.5V | power-path | — | buck sơ cấp |
| `+V5P0A_DC` | 5.0V | RT6228A U37 | — | 5VRUN, CB |
| `+5VRUN` | 5.0V | +V5P0A_DC | run | USB, audio, quạt |
| `+V3P3S_DC` / `+V3P3S` | 3.3V | RT6228A U39 | `PM_SLP_S3#` | logic 3.3V chính |
| `+V3A_DC_ATX` | 3.3V | AXP8020 U31 | always | standby, LID, GPIO |
| `+V3P3A_RTC` | 3.3V | RTC | always | RTC/CMOS |

### 12.2 Strap cấu hình
| Strap | Giá trị | Ý nghĩa |
|-------|---------|---------|
| `SATA0-1_SATA_PCIE#` | jumper | mSATA/M.2: chọn SATA hay PCIe |
| `DDIx_DDC_AUX_SEL` | 0 / 1 | 0 = DP, 1 = HDMI |
| `BIOS_DIS0/1#` | jumper `BIOS_SEL` | chọn BIOS |
| `DTR1#` (SuperIO) | 2E/2F ↔ 4E/4F | địa chỉ config SuperIO |
| `RTS1#` (SuperIO) | enable/disable | bật WDT |
| `TYPE0/1/2/10#` | NC / GND | định danh loại module (Type6: Type2#→GND) |
| `3G_DIS#` | low | tắt modem 3G |

### 12.3 IC chính
| Chức năng | IC | Sheet |
|-----------|-----|-------|
| Clock buffer PCIe | PI6C20800SAE | 5 |
| Clock 33 MHz | ICS9112-16 | 5 |
| LAN GbE | Intel WGI210AT (i210) | 12 |
| SuperIO | IT8768E | 23 |
| RS-232/485 xcvr | SP485ECN ×4 | 24 |
| DP→VGA bridge | IT6516BFN | 16 |
| Audio codec | ALC662-VD0-GR | 20 |
| Speaker amp | NS4150 ×2 | 22 |
| Battery charger | BQ24725A | 37 |
| Buck 5V/3.3V | RT6228A ×2 | 30,32 |
| Buck 3.3V standby | AXP8020 | 30 |
| BIOS flash | W25Q64 | 40 |
| TPM | SLB9665 (TPM 2.0) | 43 |
| Port-80 debug | AK2001 | 47 |
| UART buffer/AND | AiP74LV1T08 | 35 |
| LID Hall sensor | APX8132 | 40 |

---

## 13. Checklist thiết kế

Từ sheet 54 của reference + best-practice cho carrier Type 6:

- [ ] **AC caps** đủ trên mọi cặp TX PCIe/USB3/SATA, đúng giá trị theo Gen.
- [ ] **MXM connector**: cân nhắc bỏ TX caps (đã có trên MXM) để tránh double-AC.
- [ ] **Lane polarity inversion** cho M.2 khi chạy PCIe.
- [ ] Vị trí **PIN1 của AB slot** khớp với module.
- [ ] **ESD** trên mọi connector ngoài (USB, LAN, HDMI, DP, VGA, audio, PS2).
- [ ] **Clock 33R series** + tải 49.9R đúng cho từng ngõ PI6C20800.
- [ ] **Reset tree**: `COME_PLT_RST#` tới đủ endpoint, RC delay hợp lý.
- [ ] **Power sequencing**: PG chain đúng thứ tự, `COME_PWROK` chỉ H khi đủ rail.
- [ ] **Gating**: rail "S" cắt theo `PM_SLP_S3#`, USB VBUS theo `PM_SLP_S4#`.
- [ ] **JD resistor** audio đặt sát chân sense codec (layout).
- [ ] **Analog isolation**: ferrite bead cho codec/DAC/PHY LAN.
- [ ] **Ngân sách nguồn**: 3.3V ≤ 10A, 5V ≤ 6.5A — kiểm tra tải thực tế.
- [ ] **Strap/jumper** đặt đúng mặc định (SATA/PCIe, DP/HDMI, BIOS, SuperIO addr).
- [ ] **Debug**: port-80 + header LPC/SMB/GPIO + test point cho các rail chính.

---

*Tài liệu này bám theo reference schematic `kblh-e61_cb_v20` và cấu trúc project KiCad hiện tại
(`Libs/symbol/A_PC_ExtensionBoard.kicad_sym`, `Libs/footprints/PC_Components_1.pretty`).
Cập nhật khi thêm/bớt subsystem hoặc thay đổi rail/strap.*
