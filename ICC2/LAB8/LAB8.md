# LAB8 — 布线（Routing）

## 统一专业术语

| 术语 | 说明 |
|------|------|
| Routing | 布线 |
| CTS | 时钟树综合 |
| NDR | 非默认布线规则 |
| DRC | 设计规则检查 |
| skew | 时钟偏斜 |
| latency | 时钟延迟 |
| sink pin | 时钟接收端引脚 |
| balance point | 平衡点 |
| OCV | 片上工艺偏差 |
| ICG | 时钟门控 |
| LEQ | 逻辑等效单元 |
| track | 布线轨道 |
| via / cuts | 通孔 / 切口 |
| margin | 时序余量 |
| jitter | 时钟抖动 |

本实验带你掌握 IC Compiler II 中时钟树综合（CTS）的前期设置与执行。完成本实验后，你将掌握以下技能：

1. 设置时钟树平衡（Clock Tree Balancing）
2. 创建并应用非默认布线规则（Non-Default Routing Rules）
3. 应用时钟相关的时序与 DRC 约束

---

## 任务 1：加载设计与分析时钟

### 1. 切换至 CTS 实验工作目录，加载起始设计：

```shell
UNIX% cd lab8_cts
UNIX% icc2_shell -gui -f load.tcl
```

该脚本将打开一个已准备好后续操作的模块。

### 2. 打开 run.tcl 文件

在 ICC II 脚本编辑器中打开 `run.tcl` 文件（同前几个实验）。该文件包含本实验所有需要执行的命令。建议从文件中选取命令，使用 **Run Selection** 功能逐条执行，以节省时间并避免输入错误。

### 3. 打开时钟树分析窗口

选择菜单 **Window > Clock Tree Analysis Window**，当提示 `"... Do you want to continue?"` 时回答 **Yes**。

### 4. 展开 SDRAM_CLK

点击 `SDRAM_CLK` 前的 `+` 展开，可看到两个 `SD_DDR_CLK*` 时钟。注意：
- **Is Generated** 列对这两个时钟设为 `true`
- 它们的源时钟为 `sd_CK*`
- 时钟前有 **`M`** 和 **`G`** 符号，分别标识为主时钟（Master）和生成时钟（Generated）

**总结**：`SD_DDR_CLK` 和 `SD_DDR_CLKn` 是从主时钟 `SDRAM_CLK` 生成的时钟。`SDRAM_CLK` 施加于其源端口 `sdram_clk`；两个生成时钟分别施加于其源端口 `sd_CK` 和 `sd_CKn`。

### 5. 进一步分析 func 模式下的 SDRAM_CLK

右键点击 `SDRAM_CLK` → 选择 **Clock Tree Object List**。在弹出的 `Find CTS Object` 对话框中点击 **OK**。

你将看到一个新窗口，其中包含该时钟的所有接收端引脚（sink pins）。向下滚动到底部，还能看到输出端口 `sd_CK` 和 `sd_CKn`。向右滚动可查看更多引脚 / 端口属性。你可以拖动列的位置并排序以更好地显示信息。

完成后关闭 CTS 窗口。

### 6. 查看时钟树端点平衡点类型

生成时钟结构报告，查看时钟树端点上的平衡点类型（隐式忽略 / 显式忽略 / 接收端引脚）：

```tcl
report_clock_gor -type structure
```

在视图窗口中按 `Ctrl-F`（或点击 Search...），搜索字符串 `sd_CK`。应能看到以 `sd_CK [out Port]` 开头的一行，在该行末尾有一个平衡点例外标记（非默认的 SINK PIN）。

**问题 1**：`sd_CK` 上设置了什么平衡点例外？为什么？

> 解答见文末参考答案。

暂时不要关闭视图窗口。

---

## 任务 2：时钟树平衡（Clock Tree Balancing）

许多设计对时钟树有特殊或非默认要求，仅执行默认的时钟树综合是不够的。

CTS 默认只对接收端引脚（sequential cell 或 macro 的时钟引脚）进行平衡（最小化 skew）。如果有其他引脚需要与这些时钟引脚一起平衡，必须在 CTS 之前明确告知 ICC II。

### SDRAM 接口说明

下图展示了 SDRAM 接口。时钟 `SDRAM_CLK` 直接连接到 MUX 的选择引脚，这些 MUX 驱动 `ORCA_TOP` 模块的输出端口。

```
sd_DQ out [0]
      |
sdram_clk
      |
  [MUX] -- 生成时钟 --> SD_DDR_CLK
      |
    sd_CK
```

**Figure 1. SDRAM Interface**

