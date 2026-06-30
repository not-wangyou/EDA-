# LAB7 — 运行时钟树综合 (Running CTS)

## 学习目标

本实验带你熟悉 **IC Compiler II（ICC II）** 中**时钟树综合（CTS）**与**CTS 后优化**的相关功能。完成本实验后，你将掌握以下技能：

1. 掌握时钟树综合与数据路径优化的基本设置
2. 运行经典 CTS 流程或 CCD（Concurrent Clock & Data）流程
3. 分析 CTS 结果质量（QoR）
4. 使用时钟抽象图 GUI 分析时钟树拓扑

---

## 实验简介

本实验将在 ORCA_TOP 设计上，按照推荐流程执行时钟树综合，并应用多种设置。通过生成各类报告来监控和追踪设计进度。

**实验时长**：70 分钟

---

## 任务 1：加载并分析布局后的设计

本任务将加载布局优化后的设计结果，并执行 CTS 前置检查。

**1. 切换至 CTS 实验工作目录，加载布局后的设计：**

```shell
UNIX% cd lab7_cts
UNIX% icc2_shell -gui -f load.tcl
```

该脚本将复制 `place_opt` 模块并打开。

**2. 打开 `run.tcl` 文件**（ICC II 脚本编辑器），本文件包含本实验将要执行的全部命令。选择命令后使用「Run Selection」执行，可节省时间并避免输入错误。

**3. 生成时序 QoR 汇总报告：**

```tcl
report_qor -summary
```

> **问题 1**：从时序角度看，该设计是否已准备好进行 CTS？保持时间违例情况如何？

---

**4. 将当前模式（mode）设为 `func`。**

**5. 生成时钟报告：**

```tcl
report_clocks
```

> **问题 2**：定义了多少个主时钟（master clocks）？
>
> **问题 3**：定义了多少个生成时钟（generated clocks）？
>
> **问题 4**：其余时钟属于什么类型？
>
> **问题 5**：`SD_DDR_CLK` 时钟的源（source）是什么？

---

**6. 生成时钟偏差报告：**

```tcl
report_clocks -skew
```

> **问题 6**：建立时间不确定度（Setup Uncertainty）的最小值/最大值是多少？保持时间不确定度（Hold Uncertainty）的最小值/最大值是多少？

---

**7. 生成时钟组报告：**

```tcl
report_clocks -groups
```

> **问题 7**：哪些时钟组是互斥（mutually exclusive）或异步（asynchronous）的？

---

**8. 生成两种模式下的时钟树汇总报告（默认报告）：**

```tcl
report_clock_qor -modes func
report_clock_qor -modes test
```

> **问题 8**：两种模式之间的最大差异是什么？（提示：观察时钟名称）
>
> **问题 9**：`SD_DDR_CLK` 有多少个接收端（sinks）？

---

**9. 生成 `SD_DDR_CLK` 起点/源端口的报告（参见问题 5）：**

```tcl
report_ports [get_ports sd_CK]
```

> **问题 10**：为什么 `SD_DDR_CLK` 的接收端数量为零？（提示：观察端口方向）

---

## 任务 2：CTS 设置与配置

**1. CTS 设置**（时钟树平衡约束、NDR 规则等）将在下一单元中详细讲解，此处通过加载脚本来完成设置：

```tcl
source scripts/cts_ex_ndr.tcl
```

**2. 确认用于保持时间修复的场景（scenario）已正确启用。查找所有已激活且已启用保持时间修复的场景：**

```tcl
get_scenarios -filter active&&hold
report_scenarios
```

> **问题 11**：场景是否已正确配置用于保持时间修复？

完成保持时间的场景设置（参考 `run.tcl`），同时双重确认所有场景均已激活。

**3. 使用 `set_lib_cell_purpose` 控制用于保持时间修复的缓冲单元或延迟单元。从 `run.tcl` 中执行对应的三条命令。**

**4. （可选）如需提高保持时序优化的努力级别，可执行以下命令（非本设计必需）：**

```tcl
set_app_options -name clock_opt.hold.effort -value high
```

