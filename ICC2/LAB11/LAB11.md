# LAB11 — 布线及后优化（Routing and Post-Route Optimization）

## 学习目标

完成本实验后，你将掌握以下技能：

1. 对完成时钟树综合（CTS）后的设计执行可布线性检查
2. 应用布线选项（routing options）
3. 对次级 PG 网络进行布线
4. 通过优化进行控制
5. 执行布线及后优化（routing and post-route optimization）
6. 在开启信号完整性（SI）分析模式下分析设计时序，并执行增量功耗和串扰优化
7. （选做）执行签核 DRC 检查与修复、标准单元填充单元插入（standard cell filler insertion），以及签核金属填充插入（sign-off metal fill insertion）

**实验时长：** 45 分钟

---

## 任务 1：加载并检查 CTS 后的设计

1. 切换至布线（Routing）实验的工作目录，然后加载 CTS 完成后的设计：

```shell
UNIX% cd lab9_11_route_signoff
UNIX% icc2_shell -gui -f load.tcl
```

该脚本会复制 `clock_opt` 模块并打开。

2. 生成时序结果质量（QoR）汇总报告：

```tcl
report_qor -summary
```

**问题 1：** 当前时序是否满足布线要求？

---

## 任务 2：对设计进行布线

为了提供更具沉浸感和趣味性的实验体验，本实验**不提供**逐步的详细指令。

取而代之的是，请你用编辑器打开 `run.tcl` 文件（或使用 ICC II 脚本编辑器），逐行执行其中的命令。**请特别留意不要一次性 source 整个文件**，否则会失去理解各选项和命令之间如何协同工作的意义。如果某个选项让你感到困惑，请查阅其 man 手册。

以下各小节提供了补充信息和注释，按顺序排列，便于你在执行脚本时参照查阅。

如果你愿意，也可以偏离 `run.tcl` 中的命令，或者尝试不同的 effort 级别、不同设置等。不过请注意，运行时间可能会因设置不同而变化。总体而言，本实验运行速度非常快，因此鼓励你多做尝试。

以下内容旨在作为本实验的指南，包含需要配置和运行的命令相关信息，以及一些附加思考题。

如有问题，请咨询你的 instructor。

### 布线前检查（Pre-routing Checks）

在对设计进行布线之前，最好确保没有会阻止布线器正常工作的隐患。

**问题 2：** 当前设计是否已准备好进行布线？

### 天线效应（Antenna）

天线定义通常以单独的 TCL 文件提供，与工艺技术相关，使用以下命令（与 IC Compiler 中的命令相同）：

```tcl
define_antenna_rule
define_antenna_layer_rule
define_antenna_area_rule
define_antenna_accumulation_mode
define_antenna_layer_ratio_scale
```

此外，还有一些应用选项（application options）控制天线违例的处理方式。使用 `run.tcl` 中所示的 `report_app_options` 命令进行查看。

### 串扰预防（Crosstalk Prevention）

串扰预防力求确保时序关键网络不会被长距离并行布线。预防可以在全局布线（global routing）和轨道分配（track assign）阶段进行。当前的建议是仅在轨道分配阶段启用预防。为了使串扰预防生效，还必须启用串扰（或信号完整性，SI）分析。为了使后布线分析和优化更具趣味性（展示需要在 `route_opt` 期间修复的 SI 违例），你也可以选择在布线期间不启用 SI 分析来"人为地"制造 SI 违例。

### 次级 PG 布线（Secondary PG Routing）

次级 PG 引脚是特殊单元（如电平转换器或隔离单元）上的电源引脚。除了通过标准单元 rail 提供的常规电源外，这些次级电源引脚还需要通过标准布线器进行布线。有几个布线参数用于配置布线行为。此外，通常会为这些电源连接定义非默认规则（non-default rules）。

按照脚本中的说明执行次级 PG 布线后，找到 `PD_RISC_CORE` 电压区域中的低到高电平转换器，检查其电源布线，并回答以下问题：

**问题 3：** 次级 PG 引脚的名称是什么？它们连接到什么位置？

### 布线及 DRC 分析（Routing, DRC Analysis）

完成次级 PG 布线以及"关键"信号布线后，执行自动布线（auto-route）。在此阶段，所有之前未布线的信号网络都将被布线。任何已经布线的信号（时钟、次级 PG）如果 DRC 干净，则不会被重新布线。自动布线包括全局布线（global routing）、轨道分配（track assignment）和详细布线（detail routing）。

**问题 4：** 默认情况下，详细布线（detail route）迭代运行多少次？（提示：查看 `route_auto` 的 man 手册。）

**问题 5：** 如何更改默认值？如何强制布线器即使看不到改进也要跑完所有迭代？（提示：`report_app_options *iterat*`）

### 检查布线设计规则违例（Examining Routing Design Rule Violations）

使用错误浏览器（Error Browser）可视化 GUI 中遗留的布线违例。在顶部工具栏上，选择错误浏览器按钮。

