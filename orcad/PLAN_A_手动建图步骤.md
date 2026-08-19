# CAN FD 总线仿真原理图 —— 手动建图步骤（方案 A）

目标：在 OrCAD Capture CIS 17.4 里手动画出与 `si_models/can_si_2motor_transceiver.cir` 等价的原理图，
3 个 TCAN1042 收发器（板端 X1 + 电机 X2/X3），双线差分传输线（6 根 T 元件），双端 60Ω 终端，PWL 激励 + 文本指令。
**校验标准**：生成的 PSpice 网表（.net）与 .cir 的元件/节点/指令一一对应，直接可跑出同样的波形。

---

## 0. 需要准备的文件

| 文件 | 说明 |
|---|---|
| `si_models/orcad/models/TCAN1042_q1_pspice_sch.lib` | **已备好**。原模型 + 末尾追加了 MUX2X1_0 / LOGIC_IO_0 的 EVALUE 修正（对应 .cir 46–58 行的覆盖块）。原理图 `.lib` 指令指向它即可，不用再手工贴子电路。 |
| `si_models/can_si_2motor_transceiver.cir` | 目标网表，建完图后逐行对照。 |

> 原理图里的 `.lib` 路径建议用**绝对路径**（Capture netlist 对相对路径的基准目录不稳定）：
> `.lib "E:\workspace2608\03\Ragtime_Firmwares\si_models\orcad\models\TCAN1042_q1_pspice_sch.lib"`

---

## 1. 新建 PSpice 工程

1. File → New → Project
2. Name: `can_si_2motor`；Location: `E:\workspace2608\03\Ragtime_Firmwares\si_models\orcad\`
3. 选 **Analog or Mixed A/D**，勾选 **Create a blank project**（不要勾 "based on an existing project"）
4. OK → 自动打开 SCHEMATIC1 的 PAGE1
5. Options → Design Template → Page Size 保持 A（可选改 B 更宽松）

---

## 2. 自建 TCAN1042 符号（.olb）

Capture 的模拟库没有 TCAN1042，需要建一个带 PSpiceTemplate 的符号。

1. File → New → Library → 弹出 Symbol Editor，Ctrl+S 存为 `TCAN1042.olb`（放在工程目录）
2. 在 Library 里：右键 → **New Part**
   - Part Name: `TCAN1042`
   - Part Reference Prefix: `X`
   - Part Per Package: 1
   - 点击 **Pin** 逐个添加 7 个引脚（属性见下表），再 **Body/Box** 画一个方框包住引脚，保存
3. 7 个引脚（名称必须与子电路参数名一致）：

| Pin# | 名称  | 类型    | 建议位置 |
|---|---|---|---|
| 1 | TXD  | INPUT   | 左侧 |
| 2 | STB  | INPUT   | 左侧 |
| 3 | RXD  | OUTPUT  | 右侧 |
| 4 | CANH | PASSIVE | 右侧 |
| 5 | CANL | PASSIVE | 右侧 |
| 6 | VCC  | POWER   | 上 |
| 7 | GND  | POWER   | 下 |

4. 给符号添加属性（Edit → Properties，或直接选中符号加 User Property）：
   - **PSpiceTemplate** = `X^@REFDES %TXD %STB %RXD %CANH %CANL %VCC %GND TCAN1042`
   - **VALUE** = `TCAN1042`
   - 其余不用填
5. 保存并关闭 Symbol Editor，把 `TCAN1042.olb` 加进当前工程：
   - Project Manager → 右键工程 → Add → File to Project → 选 `TCAN1042.olb`

> 若想偷懒：直接用 Place Part 搜任意 7 引脚封装也行，但必须能手工加 PSpiceTemplate 属性，模板内容必须与上面一致。

---

## 3. 放置元件

用 Place Part（快捷键 P），Library 选 **ANALOG** / **SOURCE** / **TCAN1042**。

| REFDES | 库/元件 | 关键属性 |
|---|---|---|
| X1, X2, X3 | TCAN1042 | 值 TCAN1042；REFDES 前缀 X |
| T_TRK_H, T_TRK_L | ANALOG → **T** | Z0=60，TD={L_TRUNK*TDPM} |
| T_SG2_H, T_SG2_L | ANALOG → T | Z0=60，TD={L_SEG2*TDPM} |
| T_STB_H, T_STB_L | ANALOG → T | Z0=60，TD={S_STUB*TDPM} |
| RTH_A, RTL_A, RTH_M2, RTL_M2 | ANALOG → **R** | 60 |
| RPR_M1H, RPR_M1L, RPR_RXD1, RPR_RXD2, RPR_RXD3 | ANALOG → R | 1Meg |
| VCC | SOURCE → **VDC** | DC=5 |
| VSTB1, VSTB2, VSTB3 | SOURCE → VDC | DC=0 |
| VTX2, VTX3 | SOURCE → VDC | DC=5 |
| VTX1 | SOURCE → **VPWL** | 见下方 PWL 点 |

T 元件属性设置（双击打开 Properties）：
- **Z0** = `60`
- **TD** = `{L_TRUNK*TDPM}`（T_SG2_* 用 `{L_SEG2*TDPM}`，T_STB_* 用 `{S_STUB*TDPM}`）

VTX1 的 PWL 属性（对应 .cir 118–131 行，bit=200ns @5Mbps，edge 15ns，共 13 点）：
（VPWL 属性界面按列填 T1..T13 / V1..V13；T 单位默认秒）

| 序号 | T | V |
|---|---|---|
| 1 | 0 | 5 |
| 2 | 400n | 5 |
| 3 | 415n | 0 |
| 4 | 600n | 0 |
| 5 | 615n | 5 |
| 6 | 800n | 5 |
| 7 | 815n | 0 |
| 8 | 1200n | 0 |
| 9 | 1400n | 0 |
| 10 | 1415n | 5 |
| 11 | 1800n | 5 |
| 12 | 1815n | 0 |
| 13 | 2.2u | 5 |

> 若 VPWL 的 PWL 属性界面不好填，直接在 VTX1 上加属性 **PWL**（**推荐**，与 .cir 完全一致的 13 个点）：
> `0 5 400n 5 415n 0 600n 0 615n 5 800n 5 815n 0 1200n 0 1400n 0 1415n 5 1800n 5 1815n 0 2.2u 5`

放置建议（从 X1 出发往右排）：
```
          X2 [MOTOR1]  (stub 端, 不接终端)
          ↑ stub T_STB_*