**5. 为减少扫描链的保持时间违例，建议启用扫描链重排序，以最小化时钟缓冲单元之间的交叉：**

```tcl
set_app_options -name opt.dft.clock_aware_scan_reorder -value true
```

**6. 启用时钟收敛悲观性去除（CRP）功能**，从 `run.tcl` 中执行对应行。

---

## 任务 3：CTS 后的 I/O 延迟

本设计中，除 `v_PCI_CLK` 外，我们不希望更新其他 I/O 延迟。`v_PCI_CLK` 是一个虚拟时钟（virtual clock），因此需要配置延迟调整以更新该时钟：

```tcl
foreach_in_collection mode [all_modes] {
  current_mode $mode
  set_latency_adjustment_options -exclude_clocks "*"
  set_latency_adjustment_options \
    -reference_clock PCI_CLK \
    -clocks_to_update v_PCI_CLK
}
```

> **说明**：通常情况下，如果输入/输出约束的时钟与内部时钟相同，则无需额外配置——它们的 I/O 延迟将自动调整。

---

## 任务 4：运行全面时钟树检查

经过上述手动分析后，生成时钟树检查报告，查看 CTS 过程中可能出现哪些潜在问题：

```tcl
check_clock_trees
```

你应看到一份较长的报告，包含「Summary」和「Details」两部分。汇总部分显示各类问题的数量及建议解决方案，例如：

```
CTS-019      2      None     Clocks propagate to output ports
CTS-905      4      None     There are clocks with no sinks
```

详情部分会针对每条告警提供更多细节。例如 CTS-0905 报告末尾指出无接收端的时钟，即之前步骤中已分析的 `SD_DDR_CLK` 和 `SD_DDR_CLKn` 相关端口（在 func 和 test 模式下各重复一次）。

更多告警示例：

| 告警编号 | 数量 | 说明 |
|----------|------|------|
| CTS-903 | 46 | 时钟网络中例化的单元不在时钟参考列表中 |
| CTS-904 | 12 | 部分时钟参考单元未指定用于尺寸调整的 LEQ 单元 |
| CTS-009 | 12 | 时钟树中的单元实例在相同引脚间存在多个条件延迟弧 |
| CTS-013 | 35 | 时钟网络中的单元设置了 `dont_touch` 约束 |

CTS-903 和 CTS-904 表示时钟树中使用的部分单元未启用 CTS 库单元用途，或者已启用但它们的逻辑等价单元（LEQ）未启用——这意味着 CTS 将无法调整这些单元的尺寸。

CTS-013 列出了我们已应用 `dont_touch` 的单元。

> 在本实验中，上述所有告警均可接受，可以忽略。

---

## 经典 CTS 还是 CCD？

由于实验时间有限，你很可能只够运行一种 CTS 流程——请选择**经典 CTS 流程（选项 A — 任务 5 和 6）**或 **CCD 流程（选项 B — 任务 7）**。

> **注意**：对本设计而言，CCD 的运行时间比经典 CTS 更长。

若提前完成一个流程，可尝试另一个流程。

---

## 任务 5：选项 A — 执行经典 CTS

> **注意**：若你在已完成选项 B（CCD）后执行此步骤，则需要重新加载设计并重新应用所有设置。为简化操作，只需重启 ICC II 并使用脚本 `scripts/load_all.tcl`——该脚本将重新加载设计并完成之前的所有设置。

**1. 执行时钟树综合并绕线时钟树：**

```tcl
clock_opt -to route_clock
```

该命令只需几分钟即可完成，它将执行前两个阶段：`build_clock` 和 `route_clock`。

**2. 运行完成后，查看 CTS 偏差结果。观察所有模式/角点下的结果后，记录功能模式下最差角点 `ss_125c` 的结果：**

```tcl
report_clock_qor
report_clock_qor -type local_skew
report_clock_qor -type area
report_clock_qor -mode func -corner ss_125c -significant_digits 3
```

**3. 记录各项时钟 QoR 统计数据：**

- 时钟缓冲单元（时钟中继器）数量：
- 时钟树面积（中继器 + 其他网络单元）：

