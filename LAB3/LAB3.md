# LAB3

<!-- TOC -->
## 目录

- [实验 3 布局与优化](#实验-3-布局与优化)
    - [统一专业术语](#统一专业术语)
- [任务 1：启动 ICC II 并加载 ORCA_TOP 设计模块](#任务-1启动-icc-ii-并加载-orca_top-设计模块)
- [任务 2 执行布局与优化](#任务-2-执行布局与优化)
    - [模块 1：布局前检查（Pre-placement Checks）](#模块-1布局前检查pre-placement-checks)
    - [模块 2：布局前基础配置（Pre-placement Settings）](#模块-2布局前基础配置pre-placement-settings)
    - [模块 3：漏功耗 + 动态功耗优化配置](#模块-3漏功耗-动态功耗优化配置)
    - [模块 4：层优化 & 布线驱动寄生提取（RDE）](#模块-4层优化-布线驱动寄生提取rde)
    - [模块 5：集成时钟门控（ICG）优化](#模块-5集成时钟门控icg优化)
    - [模块 6：布局执行与结果分析](#模块-6布局执行与结果分析)
  - [五、 参考答案（Answers / Solutions）](#五-参考答案answers-solutions)
  - [六、 实操复现补充提示](#六-实操复现补充提示)

<!-- /TOC -->


## 实验 3 布局与优化

#### 统一专业术语

placement 布局 \| optimization 优化 \| pre-check 前置检查timing 时序 \| congestion 拥塞 \| violation 违例MCMM 多角多模式 \| Vt 阈值电压 \| RDE 路由驱动提取ICG 时钟门控 \| slack 时序裕量 \| fanout 扇出

本实验带你掌握 IC Compiler II 的布局与优化操作。完成本实验后，你将掌握以下技能：

1. 执行布局前置检查

2. 运行布局优化命令 place_opt

3. 分析布局结果（时序、拥塞、设计质量）

4. 使用设计浏览器分析各类违例问题

## 任务 1：启动 ICC II 并加载 ORCA_TOP 设计模块

- 切换至本实验工作目录，以图形界面模式启动 ICC II，并加载设计：

- shell

- \# Linux终端执行

```shell
UNIX% cd lab3_place
```

```shell
UNIX% icc2_shell -gui -f load.tcl
```

- 该脚本会将已完成前期准备的模块 ORCA_TOP/init_design 复制为 ORCA_TOP/place_opt，并自动打开该模块。

- 生成**结果质量（QoR）汇总报告**，查看初始状态：

- tcl

```tcl
- report_qor -summary
```

- 执行后可观察到：此时设计尚未执行布局与优化，**最大负时序裕量（WNS）、总负时序裕量（TNS）数值均偏大**。

- <img src="assets/media/image1.png" style="width:5.75972in;height:2.52778in" />

## 任务 2 执行布局与优化

```tcl
请逐行运行 run.tcl 中的命令,下文按功能模块拆分解读,并附带思考题。
```

#### 模块 1：布局前检查（Pre-placement Checks）

正式布局前，需排查会导致 place_opt 运行异常的问题，结合脚本命令完成以下思考题：

> 问题 1：设计中是否还存在未处理的理想网络（ideal nets）？问题 2：当前模块设置的最大布线层是哪一层？补充说明：本设计基于 SCANDEF 定义了扫描链（扫描测试电路）。问题 3：该设计包含多少条扫描链？高扇出网络会严重影响布局与时序，ICC II 中使用 report_net_fanout 分析扇出，结合脚本回答：问题 4：扇出数大于 60 的**非时钟类**高扇出网络一共有多少条？

#### 模块 2：布局前基础配置（Pre-placement Settings）

对于 12nm 及以下先进工艺，需使用 set_technology 命令配置 ICC II，该命令会批量修改工具配置项以适配对应工艺。若要在布局优化过程中插入**钳位单元（tie cells）**，需保证库中的钳位单元未被添加 dont_touch（禁止修改）属性，同时将其纳入优化单元范围。

> 问题 5：使用哪条命令 / 配置项可将单元加入优化范围？

#### 模块 3：漏功耗 + 动态功耗优化配置

默认情况下，place_opt 不会开启漏功耗、动态功耗优化；若需功耗优化，必须手动开启，并保证至少有一个工作场景（scenario）启用对应功耗分析。**工具配置项**：opt.power.mode（功耗优化模式）

#### 模块 4：层优化 & 布线驱动寄生提取（RDE）

**层优化（Layer Optimization）**工具会识别长走线、时序关键网络，并将其优先分配至高层金属走线（电阻更低）。place_opt 中定义的最大 / 最小布线层约束，会同步延续到布线后优化阶段。**工具配置项**：place_opt.flow.optimize_layers

**布线驱动寄生提取（RDE, Route Driven Extraction）**先对初步布局的设计执行全局布线，生成 RDE 寄生参数表；后续所有虚拟寄生参数提取、布局优化、时钟树综合（CTS）都会复用该数据表，**大幅提升布局前后的时序相关性**。16nm 及以下工艺默认开启 RDE（auto），也可手动强制开启。**工具配置项**：opt.common.enable_rde

#### 模块 5：集成时钟门控（ICG）优化

仅当设计存在**集成时钟门控（ICG）建立时间违规**时，才需要开启 ICG 优化；本设计无此类问题，为控制运行时长，建议关闭该优化。**工具配置项**：place_opt.flow.optimize_icgs

#### 模块 6：布局执行与结果分析

确保扫描链在 place_opt 阶段被同步优化，确认可测试性（DFT）相关配置：

### 问题 6：用于扫描链优化的工具配置项是什么？其默认值是什么？

**高级合法化工具（Advanced legalizer）**12nm 及以下工艺推荐开启高级合法化工具；本实验基于 32nm 工艺，你也可手动开启测试效果：

set place.legalize.enable_advanced_legalizer true

执行布局优化可选择分步执行各阶段，或单条命令执行完整 place_opt 全流程（完整流程运行约 10 分钟）。

布局完成后检查两项核心问题：

11. 1.  布线拥塞：本设计无严重拥塞问题；在 ** 视图设置（View Settings）** 中开启「单元禁止摆放边距（Cell Keepout Margin）」可见：RAM 宏单元之间的通道已被屏蔽，这是拥塞表现良好、运行效率高、时序达标重要原因。

2. 时序：开启高级合法化工具后，设计时序基本可满足要求。

### 五、 参考答案（Answers / Solutions）

#### 问题 1

**问题**：设计中是否还存在未处理的理想网络？**解答**：执行 report_ideal_network 报告后可见：**所有工作场景下均无残留理想网络**。若存在理想网络，需使用 remove_ideal_net 命令清除。

#### 问题 2

**问题**：当前模块设置的最大布线层是哪一层？**解答**：执行 report_ignored_layers 命令查询结果：当前设计**最大信号布线层为 M8**。补充说明：本工艺库共 9 层金属，但设计限制信号布线仅使用 M1~M8；M7、M8 同时用于电源地（PG）网格布线，M9 在本设计中不使用。

#### 问题 3

**问题**：该设计包含多少条扫描链？**解答**：设计一共包含 **8 条扫描链**。可通过 report_design `-summary` 报告末尾、或专用命令 get_scan_chain_count 查看数量。

#### 问题 4

**问题**：扇出数大于 60 的非时钟类高扇出网络一共有多少条？**解答**：

1. 全局统计：扇出数 ≥60 的网络总计 10 条；

2. 剔除时钟类网络（网络名含 clk、驱动引脚含 CLK）后：**扇出数大于 60 的非时钟高扇出网络共 2 条**。脚本中使用过滤参数 `-filter net_type`==signal 可精准筛选信号类网络。

#### 问题 5

**问题**：使用哪条命令 / 配置项可将单元加入优化范围？**解答**：

```tcl
set_lib_cell_purpose -include optimization
```

#### 问题 6

**问题**：用于扫描链优化的工具配置项是什么？其默认值是什么？**解答**：配置项名称：opt.dft.optimize_scan_chain默认值：true（默认开启扫描链优化）

### 六、 实操复现补充提示

1. **命令执行规范**ICC II 脚本编辑器操作：选中单行 / 多行命令 → 点击「Run Selection」执行，**严禁全选一次性运行**，每执行一条观察终端日志与 GUI 视图变化。

2. **日志查看**重点关注报错（Error）、告警（Warning）：布局前若出现库关联、层约束、扫描链报错，需先修复再继续。

3. **视图辅助**

1. 查看单元禁止摆放边距：View Settings → Objects → 勾选 Cell Keepout Margin；

2. 查看布线拥塞：可复用实验 2 的拥塞图功能，验证布局后拥塞状态。

4. **耗时说明**place_opt 全流程运行约 10 分钟，执行期间请勿操作工具，等待命令执行完成后再生成报告。

`Run.tcl`

\# Synopsys 客户培训服务部 \# IC Compiler II：芯片实现实战实验 \# \# 布局与优化实验 运行脚本 \# \# 设计已通过 `load.tcl` 脚本完成加载 \#################################### \## 一、设计分析阶段 \#################################### \# 输出设计结果质量(QoR)汇总报告 report_qor `-summary` \# 遍历所有工作场景，检查设计中是否存在理想网络 report_ideal_network `-scenarios` \[all_scenarios\] \# 查询当前设计被屏蔽/禁用的金属布线层 report_ignored_layers \# 查看主机硬件资源、多核运行等配置信息 report_host_options \# 输出设计整体摘要信息 report_design `-summary` \# 统计设计中扫描链的总数量 get_scan_chain_count \# 检查扫描链电路是否存在异常 check_scan_chain \# 统计芯片面积、单元利用率 report_utilization \# 执行布局前置阶段的全项设计检查 check_design `-checks pre_placement_stage` \# 在消息浏览器中查看EMS类提示/告警信息： \# 操作路径：窗口(Window) -> 消息浏览器(Message Browser Window)，再选择 文件(File) -> 打开消息库(Open Message Database) -> 选中 check_design.ems \# 点击左侧"+"展开查看详细信息。 \# 本实验中可忽略 TCK 相关告警（即便 TCK-012 提示部分端口未做约束，也不影响本次实验） check_design `-checks physical_constraints` \# 告警 DCHK-104：提示 M9、MRDL 层无电源地(PG)图形，属于正常现象。 \# 本设计仅使用 M1~M8 金属层，M9 未被使用。 \# 告警 DCHK-105：工艺文件相关提示，本实验可直接忽略。 \# 分析设计中的高扇出网络 report_net_fanout `-high_fanout` \# 统计扇出数大于60的网络 report_net_fanout `-threshold 60` \# 仅统计**纯数据通路**高扇出网络，需执行以下两条命令： mark_clock_trees \# 标记所有时钟树网络 report_net_fanout `-threshold 60` \[get_flat_nets `-filter net_type`==signal\] mark_clock_trees `-clear` \# 清除时钟树标记 \# 配置工艺节点参数 \# 本实验暂不使用该命令，此处仅作语法参考。该命令建议在**设计初始化完成后、执行布局优化(place_opt)前**执行 \# 该配置全局生效，仅需设置一次 \# 示例写法： \# set_technology `-node 7` \# 启用钳位单元(Tie-cell)相关配置 \# 将钳位单元纳入优化单元范围 set_lib_cell_purpose `-include optimization` \[get_lib_cells \`$TIE_LIB_CELL_PATTERN_LIST`\] \# 取消钳位单元的「禁止修改」属性 set_dont_touch \[get_lib_cells \`$TIE_LIB_CELL_PATTERN_LIST`\] false \# 限制单个钳位单元的最大扇出数为8 set_app_options `-list {opt.tie_cell.max_fanout` 8} \## 开启漏功耗、动态功耗、总功耗优化（开启后会增加实验运行时长）： \# 前提检查：确保至少一个工作场景(Scenario)已开启功耗分析 \# report_scenarios \# 配置功耗优化模式（工具默认：不开启任何功耗优化） \# set_app_options `-name opt.power.mode` `-value leakage` \# 仅开启漏功耗优化 \# set_app_options `-name opt.power.mode` `-value dynamic` \# 仅开启动态功耗优化 \# set_app_options `-name opt.power.mode` `-value total` \# 开启漏功耗+动态功耗（总功耗）优化 \# report_app_options opt.power.effort \# \## 额外开启低阈值单元(LVt)替换优化：需要提前对低阈值单元做属性标记 \# (注：低阈值单元替换会对所有已激活的工作场景生效) \# set_threshold_voltage_group_type `-type low_vt` saed32cell_lvt \# 层优化配置：设置为 true 时，工具会为长线网、时序关键网分配最大/最小布线层约束； \# 同时对长缓冲链做**金属层提升**与重缓冲优化 \# set_app_options `-name place_opt.flow.optimize_layers` `-value true` \# 面向先进工艺，开启「布线驱动寄生提取(RDE)」 \# 该功能用于缩小**布局阶段**与**布线阶段**的时序偏差，同时减少缓冲器数量、优化芯片面积。 \# 开启RDE后，传统的层分区优化会自动失效。 \# 工具默认配置为 auto：仅在16nm及以下先进工艺自动开启RDE \# 本次实验手动强制开启该功能： set_app_options `-name opt.common.enable_rde` `-value true` \# Synopsys 物理导向(SPG)功能支持 \# 若前端综合(DC)阶段使用了SPG物理导向功能，此处需设为true；本实验不涉及该功能 \# set_app_options `-list {place_opt.flow.do_spg` true} \# 在布局优化流程中开启集成时钟门控(ICG)优化 \# set_app_options `-name place_opt.flow.optimize_icgs` `-value true` \# 若开启了上述ICG优化，时钟会被做传播处理。因此建议同步开启： \# 时钟重收敛悲观值移除(CRPR)，并根据预估偏差降低时钟不确定度（本实验不做该配置） \# set_app_options `-name time.remove_clock_reconvergence_pessimism` `-value true` \# 单元密度自动控制：工具默认开启(place.coarse.auto_density_control) \# 开启后以下两个工具参数会被自动赋值： \# - place.coarse.max_density 全局最大单元密度 \# - place.coarse.congestion_driven_max_util 拥塞驱动下的最大利用率 \# 自 2018.06 版本起，**自动时序控制**默认开启。该功能可全面提升全通路的时序驱动布局效果 report_app_options place.coarse.auto_timing_control \# 界面设置：关闭电源网格在布局视图中的显示 gui_set_setting `-window` \[gui_get_current_window `-types Layout` `-mru`\] `-setting showRoutedGround` `-value false` gui_set_setting `-window` \[gui_get_current_window `-types Layout` `-mru`\] `-setting showRoutedPower` `-value false` \# 界面设置：开启电压域(VA)在布局视图中的显示 gui_set_setting `-window` \[gui_get_current_window `-types Layout` `-mru`\] `-setting showVoltageArea` `-value true` \# 重置「扫描链优化」相关参数为工具默认值 reset_app_options opt.dft.optimize_scan_chain \# 查看扫描链优化功能的当前配置 report_app_options opt.dft.optimize_scan_chain \# 高级合法化工具：推荐用于12nm及以下先进工艺 \# 参数 place.legalize.enable_advanced_legalizer 默认值为 false（关闭） \# 其他工艺节点也可手动开启，但需先向Synopsys确认该工具已适配当前工艺节点 set_app_options `-name place.legalize.enable_advanced_legalizer` `-value true` \# 若拥有Fusion协同许可，可开启高级逻辑重构（兼顾面积与时序） \# set_app_options `-name opt.common.advanced_logic_restructuring_mode` `-value area_timing` \# 布局优化主命令 place_opt 语法说明 \# \[`-list_only`\] \# 仅罗列执行阶段，不实际运行 \# \[`-from` \<startStage>\] \# 指定起始阶段 可选：initial_place \| initial_drc \| initial_opto \| final_place \| final_opto \# \[`-to` \<endStage>\] \# 指定结束阶段 可选：initial_place \| initial_drc \| initial_opto \| final_place \| final_opto \# 执行完整布局优化全流程 place_opt \# 调用工具自定义函数 \_run_stage，分阶段执行布局优化并生成独立日志（分步调试使用） \# \_run_stage place_opt initial_place \# 初始布局 \# \_run_stage place_opt initial_drc \# 初始设计规则检查 \# \_run_stage place_opt initial_opto \# 初始优化 \# \_run_stage place_opt final_place \# 最终布局 \# \_run_stage place_opt final_opto \# 最终优化 \# 检查布局后所有单元位置是否合法（无重叠、符合位单元规则） check_legality \# 检查多电压域(MV)设计的电源、电平转换等相关规则 check_mv_design \# 保存当前模块数据 save_block \# 输出详细布局报告：包含硬宏布线重叠、窄通道区域等信息 v report_placement `-verbose high` `-hard_macro_route_over -thin_channel_area` \# 执行全局布线（仅做拥塞分析，不完成实际布线），优化等级设为最低 route_global `-effort_level minimum` `-congestion_map_only true` \# 分析全局布线拥塞图，检查通道拥塞问题 \# 如需区分「寄存器到寄存器通路」和「IO通路」做时序报告，开启以下配置： \# set_app_options `-list {time.enable_io_path_groups` true} \# 再次输出QoR汇总报告，对比布局前后差异 report_qor `-summary` \# 报告不同阈值电压单元(LVT/RVT/HVT)的使用分布 report_threshold_voltage_groups \# 输出整体时序报告 v report_timing \# 按工作场景(Scenario)分类输出时序报告 v report_timing `-report_by scenario` \# 按时序分组分类输出时序报告 v report_timing `-report_by group` \# 可通过图形界面查看最坏时序路径： \# 操作路径：窗口(Window) -> 时序分析窗口(Timing Analysis Window) \# 若时序数据未更新，点击界面上的 update（更新）按钮。 \# 选中左侧红色柱状条（时序违规路径） \# 选中底部列表中的最坏路径，右键选择 Inspect Worst Paths（查看最坏路径） \# 在路径查看器中选中数据通路，布局视图会自动定位到对应位置 \# \# 2017.09-SP4 及更早版本：布局后增量优化需使用 refine_opt 命令 \# 新版本工具搭载新一代布局优化(NPO)引擎，已无需该命令 \# \# 若此前开启了「层优化」功能，可导出脚本查看层提升结果： write_script `-force` \# 查看导出文件 `wscript/design.tcl`，确认是否存在网络被执行金属层提升操作 \# 统计整体单元利用率 report_utilization \# 单独统计 PD_RISC_CORE 电压域内的单元利用率 report_utilization `-of_objects` \[get_voltage_areas PD_RISC_CORE\] \# 如上命令默认排除列表为空，统计结果会包含宏单元； \# 因此手动创建名为 dflt 的利用率统计规则，复刻工具默认的排除逻辑 create_utilization_configuration dflt `-exclude {hard_macros` macro_keepouts soft_macros io_cells hard_blockages} \# 使用自定义规则，重新统计 PD_RISC_CORE 电压域利用率 report_utilization `-config dflt` `-of_objects` \[get_voltage_areas PD_RISC_CORE\] \# 对比多份利用率报告时，重点关注统计类型(capacity_type)，可选类型： \# core_area（核心区域）、boundary（边界区域）、site_array（位单元阵列）、site_row（位单元行）
