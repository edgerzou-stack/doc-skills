# FM 工程总说明（收敛版）

这份文档是本工程**唯一保留的总说明文档**。

本次已经把原先分散的工程说明 md 收敛到这里，后续新增说明、命令、debug 步骤也优先追加到这一个文件里。

这份文档覆盖了原先分散的几份说明：

- `README_user_match_zh.md`
- `FM_CONFIG_GUIDE_zh.md`
- `FM_FLOW_RUN_ORDER_zh.md`
- `VCS_VPI_RESOLVE_RUNBOOK_zh.md`
- `README_debug_ppe_top_zh.md`

目标是只保留一份“够用、可维护、能 debug”的总入口。

## 1. 这套工程现在到底在做什么

主流程分三段：

1. `make resolve`
   - 用 `resolve_module_tops.py` 找每个目标 module 对应的 `context top`
   - 当前默认走 `VCS+VPI typed hierarchy`
2. `make generate`
   - 默认按 **instance 粒度** 生成 `*_fm.tcl`
   - `resolve` 阶段会先产出 `instance_target_map.tsv` 和 `resolved_targets.list`
   - 再由 `auto_resolve_and_gen_fm.py` 把每个实例任务对应的 `topModule_ref/context_top_ref/context_top_imp/target_ref_inst/target_imp_inst` 回填进去
3. `make submit` / `make detect`
   - 提交 FM 批量任务
   - 收集 pass/fail 结果

现在日常使用建议：

```bash
make all
```

或拆开执行：

```bash
make resolve
make generate
make submit
make detect
```

复用已有 graph-cache 时：

```bash
make daily
```

失败模块重跑：

```bash
make rerun
```

当前默认：

```text
TARGET_GRANULARITY=instance
```

也就是默认不再是“每个 module 一份任务”，而是“每个 elaborated instance 一份任务”。

## 2. 现在建议保留的核心脚本

### 2.1 主流程必须

- `Makefile`
- `resolve_module_tops.py`
- `dump_vcs_instance_types_vpi.c`
- `auto_resolve_and_gen_fm.py`
- `gen_module_fm.py`
- `fm_template_instance_aware.tcl`
- `user_match.tcl`
- `gen_bsub_submit.py`
- `detect_fm_failures.py`

### 2.2 输入/模板相关

- `ref.tcl`
- `imp.tcl`
- `user_match.tcl`
- `rtl_graph.f`
- `rtl_fm.f`
- `top_candidates.list`
- `../exclude.list`

### 2.3 可选 debug / probe 脚本

这些不是日常主流程必须，建议视为“工具箱”而不是“主链路”：

- `probe_verdi_instance_type.tcl`
- `probe_verdi_typed_hierarchy_more.tcl`
- `probe_verdi_hier_apis.tcl`
- `probe_verdi_npisrcfind.tcl`
- `dump_verdi_cmds.tcl`

这些 probe 的使用说明也已经并入当前这份总文档，不再单独保留额外 md。

也就是说，**以后你维护时主要盯前两组文件就够了**。

## 3. 为什么现在 parameter 能被 Formality 正确识别

这是本次改造里最关键的一点。

以前容易出问题的原因是：  
如果直接把某个 module 当成独立顶层去 `read_sverilog` / `set_top`，那么它的 parameter 值来自“裸模块 elaboration”，不一定等于它在真实 SoC top 下被例化时的参数值。

现在的模板逻辑变成了“**先读真实 top，再选目标 instance 做 compare**”。

关键文件是：

[fm_template_instance_aware.tcl](/Users/zouzhengting/Workplace/fm_template_instance_aware.tcl)

核心步骤如下。

### 3.1 先读完整设计，并把真实 top 设为上下文

模板里先读：

```tcl
read_sverilog -container r -libname WORK -17 -vcs "+define+SYNTHESIS -f rtl_ref.f"
set_top r:/WORK/${context_top_ref}

read_sverilog -container i -libname WORK -17 -vcs "+define+SYNTHESIS -f rtl_imp.f"
set_top i:/WORK/${context_top_imp}
```

这里的 `context_top_ref/context_top_imp` 不是目标 module，而是 resolver 算出来的真实 top。

### 3.2 在真实 top 下找目标 module 对应的 instance

模板里通过 `_fm_find_first_inst_by_module` 在：

```tcl
r:/WORK/${context_top_ref}
i:/WORK/${context_top_imp}
```

下面找 `ref_name == $topModule_ref` / `ref_name == $topModule_imp` 的实例。

如果你手动指定了：

```tcl
set target_ref_inst ...
set target_imp_inst ...
```

则直接用你指定的实例路径；否则默认找“第一个匹配实例”。

### 3.3 compare 的对象不再是裸 module，而是 elaborated instance

真正起作用的是这两句：

```tcl
set_reference $ref_target_obj
set_implementation $imp_target_obj
```

也就是说，Formality 现在比较的是：

- `KN_top/.../u_xxx`
- `KN_top/.../u_xxx`

这样的真实实例，而不是“孤立的 module 定义”。

### 3.4 这意味着 parameter 从哪里来

parameter 的值来自：

```text
真实 top + 真实例化路径 + 真实 elaboration
```

所以：

- `parameter override`
- `defparam`
- generate 展开
- 顶层条件编译后影响到的例化结构

都会在 `read_sverilog + set_top + set_reference/set_implementation(instance)` 这条链路里被带进去。

### 3.5 现在这个边界怎么处理

之前的边界是：

- 同一个 module 在同一个 top 下被例化多次
- 并且这些实例 parameter 不同
- 旧 flow 只会自动选第一个实例

现在默认主流程已经切到 **per-instance**，所以这类场景不再需要手工挑实例。

也就是说，`resolve` 会自动把：

```text
module_type + context_top + instance_path
```

展开成多个独立任务，再分别生成 Tcl。

只有在你显式把 `TARGET_GRANULARITY=module` 切回旧模式时，才会重新回到“默认找第一个实例”的行为。

## 4. `make resolve` 现在是怎么工作的

主脚本：

[resolve_module_tops.py](/Users/zouzhengting/Workplace/resolve_module_tops.py)

当前默认引擎：

```text
VCS compile/elab
-> 加载 dump_vcs_instance_types_vpi.c 生成的 VPI so
-> simv 输出 instance_path<TAB>module_type
-> Python 直接用 typed hierarchy 建图
```

这样避免了老问题：

- 靠静态 RTL parser 猜 instance type
- generate / macro / parameter override / bind / interface 导致的 parent-child 缺边

### 4.1 为什么一定要带 `-debug_access+all`

VPI 需要看完整 elaborated hierarchy。  
如果没有：

```bash
-debug_access+all
```

经常会出现：

- typed hierarchy 不完整
- 某些实例看不到
- 最终 graph 少边，导致 resolver 误判 `not_found` / `no_candidate_top`

现在脚本已经两层兜底：

1. `Makefile` 默认带 `-debug_access+all`
2. `resolve_module_tops.py` 如果发现你没传任何 `-debug_access...`，也会自动补

### 4.2 第一次 `make resolve` 为什么慢，以及怎么提速

第一次慢的主要原因不是 Python 查 graph，而是 VCS+VPI 建 hierarchy：

```text
每个 top candidate
-> 单独 VCS compile/elab
-> 运行 simv
-> VPI dump instance_path -> module_type
```

如果 `top_candidates.list` 里有多个 top，旧脚本是串行逐个 top 跑 VCS。现在新增了：

```make
TOP_WORKERS ?= 1
```

你可以小步试：

```bash
make resolve TOP_WORKERS=2
```

机器内存和 VCS license 允许的话，再试：

```bash
make resolve TOP_WORKERS=4
```

不建议一开始开很大，因为每个 worker 都是一份 VCS compile/elab，容易同时吃掉大量内存和 license。

经验建议：

- `TOP_WORKERS=1`：最稳，默认值
- `TOP_WORKERS=2`：优先尝试，通常风险较低
- `TOP_WORKERS=4`：机器内存和 license 明确够时再试
- `TOP_WORKERS>4`：除非确认环境承受得住，否则不建议

还有两个不改脚本的提速办法：

- 减少 `top_candidates.list` 里的 top 数量；每少一个 top，就少一次 VCS compile/elab
- smoke 时临时只放 1 个 top candidate，确认 flow 后再放大全量 top

注意：`RESOLVE_WORKERS` 和 `TOP_WORKERS` 不是一回事。

```make
RESOLVE_WORKERS ?= 32
TOP_WORKERS ?= 1
```

- `RESOLVE_WORKERS`：Python 里 module -> top 查询并行，比较轻
- `TOP_WORKERS`：VCS compile/elab 并行，很重

所以不要把 `RESOLVE_WORKERS=32` 当成 `TOP_WORKERS=32` 来用。

## 5. `make generate` 现在怎么工作

现在默认是 **instance 粒度**，核心中间产物变成：

- `scripts/instance_target_map.tsv`
- `scripts/resolved_targets.list`
- `scripts/resolved_modules.list`

其中：