在慢角点 `ss_125c` 下，记录以下时钟的全局偏差/局部偏差/最大延迟数值：

| ss_125c Corner | Global Skew | Local Skew | Max Latency |
|----------------|-------------|------------|-------------|
| SYS_2x_CLK     |             |            |             |
| SDRAM_CLK      |             |            |             |

---

**4. 另一个非常有用的报告是鲁棒性（robustness）报告：**

```tcl
report_clock_qor -type robustness -mode func \
  -corner ff_m40c -robustness_corner ss_125c
```

**多角点鲁棒性**衡量每个时钟接收端的延迟在测量角点与参考角点（通过 `-robustness_corner` 指定）之间的缩放关系。每对角点都有一个缩放因子，即测量角点的平均延迟除以参考角点的平均延迟。单个接收端的鲁棒性值是其测量角点延迟与参考角点延迟的比值，再相对于该对角点的缩放因子进行归一化。

- 鲁棒性值为 1：表示该接收端在测量角点和参考角点之间呈现典型的缩放特性。
- 鲁棒性值大于 1：表示该接收端在测量角点的延迟高于平均水平。
- 鲁棒性值小于 1：表示该接收端在测量角点的延迟低于平均水平。

最大和最小鲁棒性值的接收端具有最差的多角点鲁棒性，可能在测量角点引发时序问题。若某时钟或偏差组的所有接收端具有相似的鲁棒性值，则认为该时钟树具有多角点鲁棒性，偏差可在这些角点间保持稳定。

---

**5. 使用时钟时序报告生成不同的偏差报告。关注 func 模式和最差角点：**

```tcl
report_clock_timing -type skew -modes func \
  -corners ss_125c -significant_digits 3
```

报告的偏差是最大与最小延迟数值之差，加上或减去时钟收敛悲观性（CRP）。

记录指定时钟的偏差和最大延迟：

|            | Skew | Latency (max) |
|------------|------|---------------|
| SYS_2x_CLK |      |               |
| SDRAM_CLK  |      |               |

> **说明**：报告右侧符号的定义
>
> - W — Worst-case operating condition（最差工况）
> - B — Best-case operating condition（最佳工况）
> - R — Rising transition（上升沿）
> - F — Falling transition（下降沿）
> - P — Propagated clock to this pin（传播时钟至该引脚）
> - I — Clock inversion to this pin（时钟反相至该引脚）
> - + — Launching transition（发射沿）
> - - — Capturing transition（捕获沿）

> **问题 12**：为什么 `report_clock_qor` 和 `report_clock_timing` 报告的偏差值不同？

---

## 任务 6：执行 CTS 后优化

本任务将对非时钟网络逻辑进行优化以解决时序违例，并首次执行保持时间修复。

**1. 生成优化前的时序汇总报告：**

```tcl
report_qor -summary
```

记录最差情况（全设计）的建立时间和保持时间的 WNS/TNS/NVE 数值：

|       | WNS | TNS | NVE |
|-------|-----|-----|-----|
| Setup |     |     |     |
| Hold  |     |     |     |

---

**2. 执行 CTS 后优化：**

```tcl
clock_opt -from final_opto
```

---

**3. CTS 后优化完成后，生成另一份时序汇总报告：**

> **问题 13**：是否还有剩余的建立时间或保持时间违例？

---

**4. 确认设计无拥塞问题。**

**5. 保存模块，继续任务 8。**

---

## 任务 7：选项 B — 并发时钟与数据流（CCD）

本任务使用 CCD 流程构建时钟树并优化逻辑。请注意，CCD 相比经典 CTS 运行时间更长。如果你一开始就选择了选项 B，则从第一步继续。

> **注意**：若你在已完成选项 A 后执行此步骤，则需要重新加载设计并重新应用所有设置。为简化操作，只需重启 ICC II 并使用脚本 `scripts/load_all.tcl`——该脚本将重新加载设计并完成之前的所有设置，准备运行 CCD。

**1. 加载以下脚本**——该脚本将通过引入一些建立时间违例来增加 CCD 的挑战性，这些违例需要由 CCD 算法修复。之后查看时序汇总：

