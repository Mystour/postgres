# 大作业一 PostgreSQL 哈希表构建 CPU PMC 观测

基于 PostgreSQL 15.2（`REL_15_2`）。实现参考《PG北大内核课程_实践课》第 9–13、15 页。基础模式为 Private Hash 构建阶段输出硬件计数，低开销模式用配对 RDPMC 快照累计循环中计算哈希和插入的计数。

## 源码位置

- [PMC 事件初始化、读取和清理](../src/backend/executor/halo_pmc.c)
- [PMC 接口](../src/include/executor/halo_pmc.h)
- [私有哈希表观测点](../src/backend/executor/nodeHash.c)
- [两个配置参数的注册](../src/backend/utils/misc/guc.c)

## 构建与复现

在支持 PMU 和用户态 RDPMC 的 x86-64 Linux 上，以有 perf 访问权限的普通账号执行。Git 源码构建需要 GCC、Make、Bison、Flex、Perl 等工具。从仓库根目录运行：

```bash
./configure --prefix="$PWD/install" --enable-debug --without-readline --without-zlib
make -j4
make install

work=$(mktemp -d)
mkdir -m 700 "$work/socket"
install/bin/initdb -A trust -D "$work/data"
install/bin/pg_ctl -D "$work/data" -l "$work/server.log" \
  -o "-k '$work/socket' -p 55493 -c listen_addresses=''" -w start
install/bin/psql -X -h "$work/socket" -p 55493 -d postgres \
  -f course-pmc/init-pmc.sql -f course-pmc/verify.sql
install/bin/pg_ctl -D "$work/data" -w stop
```

`init-pmc.sql` 建立 `public.pmc_name` 并写入通用计数事件和课件的 Intel raw 事件示例。每次开始观测时，通过 SPI 读取当前事件定义，再按 `halo_pmc_list` 打开当前线程的用户态事件组。这样修改实验表后，下一次哈希构建会使用新定义。空列表关闭观测。两个观测参数由 PG 超级用户设置。Linux 上运行数据库的操作系统账号也必须有权使用 perf。

在 psql 中可以用如下语句选择事件。最大八项，实际可同时调度的事件数取决于 CPU。Intel raw 事件的编码依赖 CPU 型号。先测 `cycles,instructions`，再测试 L1D 事件或已确认支持的 raw 事件。

```sql
SET halo_pmc_list = cycles,instructions;
SET halo_pmc_list = Instructions,PERF_COUNT_HW_CACHE_L1D_RD,PERF_COUNT_HW_CACHE_L1D_MISS_RD;
SET halo_pmc_segment = on;
```

## 测量范围

基础模式测量 `MultiExecPrivateHash()` 的执行区间，包括从子节点取行、哈希计算及插入等操作。事件组打开和启用一次，结束后按事件 ID 对应并输出真实原始计数、调度比例和缩放计数。执行计划报错也会关闭描述符和解除映射。共享并行哈希表构建不在本实现范围。

低开销模式只累加取行以后的循环体。每一轮在计算哈希和插入之前、之后各读一次用户态映射页及 x86 RDPMC。计数器一直启用，前后快照相减以排除轮与轮之间的取行和其他工作。循环内没有 perf 读系统调用、启停系统调用或日志输出。遇到无直接读取权限、事件未调度或快照不一致时跳过该轮，结束时单独输出成功和跳过的轮数。若驱动仅允许组读取，基础模式仍可以出数。快照和读指令会产生少量测量开销，多事件的边界会略有差异。可先以单事件运行并对比 segment 开关对耗时的影响。

预计查询结果为 9900。硬件路径验收时应看到阶段计数、调度比例及 `RDPMC measured` 大于 0。若只想在无 PMU 的 Linux 环境验证读计数流程，可传 `psql -X -d postgres -v pmc_event=task_clock,cpu_clock -f verify.sql`。这两种属于软件事件，只能检查初始化、开关、读取和 SQL 行为。


## 硬件实测

2026 年 11 月 23 日在 Intel Xeon Silver 4316（Linux x86-64）上，将同一份源码补丁应用到 PostgreSQL 15.2 发布源码后编译并初始化。测试关闭共享并行哈希路径，执行计划使用 Hash Join，三次查询均返回 9,900。

| 观测模式 | 指标 | 结果 |
| --- | --- | ---: |
| 构建阶段，段内观测关闭 | 周期 | 2,292,055 |
| 构建阶段，段内观测关闭 | 指令 | 9,056,828 |
| 循环体 RDPMC | 采样轮数 | 10,000 |
| 循环体 RDPMC | 跳过轮数 | 0 |
| 循环体 RDPMC | 周期差值之和 | 4,163,347 |
| 循环体 RDPMC | 指令差值之和 | 6,232,696 |

两种模式的阶段计数调度比例均为 100%。计数结果证明采集路径工作。一轮日志不足以估计开启段内观测后的稳定开销。详见 [实测日志](postgresql-pmc.log)，其中 SQL 提示中的临时绝对路径已缩短为脚本名，行尾空白已清理。