- `instance_target_map.tsv`：每个实例任务的明细
- `resolved_targets.list`：真正用于生成/提交/重跑的任务列表
- `resolved_modules.list`：只是辅助统计，表示涉及到哪些 module type

### 5.1 `generate` 默认不再吃派生 target list

现在默认：

```make
GENERATE_MODULE_LIST ?=
```

也就是说，普通：

```bash
make generate
```

不会再把 `scripts/resolved_targets.list` 作为过滤名单传回脚本，而是直接以：

```text
scripts/instance_target_map.tsv
```

为准生成全部当前实例任务，并顺手重写：

```text
scripts/resolved_targets.list
```

这样可以避免旧版本或上一次 run 留下来的 stale `resolved_targets.list` 把 generate 卡住。

注意：`module_top_map.tsv` 里仍然可能有 module-level unresolved；在 instance 模式的普通 `make generate` 中，这些 unresolved module 不会阻止已生成的 instance target 继续生成 Tcl。

### 5.2 `generate-only` 现在真正按传入 target list 过滤

`auto_resolve_and_gen_fm.py --flow-stage generate-only` 现在会：

1. 在 instance 模式下读 `instance_target_map.tsv`
2. 再按 `--module-list` 传入的 target list 过滤
3. unresolved 检查也只针对这批 target

所以如果你显式传入失败任务列表或小规模试跑列表，unresolved 检查只针对这批 target。普通 `make generate` 不再显式传 target list。

## 6. 如果以后目录变了，优先改哪里

主入口是：

[Makefile](/Users/zouzhengting/Workplace/Makefile)

最常改的是这些变量。

### 6.1 顶层输入

```make
GRAPH_FILELIST ?= ./rtl_graph.f
FM_FILELIST ?= ./rtl_fm.f
REF_TCL ?= ./ref.tcl
IMP_TCL ?= ./imp.tcl
MODULE_LIST ?= ../../module.list
TOP_CANDIDATES_LIST ?= ./top_candidates.list
EXCLUDE_LIST ?= ../exclude.list
```

### 6.2 模板和字典 Tcl 路径

```make
TEMPLATE ?= ./fm_template_instance_aware.tcl
REF_MODULES_TCL ?= ./ref.tcl
IMP_MODULES_TCL ?= ./imp.tcl
USER_MATCH_TCL ?= ./user_match.tcl
```

### 6.3 生成和 resolve 的默认行为

```make
RESOLVE_ENGINE ?= vcs-vpi
TARGET_GRANULARITY ?= instance
RESOLVED_TARGET_LIST ?= $(OUTPUT_DIR)/resolved_targets.list
GENERATE_MODULE_LIST ?=
ON_UNRESOLVED ?= error
```

经验规则：

- 文件位置变了，优先改 Makefile 变量
- 只有模板行为本身变了，才去改 `fm_template_instance_aware.tcl`
- 只有解析逻辑本身变了，才去改 Python

## 7. 脚本用法速查

日常优先用 `Makefile`，不要手敲 Python 参数。只有 debug 或单独验证某一层时，才直接跑脚本。

### 7.1 Makefile 主入口

首次或 RTL / top candidate / VCS 参数变化后：

```bash
make all
```

等价于：

```bash
make resolve
make generate
make submit
make detect
```

只复用已有 resolve 结果继续生成和提交：

```bash
make daily
```

失败任务重跑：

```bash
make rerun
```

只重建 hierarchy / instance target：

```bash
make resolve REFRESH_GRAPH=1
```

debug 某个 module 为什么找不到：

```bash
make resolve REFRESH_GRAPH=1 DEBUG_MODULE=ppe_top
```

临时改常见路径：

```bash
make generate \
  TEMPLATE=./fm_template_instance_aware.tcl \
  FM_FILELIST=./rtl_fm.f \
  REF_MODULES_TCL=./ref.tcl \
  IMP_MODULES_TCL=./imp.tcl \
  USER_MATCH_TCL=./user_match.tcl
```

查看 make 实际会执行什么：

```bash
make -n resolve
make -n generate
make -n submit
make -n rerun
```

### 7.2 `resolve_module_tops.py`

用途：

- 用 VCS+VPI elaboration 得到 `instance_path -> module_type`
- 建 module parent graph
- 默认同时输出 instance 粒度任务

一般不直接跑，`make resolve` 会调用它。

手工 smoke 示例：

```bash
python3 resolve_module_tops.py \
  --filelist ./rtl_graph.f \
  --module-list ../../module.list \
  --top-candidates-list ./top_candidates.list \
  --resolve-engine vcs-vpi \
  --vcs-bin vcs \
  --vcs-arg=+define+SYNTHESIS \
  --vcs-arg=-timescale=1ns/1ps \
  --vcs-arg=+delay_mode_zero \
  --vcs-arg=+nospecify \
  --vcs-arg=-debug_access+all \
  --refresh-graph \
  --output scripts/module_top_map.tsv \
  --dump-context-top-map scripts/module_context_top.map \
  --dump-unresolved scripts/unresolved_modules.list \
  --dump-instance-targets scripts/instance_target_map.tsv \
  --dump-target-list scripts/resolved_targets.list
```

关键输出：

```text
scripts/module_top_map.tsv
scripts/module_context_top.map
scripts/unresolved_modules.list
scripts/instance_target_map.tsv
scripts/resolved_targets.list
```

### 7.3 `auto_resolve_and_gen_fm.py`

用途：

- 串联 resolve 和 generate
- `resolve-only`：只生成 map/list
- `generate-only`：复用已有 map/list 生成 Tcl
- `all`：resolve + generate 一次完成

`make resolve` 调的是：

```bash
python3 auto_resolve_and_gen_fm.py \
  --flow-stage resolve-only \
  --target-granularity instance \
  --filelist ./rtl_graph.f \
  --module-list ../../module.list \
  --top-candidates-list ./top_candidates.list \
  --template ./fm_template_instance_aware.tcl \
  --output scripts/
```

`make generate` 默认调的是：

```bash
python3 auto_resolve_and_gen_fm.py \
  --flow-stage generate-only \
  --target-granularity instance \
  --instance-pick-mode one-per-module \
  --instance-random-seed 12345 \
  --template ./fm_template_instance_aware.tcl \
  --ref-modules-tcl ./ref.tcl \
  --imp-modules-tcl ./imp.tcl \
  --user-match-tcl ./user_match.tcl \
  --fm-filelist ./rtl_fm.f \
  --output scripts/ \
  --dedup \
  --on-unresolved error \
  --save-session never \
  --workers 0
```

如果你只想小规模试一批 target，可以先准备一个 list：

```text
ppe_top__KN_top__u0__995ee43a
ppe_top__KN_top__u1__216b6001
```

然后跑：

```bash
make generate GENERATE_MODULE_LIST=./my_targets.list
```

注意：instance 模式下，这个变量名历史上叫 `GENERATE_MODULE_LIST`，但内容实际是 target task list。

### 7.3.1 默认 target 策略：每个 module 随机选一个 instance

现在默认仍然是 instance-aware flow，但不会再把同一个 module 的所有 instance 都拿去比。默认策略是：

```make
TARGET_GRANULARITY ?= instance
INSTANCE_PICK_MODE ?= one-per-module
INSTANCE_RANDOM_SEED ?= 12345
INSTANCE_TARGET_ORDER ?= hierarchy
```

含义：

- `scripts/instance_target_map.tsv` 仍然保留全量 instance target
- `scripts/resolved_targets.list` 只保留每个 module 随机选中的一个 instance target
- 随机选择由 `INSTANCE_RANDOM_SEED` 固定，所以同一个输入下结果可复现
- `resolved_targets.list` 默认按真实 instance 层级深度排序，越靠近 top 的 target 越靠前
- `make submit` 默认只提交 `scripts/resolved_targets.list`

这能同时保留真实 instance parameter，又避免几十万 instance 全部比对。

如果想换一批随机样本：

```bash
make resolve INSTANCE_RANDOM_SEED=20260424
make generate INSTANCE_RANDOM_SEED=20260424
```

如果确实想回到“所有 instance 都比”的旧行为：

```bash
make resolve INSTANCE_PICK_MODE=all
make generate INSTANCE_PICK_MODE=all
```

如果想保留原始顺序，不按层级排序：

```bash
make generate INSTANCE_TARGET_ORDER=original
```

`make rerun` 传入的是 `failed_module.list` 里的显式 target id，所以不会再二次随机抽样，失败哪个就重跑哪个。

`make generate` 提速建议：

```bash
make generate GEN_WORKERS=32
make generate GEN_WORKERS=64
```

`GEN_WORKERS=0` 是自动模式，脚本会按机器 CPU 自动取一个并行度。现在 `make generate` 有两段都会并行：

- 第一段：`gen_module_fm.py` 从模板批量写出 `*_fm.tcl`
- 第二段：`auto_resolve_and_gen_fm.py` 把每个 Tcl patch 成正确的 `context_top_ref/context_top_imp/target_ref_inst/target_imp_inst`

