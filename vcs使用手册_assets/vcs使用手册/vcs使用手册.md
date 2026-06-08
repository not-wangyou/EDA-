# VCS 编译仿真实验记录

> 本文档基于VCS lab1的parta编写
示例文件https://pan.quark.cn/s/e4a5cbea7ed8

VCS是编译型verilog仿真器，处理verilog的源码过程如下：

![图 1（1408x296）](vcs使用手册_assets/image-01.png)

本次实验使用VCS labs里面lab1的verilog源码

![图 2（1011x947）](vcs使用手册_assets/image-06.png)

此电路为一位加法器 fa.v 组成4位加法器 add4.v，再组成一个8位加法器，使用资源换性能的思路，减小了行波进位加法器的进位延迟。顶层文件为add8.v，testbench为addertb.v。

![图 3（687x123）](vcs使用手册_assets/image-07.png)

需要注意的是，使用手册源码给出的编译指令是

```bash
vcs addertb.v fa.v add4.v add8.v
```

实际运行会报错

```bash
/usr/include/gnu/stubs.h:7:27: fatal error: gnu/stubs-32.h: No such file or directory
```

这是因为gnu/stubs-32.h 是 32 位 glibc 开发包 中的头文件。VCS 在编译过程中试图生成 32 位目标代码（可能是默认行为或由于某些选项），但你的 CentOS 系统缺少对应的 32 位兼容库和头文件。

CentOS 默认倾向于 64 位环境，但 VCS 旧版本或某些配置下会默认以 -m32 模式编译，从而触发该错误。

![图 4（962x414）](vcs使用手册_assets/image-08.png)

在 VCS 编译命令中增加 -full64 选项，强制生成 64 位可执行文件，避免依赖 32 位库。

```bash
vcs -full64 addertb.v fa.v add4.v add8.v
```

![图 5（968x406）](vcs使用手册_assets/image-09.png)

可以看到报错已经消失，此时目录有以下文件

![图 6（689x56）](vcs使用手册_assets/image-10.png)

| 文件 / 目录 | 类型 | 说明 |
| --- | --- | --- |
| simv | 可执行文件 | VCS 生成的仿真二进制可执行文件。运行 ./simv 即可启动仿真，执行测试平台并输出结果。这是仿真流程的核心产物。 simv 是最重要的输出，它可以直接执行。如果在编译加了 -R ，VCS 会在编译完成后自动运行它，否则你需要手动 ./simv。 |
| simv.daidir | 目录 | VCS 的中间设计数据库目录。存放编译过程中生成的抽象语法树（AST）、符号表、设计层次信息等。仿真器（simv）运行时需要读取该目录中的信息。 simv.daidir 在调试或使用 VCS 的 DVE/Verdi 等功能时需要保留；若仅进行批处理仿真，也可以删除，但重新运行 simv 时会自动重建。 |
| csrc | 目录 | 临时 C 源代码及编译中间文件目录。VCS 会将 Verilog 设计翻译为 C/C++ 代码，再调用系统编译器（如 gcc）生成 simv。csrc 中包含了这些中间 .c、.o、共享库等文件，通常在编译后可以删除，不影响后续仿真。 csrc 是编译过程中的临时文件，可以安全删除。如果你执行 make clean 或手动删除 csrc，下次重新编译 VCS 时会重新生成。 |

在此时编译已经完成了，接下来开始仿真

```bash
./simv
```

仿真结束输出：

```bash
$finish at simulation time 13107200
V C S  S i m u l a t i o n  R e p o r t
...
*** Testbench Successfully completed! ***
```

表示加法器功能正确

至此vcs我们就完成了vcs的一次编译仿真

## DVE 查看波形

DVE（Discovery Visual Environment） 是 Synopsys VCS 自带的图形化调试工具，用于帮助工程师直观地分析仿真行为、定位设计缺陷。它通过波形图、源代码高亮、信号追踪等方式，将仿真数据可视化，大幅提升调试效率。

在刚才的编译仿真过程中，我们没有开启调试支持。VCS 默认生成的 simv 只支持批处理模式（无波形、无 GUI）。当你想用 -gui 启动 DVE 图形界面时，需要编译时就加入相应的调试选项，使 simv 包含调试接口和波形记录能力。

在 VCS 命令中加入 -debug_access+all

```bash
vcs -full64 -debug_access+all addertb.v fa.v add4.v add8.v
```

编译完成后，执行

```bash
./simv -gui
```

![图 7（2540x1338）](vcs使用手册_assets/image-11.png)

此时dve的图形化界面就显示出来了

### 如果想在编译后立即打开 GUI

```bash
vcs -full64 -debug_all addertb.v fa.v add4.v add8.v -R -gui
```

按住shift选中所有信号，右键选中所有信号 --> 右键Add to Waves --> New Wave View

![图 8（848x522）](vcs使用手册_assets/image-12.png)

此时dve界面也没有任何波形跳变，因为DVE 启动时，VCS 仿真进程可能处于 暂停状态