驱动 `sd_CK` 的虚拟 MUX 是必需的，因为 DDR SDRAM 接口的时序要求严格——其在输出数据端口上同时使用时钟上升沿和下降沿产生数据。本设计要求优化从 `SDRAM_CLK` 到 `sd_DQ_out` 和 `sd_CK` 的时钟偏斜。

默认情况下，MUX 的 select 引脚（S0）被标记为**隐式忽略引脚（implicit ignore pins）**。要让 CTS 平衡这些引脚的 skew，需将其重新定义为接收端引脚（sink pins）。

### 操作步骤

**1.** 在之前打开的视图窗口中（任务 1 最后一步），注意 `sd_CK` 行上下方的 MUX select 引脚（`I_SDRAM_TOP/I_SDRAM_IF/sd_mux_CK*/S0`），它们同样是隐式忽略引脚。

**2.** 点击 **Dismiss Search**，然后退出视图窗口。

**3.** 为所有 mode 下的 S0 引脚应用平衡约束：

```tcl
foreach_in_collection mode [all_modes] {
    current_mode $mode
    set_clock_balance_points \
        -consider_for_balancing true \
        -balance_points [get_pins "I_SDRAM_TOP/I_SDRAM_IF/sd_mux_*/S0"]
}
```

**4.** 生成以下报告，验证用户定义的平衡点：

```tcl
report_clock_balance_points
```

你应能在以下标题下方看到这些引脚：

```
Clock Independent:
  Balance Points:
```

这些平衡点显示为 **Clock Independent**，因为未指定与例外关联的时钟。如果例外需要针对特定时钟进行平衡，且该点有多个时钟到达，则应指定时钟——此处不适用。

**5.** 再次查看时钟结构报告，搜索 `sd_mux`：

```tcl
report_clock_gor -type structure
```

**6.** 在视图窗口中搜索 `sd_CK`。

**问题 2**：MUX 的 S0（select）引脚现在如何标识？

**问题 3**：`sd_CK` 端口现在如何标识？这意味着什么？

**7.** 在视图窗口中点击 **Dismiss Search**，然后退出视图窗口。

**8.** 指示 CTS 不要更改 SDRAM MUX——它们已被选定以达到最佳时序，应保持不变：

```tcl
set_dont_touch [get_cells "I_SDRAM_TOP/I_SDRAM_IF/sd_mux_*"]
```

用以下命令验证设置：

```tcl
report_dont_touch I_SDRAM_TOP/I_SDRAM_IF/sd_mux_*
```

**9.** 返回 **Window > Clock Tree Analysis Window**，展开 `SYS_2x_CLK`（点击 `+`），应能看到 `SYS_CLK` 时钟。

**问题 4**：该生成时钟的源是什么？

**10.** 指示 CTS 不要更改用作时钟分频器的寄存器：

```tcl
set_dont_touch [get_cells "I_CLOCKING/sys_clk_in_reg"]
```

用上一步使用的报告命令验证。

**11.** 设置 skew 目标：所有慢 corner（ss）设为 0.05ns，所有快 corner（ff）设为 0.02ns。

**12.** 生成以下报告确认：

```tcl
report_clock_tree_options
```

**13.** 执行 CTS 时，通常希望使用特定单元进行综合，而非让 ICC II 从库中任意选择。例如：有助于减少 skew 的单元（相同的上升 / 下降沿转换时间）；有助于在功耗与速度 / 驱动强度、尺寸等之间更好平衡的单元。

CTS 专用单元通过以下命令定义：

```tcl
set_lib_cell_purpose -include cts
```

**14.** 首先，自动识别已在时钟网络上的门电路和 ICG，以及它们的**逻辑等效单元（LEQ）**：

```tcl
derive_clock_cell_references -output cts_leq_set.tcl
```

> 注意：上述命令仅在所有库单元已具有 `cts` purpose 时才能正常工作。

**15.** 查看生成的文件 `cts_leq_set.tcl`。所有当前在时钟网络上的单元及其 LEQ 均已被识别。你可以复制粘贴命令到新文件，取消注释适当行并稍后 source 执行——但**不需要**这样做，已为你准备好。

接下来，选择用于时钟树的缓冲器 / 反相器：

```tcl
set CTS_CELLS [get_lib_cells \
    "*/NBUFF*LVT */NBUFF*RVT */INVX*_LVT */INVX*_RVT */*DFFF*"]
set_dont_touch $CTS_CELLS false
set_lib_cell_purpose -exclude cts [get_lib_cells]
set_lib_cell_purpose -include cts $CTS_CELLS
```

从 `run.tcl` 中选择 / 运行这些行。然后 source 我们提供的版本（而非编辑 `cts_leq_set.tcl`）：

```tcl
source scripts/cts_include_refs.tcl
```

生成报告，确认关键 CTS 单元的 lib_cell purpose 已正确设置且 dont_touch 已移除：