之前只有第一段并行，第二段会串行读写所有生成 Tcl。instance target 接近百万级时，慢点主要在第二段 I/O。现在第二段也会复用 `GEN_WORKERS` 并行 patch，日志里会看到：

```text
patch_workers=32
```

如果目录在 NFS 或共享盘上，`GEN_WORKERS` 不是越大越好。建议先试 `32`，再试 `64`，如果发现负载高但速度不升，退回 `16/32`。

生成文件名现在默认会压短。旧规则会把完整 instance task id 展开成文件名，例如：

```text
<very_long_target_id>_fm.tcl
```

新规则是：

```text
<module_prefix>__<16hex_hash>_fm.tcl
```

生成后的 Tcl 内部仍然保留完整 target id：

```tcl
set module "<full_target_id>"
set module_file_id "<module_prefix>__<16hex_hash>"
```

`submit/detect` 也会使用同一套短名，所以不需要手工维护映射。短名会同时用于：

- `scripts/*_fm.tcl`
- `bsub_logs/*`
- `task_logs/*`
- `runner_logs/*`
- `fm_work/*`
- `outputs/rpts` 目录

如果你在同一个 `scripts/` 目录里已经有很多旧的长文件名，重新 `make generate` 不会自动删除旧文件。确认没有旧 job 还依赖这些 Tcl 后，可以先清掉旧生成 Tcl：

```bash
make clean-generated-tcl
make generate
```

如果还有旧 job 在 PEND/RUN，不要清理旧 Tcl，否则旧 job 启动时可能找不到原文件。

### 7.4 `gen_module_fm.py`

用途：

- 从模板生成一批 `*_fm.tcl`
- 当前通常由 `auto_resolve_and_gen_fm.py` 调用
- 它只负责套模板，不负责决定 instance path

手工使用示例：

```bash
python3 gen_module_fm.py \
  --module-list scripts/resolved_targets.list \
  --template ./fm_template_instance_aware.tcl \
  --output scripts/ \
  --ref-modules-tcl ./ref.tcl \
  --imp-modules-tcl ./imp.tcl \
  --user-match-tcl ./user_match.tcl \
  --dedup \
  --workers 0
```

注意：如果直接跑它，不会自动 patch `context_top_ref/target_ref_inst`。正常流程请用 `make generate` 或 `auto_resolve_and_gen_fm.py --flow-stage generate-only`。

### 7.5 `gen_bsub_submit.py`

用途：

- 生成并执行 LSF 提交脚本
- 当前默认按 `scripts/resolved_targets.list` 提交 instance task

`make submit` 调的是：

```bash
python3 gen_bsub_submit.py \
  --module-list scripts/resolved_targets.list \
  --tcl-dir scripts/ \
  --output submit_fm_array.sh \
  --job-prefix fm_obf \
  --max-inflight 100 \
  --poll-sec 100 \
  --bsub-log-dir bsub_logs \
  --task-log-dir task_logs \
  --runner-log-dir runner_logs \
  --fm-work-dir-root fm_work \
  --skip-passed \
  --extra-bsub '-R select[mem>16000] -R rusage[mem=16000]'

bash submit_fm_array.sh
```

如果只想看每个 target 对应的 bsub 命令：

```bash
make -n submit
```

如果确实要把每个 target 对应的完整 `bsub ...` 命令落盘：

```bash
make submit DUMP_FLAT_CMDS=debug_bsub_cmds.txt
```

生成后看：

```text
debug_bsub_cmds.txt
```

注意：instance 模式下 target 可能接近百万级，`debug_bsub_cmds.txt` 会非常大。默认已经不再生成它，否则你会看到 `make submit` 一开始长时间没有任何 bsub 输出，其实是在 Python 阶段写这个 debug 文件。

`make submit` 开始阶段可能慢的常见原因：

- Python 生成阶段：读取 `scripts/resolved_targets.list`，写 `submit_fm_array.sh.modules` 和 `submit_fm_array.sh`
- debug 命令 dump：如果设置了 `DUMP_FLAT_CMDS=debug_bsub_cmds.txt`，会为每个 target 额外生成一条完整 bsub 命令，百万级 target 会很慢
- LSF 限流：真正执行 `bash submit_fm_array.sh` 后，每次提交前会查 `bjobs`，如果 `MAX_INFLIGHT` 已满，会按 `POLL_SEC` 等待

如果你只是正常提交，推荐：

```bash
make submit
```

如果你想更快看到 bsub 发射，但能接受更多排队任务，可以调高：

```bash
make submit MAX_INFLIGHT=300 POLL_SEC=20
```

这个值要看集群配额，太大可能导致 LSF 查询慢或者被管理员限流。

### 7.6 `detect_fm_failures.py`

用途：

- 扫描 `bsub_logs/task_logs/runner_logs`
- 生成失败列表、通过列表和状态表
- instance 模式下，失败列表里是 target task id，不是 module 名

`make detect` 调的是：

```bash
python3 detect_fm_failures.py \
  --module-snapshot submit_fm_array.sh.modules \
  --bsub-log-dir bsub_logs \
  --task-log-dir task_logs \
  --runner-log-dir runner_logs \
  --failed-list failed_module.list \
  --pass-list pass_modules.list \
  --status-csv fm_status.csv \
  --workers 64
```

输出：

```text
failed_module.list
pass_modules.list
fm_status.csv
```

虽然文件名还叫 `failed_module.list`，instance 模式下它实际保存的是失败的 target task id。

### 7.6.1 Formal 比对慢怎么判断

现在这个 flow 的 `make submit` 是“一 target 一次 fm_shell”。也就是说，每个 instance target 都会单独启动一次 Formality，然后执行：

```tcl
read_sverilog -container r ...
set_top r:/WORK/${context_top_ref}
read_sverilog -container i ...
set_top i:/WORK/${context_top_imp}
set_reference ...
set_implementation ...
match
verify
```

所以慢的最大原因通常不是 `verify` 本身，而是每个 target 都重复做完整 RTL 读取、elaborate、set_top 和 match。instance 模式能保证 parameter 是真实实例参数，但代价就是同一个 top 下很多小 instance 也会反复冷启动。

如果 2 小时只完成 300 个，先分清三类原因：

1. LSF 并发不够
   - 默认 `MAX_INFLIGHT=100`
   - 如果大量任务在 PEND，真实 RUN 数可能远小于 100
   - 这种情况不是 FM 慢，是队列慢
2. 单个 fm_shell 冷启动太重
   - 每个 task 都读完整 ref/imp filelist
   - 大 top、宏多、SV elaborate 慢时，小模块也会慢
   - 这种情况需要减少重复读 RTL，或按 top/session 分组
3. target 粒度太细
   - instance 模式下同一个 module 的多个不同参数实例会拆成多个 target
   - 大部分 instance 可能很小，但每个仍然付一次完整 fm_shell 启动成本

现场快速诊断命令：

```bash
bjobs -u "$USER" -o 'stat job_name queue from_host exec_host submit_time start_time' -noheader | head -40
```

看当前 RUN/PEND 数：

```bash
bjobs -u "$USER" -o 'stat' -noheader | awk '{n[$1]++} END{for (s in n) print s,n[s]}'
```

看最近完成的 Formality elapsed time：

```bash
grep -R "Elapsed time:" runner_logs 2>/dev/null | tail -30
```

看是不是大量任务还在排队：

```bash
bjobs -u "$USER" -p | head -80
```

如果 RUN 数远小于 `MAX_INFLIGHT`，优先查队列/资源请求/License。当前默认资源是：

```make
EXTRA_BSUB ?= -R select[mem>16000] -R rusage[mem=16000]
```

资源要求越高，PEND 越可能久。可以试：

```bash
make submit MAX_INFLIGHT=300 POLL_SEC=20
```

但如果 license 或队列配额只有几十个，这个命令只会增加 PEND，不会真正提速。

如果 RUN 数接近上限，但每个 `runner_logs/*run.log` 里 `Elapsed time` 都很长，说明瓶颈在 FM 单任务。后续优化方向是：

- 按 context top 分组，在一个 fm_shell 里读一次 design 后连续 compare 多个 instance
- 或者做 session/cache 复用，避免每个 target 都重复 `read_sverilog`
- 对已经 pass 的 target 用 `make submit` 默认 `--skip-passed` 跳过
- 对失败/疑难 target 再开 `SAVE_SESSION=always`，不要对全量 target 保存 session

### 7.6.2 磁盘空间和异常中止策略

当前默认已经按“PASS 不保留重型 session”的思路处理：

- 普通 `make generate` 默认 `SAVE_SESSION=never`
- `make rerun` 默认 `RERUN_SAVE_SESSION=always`，方便失败 case debug
- bsub wrapper 里如果 `fm_shell` 返回 0，会删除 `fm.session` / `*.session`
- 如果没有设置 `KEEP_FM_WORK_DIR_FLAG`，PASS job 的整个 `fm_work/<target>.idxN.work` 会被删除
- `make detect` 默认带 `--cleanup-pass-sessions`，会清理历史 PASS case 残留的 session