在弹出的窗口中，选择 `zroute.err` 并点击 **Open Selected**。在新窗口中，你可以从列表中选中错误，布局视图将自动缩放至该违例位置。

关闭错误浏览器（Error Browser）。

### 信号完整性（Signal Integrity）

信号完整性分析应在布线之前开启。这将指示提取引擎提取交叉耦合电容，并指示布线器使用这些耦合电容执行包含 delta delay 的时序分析。这在执行时序驱动布线（timing-driven routing）时非常重要。

在本实验中，如果你在自动布线前关闭了 SI 分析，可以在自动布线后将其开启，以观察 SI 效应，并允许在 `route_opt` 期间修复 SI 相关违例。

为了更好地与 PrimeTime 关联，还应启用时序窗口分析（timing window analysis），如脚本中所示。

### 后布线优化（Post-Route Optimization）

后布线优化通过 `route_opt` 执行，该命令执行时序、逻辑 DRC、面积以及（可选）CCD 和功耗优化。

需要设置一些应用选项来启用 CCD 和功耗优化。

为了获得与 PrimeTime 的最佳相关性，应启用 PT 延迟计算（PT delay calculation）。在本实验中，不要在 `route_opt` 中执行 StarRC InDesign 提取。后续在执行 ECO Fusion 时会使用 StarRC。

### ECO Fusion

完成 `route_opt` 后，使用 StarRC 寄生参数提取在 PrimeTime 中分析签核时序非常重要。PrimeTime 发现的任何违例都可以通过 PT 的物理 ECO（physical ECO）进行修复。

借助 Fusion 技术，整个 ECO 过程可以在 ICC II 内部完全控制。

查看 `run.tcl` 文件中的相关命令，并执行一轮 ECO 修复。对于 Fusion，你需要 ICC II、StarRC 和 PrimeTime SI。

请注意，在 ECO 实施完成后，必须使用 `check_pt_qor` 命令分析时序。**不应再使用 ICC II 自身的时序报告命令**（`report_qor`, `report_timing` 等）。

### 标准单元填充、ICV DRC、金属填充（Std Cell Fillers, ICV DRC, Metal Fill）

在所有后布线优化完成且任何必要的 ECO 执行完毕后，按 `run.tcl` 脚本中的说明执行标准单元填充单元插入（standard cell filler insertion）。

如果你想要执行签核 DRC 检查，请查看后续步骤。你将找到执行 DRC 检查和 ICV DRC 自动修复的命令。

请按所示使用 `select_rules` 过滤器，否则你会看到大量因我们的 techfile 与 ICV DRC runset 不匹配而产生的违例。

你可以使用错误浏览器查看 ICV 生成的错误。如果错误浏览器已打开，请使用 **File → Open...** 并选择 `signoff_check_drc.err`。否则，在错误浏览器弹窗中选择该错误文件（Checker 列会显示 "IC Validator"）。

作为最后一步，使用 IC Validator 执行金属填充（metal filling）。查看 `run.tcl` 脚本的最后部分。

按照讲义中的说明，使用 GUI 检查金属填充。

**恭喜！你已成功完成本布线实验。**

---

## 参考答案

### 问题 1

**问题：** 当前时序是否满足布线要求？

**解答：** 你应该会看到仅剩少量小幅时序违例。此外还有少量最大 transition 违例和最大 capacitance 违例（使用 `report_constraints -all` 可以看到它们）。

### 问题 2

**问题：** 当前设计是否已准备好进行布线？

**解答：** 使用 `check_design` 命令会显示一些问题。请在消息窗口中查看 EMS 问题：使用 **Window → Message Browser Window**，然后使用 **File → Open Message Database...** 打开 `check_design.ems` 文件。

对于本实验，你可以忽略 EMS 消息。

对于其余非 EMS 消息，你需要查看 `check_design*.log` 文件。

- ZRT-022 警告缺少层 `co` 的默认接触孔（default contact），这在当前工艺下是完全正常的。
- ZRT-044 列出了几个没有有效 via 区域的单元，这应在库准备阶段在对应的物理库中处理。
- ZRT-585 报告了单元内部引脚的问题，这对布线器不相关。
- 最后，请忽略 ZRT-511。

### 问题 3

**问题：** 次级 PG 引脚的名称是什么？它们连接到什么位置？

**解答：** 低电压的 PG 引脚称为 **VDDL**。布线器将它们连接到最近的 VDD 垂直 strap 或水平 rail，这些 rail 位于电压区域边界外侧。你可以使用 `report_power_domains` 验证电压值。

### 问题 4

**问题：** 默认情况下，详细布线迭代运行多少次？

**解答：** 默认值为 **40**。

### 问题 5

**问题：** 如何更改默认值？如何强制布线器即使看不到改进也要跑完所有迭代？

**解答：** 使用以下命令更改默认值：

```tcl
route_auto -max_detail_route_iterations <#>
```

要强制 `route_auto` 实际跑完所有迭代，需要将一个应用选项从其默认值（false）更改为 true：

```tcl
set_app_options -name route.detail.force_max_number_iterations -value true
```
