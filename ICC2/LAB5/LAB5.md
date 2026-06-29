# LAB5

<!-- TOC -->
## 目录

- [实验 5 设计搭建](#实验-5-设计搭建)
  - [学习目标](#学习目标)
  - [实验简介](#实验简介)
  - [相关文件与目录](#相关文件与目录)
  - [任务 1 目录结构与启动 ICC II](#任务-1-目录结构与启动-icc-ii)
  - [任务 2 全局变量与基础配置](#任务-2-全局变量与基础配置)
  - [任务 3 创建设计库、加载网表与 UPF 约束](#任务-3-创建设计库加载网表与-upf-约束)
    - [步骤 1 创建设计库](#步骤-1-创建设计库)
    - [步骤 2 加载 Verilog 网表](#步骤-2-加载-verilog-网表)
    - [步骤 3 模块链接（Link Block）](#步骤-3-模块链接link-block)
    - [两种修复方案（依次执行）](#两种修复方案依次执行)
      - [方案一：临时追加参考库（快速修复）](#方案一临时追加参考库快速修复)
      - [方案二：修改源头配置（根治问题）](#方案二修改源头配置根治问题)
    - [步骤 4 查看当前模块信息 & 加载 UPF](#步骤-4-查看当前模块信息-加载-upf)
  - [任务 4 加载布局文件与扫描链 DEF 文件](#任务-4-加载布局文件与扫描链-def-文件)
  - [任务 5 校验摆放位单元与布线层配置](#任务-5-校验摆放位单元与布线层配置)
  - [任务 6 保存模块与设计库](#任务-6-保存模块与设计库)
- [参考答案（Answers / Solutions）](#参考答案answers-solutions)
  - [任务 2 参考答案](#任务-2-参考答案)
  - [任务 3 参考答案](#任务-3-参考答案)
  - [任务 4 参考答案](#任务-4-参考答案)
  - [任务 5 参考答案](#任务-5-参考答案)
  - [任务 6 参考答案](#任务-6-参考答案)
  - [额外实操补充（复现提示）](#额外实操补充复现提示)

<!-- /TOC -->


## 实验 5 设计搭建

### 学习目标

本实验带你掌握在 **IC Compiler II（ICC II）** 中完成**设计基础搭建**的全流程。完成本实验后，你将能够：

1. 创建 NDM 格式设计库

2. 加载门级网表并应用 UPF 多电压设计约束

3. 载入布局文件与扫描链 DEF 文件

4. 校验摆放位单元、布线层的相关配置

5. 排查并修复设计搭建阶段的常见问题

### 实验简介

设计搭建是芯片后端实现**最核心、最基础**的步骤。若搭建环节出现错误，会导致后续布局、时钟树综合、布线等全流程出现各类问题，严重影响项目进度。设计搭建工作一旦完成（除非设计本身、约束文件发生变更），一般无需重复修改，后续可专注于布局、时钟树综合、布线等核心实现环节。

本实验涵盖的搭建步骤：

1. 创建 NDM 设计库

2. 加载网表、布局文件、扫描链 DEF 文件

3. 载入多电压设计文件：UPF 约束与电压域定义

4. 校验摆放位单元、布线层的配置规则

### 相关文件与目录

本实验所有文件均存放于宿主目录下的 lab56_setup 文件夹中。

lab56_setup/ ├─ `run5.tcl` \# 本实验所有执行命令汇总脚本 ├─ `setup.tcl` \# 全局变量与基础配置脚本 ├─ .synopsys_icc2.setup \# ICC II 启动自动加载配置文件 ├─ ORCA_TOP_design_data/ \# 设计数据源目录 │ ├─ `ORCA_TOP.v` \# Verilog 门级网表 │ ├─ `ORCA_TOP.upf` \# UPF 多电压设计约束文件 │ ├─ ORCA_TOP.fp/ \# ICC II 导出的布局文件目录 │ └─ `ORCA_TOP.scandef` \# 扫描链定义 DEF 文件 └─ ../ref/CLIBs \# SAED32nm 工艺单元库目录

### 任务 1 目录结构与启动 ICC II

在 Linux 终端切换至实验目录，查看目录内文件：

```shell
UNIX% cd lab56_setup
```

```shell
UNIX% ls -al
```

该目录同时供本实验与下一个**时序搭建实验**共用。

启动图形界面版 ICC II：

```shell
UNIX% icc2_shell -gui
```

和之前实验操作一致，在 ICC II 内置脚本编辑器中打开 `run5.tcl` 文件。**选中单行 / 多行命令，点击「运行所选内容」执行**，不要手动逐字输入，减少拼写错误。

额外打开一个 Linux 终端，同样切换至 lab56_setup 目录，用于查看、编辑实验所需各类文件。

### 任务 2 全局变量与基础配置

```tcl
1. 在新打开的终端中查看 setup.tcl 文件内容,并结合工具命令回答以下思考题:
```

> **问题 1**：系统变量 search_path 的默认取值是什么？提示：可使用以下任意命令查询printvar sear \[Tab\] / echo \`$search_path` / get_app_var search_path / report_app_var search_path
>
```tcl
> **问题 2**:run5.tcl 中哪些命令会调用 search_path 变量来检索文件？(仅列出命令名,无需带参数)
```
>
```tcl
> **问题 3**:run5.tcl 中哪条命令调用了自定义变量 TECH_LIB(工艺库)与 REFERENCE_LIBRARY(参考单元库)？
```
>
```tcl
> **问题 4**:从 setup.tcl 中统计,本设计一共调用了多少个以 saed32 命名的参考单元库？
```
>
```tcl
> **问题 5**:setup.tcl 中配置的多线程最大核心数为多少？
```

```tcl
1. 查阅 set_host_options 命令的帮助手册(参考手册底部「参见」板块),回答下一题:
```

> **问题 6**：工具默认启用的 CPU 核心数量是多少？

`setup.tcl` 末尾的 suppress_message 命令用于**屏蔽已知的提示信息、无害告警**。在批量运行脚本时，屏蔽冗余信息可以精简日志，便于快速定位真正需要关注的报错与异常。查看完毕后，关闭 `setup.tcl`，**不要保存文件**。

在 ICC II 脚本编辑器中，选中并执行命令加载配置脚本：

```tcl
source -echo setup.tcl
```

执行以下命令，校验已加载的配置：

printvar search_path \# 查看文件搜索路径

printvar REFERENCE_LIBRARY \# 查看参考库列表

```tcl
report_host_options \# 查看主机/线程配置
```

print_suppressed_messages \# 查看当前已屏蔽的告警列表

除了 `setup.tcl` 中手动屏蔽的信息外，工具本身也会默认屏蔽部分日志（如 CTS-725、POW-001 等），目的是减少日志冗余。若需要重新显示被屏蔽的信息，可使用 unsuppress_message 命令。

### 任务 3 创建设计库、加载网表与 UPF 约束

#### 步骤 1 创建设计库

执行创建设计库命令：

```tcl
create_lib\\use_technology_lib \$TECH_LIB \\ref_libs \$REFERENCE_LIBRARY \\
```

`ORCA_TOP.dlib`

执行后会弹出报错：

```tcl
> **错误提示(LIB-059)**:工艺库 ../ref/CLIBs/saed32_1p9m_tech.ndm 不在参考库列表中。
```

```tcl
需要修改 setup.tcl 修复该问题,修复后重新执行创库命令,直至成功。
```

> **问题 7**：本次新建的设计库名称是什么？**问题 8**：此时 lab56_setup 目录下能看到该设计库文件 / 目录吗？

#### 步骤 2 加载 Verilog 网表

执行加载网表命令：

```tcl
read_verilog -top ORCA_TOP ORCA_TOP.v
```

执行后弹出报错：

```tcl
> **错误提示(FILE-002)**:在当前搜索路径 . scripts ORCA_TOP_constraints 下,未找到文件 ORCA_TOP.v。
```

根据前文给出的目录结构修正配置，重新执行命令，直至网表加载成功。

#### 步骤 3 模块链接（Link Block）

图形界面模式下工具会**自动完成模块链接**，可跳过手动命令 link_block。链接完成后出现多条告警：

> **告警（LNK-005）**：无法解析单元 SRAMLP2RW32x4（首次在模块 PCI_TOP 中调用）**告警（LNK-005）**：无法解析单元 SRAMLP2RW64x8（首次在模块 CONTEXT_MEM 中调用）

**问题原因**：设计库关联的参考库列表中，缺失网表实例化用到的宏单元库（本设计调用了 4 款不同的 SRAM 存储器宏单元）。

> **问题 9**：存放上述未解析宏单元的参考库名称与文件路径是什么？

#### 两种修复方案（依次执行）

##### 方案一：临时追加参考库（快速修复）

无需修改配置文件，直接为当前设计库补充缺失的参考库，再重新链接模块：

```tcl
set_ref_libs -add ../ref/CLIBs/saed32_sram_lp.ndm
```

link_block `-force`

执行后链接告警消失。

##### 方案二：修改源头配置（根治问题）

方案一仅临时生效，若后续重新搭建设计会再次出现报错。因此需要修改核心配置文件：

1. 关闭当前设计库：close_lib

```tcl
2. 编辑 setup.tcl,将缺失的 SRAM 库追加到 REFERENCE_LIBRARY 列表中
```

3. 重新加载配置、创建设计库、加载网表、执行模块链接

#### 步骤 4 查看当前模块信息 & 加载 UPF

1. 执行以下命令查看模块句柄（名称）：

```tcl
get_blocks ORCA_TOP
```

```tcl
get_blocks ORCA_TOP.design
```

```tcl
get_blocks ORCA_TOP.dlib:ORCA_TOP
```

```tcl
get_blocks ORCA_TOP.dlib:ORCA_TOP.design
```

补充说明：

4. 1.  模块默认句柄格式：库名:模块名.视图名，默认视图名为 design

2. 若模块额外标注别名（label），句柄格式为：库名:模块名/别名.视图名

3. 若一个库内仅有一个同名模块，可简写模块名调用。

> **问题 10**：当前设计库中新建模块的完整句柄是什么？

```tcl
简要查看 ORCA_TOP_design_data/ORCA_TOP.upf 文件内容(查看后关闭,**不保存**):该 UPF 文件定义内容:
```

2. 1.  3 路电源端口 / 网络：VSS、VDD、VDDH

2. 2 个电压域：顶层电压域 PD_ORCA_TOP、内核电压域 PD_RISC_CORE

3. 内核电压域输入 / 输出对应的**电平转换单元**

4. 电源网络的工作状态（本设计仅定义上电状态）

加载并生效 UPF 多电压约束：

```tcl
load_upf ORCA_TOP.upf
```

```tcl
commit_upf
```

### 任务 4 加载布局文件与扫描链 DEF 文件

载入 ICC II 导出的布局文件：

```tcl
source ORCA_TOP.fp/floorplan.tcl
```

此时布局视图会显示完整的**倒 L 型**模块轮廓。

视图优化（电源网格遮挡视线，关闭相关显示）：在「视图设置（View Settings）」面板中做如下调整：

7. 1.  布线（Route）→ 网络类型（Net Type）→ 电源 / 地（Power and Ground）：取消可见性

2. 物理端口（Terminal）：开启可选中属性

3. 逻辑端口（Port）：取消可见性

4. 电压域（Voltage Area）：开启可见性

切换至「设置（Settings）→ 视图（View）」，开启「标签缩放（Scale fonts）」，提升文字可读性。

调整后视图可清晰看到：

10. 1.  所有硬宏单元已完成摆放

2. 芯片边界分布着 IO 端口、电源地对应的物理引脚

3. 整个核心区域默认归属电压域 DEFAULT_VA（紫色虚线框）

4. 右下角单独划分出电压域 PD_RISC_CORE

5. 标准单元仍堆叠在布局左下角

加载扫描链定义 DEF 文件：

```tcl
read_def ORCA_TOP.scandef
```

### 问题 11**：本设计一共包含多少条扫描链？

将物理引脚与电源网络做关联，并检查多电压设计规则：

```tcl
connect_pg_net \# 绑定电源地引脚与电源网络
```

```tcl
check_mv_design \# 检查多电压设计合规性
```

### 任务 5 校验摆放位单元与布线层配置

校验默认摆放位单元：确认 unit 位单元是否为全局默认位单元

```tcl
get_attribute \[get_site_defs unit\] is_default
```

校验标准单元的对称性规则：

```tcl
get_attribute \[get_site_defs unit\] symmetry
```

补充说明：该对称性规则已在工艺库阶段提前配置，无需手动修改。若工艺库未配置，需执行命令：set_attribute \[get_site_defs unit\] symmetry {Y}

> **问题 12**：**Y 轴对称**代表标准单元可以沿哪个轴翻转？（沿 X 轴上下翻转 / 沿 Y 轴左右翻转）

查看各金属层默认布线方向：

```tcl
get_attribute \[get_layers M?\] routing_direction
```

该规则同样在工艺库中预先定义。

校验布线层限制，并修改最大布线层：

```tcl
report_ignored_layers \# 查看当前被禁用的布线层
```

```tcl
set_ignored_layers -max_routing_layer M6 \# 限制信号布线最大层为M6
```

```tcl
report_ignored_layers \# 再次校验
```

### 任务 6 保存模块与设计库

在 Linux 终端查看当前目录文件：

```shell
ls -l
```

### 问题 13**：此时 lab56_setup 目录下能看到 ORCA_TOP.dlib 设计库吗？

为模块添加别名并保存（标记当前为「初始化完成版」设计）：

```tcl
rename_block -to_block ORCA_TOP/init_design \# 给模块添加别名 init_design
```

```tcl
save_block \# 保存当前模块 或使用 save_lib 保存整个设计库
```

### 问题 14**：执行保存后，目录中是否出现 ORCA_TOP.dlib 设计库？

退出 ICC II：菜单选择 File → Exit，弹窗确认后退出工具。

> 至此，**实验 5 设计搭建** 全部完成。

## 参考答案（Answers / Solutions）

### 任务 2 参考答案

#### 问题 1

search_path 默认值：**当前工作目录（.）**

#### 问题 2

```tcl
会调用 search_path 检索文件的命令:source、read_verilog、load_upf、read_def
```

```tcl
> 补充:create_lib 命令直接使用绝对路径调用库文件,不依赖搜索路径。
```

#### 问题 3

```tcl
调用 TECH_LIB 与 REFERENCE_LIBRARY 变量的命令:create_lib
```

#### 问题 4

`setup.tcl` 中一共引入 **3 个** saed32 系列参考库：高阈值库 (HVth)、低阈值库 (LVth)、常规阈值库 (RVth)（标准单元 + 电平转换单元库）。

#### 问题 5

脚本配置：多线程最大核心数**不超过 8 核**（会自动取用设备空闲核心，但上限为 8）。

#### 问题 6

```tcl
工具默认启用的 CPU 核心数:**1 核**(查看 report_host_options 结果中 max_cores 字段)。
```

### 任务 3 参考答案

#### 问题 7

```tcl
新建设计库名称:**ORCA_TOP.dlib**(也可执行 current_lib 命令查看)
```

#### 问题 8

**不能**。设计库文件 / 目录在**执行保存操作前**，不会出现在 Linux 目录中。

#### 问题 9

```tcl
缺失的宏单元参考库:路径 & 名称:../ref/CLIBs/saed32_sram_lp.ndm
```

#### 问题 10

```tcl
当前模块完整句柄:ORCA_TOP.dlib:ORCA_TOP.design(执行 current_block 可直接查询)
```

### 任务 4 参考答案

#### 问题 11

```tcl
扫描链总数:**8 条**查询依据:read_def 日志中 SCANCHAINS:8/8;也可使用命令 get_scan_chain_count 统计。
```

### 任务 5 参考答案

#### 问题 12

Y 轴对称（symmetry {Y}）：标准单元可**沿 Y 轴左右翻转（X 方向镜像）**。

### 任务 6 参考答案

#### 问题 13

**不能**。尚未执行保存，设计库不会落地为本地文件。

#### 问题 14

```tcl
**可以**。执行 save_block/save_lib 后,ORCA_TOP.dlib 设计库目录正常生成。
```

### 额外实操补充（复现提示）

1. **文件路径易错点**网表、UPF 文件都存放在 ORCA_TOP_design_data 目录，必须在 search_path 中追加该路径，否则会报文件找不到错误。

2. **参考库补全规则**只要网表中实例化了新的宏单元 / IP，就必须把对应库加入 REFERENCE_LIBRARY，否则会出现模块链接失败。

3. **视图切换技巧**布局视图中：蓝色矩形为硬宏单元、左下角紫色小块为标准单元、芯片边缘浅蓝色图标为 IO 物理端口。

```tcl
4. **多电压设计检查**check_mv_design 是多电压设计必跑命令,用于排查电平转换缺失、电源短路、电压域交叉等致命问题。
```