如果你确实想保留 PASS 的 work dir：

```bash
make submit KEEP_FM_WORK_DIR_FLAG=--keep-fm-work-dir
```

如果你连 PASS session 也想保留：

```bash
make submit KEEP_PASS_SESSION_FLAG=--keep-pass-session
```

如果想让 detect 把 PASS work dir 整个删掉，而不只是删 session：

```bash
make detect CLEANUP_PASS_WORK_DIR=1
```

其它省空间建议：

- 日常全量不要开 `SAVE_SESSION=always`
- 只对失败列表 rerun 时保存 session
- 不要默认开 `DUMP_FLAT_CMDS`，百万级 target 会生成很大的 debug 命令文件
- PASS 后保留 `runner_logs/task_logs/bsub_logs/fm_status.csv` 即可，重型 `fm_work` 优先删除
- 旧的长文件名 Tcl 确认没 job 依赖后，用 `make clean-generated-tcl` 清掉

异常中止策略：

- `make submit` 默认不启用 fail-fast：`ABORT_ON_ERROR=0`
- 默认提交完就退出，不在 submit 脚本里常驻监控；后续用 `make detect` 汇总结果
- 需要定位第一个 fatal 时，手动开启 `ABORT_ON_ERROR=1`
- `MONITOR_AFTER_SUBMIT` 和 `DEBUG_RERUN_ON_ERROR` 默认跟随 `ABORT_ON_ERROR`
- submit 仍然使用一份 `resolved_targets.list`，但这个 list 默认已经按真实层级排序；top 附近的 target 会先被发射
- 当 `ABORT_ON_ERROR=1` 时，生成的 `submit_fm_array.sh` 会监控本次提交后新产生的 bsub/task/runner logs；一旦发现 `ERROR:` / `Error:` / `Fatal:` / `internal system error` / `segmentation fault` / license failure / `bsub:` 等错误行，会立即：
  - 写入 `submit_fm_array.sh.abort.log`
  - 记录触发 abort 的具体错误行
  - `bkill` 同一个 `JOB_PREFIX` 下仍在 RUN/PEND/挂起状态的 job
  - 从出错 log 文件反查 target，单独提交一个 debug bsub 重跑这个 target
  - debug rerun 会把 Tcl 临时改成 `set enable_save_session 1`，并保留 debug workdir/session
  - 以非零码退出

注意：这是“单 list + 层级排序 + 出错即停”的策略，不是严格的 level barrier。也就是说，如果 `MAX_INFLIGHT` 很大，低层级 target 可能已经被提前提交；一旦上层 target 报错，脚本会把这些仍在 RUN/PEND 的 job kill 掉。

如果想打开出错即停：

```bash
make submit ABORT_ON_ERROR=1
```

如果打开 fail-fast，但不想自动提交单个 save-session debug rerun：

```bash
make submit DEBUG_RERUN_ON_ERROR=0
```

如果打开 fail-fast，但只想提交完就退出，不在 submit 脚本里等待并监控到所有 job 结束：

```bash
make submit MONITOR_AFTER_SUBMIT=0
```

### 7.7 `fm_template_instance_aware.tcl`

用途：

- Formality 实际执行的模板
- 先读真实 top
- 再用 `target_ref_inst/target_imp_inst` 选择目标实例
- 最后 `set_reference/set_implementation`

正常不要手工执行它。它会被 `gen_module_fm.py` 包装成每个 target 的 `*_fm.tcl`。

如果模板位置变化，改：

```make
TEMPLATE ?= ./fm_template_instance_aware.tcl
```

或临时覆盖：

```bash
make generate TEMPLATE=./scripts/fm_template_instance_aware.tcl
```

### 7.8 `user_match.tcl`

用途：

- 在 Formality 里做 ref/imp 的手工匹配
- 匹配子实例、寄存器、端口
- 由生成后的 `*_fm.tcl` 自动 `source`

常用 black-box 开关：

```tcl
set USER_MATCH_ENABLE_CHILD_BLACKBOX 1
set USER_MATCH_MATCH_CHILD_PORTS 1
```

当前 `user_match.tcl` 是 instance-aware black-box 风格：只处理当前选中 instance 这一层，把 child bbox 设成 black box，不再递归展开 child 内部逻辑。

instance 模式下要注意：模板会先解析真实 instance：

```tcl
set ref_target_obj r:/WORK/<context_top>/<target_ref_inst>
set imp_target_obj i:/WORK/<context_top>/<target_imp_inst>
set_reference $ref_target_obj
set_implementation $imp_target_obj
```

因此 `user_match.tcl` 的匹配起点必须是 `ref_target_obj/imp_target_obj`，不能再用旧的裸 module 根：

```tcl
r:/WORK/${topModule_ref}
i:/WORK/${topModule_imp}
```

否则会出现类似：

```text
Error: Unknown name: 'r:/WORK/KN_Com/.../u_atmoic_float' (FM-036)
```

这不是 Verdi/VCS instance map 错，而是 `user_match.tcl` 在 instance-aware flow 里还按旧 scope 做匹配。当前脚本已改成：

- 如果存在 `ref_target_obj/imp_target_obj`，当前 module 的 port/reg 和 child bbox 匹配都从这两个真实 instance scope 开始
- child bbox 默认 `set_black_box`，并做 child cell / child port user match
- 不再使用裸 `regsub` 从 ref path 推导 imp path，避免 bus/special char 名字触发正则解析问题

这个修改只需要覆盖 `user_match.tcl`，已经生成的 `*_fm.tcl` 通常不需要重新生成，因为它们是运行时 `source` 这个文件。

## 8. 日常 debug 怎么做

### 8.1 我明明知道某模块在 top 下，但 resolve 找不到

直接跑：

```bash
make resolve REFRESH_GRAPH=1 DEBUG_MODULE=ppe_top
```

优先看：

```text
scripts/debug_resolve/ppe_top.vcs_vpi.screen.txt
```

它会告诉你：

- typed hierarchy 里有没有 exact module type
- 有没有 fuzzy type name
- 有没有只出现在 instance path 里
- 命中实例的 parent path / parent type 是什么
- graph parent 有没有断

如果你还想保留完整 typed hierarchy：

```bash
make resolve REFRESH_GRAPH=1 DEBUG_MODULE=ppe_top DEBUG_DUMP_TYPED=1
```

### 8.2 生成出来的 Tcl 路径不对

先看模板和 Makefile 变量：

- `TEMPLATE`
- `REF_MODULES_TCL`
- `IMP_MODULES_TCL`
- `USER_MATCH_TCL`
- `FM_FILELIST`

### 8.3 遇到 `extra characters after close-quote`

这个问题之前已经修过，根因是：

- Tcl 行里已经有 `-vcs "..."` 字符串
- 旧 patch 逻辑又把 `-f` 路径套了一层双引号
- 变成嵌套引号，Formality Tcl 直接报错

现在 `gen_module_fm.py` 和 `auto_resolve_and_gen_fm.py` 都会对：

```tcl
-vcs "+define+SYNTHESIS -f rtl_ref.f"
```

这种写法做特殊处理，正确改成：

```tcl
-vcs "+define+SYNTHESIS -f /abs/path/rtl_fm.f"
```

### 8.4 遇到 `FM-008 current design is not set`

典型日志形态：

```text
Implementation design set to 'i:/WORK/KN_top'
Error: The current design is not set. A design must be specified. (FM-008)
... while executing
"get_cells -quiet -hier ${scope_root}/*"
... procedure "_fm_find_first_inst_by_module"
```

根因：

- 模板先 `set_top r:/WORK/<top>`
- 又 `set_top i:/WORK/<top>`
- 这时 Formality 的当前设计状态已经偏向 implementation
- 旧模板随后调用 `_fm_find_first_inst_by_module r:/WORK/<top> ...`
- 里面直接 `get_cells -hier r:/WORK/<top>/*`
- 某些 Formality 版本即使路径里写了 `r:/...`，仍然要求 current design 已经切到对应 design，否则就报 `FM-008`

现在模板已经补了显式 current design 切换：

```tcl
_fm_set_current_design_if_possible r:/WORK/${context_top_ref}
_fm_set_current_design_if_possible i:/WORK/${context_top_imp}
```

并且 `_fm_find_first_inst_by_module` 入口也会先尝试：

```tcl
_fm_set_current_design_if_possible $scope_root
```

所以 module 模式下继续自动找实例时，也不会因为 current design 没切好而直接炸。

另外，当前默认已经是：

```text
TARGET_GRANULARITY=instance
```

per-instance 生成出来的 Tcl 会直接写入：

```tcl
set target_ref_inst "..."
set target_imp_inst "..."
```

因此默认路径不会再进入 `_fm_find_first_inst_by_module` 的 `get_cells` 自动搜索分支，触发这个问题的概率更低。模板里保留修复，是为了兼容你临时切回 `TARGET_GRANULARITY=module` 或某些 target 没有 instance path 的情况。

## 9. 这次做完后，脚本怎么理解最省脑子

可以把整个工程理解成下面这三层。