```tcl
source scripts/margins_for_ccd.tcl
report_qor -summary
```

**2. 记录最差情况（全设计）的建立时间和保持时间的 WNS/TNS/NVE 数值：**

|       | WNS | TNS | NVE |
|-------|-----|-----|-----|
| Setup |     |     |     |
| Hold  |     |     |     |

---

**3. 启用 CCD 流程。**

**4. 运行默认的 `clock_opt` 命令。** 这是运行 CCD 流程的推荐方式，将执行全部三个阶段（`build_clock`、`route_clock` 和 `final_opto`）。运行时间至少 15 分钟。可以趁此休息一下。

当然，你也可以选择逐个阶段独立运行，并执行中间分析。

**5. 运行完成后，记录设计 QoR，查看 CCD 是否修复了所有人为引入的违例。** 注意：不要与经典 CTS 的结果进行比较！为了 CCD 演示，时序被人为恶化。

**6. 保存模块，继续任务 8。**

---

## 任务 8：分析

**1. 使用时钟抽象图 GUI 查看综合后的时钟树。** 在 GUI 中选择 `Window > Clock Tree Analysis Window`。

**2. 在新窗口（CTS Window）中，勾选右上角「Clocks associated with corner」旁边的小方框。** 这样可以分析随角点变化的时钟延迟。确保选中 `ss_125c` 角点。

**3. 在窗口主区域列出所有时钟，进入 func 场景部分，右键点击 `SYS_2x_CLK`，然后选择「Clock Tree Latency Graph of selected Corner」。**

下图展示了经典 CTS 流程的 `SYS_2x_CLK` 延迟图（CCD 截图见下一页）：

> （截图占位区域 — 经典 CTS 延迟图）

下图展示了 CCD 流程的 `SYS_2x_CLK` 延迟图：

> （截图占位区域 — CCD 延迟图）

正如预期，CCD 的寄存器端点延迟分布比经典 CTS 更大。

**4. 关闭 CTS Window。**

**5. 查看时钟树绕线拓扑结构。** 选择「Clock Tree」可视化模式。在右侧打开的面板中，点击右上角的小齿轮图标显示面板设置，确保勾选「Auto Apply」。

从时钟列表中选择单个时钟，即可在布局窗口中查看其绕线拓扑结构。你还可以选择显示拓扑结构的层级数量和具体级别。

> **注意**：Level 0 是连接到时钟根或时钟源的线网，Level 1 是由第一个驱动单元驱动的线网，以此类推。

**6. 关闭 Clock Tree 可视化模式面板。**

**7. 查看受 `v_PCI_CLK` 约束的某个时钟的时序：**

```tcl
report_timing -from [get_clocks v_PCI_CLK]
```

> **问题 14**：输入端口 `v_PCI_CLK` 上的网络延迟是否已传播（propagated）？

🎉 **恭喜你！你已成功完成 Running CTS 实验。**

---

## 参考答案

### 问题 1

**问题**：从时序角度看，该设计是否已准备好进行 CTS？保持时间违例情况如何？

**解答**：不应存在建立时间违例。存在保持时间违例，这些违例将在 CTS 过程中处理。

---

### 问题 2

**问题**：定义了多少个主时钟（master clocks）？

**解答**：定义了 **3 个主时钟**。主时钟具有时钟源，且不是生成时钟（无「G」标记）。

`PCI_CLK`、`SDRAM_CLK` 和 `SYS_2x_CLK`

---

### 问题 3

**问题**：定义了多少个生成时钟（generated clocks）？

**解答**：**3 个生成时钟**：`SD_DDR_CLK`、`SD_DDR_CLKn` 和 `SYS_CLK`。查看报告顶部区域：Attrs 列显示字母 G（表示 generated）。时钟列表下方的区域包含生成时钟的详细信息。

---

### 问题 4

**问题**：其余时钟属于什么类型？

**解答**：其余时钟是**虚拟时钟（virtual clocks）**，它们是 `v_SDRAM_CLK` 和 `v_PCI_CLK`。虚拟时钟在 Sources 列中显示为空字段。