```tcl
report_lib_cells -objects [get_lib_cells] \
    -columns {name:20 valid_purposes dont_touch}
```

在视图窗口中搜索字符串 `cts`。也可使用 workshop 提供的别名 `full_lib_report`。

---

## 任务 3：定义 CTS 非默认布线规则（NDR）

在本任务中，你将指定 CTS 的非默认布线规则以及时钟单元间距规则。

### 1. 查看 NDR 文件

用编辑器打开文件 `scripts/ndr.tcl`。

### 2. 审阅文件并回答问题

**问题 5**：两个时钟布线规则分别应用于时钟树的哪些网线段？

**问题 6**：两个规则之间有哪些关键区别？

### 3. 应用时钟 NDR

```tcl
source -echo scripts/ndr.tcl
```

### 4. 验证已创建的布线规则

```tcl
report_routing_rules -verbose
```

应能看到每个规则的金属层详细信息，以及通孔（via）部分。

### 5. 验证规则的应用位置

```tcl
report_clock_routing_rules
```

该报告显示各规则应用于哪些网线段（sink 覆盖 all），以及每个时钟线段的最小 / 最大层约束。

---

## 任务 4：时序与 DRC 约束

### 1. 生成主时钟源端口报告

```tcl
report_ports -verbose [get_ports *clk]
```

验证主时钟源均为输入端口，且均受 Driving Cell 或 input Transition 约束。

**问题 7**：是否所有时钟端口都受 Driving Cell 或 input Transition 约束？

**问题 8**：为什么时钟输入端口必须通过 `set_driving_cell` 或 `set_input_transition` 约束？

### 2. 修复发现的问题

为 `ate_clk` 端口添加 driving cell。Driving cells 需在所有 scenario 中添加，因为该约束是 scenario 相关的。

指定 `NBUFFX16_RVT` 作为 `ate_clk` 端口的 driving cell，然后重新报告时钟端口：

```tcl
foreach_in_collection scen [all_scenarios] {
    current_scenario $scen
    set_driving_cell \
        -lib_cell NBUFFX16_RVT [get_ports ate_clk]
}
report_ports -verbose [get_ports *clk]
```

### 3. 调整时钟不确定性（Clock Uncertainty）

首先查看当前时钟不确定性值：

```tcl
report_clocks -skew
```

接下来，更改时钟不确定性值以考虑 CTS 后的传播时钟时序。这是通过减小代表 skew 的数值来实现的。不确定性值仍应建模时钟抖动或额外时序余量的影响。

应用以下命令：

```tcl
foreach_in_collection scen [all_scenarios] {
    current_scenario $scen
    set_clock_uncertainty 0.1 -setup [all_clocks]
    set_clock_uncertainty 0.05 -hold [all_clocks]
}
```

### 4. 设置最大转换时间约束

在所有 func mode 的 corner 中，对所有时钟施加 0.15ns 的最大转换时间约束。

### 5. 启用时钟悲观移除

启用时钟重新收敛悲观移除（CRPR），消除 OCV 时序降额在共享 launch/capture 时钟树分支上的时序悲观。

### 6. 确认设置

生成以下报告确认最大转换时间设置：

```tcl
report_clock_settings
```

> **注意**：要查看正确的 max transition 信息，需向下滚动到 `##Global` 部分之后，搜索 `Mode = func` 部分，其中列出了所有独立时钟。报告先列出所有 mode/corner 的 Global 设置（本例中未设置），我们通过在当前 func mode 下使用 `-clock_path [get_clocks]`，对时钟应用了特定设置（应用于所有时钟）。

---

## 任务 5：执行 CTS 并分析结果

### 1. 构建并布线时钟树

```tcl
clock_opt -to route_clock
```

### 2. 在 GUI 中查看时钟布线

关闭电源和地网络的可见性，放大查看时钟布线。将鼠标悬停在网络上，弹出的查询窗口会显示应用的 NDR 布线规则。

### 3. 报告 skew

报告之前定义为 sink pins 的所有 `sd_mux*` 引脚之间的 skew：

```tcl
report_clock_gor \
    -to I_SDRAM_TOP/I_SDRAM_IF/sd_mux_*/S0 \
    -corners ss_125c
```

应发现全局 skew 非常小。

### 4. 查看时钟树延迟图

查看 SDRAM_CLK 的时钟树延迟图（latency graph）：
- **a.** GUI 中：**Window > Clock Tree Analysis Window**
- **b.** 勾选右上角的 **Clocks associated with corner** 复选框。这使得我们可以分析延迟，因为延迟计算是基于每个 corner 进行的（不选择 corner 时，延迟图的 x 轴显示 "levels of logic"）。
- **c.** 在 func scenario 下右键点击 **SDRAM_CLK**，选择 **Clock Tree Latency Graph of selected Corner**。