### 9.1 第一层：核心业务链路

- `resolve_module_tops.py`
- `auto_resolve_and_gen_fm.py`
- `gen_module_fm.py`
- `fm_template_instance_aware.tcl`
- `user_match.tcl`

### 9.2 第二层：批量运行外壳

- `Makefile`
- `gen_bsub_submit.py`
- `detect_fm_failures.py`

### 9.3 第三层：一次性 debug / probe

- 各种 `probe_verdi_*.tcl`
- 各种 Verdi probe runbook

以后如果你要继续精简，我建议原则是：

1. 不删第一层
2. 第二层只保留常用入口
3. 第三层全部视为工具箱，不放进主 README 主线

## 10. 这次做过的“保守精简”

这次没有去删已经验证过有用的逻辑，而是做了两种更稳的收敛：

1. 使用层精简
   - 新增 `make all`
   - 新增 `make daily`
   - `generate/submit/detect/rerun` 的公共参数在 Makefile 里收成公共块，后面改变量时不用到处改
2. 认知层精简
   - 主流程默认只强调 `VCS+VPI`
   - Verdi probe 相关内容转为 debug 辅助，不再当主线

## 11. 推荐命令速查

首次或 graph 变化后：

```bash
make all
```

只复用已有 resolve 结果继续跑：

```bash
make daily
```

只做 resolve：

```bash
make resolve REFRESH_GRAPH=1
```

只做 generate：

```bash
make generate
```

失败模块重跑：

```bash
make rerun
```

debug 某模块：

```bash
make resolve REFRESH_GRAPH=1 DEBUG_MODULE=ppe_top
```

如果你真的想临时退回旧的 module 粒度，也可以：

```bash
make resolve TARGET_GRANULARITY=module
make generate TARGET_GRANULARITY=module
```

但现在默认不建议这么做，除非你明确想做一个更粗粒度的快速实验。

## 12. 你后面最值得记住的三句话

1. `parameter` 现在不是靠“裸 module”识别，而是靠“真实 top 下的真实 instance”识别。  
2. `make generate` 默认应该以 `scripts/instance_target_map.tsv` 为准，并重写 `scripts/resolved_targets.list`。  
3. 以后目录变了，先改 `Makefile` 变量，再考虑改模板或 Python。

## 13. 后续高吞吐方向：长驻 fm_shell worker pool

如果 instance target 是几十万级，而当前吞吐只有大约 `150/h`，继续用“一 target 一个 fm_shell”的模型不现实。因为每个 target 都在重复：

```text
启动 fm_shell
read_sverilog ref 全量 RTL
read_sverilog imp 全量 RTL
set_top
match/verify 一个 target
退出
```

更合理的高吞吐模型是：

```text
启动 100 个 worker bsub job
每个 worker 内部启动一个长驻 fm_shell
每个 worker 只 read_sverilog 一次
worker 循环领取 target
每个 worker 同一时刻 compare 一个 target
100 个 worker 合起来仍然保持约 100 路并行
```

这和“按 top 分组后组内串行”不同。worker pool 的并行单位不是“top group”，而是“长驻 worker”。如果开 100 个 worker，那么同一时间仍然可以有约 100 个 target 在 compare，只是每个 worker 做完一个 target 后不退出、不重读 RTL，而是继续做下一个 target。

### 13.1 推荐落地架构

生成三个新东西：

```text
worker_tasks.tsv
submit_fm_workers.sh
fm_worker_pool.tcl
```

`worker_tasks.tsv` 每行保存一个 target：

```text
task_id    module    context_top_ref    context_top_imp    target_ref_inst    target_imp_inst
```

`submit_fm_workers.sh` 提交固定数量 worker：

```bash
make submit-workers FM_WORKERS=100
```

每个 worker 运行：

```bash
fm_shell -f fm_worker_pool.tcl
```

`fm_worker_pool.tcl` 做：

```tcl
source fm_setup.tcl
source util.tcl
source ref.tcl
source imp.tcl
source user_match.tcl

read_sverilog -container r ...
read_sverilog -container i ...

while {还有 target} {
  领取一个 target
  set_top 当前 context top
  set_reference 当前 ref instance
  set_implementation 当前 imp instance
  match
  verify
  写单 target 结果和 log
  清理本轮 match/verify 状态
}
exit
```

### 13.2 target 分配方式

第一版建议用“静态分片”，最简单稳定：

```text
worker_000.list
worker_001.list
...
worker_099.list
```

生成时把 `resolved_targets.list` round-robin 分到 100 个 list。优点是实现简单，不需要 NFS 锁。缺点是不同 worker 工作量可能不均衡，最后会有尾巴。

第二版再做“动态领取”：

```text
queue/pending/*.task
queue/running/*.task
queue/done/*.task
queue/fail/*.task
```

worker 用原子 `mv` 或 lock directory 领取任务。优点是负载均衡好，缺点是 NFS 原子性和异常恢复要更谨慎。

建议路线：

```text
先静态分片 pilot -> 确认 Formality session 状态可复用 -> 再做动态队列
```

### 13.3 最大风险：连续 compare 的状态隔离

worker pool 能不能安全，核心不在 Python，而在 Formality session 里连续 compare 多个 target 时状态是否干净。必须先做小样本验证：

```text
选 20 个 target
旧模式逐个跑，得到 PASS/FAIL
worker 模式一个 fm_shell 连续跑同样 20 个 target
对比 PASS/FAIL 是否一致
反过来换一个顺序再跑一次
```

重点看：

- `set_reference/set_implementation` 切到新 target 后，旧 compare point 是否残留
- `match` 后的手工匹配是否污染下一个 target
- 一个 target fail 后，worker 能否继续跑下一个 target
- 每个 target 的 log/status/report 是否能单独落盘
- 是否需要在每个 target 前后调用 Formality 的 reset/cleanup 命令

如果这一步不过，worker pool 不能直接全量使用。

### 13.4 预期收益

如果现在单个 target 平均 40 分钟，其中大部分花在 `read_sverilog/elaborate/set_top`，worker pool 会把这部分成本从“每个 target 一次”降到“每个 worker 一次”。

粗略理解：

```text
旧模式：几十万个 target * read_sverilog 成本
worker 模式：100 个 worker * read_sverilog 成本 + 几十万个 target * compare 成本
```

真实吞吐取决于 `match/verify` 本身耗时、license、host 资源和状态清理成本，但方向上它是目前最值得做的架构升级。

## 14. 单脚本 top/subsystem + set_black_box 方案评估

这句话：

```text
搞一个脚本，就一个，把所有的 set_user_match 都设上。
然后通过 set_top 和 set_black_box 来指定跑顶层和底下几个大module。
```

本质是建议从“逐 module/逐 instance 批量等价”切到“顶层或大子系统等价 + black box 分层控制”。

### 14.1 它想解决什么

当前 instance flow 的主要成本是：

```text
每个 target 单独 fm_shell
每个 target 重复 read_sverilog/ref+imp
每个 target 重复 elaborate/set_top/match/verify
```

单脚本 top/subsystem flow 会变成：

```text
一个 Tcl
read_sverilog ref 一次
read_sverilog imp 一次
set_top 到 KN_top 或某个大子系统
一次性 source/ref.tcl/imp.tcl/user_match.tcl
用 set_black_box 控制哪些大模块不展开
match
verify
```

它能显著减少调度和重复读 RTL 的成本。

### 14.2 它适合验证什么

适合：

- 顶层连线、端口、层级连接是否等价
- 大子系统边界是否等价
- 用 black box 把已知复杂模块挡住，只验证外部集成
- 快速发现“结构连接类”的大问题

不适合单独替代：

- 每个 leaf module 内部逻辑都必须被证明的场景
- black box 内部逻辑变化需要被覆盖的场景
- top-level 状态空间过大、black box 切得不干净的场景

关键点是：`set_black_box` 会降低证明复杂度，但被 black box 的内部逻辑就不再被这次 top verify 证明。它证明的是 black box 边界和外部逻辑，不是 black box 内部。

### 14.3 推荐做法

不要一上来跑完整 `KN_top` 全展开。建议做分层 pilot：

1. 先做顶层 smoke
   - `set_top KN_top`
   - 把几个超大模块 black box
   - source `user_match.tcl`
   - 看 match/verify 能不能跑通
2. 再做子系统级 verify
   - 选择 `KN_Com`、`KN_cache`、`dma_subsys_top` 这种大模块
   - 每次只展开一个大子系统
   - 其它 sibling block black box
3. 最后对可疑模块再回到 instance flow
   - 对 top/subsystem fail 的区域，用现在的 instance-aware flow 精查

这会形成两层验证：

```text
top/subsystem flow: 少数大脚本，验证集成和大边界
instance flow: 少数失败/可疑 target，做精细定位
```

### 14.4 单脚本 Tcl 大概长什么样

伪代码：