---

### 问题 5

**问题**：`SD_DDR_CLK` 时钟的源（source）是什么？

**解答**：从报告顶部区域的 Sources 列可以看出：`sd_CK`，它是 `SDRAM_CLK` 的一个生成时钟。

---

### 问题 6

**问题**：建立时间不确定度（Setup Uncertainty）的最小值/最大值是多少？保持时间不确定度（Hold Uncertainty）的最小值/最大值是多少？

**解答**：根据 `report_clocks -skew` 报告：

- **Setup Uncertainty**：0.10 ns（`*ff*` 场景），0.30 ns（`*ss*` 场景）
- **Hold Uncertainty**：0.05 ns（`*ff*` 场景），0.10 ns（`*ss*` 场景）

---

### 问题 7

**问题**：哪些时钟组是互斥或异步的？

**解答**：报告列出了三个异步时钟组：

1. `SYS_2x_CLK` 和 `SYS_CLK`
2. `PCI_CLK` 和 `v_PCI_CLK`
3. `SDRAM_CLK`、`v_SDRAM_CLK` 和 `SD_DDR_CLK*`

同一组内的所有时钟彼此同步，但不同组之间的时钟彼此异步。时钟组可以是 `logically_exclusive`、`physically_exclusive` 或 `asynchronous`。互斥或异步时钟之间的路径在时序分析中不被考虑（类似于 `set_false_path`）。此外，`logically_exclusive`、`physically_exclusive` 和 `asynchronous` 关系还会影响时钟间的串扰分析方式。

---

### 问题 8

**问题**：两种模式之间的最大差异是什么？（提示：观察时钟名称）

**解答**：测试模式（test mode）多了一个时钟：`ate_clk`。

---

### 问题 9

**问题**：`SD_DDR_CLK` 有多少个接收端？

**解答**：**零个接收端**。

---

### 问题 10

**问题**：为什么 `SD_DDR_CLK` 的接收端数量为零？（提示：观察端口方向）

**解答**：`SD_DDR_CLK` 是一个在端口 `sd_CK` 上定义的生成时钟（如 `report_clocks` 所示）。通过 `report_ports [get_ports sd_CK]` 可以看到这是一个**输出端口**，因此没有接收端。

---

### 问题 11

**问题**：场景是否已正确配置用于保持时间修复？

**解答**：你会发现有三个 `*ff*` 场景，其中有一个未配置保持时间修复：`test_ff_125c`。由于这是一个快速场景（fast scenario），需确保它已启用保持时间修复。

---

### 问题 12

**问题**：为什么 `report_clock_qor` 和 `report_clock_timing` 报告的偏差值不同？

**解答**：`report_clock_qor` 命令默认报告**全局偏差（global skew）**，即整个时钟域中所有接收端之间的最大偏差（最长与最短插入延迟之差），即使这些极端时钟路径之间没有关联关系。此外，`report_clock_qor` 不使用时序缩退因子（timing derates），即使你的角点中已定义了它们。你也可以使用 `report_clock_qor -type local_skew` 生成局部偏差报告。

而 `report_clock_timing` 报告仅计算**局部时钟偏差（local clock skew）**：报告的最大偏差是在共享时序路径的两个触发器之间（一个时钟分支发射数据，另一个分支捕获数据）。`report_clock_timing` 命令还会考虑时序缩退因子——对于建立时间分析，它对发射时钟路径应用 late derates，对捕获时钟路径应用 early derates。后一种报告在确定真实的最差情况时钟偏差时更加精确。

---

### 问题 13

**问题**：是否还有剩余的建立时间或保持时间违例？

**解答**：应仅剩少量小的建立时间和保持时间违例。

---

### 问题 14

**问题**：输入端口 `v_PCI_CLK` 上的网络延迟是否已传播？

**解答**：**否**。在报告中，「clock network delay (ideal)」下列出了一个延迟数值。这是 ICC II 为 `PCI_CLK` 计算并自动更新的理想延迟值，已应用于 `v_PCI_CLK`（因为我们在任务 3 中做了相应配置）。

---