X1 [BOARD] ──T_TRK_*──● HUB 节点──T_SG2_*──→ X3 [MOTOR2] + RTH_M2/RTL_M2
   + RTH_A/RTL_A          (差分两根线都画)
```

---

## 4. 连线（关键）

所有连线用 **Place Wire**（快捷键 W）。差分总线 = 两根平行线（CANH 与 CANL），每根线之间隔 1~2 格，别交叉。

### 4.1 节点命名（Net Alias）

每个网络在任意一段线上放 Net Alias（Place → Net Alias，快捷键 N），名字必须与下表一致：

| 网络 | 连接到 |
|---|---|
| `D_TXD1` | VTX1 +，X1.TXD |
| `D_TXD2` | VTX2 +，X2.TXD |
| `D_TXD3` | VTX3 +，X3.TXD |
| `STB1` | VSTB1 +，X1.STB |
| `STB2` | VSTB2 +，X2.STB |
| `STB3` | VSTB3 +，X3.STB |
| `RXD1` | X1.RXD，RPR_RXD1 上端 |
| `RXD2` | X2.RXD，RPR_RXD2 上端 |
| `RXD3` | X3.RXD，RPR_RXD3 上端 |
| `VCCP` | VCC +，X1/X2/X3.VCC |
| `CANH_B` | X1.CANH，T_TRK_H A+，RTH_A 上端 |
| `CANL_B` | X1.CANL，T_TRK_L A+，RTL_A 上端 |
| `HUB` | T_TRK_H B+，T_SG2_H A+，T_STB_H A+ |
| `HUBL` | T_TRK_L B+，T_SG2_L A+，T_STB_L A+ |
| `CANH_M1` | T_STB_H B+，X2.CANH，RPR_M1H 上端 |
| `CANL_M1` | T_STB_L B+，X2.CANL，RPR_M1L 上端 |
| `CANH_M2` | T_SG2_H B+，X3.CANH，RTH_M2 上端 |
| `CANL_M2` | T_SG2_L B+，X3.CANL，RTL_M2 上端 |

### 4.2 T 元件引脚顺序

T 的引脚为 **A+ A- B+ B-**（PSpice 模板 `T^@REFDES %A+ %A- %B+ %B-`）。
- **A 端接来向**，**B 端接去向**（T_TRK: A=BOARD侧, B=HUB侧；T_SG2/T_STB: A=HUB侧, B=节点侧）。
- T 的 A- / B-（接地回线）全部连到 `0` 地。

### 4.3 连线动作清单

1. 板端：X1.CANH → 短横线；X1.CANL → 其下方短横线（两线平行）。
2. 终端：RTH_A 上端接 CANH_B 线，下端接地；RTL_A 同理接 CANL_B。
3. 干线：CANH_B 线向右引到 HUB 点，CANL_B 平行引到 HUBL 点。
4. 分支点（HUB/HUBL）：在两线各放一个 **junction**（Place → Junction，快捷键 J），T_TRK_H.B+、T_SG2_H.A+、T_STB_H.A+ 三段在此汇合；CANL 同理。
5. seg2：HUB → 右下方 X3；HUBL 平行。
6. stub：HUB → 右上方 X2；HUBL 平行。
7. 电机2 终端：RTH_M2 / RTL_M2 分别接 CANH_M2 / CANL_M2，下端接地。
8. 探测电阻 RPR_*：上端接对应节点（CANH_M1/CANL_M1/RXD1/RXD2/RXD3），下端接地。
9. 电源/激励：VCC + 连 VCCP；VSTB1..3 + 连 STB1..3；VTX1 + 连 D_TXD1；VTX2/3 + 连 D_TXD2/3。
10. 所有电源/信号源负端、所有电阻下端、所有 T 的 A-/B-、X1..3 的 GND 引脚 → 地符号（`0`，Place → Ground，选 SOURCE/0 或直接 Net Alias 命名 `0`）。

---

## 5. 仿真指令文本

用 Place → Text（快捷键 T）把以下内容放到原理图空白处（**每行以 `.` 开头**，Capture 会自动把这些行并入 PSpice 网表）：

```
.PARAM L_TRUNK=1.0
.PARAM L_SEG2 =1.5
.PARAM S_STUB =0.3
.PARAM TDPM   =4.27n
.TRAN 0.1n 2.5u
.PROBE
.lib "E:\workspace2608\03\Ragtime_Firmwares\si_models\orcad\models\TCAN1042_q1_pspice_sch.lib"
.PRINT TRAN V(RTH_A) V(RTL_A) V(RTH_M2) V(RTL_M2)
.PRINT TRAN V(RPR_M1H) V(RPR_M1L) V(RPR_RXD1) V(RPR_RXD2) V(RPR_RXD3)
```

可选 stub 长度扫描（默认注释掉，需要时把下面两行加上并把上面 .TRAN 注释掉）：
```
.STEP PARAM S_STUB 0 3 0.25
.TRAN 0.1n 2.5u
```

> 不要在原理图里写 `.MEAS/.MEASURE` —— PSpice 不支持（会报 ORPSIM-16056），用 Probe 的 Evaluate Measurement 即可。

---

## 6. 仿真设置与运行

1. PSpice → New Simulation Profile → Name: `can_si`，OK
2. Analysis 页：Analysis type = **Time Domain (Transient)**，Run to time = `2.5u`，Maximum step = `0.1n`，勾选 **Skip the initial transient bias point calculation (SKIPBP)**（可选，避免 0 时刻收敛问题；如不勾也没关系）
3. 若启动仿真报 "can't find library"：检查第 5 节 `.lib` 绝对路径是否正确、文件是否存在
4. PSpice → Run（快捷键 F11）
5. Probe 里看：Trace → Add → `V(RTH_A)` / `V(RTL_A)`（= 板端 CANH/CANL），`V(RTH_M2)-V(RTL_M2)`（电机2 差分），`V(RPR_RXD1)`（RXD 数字回读）
   - 预期：RXD1 回放出 TXD 位序（dominant=低），TXD1 长 dominant 后 1415n 那一位 recessive 会出现最大过冲/振铃

---

## 7. 完成后的网表对照（自查）

PSpice → Create Netlist 生成的 .net 应与 .cir 一致，抽查关键行：

```
X1 D_TXD1 STB1 RXD1 CANH_B CANL_B VCCP 0 TCAN1042
T_TRK_H CANH_B 0 HUB 0 Z0=60 TD={L_TRUNK*TDPM}
T_SG2_H HUB 0 CANH_M2 0 Z0=60 TD={L_SEG2*TDPM}
T_STB_H HUB 0 CANH_M1 0 Z0=60 TD={S_STUB*TDPM}
RTH_A CANH_B 0 60
RTH_M2 CANH_M2 0 60
RPR_RXD1 RXD1 0 1Meg
VCC VCCP 0 5
VTX1 D_TXD1 0 PWL(0 5 400n 5 415n 0 600n 0 615n 5 800n 5 815n 0 1200n 0 1400n 0 1415n 5 1800n 5 1815n 0 2.2u 5)
```

若某行网表出现 `X1 1 2 3 4 5 6 7 TCAN1042`（数字引脚号）而没出现名字 —— 说明 TCAN1042 的 PSpiceTemplate 没生效，请回去检查第 2 步的属性。

---

## 8. 常见坑

1. **节点名不能以 A/B/J/K/M/N/P/Q/T/U/X/Z/_\ 开头**（PSpice 17.4 输出变量引用 bug），本图已全部用 C/D/H/R/S/V 开头，别改。
2. **T 元件 A-/B- 必须接地**，否则模型悬空报错。
3. `.lib` 路径建议绝对路径 + 双反斜杠 `\\` 或正斜杠 `/`。
4. 有 lossless T 元件时 `.PRINT V(节点名)` 会报 "PRINT device X is undefined"，所以探测一律用电阻 + `V(Rxx)`，本方案已内置 RPR_*。
5. RXD 极性：TXD 低 = dominant（RXD 低），若发现 RXD1 与 TXD1 同极性说明画图时把 VTX1 极性画反。