```tcl
source -echo ../common/scripts/fm_setup.tcl
source -echo ../common/scripts/util.tcl
set_app_var verification_passing_mode Equality

source ./ref.tcl
set ref_modules $modules
unset modules

source ./imp.tcl
set imp_modules $modules
unset modules

set topModule_ref KN_top
set topModule_imp KN_top

read_sverilog -container r -libname WORK -17 -vcs "+define+SYNTHESIS -f ./rtl_fm.f"
set_top r:/WORK/KN_top

read_sverilog -container i -libname WORK -17 -vcs "+define+SYNTHESIS -f ./rtl_fm.f"
set_top i:/WORK/KN_top

set_reference r:/WORK/KN_top
set_implementation i:/WORK/KN_top

# Black box 大块，注意 ref/imp 两边都要设置
set_black_box r:/WORK/KN_top/u_big_block0
set_black_box i:/WORK/KN_top/u_big_block0
set_black_box r:/WORK/KN_top/u_big_block1
set_black_box i:/WORK/KN_top/u_big_block1

set USER_MATCH_MAX_DEPTH 2
source ./user_match.tcl

match
verify
report_unmatched_points > ./rpts/top_unmatched.rpt
report_failing_points > ./rpts/top_failing.rpt
exit
```

实际命令名可能要按你们 Formality 版本微调，尤其是 `set_black_box` 对 instance path/module name 的支持形式。

### 14.5 black box 选 module 还是 instance

优先建议用 instance path：

```tcl
set_black_box r:/WORK/KN_top/u_xxx
set_black_box i:/WORK/KN_top/u_xxx
```

这样范围最明确。

不建议一开始用“按 module type 全局 black box”，因为同一个 module type 可能在多个地方例化。全局 black box 可能挡掉你本来想验证的实例。

### 14.6 风险点

1. `set_user_match` 规模可能很大
   - 顶层递归深度太大时，会生成大量 match
   - 可以先 `USER_MATCH_MAX_DEPTH=1/2`，不要一上来 `-1`
2. black box 后覆盖会变小
   - black box 内部不被验证
   - 需要配合子系统级 verify 补覆盖
3. 顶层 constraints 更重要
   - reset、clock、scan/test mode、tie-off 如果没约束，top verify 容易失败或跑很久
4. ref/imp 层级名差异会放大
   - 顶层一次性 match 时，层级差异会集中爆出来
   - 需要 report unmatched/failing 后迭代 `user_match.tcl`

### 14.7 我的结论

这个方案值得做，但不要把它理解为“一个脚本证明所有 leaf module”。更合理的定位是：

```text
top/subsystem-level smoke + 分层集成证明
```

建议先实现一个最小 pilot：

```text
make topfm TOP=KN_top BLACKBOX_LIST=./blackbox_instances.list
```

只跑一个 top，black box 几个最大 block，先看：

- read/elab 是否能稳定通过
- user_match 是否能在顶层 scope 下完成
- unmatched/failing 点规模是否可控
- runtime 是否比 instance 批量方案好很多

如果 pilot 结果好，再把它变成正式 flow；如果 top 证明爆炸，就退成“每次展开一个大子系统，其它 sibling black box”的分层验证。

## 15. 按层级逐层 Formal 的方案

这个思路可以做，而且比“所有 instance 全部平铺提交”更符合大设计收敛方式。

你说的策略可以定义成：

```text
level 0: top 本身
level 1: top 下面第一层子模块
level 2: level 1 模块下面的子模块
...

每次只跑当前 level 的 formal。
当前 level 全部 PASS -> 继续下一层。
当前 level 任意 FAIL/UNKNOWN -> 停止，报错退出，先定位这一层。
```

### 15.1 它和 black box 的关系

每层 formal 不应该无限展开所有下级逻辑，否则 level 1 就可能已经把整个设计都吃进去。更合理的是：

```text
当前要验证的 module/instance 展开
它下面更深一层的大子模块先 set_black_box
```

也就是当前层只证明：

- 当前 block 自己的组合/寄存器逻辑
- 当前 block 到 child block 的连接关系
- 当前 block 的接口映射

child block 内部逻辑留到下一层再验证。

这就是 compositional / hierarchical formal 的味道。

### 15.2 为什么“全部 pass 再下一级”是合理的

好处：

- 失败定位更清晰：level 1 没过，就不用浪费时间跑 level 5 的小模块
- 每个 formal 的状态空间更小：深层 child 被 black box
- 不需要几十万个 instance 全部排队
- 和工程 debug 习惯一致：先大边界，再逐层下钻

代价：

- 需要明确每层的 black box 边界
- 被 black box 的内部逻辑必须在后续 level 继续验证
- 如果某一层约束不完整，可能误报 fail 或 inconclusive

### 15.3 层级怎么定义

建议用真实 instance hierarchy，而不是静态 module parser：

```text
scripts/instance_target_map.tsv
```

它里面已经有：

```text
task_id    module    context_top    ref_inst    imp_inst    full_inst
```

可以从 `ref_inst/full_inst` 的层级深度推导 level：

```text
KN_top                                      -> level 0
KN_top.u_a                                  -> level 1
KN_top.u_a.u_b                              -> level 2
KN_top.u_a.u_b.u_c                          -> level 3
```

对于同一个 module 在同一层出现很多实例，默认还是只随机挑一个实例：

```text
same module + same level -> pick one instance
```

如果后续发现某个 module 有不同 parameter 变体，再扩展成：

```text
same module + same parameter signature + same level -> pick one instance
```

### 15.4 每个 level 跑什么

建议生成：

```text
layered_targets/level_00.list
layered_targets/level_01.list
layered_targets/level_02.list
...
```

每个 list 是这一层要跑的 target。然后调现有生成/提交流程：

```bash
make generate GENERATE_MODULE_LIST=layered_targets/level_01.list
make submit
make detect
```

如果 `failed_module.list` 或 `fm_status.csv` 里有 FAIL/UNKNOWN，就停止。

如果全 PASS：

```bash
make generate GENERATE_MODULE_LIST=layered_targets/level_02.list
make submit
make detect
```

### 15.5 推荐新增 make 入口

后续可以做成：

```bash
make layer-plan
make layer-run LEVEL=1
make layer-detect LEVEL=1
make layer-next
```

或者更自动一点：

```bash
make layer-all
```

`layer-all` 的逻辑：

```text
生成所有 level list
for level in 0..N:
  generate 当前 level
  submit 当前 level
  detect 当前 level
  如果有 FAIL/UNKNOWN:
    打印失败列表
    exit 1
  否则继续下一层
```

### 15.6 我建议的 pilot

不要一上来全自动跑所有层，先做三层 pilot：

```text
level 1: top 第一层子模块
level 2: 第二层子模块
level 3: 第三层子模块
```

每层仍然限制：

```text
每个 module 随机一个 instance
MAX_INFLIGHT=100
当前 block 的下一级大 child black box
```

先观察：

- 每层 target 数量是多少
- 每层 pass/fail 比例
- 单个 target runtime 是否明显下降
- black box 后 unmatched/failing 是否可控

### 15.7 我的判断

这个方案可以做，而且比 worker pool 更容易落地：

- 不需要长驻 fm_shell
- 不需要动态队列
- 可以复用当前 `make generate/submit/detect`
- 失败定位比全量 instance 平铺更清楚

但它不是简单地“按 depth 筛 target”就完事。关键是要配套 black box 策略，否则每层 formal 仍可能展开太深，runtime 不一定降。

我建议下一步先实现一个只做 level list 的脚本：

```text
build_layered_targets.py
```

输入：

```text
scripts/instance_target_map.tsv
```

输出：

```text
layered_targets/level_01.list
layered_targets/level_02.list
layered_targets/summary.tsv
```

先只做分层和每 module 抽样，不急着自动 black box。拿到每层 target 数量后，再决定 black box 怎么接入模板。

## 16. 云端 `user_match.tcl` 截图版评估

这次云端截图里的 `user_match.tcl` 可以理解成一版旧的 `.bak` 风格匹配脚本，核心思路是：

1. 从 `ref_modules` / `imp_modules` 字典里取当前 `topModule_ref/topModule_imp` 的三类信息：
   - ports：端口名列表
   - regs：寄存器名列表
   - bboxs：子实例列表，每项包含 child module 和 child instance
2. 先按 bbox list 的相同 index 找 ref/imp 子实例：
   - 取 instance leaf name
   - 在 `r:/WORK/${topModule_ref}/...` 和 `i:/WORK/${topModule_imp}/...` 下查找
   - exact 找不到时用 wildcard fallback
3. 对找到的子实例做：
   - `set_black_box <ref_child_inst>`
   - `set_black_box <imp_child_inst>`
   - `set_user_match -noninverted -type cell <ref_child_inst> <imp_child_inst>`
   - 再按 ports list 的相同 index 匹配 child instance port
4. 最后对当前 module 顶层 port 和 register 再按字典里的 index 做 `set_user_match`。

这版脚本的定位更像“验证当前 module 边界 + 把下级 bbox 挡住”。它适合把 child block 当成已验证/待后续验证的黑盒，只证明当前层的边界连接和当前层寄存器/端口映射；它不是 full recursive 展开验证。

### 16.1 截图里指出的 bug