解决方法：
在 DVE 菜单栏点击 Simulator → Run（或直接按快捷键 F5），也可以在dve命令行输入run让仿真从时间 0 开始执行。

![图 9（2483x1369）](vcs使用手册_assets/image-13.png)

运行后，你应该能看到波形变化，并且控制台会出现 $finish 信息和 *** Testbench Successfully completed! *** 消息。

![图 10（2559x287）](vcs使用手册_assets/image-02.png)

![图 11（2469x1366）](vcs使用手册_assets/image-03.png)

补充：在波形处点标记1处的小箭头，也能使仿真开始，可以使用2处的三个按钮(预览全局、放大和缩小)调整波形

![图 12（2484x1359）](vcs使用手册_assets/image-04.png)

以上已完成VCS基本的使用，下面我们考虑一个更复杂的情形。假设现在顶层模块变得十分复杂，里面包含很多的 .v 文件，那样将所有文件敲在终端上便很麻烦。编译选项-f可以解决这个问题。

创建一个 verilog_file.f 文件，内容如下：

```bash
addertb.v
fa.v
add4.v
add8.v
```

可以在服务器终端上使用cat指令

```bash
cat > verilog_file.f << EOF
addertb.v
fa.v
add4.v
add8.v
EOF
```

验证文件内容

```bash
cat verilog_file.f
```

![图 13（657x124）](vcs使用手册_assets/image-05.png)

后续编译使用

```bash
vcs -full64 -debug_all -f verilog_file.f -R
```

# VCS 与 DVE 通用指令参考手册

## 1. VCS 编译命令通用格式

```bash
vcs [编译选项] [源文件列表] [运行时选项]
```

### 常用编译选项

| 选项 | 说明 |
| --- | --- |
| -sverilog | 启用 SystemVerilog 支持（UVM 必须） |
| -debug_all 或 -debug_access+all | 开启全部调试功能（波形、断点、单步） |
| -debug_access+r | 最小调试支持（仅波形记录） |
| -full64 | 使用 64 位编译模式（推荐） |
| -lca | 启用实验性功能（需 license） |
| -ntb_opts uvm-1.2 | 自动链接 UVM 1.2 库 |
| -R | 编译后立即执行仿真 |
| -gui | 启动 DVE 图形界面（需配合 -debug_all） |
| -o <filename> | 指定输出的可执行文件名（默认 simv） |
| -f <filelist> | 从文件中读取源文件列表和编译选项 |
| -y <dir> | 指定库目录 |
| +libext+.v | 指定库目录中源文件的扩展名 |
| +incdir+<dir> | 指定 include 文件搜索路径 |
| +define+<macro> | 定义预处理宏 |
| -l <logfile> | 将编译日志输出到文件 |
| -Mupdate | 增量编译（仅重编修改过的文件） |
| -cm line+cond+fsm+tgl | 开启覆盖率收集 |

### 源文件列表方式

直接列出：vcs top.sv sub.sv

使用文件列表：vcs -f filelist.f

组合使用：vcs -f rtl.f -f tb.f -debug_all

## 2. DVE 启动与使用通用指令

### 启动方式

| 命令 | 说明 |
| --- | --- |
| dve & | 启动 DVE 主界面（无波形加载） |
| dve -vpd <waveform.vpd> & | 打开已有的 VPD 波形文件 |
| dve -covdir <cov_dir.vdb> & | 打开覆盖率数据库 |
| 从仿真中启动 | ./simv -gui 或 vcs ... -R -gui |

### DVE 常用操作

| 操作 | 快捷键 / 菜单 |
| --- | --- |
| 运行仿真 | F5 或 Simulator > Run |
| 暂停仿真 | F6 或 Simulator > Break |
| 复位仿真 | Simulator > Reset |
| 添加信号到波形 | 选中信号 → Add to Waves |
| 设置断点 | 在源代码行号左侧双击 |
| 单步执行 | F10（Step）、F11（Step Into） |
| 查看变量值 | 鼠标悬停或 Select > Show Value |
| 保存波形配置 | File > Save Signal Set |
| 退出 DVE | File > Exit |

## 4. 常见错误与解决方法

| 错误信息 | 原因 | 解决方法 |
| --- | --- | --- |
| DVE/VERDI debugging is not enabled | 编译时未加 -debug_all | 重新编译加上 -debug_all 或 -debug_access+all |
| gnu/stubs-32.h: No such file or directory | 缺少 32 位库且未使用 -full64 | 添加 -full64 或安装 32 位 glibc 开发包 |
| undefined reference to `uvm_*` | UVM 库未链接 | 添加 `-ntb_opts uvm-1.2` 或手动指定 UVM 路径 |
| Error-[MPD] Module previously defined | 模块重复定义 | 检查文件列表，确保没有重复包含同一文件 |
| Cannot find include file | include 路径未指定 | 使用 +incdir+<path> |
