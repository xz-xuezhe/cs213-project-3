# CS213 Project 3 Report

薛哲

SID: 12311012

### 1 引言

本项目旨在从性能与功能两方面出发，对 openGauss 与 PostgreSQL 进行对比研究。

### 2 性能比较

#### 2.1 配置

##### 2.1.1 硬件配置

本次测试中使用的服务器配置为

- 规格名称：kc1.large.2
- vCPUs: 2vCPUs
- 内存：4GiB
- CPU: Huawei Kunpeng 920 2.6GHz
- 基准/最大带宽：0.8 / 3 Gbit/s
- 内网收发包：30万PPS
- 系统：openEuler 20.03 64bit with ARM
- 系统盘：通用型SSD 40GiB

若无特殊说明，下文中用到的服务器均为上述配置。

##### 2.1.2 软件配置

openGauss 采用当前最新版本 6.0.0，安装过程参考 [单节点安装 | openGauss文档 | openGauss社区](https://docs.opengauss.org/zh/docs/6.0.0/docs/InstallationGuide/%E5%8D%95%E8%8A%82%E7%82%B9%E5%AE%89%E8%A3%85.html)。

PostgreSQL 采用当前最新版本 17，由于 openEuler 下暂无安装包，编译安装过程参考 [从源码编译安装PostgreSQL 16.x | 青蛙小白](https://blog.frognew.com/2023/11/install-postgresql-16-from-source-code.html)。

为了确保 openGauss 与 PostgreSQL 互不影响，二者被分别装在两台服务器上。

#### 2.2 测试标准

[TPC-C](https://www.tpc.org/tpcc/) 通过对处理仓库订单中的各个流程进行模拟，通过衡量每分钟处理订单数（tpmC），达到对数据库进行实际场景测试的效果。

这里使用 [BenchmarkSQL](https://sourceforge.net/projects/benchmarksql/) 进行 TPC-C 测试。为了避免测试程序对数据库性能造成影响，同时更好地模拟数据库服务器独立的情况，这里额外使用服务器作为测试服务器，型号为 c6s.large.2，操作系统为 CentOS 8.2 64bit。配置过程主要参考了 [BenchmarkSQL性能测试(openGauss) - 知乎](https://zhuanlan.zhihu.com/p/396651167)，由于本次测试没有用到多块磁盘，跳过了其中调整表结构的步骤。

#### 2.3 初步测试

测试使用的配置文件如下：

```properties
db=postgres
driver=org.postgresql.Driver

conn=jdbc:postgresql://<ip_address>:<port>/<database>

user=<username>
password=<password>

warehouses=20
loadWorkers=4
terminals=6
runTxnsPerTerminal=0
runMins=5
limitTxnsPerMin=0

terminalWarehouseFixed=false

newOrderWeight=45
paymentWeight=43
orderStatusWeight=4
deliveryWeight=4
stockLevelWeight=4

resultDirectory=my_result_%tY-%tm-%td_%tH%tM%tS
osCollectorScript=./misc/os_collector_linux.py
osCollectorInterval=1

osCollectorSSHAddr=<user>@<ip_address>
osCollectorDevices=net_eth0 blk_vda
```

openGauss 初步测试结果图像如下（[详细结果](https://xz-xuezhe.github.io/cs213-project-3/gsql/report.html)）：

![openGauss 初步测试结果](https://xz-xuezhe.github.io/cs213-project-3/gsql/tpm_nopm.png)

PostgreSQL 初步测试结果图像如下（[详细结果](https://xz-xuezhe.github.io/cs213-project-3/psql/report.html)）：

![PostgreSQL 初步测试结果](https://xz-xuezhe.github.io/cs213-project-3/psql/tpm_nopm.png)



可以观察到 PostgreSQL 的速度是高于 openGauss 的，在本轮初步测试中 PostgreSQL 的 tpmC 约为 openGauss 的**四倍**。

但 PostgreSQL 在运行过程中会突然进行大规模的磁盘 IO，导致在一段时间内速度大幅度下降，通过重复测试可以发现这并非个例。我个人尝试调整缓存大小，但并无影响，故原因暂时未知。

#### 2.4 正式测试

此次测试前，我对比了二者配置文件 `postgresql.conf`，并调整了以下参数：

- `max_connnections`：最大连接数，二者均调整至 1000。
- `shared_buffers`：内存中的缓存大小，这里取二者默认设置的最大值 128MB。

并另外调整 BenchmarkSQL 运行的线程数量，对运行结果进行比较。

| 线程数量 | openGauss | PostgreSQL | 比值 |
| :------: | :-------: | :--------: | :--: |
|    1     |  2550.60  |  4160.20   | 1.63 |
|    10    |  5569.60  |  17476.60  | 3.14 |
|   100    |  2182.40  |  11621.80  | 5.33 |

可以注意到注意到二者的 tpmC 随线程数量均呈先上升后下降的趋势，且随线程数增加，PostgreSQL 相比 openGauss 的性能优势越发明显。

根据 [知乎回答](https://www.zhihu.com/question/473422324/answer/2010663960)，openGauss 内核源于 PostgreSQL 9.2.4。而 PostgreSQL 9.2.4 本身在 2013 年 4 月 4 日发布，经过尝试无法在相同配置的服务器中成功运行，故未做额外测试。

### 3 功能比较

openGauss 官网网站中提到其一大优势：易运维——基于AI的智能参数调优。

openGauss 文档中的 [AI特性](https://docs.opengauss.org/zh/docs/6.0.0/docs/AIFeatureGuide/AI%E7%89%B9%E6%80%A7.html) 一节对于 openGauss 与 AI 的融合进行了详细介绍。

我本打算进行 AI4DB 部分的测试，但即使我安装了 prometheus-client 包，仍然出现了报错提示：

```
FATAL: Require dependency prometheus-client. You should use pip to install it, following:

        /opt/software/openGauss-DBMind/python/bin/python3 -m pip install -r /opt/software/openGauss-DBMind/dbmind/../requirements-x86.txt
```

DB4AI 部分则主要聚焦于直接在数据库内进行机器学习，使得模型能够更好地直接利用数据库中的数据，这里不做额外测试。

### 4 结论

根据上面的比较，同取当前最新版本，在 BenchmarkSQL 的测试下，openGauss 的性能低于 PostgreSQL，但测试过程中速度的稳定性较好。

同时，相比于 PostgreSQL，openGauss 将 AI 融合进数据库中，形成了其独特优势，但 AI4DB 部分目前疑似处于不可用状态。