原脚本里用类似下面的写法从 ref 名字推导 imp 名字：

```tcl
regsub $x $my_ref_port $y my_imp_port
regsub r: $my_imp_port i: my_imp_port
regsub ${topModule_ref} $my_imp_port ${topModule_imp} my_imp_port
```

问题是 `regsub` 的第一个参数是正则表达式，不是普通字符串。如果 `$x` 里包含 `[ ] ( ) . + * \` 等特殊字符，例如 bus 名 `dat_m_i[0]`，就可能被当成正则解析，触发 `invalid escape \ sequence` 或替换错对象。

更稳的写法是先复制 ref path，再用 `string map` 做字面量替换：

```tcl
set my_imp_port $my_ref_port
set my_imp_port [string map [list $x $y] $my_imp_port]
set my_imp_port [string map [list "r:" "i:"] $my_imp_port]
set my_imp_port [string map [list $topModule_ref $topModule_imp] $my_imp_port]
```

register 部分同理，凡是用 ref full path 推导 imp full path 的地方，都应该用 `string map [list ...]`，不要用裸 `regsub`。

### 16.2 和当前 instance-aware flow 的兼容性

当前主 flow 不是把目标 module 当裸 top 比，而是先读真实 context top，再通过 `target_ref_inst/target_imp_inst` 设置：

```tcl
set_reference $ref_target_obj
set_implementation $imp_target_obj
```

因此 `user_match.tcl` 的匹配根不能固定写成：

```tcl
r:/WORK/${topModule_ref}
i:/WORK/${topModule_imp}
```

在 instance 模式下，它应该从：

```tcl
$ref_target_obj
$imp_target_obj
```

开始解析 ports / regs / child instances。否则同一个 module 被选中的真实 instance 可能是：

```text
r:/WORK/KN_top/u_a/u_target
i:/WORK/KN_top/u_a/u_target
```

而不是：

```text
r:/WORK/<target_module>
i:/WORK/<target_module>
```

所以不能直接把云端截图版脚本原样替换进当前 flow。正确做法是保留它的“按字典 index 映射 ports/regs/bboxs”和“必要时 black box child instance”的原理，但把所有搜索 scope 改成 instance-aware root，也就是优先使用 `ref_target_obj/imp_target_obj`。

### 16.3 Makefile 判断

如果新的 `user_match.tcl` 仍然放在工程根目录，和 `Makefile`、`gen_module_fm.py`、`fm_template_instance_aware.tcl` 在同一层，那么 Makefile 默认值已经够用：

```make
USER_MATCH_TCL ?= ./user_match.tcl
```

`make generate` 会通过 `--user-match-tcl $(USER_MATCH_TCL)` 把模板里的 `source ./user_match.tcl` 改成实际路径。因此同目录替换时不需要改 Makefile。

只有两种情况需要改：

- 新脚本放到其它目录：运行时覆盖 `USER_MATCH_TCL=/path/to/user_match.tcl`
- 想同时保留多版脚本：可以新增变量或用命令行覆盖，不建议硬改模板

### 16.4 当前 workspace 实现

当前 workspace 的 `user_match.tcl` 已经按“云端 black-box 原理 + instance-aware root + 避免 regsub bug”的方向改成合并版：

- root scope 使用 `ref_target_obj/imp_target_obj`
- port/register 不再用裸 `regsub` 推导路径，改为在 instance scope 下直接解析 ref/imp 对象
- 子实例解析保留 exact + wildcard fallback
- 是否 `set_black_box` 做成开关，例如 `USER_MATCH_ENABLE_CHILD_BLACKBOX`
- instance 模式下不再跑裸 module top port matching，而是在选中的真实 instance scope 下匹配当前 module ports/regs

这样才能同时满足：

- 每个 module 随机抽一个真实 instance 比对
- parameter 来自真实 context top
- user match 从选中的真实 instance 开始
- 避免 `regsub` 被 bus/special char 名字打爆

## 17. 正式 Wrapper Flow：VCS 参数前置解析，Formality 只比当前 module

**最高优先级约束：`ref.tcl` / `imp.tcl` 是 module 映射字典，不是 read-design Tcl。**

- `ref.tcl` 只有 `set modules [dict create ...]`，是 reference 侧 module/port/reg/bbox 映射字典。
- `imp.tcl` 只有 `set modules [dict create ...]`，是 implementation 侧 module/port/reg/bbox 映射字典。
- read design 入口沿用旧 flow 的 VCS-style `read_sverilog -vcs "... -f <filelist>"` 机制。
- `rtl_graph.f` 只用于 VCS 编译、层级解析、instance/parameter/port detail dump。
- `rtl_fm.f` 用于 Formality/VCS read 和候选 module/source 定位。

当前主流程已经切到 wrapper 模式。目标是避免每个 target 都让 Formality 重复读完整设计、重复 elaboration 全设计 parameter：`make generate` 先完成切片，每个 module 只选一个 instance，生成 wrapper，并把 child module 变成 blackbox stub。VCS 负责先拿到 parameter/port/child detail，`ref.tcl/imp.tcl` 负责提供 ref/imp module 映射关系。

当前实现采用 wrapper 专用 VCS read 生成策略：

- `make generate` 阶段读一次完整 `ref.tcl/imp.tcl`，解析 `modules` dict。
- 对每个 target 生成小 dict：`<target>.ref_dict.tcl` 和 `<target>.imp_dict.tcl`。
- 小 dict 只包含当前 ref/imp module 的 entry，便于 debug，也避免每个 target source 全量 dict。
- 对每个 target 生成 wrapper filelist：`<target>.ref_wrapper.f` 和 `<target>.imp_wrapper.f`。
- per-target Formality Tcl 使用 `read_sverilog -container r/i -libname WORK -17 -vcs "... -f <target>.wrapper.f"`。
- 因此 `ref.tcl/imp.tcl` 不会被误当作 read RTL 入口，`rtl_graph.f` 也不会被当作 imp Formality 输入。

注意：`gen_module_fm.py`、`fm_template_instance_aware.tcl`、`user_match.tcl` 属于旧的完整设计 instance-aware flow。正式 wrapper flow 不再把它们作为主路径；除非专门 debug 旧流程，否则不要混用旧模板和当前 `make generate` 输出。

仍然按原来的节奏分步运行：

```bash
make resolve
make generate
make submit
make detect
```

默认行为/职责边界：

- 从 `scripts/instance_target_map.tsv` 读取 instance target
- 从 `scripts/module_top_map.tsv` 补充 top module 本身的 TOP target
- `rtl_fm.f` 用于 Formality/VCS read 和候选 module/source 范围
- `rtl_graph.f` 用于 VCS detail fallback 和层级/参数解析
- ref/imp module 映射来自 `ref.tcl` / `imp.tcl`
- RTL module source cache 只能作为生成 wrapper 的辅助缓存，不能替代 VCS/detail/dict 语义
- 接入 `../exclude.list`，module 名或 task id 命中都会跳过
- 每个 module 只选一个 instance
- 优先复用已有 detail TSV；只有缺少选中 target 的 detail 时才用 VCS+VPI 补跑
  - resolved parameter
  - elaborated port direction/width
  - child instance module/port 信息
- 生成 wrapper/stub/FM Tcl 到 `scripts/`
- `make submit` 复用 `gen_bsub_submit.py`，通过 bsub 发射
- `make detect` 复用 `detect_fm_failures.py`，产出 pass/fail/status 并清理 PASS session

主要输出：

```text
scripts/
  wrapper_targets.tsv
  wrapper_summary.tsv
  unsupported_targets.tsv
  vcs_instance_details.tsv
  module_source_index.cache.json
  resolved_targets.list
  <task_id>_fm.tcl
  wrappers/
    <target>.ref_target.sv
    <target>.imp_target.sv
    <target>.ref_wrap.sv
    <target>.imp_wrap.sv
    <target>.blackbox_stubs.sv
    <target>.ref_wrapper.f
    <target>.imp_wrapper.f
    <target>.ref_dict.tcl
    <target>.imp_dict.tcl

submit_fm_array.sh
submit_fm_array.sh.modules
failed_module.list
pass_modules.list
fm_status.csv
bsub_logs/
task_logs/
runner_logs/
fm_work/
```

wrapper 形态：

```systemverilog
module fm_ref_wrap_xxx(...);
  fm_ref_target_xxx #(
    .P0(<VCS resolved value>)
  ) u_dut (...);
endmodule

module fm_imp_wrap_xxx(...);
  fm_imp_target_xxx #(
    .P0(<VCS resolved value>)
  ) u_dut (...);