### 5. 进一步分析

时间允许的情况下，可自行进行其他感兴趣的分析。

---

## 参考答案

### 问题 1

**问题**：`sd_CK` 上设置了什么平衡点例外？为什么？

**解答**：设置为 **[IMPLICIT IGNORE PIN]**。时钟树接收端引脚端点默认为寄存器或宏单元的时钟引脚。`sd_CK` 是输出端口，因此默认被忽略，不参与平衡。

---

### 问题 2

**问题**：MUX 的 S0（select）引脚现在如何标识？

**解答**：标识为 **[BALANCE PIN]**，代表显式的、用户定义的接收端引脚例外。

---

### 问题 3

**问题**：`sd_CK` 端口现在如何标识？这意味着什么？

**解答**：`sd_CK` 端口现在标识为 **[BEYOND EXCEPTION]**，因为该端口位于 MUX 上 BALANCE PIN 例外的扇出之外（beyond）。CTS 的非默认设计规则以及通用的最大转换时间和电容设计规则将应用于时钟网络的 "beyond exception" 部分。

---

### 问题 4

**问题**：该生成时钟的源是什么？

**解答**：源是 `I_CLOCKING/sys_clk_in_reg/Q`。这是一个**除以 2 的寄存器**，用于对 `SYS_2x_CLK` 分频以生成 `SYS_CLK`。

---

### 问题 5

**问题**：两个时钟布线规则分别应用于时钟树的哪些网线段？

**解答**：

- **底部规则** `$CTS_LEAF_NDR_RULE_NAME`（`cts_w1_s2`）：应用于时钟网络的**接收端线段（sink segments）**（`-net_type sink`）。
- **顶部规则** `$CTS_NDR_RULE_NAME`（`cts_w2_s2_vlg`）：应用于时钟网络的**根线段和内部线段（root and internal segments）**，因为未指定 `-net_type`。

---

### 问题 6

**问题**：两个规则之间有哪些关键区别？

**解答**：

| 特性 | 接收端规则（cts_w1_s2） | 根 / 内部规则（cts_w2_s2_vlg） |
|------|------------------------|-------------------------------|
| 线宽 | 使用默认宽度 | 非默认宽度 |
| 间距 | 非默认间距 | 非默认间距 |
| 通孔规则 | 无 | 有（cuts 规则） |
| 布线层 | M1~M5 | 主要在 M4~M5 |
| M1 非默认间距 | 未指定（避免连接标准单元引脚时产生 DRC 违例） | 包含 |

根和内部时钟树规则为较低的非主要 CTS 布线层 M1~M3 也包含了 NDR——这通常是推荐的，因为这些网络最终需要在这些较低层上布线以连接到时钟引脚。

---

### 问题 7

**问题**：是否所有时钟端口都受 Driving Cell 或 input Transition 约束？

**解答**：**否**。你会发现时钟端口 `ate_clk` 出现在大多数报告部分，但不在 Driving Cell 或 Transition 部分（注意：Transition 部分仅当至少一个输入应用了 input transition 时才会出现，此处不适用）。这意味着该时钟端口没有 driving cell 或 input transition 约束。之前未看到该端口是因为它仅在 test mode 下被定义为时钟。

---

### 问题 8

**问题**：为什么时钟输入端口必须通过 `set_driving_cell` 或 `set_input_transition` 约束？

**解答**：在 CTS 期间，时钟插入延迟（clock insertion delay）会被计算以优化 skew 和 latency。上述两种约束中的任何一种都能让 CTS 更准确地计算连接到时钟输入端口的第一个门电路的延迟，从而得到更精确的时钟插入延迟计算（单元的延迟是其输入转换时间的函数）。默认情况下，输入端口的转换时间为 0ns，这可能导致结果不够准确。

---

## 实操复现补充提示

1. **命令执行规范**：ICC II 脚本编辑器中，选中单行 / 多行命令 → 点击 **Run Selection** 执行，逐条观察终端日志与 GUI 视图变化。

2. **日志查看**：重点关注 Error、Warning 信息。CTS 设置阶段若出现库关联、时钟约束或 NDR 报错，需先修复再继续。

3. **视图辅助**：
   - **Clock Tree Analysis Window**：查看时钟结构、延迟图、平衡点
   - **查询窗口**：鼠标悬停网络可查看应用的 NDR 规则
   - 关闭电源 / 地网络显示可更清晰地查看时钟布线

4. **耗时说明**：`clock_opt -to route_clock` 执行时间视设计复杂度而定，执行期间请勿操作工具，等待命令完成后生成报告。