endmodule
```

child module 会生成 blackbox stub：

```systemverilog
(* black_box *) module child_module(...);
endmodule
```

正确的 wrapper Formality Tcl 应满足：

- reference/implementation container 使用 VCS-style read，filelist 指向当前 wrapper source
- per-target 小 dict 来自 `ref.tcl/imp.tcl` 的当前 module entry
- wrapper 只限制比较 top 和 blackbox 边界
- 当前 module 内部的 wire/reg/always/assign 参与比较
- 直接 child module 只保留 blackbox 边界，不比较 child 内部实现

然后：

```tcl
set_top r:/WORK/fm_ref_wrap_xxx
set_top i:/WORK/fm_imp_wrap_xxx
set_reference ...
set_implementation ...
match
verify
```

也就是说，parameter 解析可以前置到 VCS；`ref.tcl/imp.tcl` 提供映射字典；每个 target 的 Formality 只读当前 wrapper source 和 blackbox stub。

速度相关策略：

- `make resolve` 的 `module_graph.cache.json` 会同时保存 typed instance rows；instance flow 需要 `instance_target_map.tsv` 时，如果 cache meta 一致，会直接复用 cache，不再因为 `--dump-instance-targets` 强制重跑 VCS
- `make generate` 以 `scripts/resolved_targets.list` 为权威 target list；`scripts/instance_target_map.tsv` 只提供这些 target 的 module/context/instance metadata。这样 `make resolve` 已经筛出的 one-per-module 结果会原样进入 wrapper generate，不再被二次 canonical 合并
- wrapper flow 会按 `ref.tcl` 与 `imp.tcl` 中 `dict create` entry 的原始顺序建立 ref/imp module 配对，和旧 user_match 的按 index 对齐思路一致；不要先把两侧 dict 去重后再配对，否则重复 key 或特殊命名会导致 imp module 错位
- wrapper generate 不再覆盖 `scripts/resolved_targets.list`；它自己的成功生成列表写到 `scripts/wrapper_resolved_targets.list`，`make submit` 使用这个 wrapper list
- resolve/generate 阶段的 VCS 编译工作目录默认放在本地 `/tmp/fm_vcs_work_<user>/<cwd>`，减少共享盘小文件压力；最终 `module_graph.cache.json`、`scripts/vcs_details_work/<top>/details.tsv` 仍回写到工程目录用于复用
- 默认 `WRAPPER_DETAIL_MODE=auto`
- 先尝试复用 `WRAPPER_REUSE_DETAILS_TSV`，默认是 `scripts/vcs_instance_details.tsv`
- 再尝试复用 `scripts/vcs_details_work/<top>/details.tsv` cache
- 只有仍然缺 detail 的 target 才按 `context_top` 分组补跑 VCS
- 补跑 VCS 时会传 `+VPI_INSTANCE_DETAILS_FILTER_FILE`，VPI 只输出目标 instance 和它的直接 child instance
- 多个缺失 top 可以用 `WRAPPER_TOP_WORKERS` 并行，默认跟随 `TOP_WORKERS`
- `make generate` 的 `all` 模式中，resolve 和 generate 可以共享辅助 module/source index
- 单独跑 `generate-only` 时，可以优先只扫描 `wrapper_targets.tsv` 里记录的辅助 RTL 文件
- wrapper generate 会预建 direct-child 查找表，不再对每个 target 全量扫描 detail TSV
- wrapper 文件、summary/list/TSV 生成内容未变化时不会重写文件，减少 NFS 写入和后续无效增量
- 每个 target 的 ref/imp target、blackbox stub 和 wrapper 会合并成一个 `scripts/wrappers/<target>.wrapper.sv`，r/i 两边共用同一个 wrapper filelist，以减少 NFS 小文件数量；`ref_dict.tcl/imp_dict.tcl` 仍单独保留方便审计
- Formality `read_sverilog -vcs` 会过滤 `-debug_access+all` 等 VCS elaboration/debug-only 参数，避免 `FM-241 unrecognized VCS option`；这些参数仍可用于 resolve/detail 阶段的 VCS 编译
- wrapper 文件生成支持 `GEN_WORKERS` 并行
- `WRAPPER_COUNT=0` 表示正式全量；可临时设成 `WRAPPER_COUNT=10` 做 smoke

如果需要指定本地 VCS 编译目录：

```bash
make resolve LOCAL_VCS_WORK_ROOT=/local_scratch/$USER/fm_vcs_work
make generate LOCAL_VCS_WORK_ROOT=/local_scratch/$USER/fm_vcs_work
```

如果改了 `rtl_graph.f`、VCS 参数、top candidate 或 VPI C 文件，需要刷新 cache：

```bash
make clean-cache
make resolve
```

默认提交时会把 `fm_shell` 的工作目录放在执行机本地 scratch，而不是共享盘 `fm_work/`：

```bash
USE_LOCAL_FM_WORK=1
```

具体行为：

- PASS case：本地 scratch 创建、本地 scratch 删除，默认不落到共享 `fm_work/`
- FAIL case：如果 `fm_shell` 非 0 退出，或者 runner log 出现 `Verification FAILED` / `Failing compare points`，失败现场会从本地 scratch 复制回共享 `fm_work/<target>.idxN.work`
- debug save-session rerun：直接保存在共享 `fm_work/`，方便定位问题
- `runner_logs/`、`task_logs/`、`bsub_logs/` 仍然写共享目录，供 `make detect` 汇总

默认本地 scratch 根目录由执行机环境决定：

```bash
${TMPDIR:-/tmp}/fm_work_${USER:-user}
```

如果集群有更合适的本地盘，可以指定：

```bash
make submit LOCAL_FM_WORK_ROOT=/local_scratch/$USER/fm_work
```

如果某些机器本地盘空间不足，或需要退回旧行为：

```bash
make submit USE_LOCAL_FM_WORK=0
```

清理旧运行产物，避免 `make detect` 读到历史 bsub log：

```bash
make clean-run
```

`clean-run` 会用并行方式清理 `fm_work/` 下的一级 case 目录，默认并发数：

```bash
CLEAN_WORKERS=32
```

如果共享盘压力大，可以调小：

```bash
make clean-run CLEAN_WORKERS=8
```

如果想先把 `fm_work/` 改名挪走、后台慢慢删，让命令尽快返回：

```bash
make clean-run-async
```

只清 wrapper 生成物，包括 `scripts/*_fm.tcl`、`scripts/wrappers/`、wrapper summary/list：

```bash
make clean-wrapper
make clean-generated
```

只清 resolve 阶段生成的目标映射，不清 VCS/detail cache：

```bash
make clean-resolve
```

只清 cache，包括 `module_graph.cache.json`、`scripts/vcs_instance_details.tsv`、`scripts/vcs_details_work/`、`scripts/module_source_index.cache.json` 和本地 `LOCAL_VCS_WORK_ROOT`：

```bash
make clean-cache
```

清运行日志、wrapper 生成物和 resolve 产物，但保留 cache，适合大多数重新跑 flow 的场景：

```bash
make clean-all
```

`make clean-all` 同样会通过并行删除清 `fm_work/`。如果希望 `clean-all` 也快速返回、后台删除旧 `fm_work/`：

```bash
make clean-all-async
```

彻底从零重跑，连 cache 一起清：

```bash
make clean-fresh
```

选择建议：

- 只想重新提交/重新 detect：用 `make clean-run`
- 改了 wrapper 生成代码或 `ref.tcl/imp.tcl/rtl_fm.f`：用 `make clean-all`
- 改了 RTL 层级、`rtl_graph.f`、VCS 参数，或怀疑 parameter/port detail 过期：用 `make clean-fresh`

### 17.1 运行开关

分阶段运行：

```bash
make resolve
make generate
make submit
make detect
```

如果暂时不想跑 VCS detail dump，只想用已有 detail TSV：

```bash
make generate WRAPPER_DETAIL_MODE=reuse-only
```

如果想强制刷新 wrapper detail cache：

```bash
make generate WRAPPER_REFRESH_DETAILS=1
```

如果确认 license/内存允许，可以提高 fallback VCS top 并行度：

```bash
make generate WRAPPER_TOP_WORKERS=2
```

如果只想先跑 10 个最小 module：

```bash
make generate WRAPPER_COUNT=10
```

直接调用脚本时可以传：

```bash
python3 wrapper_flow.py \
  --flow-stage all \
  --instance-map scripts/instance_target_map.tsv \
  --graph-filelist ./rtl_graph.f \
  --fm-filelist ./rtl_fm.f \
  --exclude-list ../exclude.list \
  --module-top-map scripts/module_top_map.tsv \
  --output scripts \
  --count 0 \
  --detail-mode auto \
  --reuse-detail-tsv scripts/vcs_instance_details.tsv \
  --module-index-cache scripts/module_source_index.cache.json
```

### 17.2 第一版边界

第一版优先覆盖小而简单的 ANSI module：

- 普通 parameter
- 普通 packed vector port
- signed/unsigned 暂按 VPI width 生成 wire vector
- child module blackbox stub
- escaped identifier

这些复杂 SV 形态先识别并写入 `unsupported_targets.tsv`，不硬生成错误 wrapper：

- interface / modport
- struct / union / typedef / enum port
- 无法解析 ANSI header 的老式 module
- VPI 没能提供可靠 port 信息的 target

后续如果 pilot 的普通 module 跑通，再扩展 interface/struct 支持。更稳的路线是继续依赖 VCS elaboration 信息，而不是手写 SV parser。
