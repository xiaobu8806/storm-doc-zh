---
title: CGroup Enforcement
layout: documentation
documentation: true
---

# CGroups in Storm

Storm 使用 CGroup 来限制 worker 的资源使用, 以保证公平和 QOS.

**请注意：CGroups 目前仅支持 Linux 平台（内核版本 2.6.24 及更高版本）**

## 设置

要使用 CGroups, 请确保正确安装 cgroups 并配置 cgroup.有关设置和配置的更多信息, 请访问:

https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Resource_Management_Guide/ch-Using_Control_Groups.html

一个 示例/默认 的 cgconfig.conf 文件 <stormroot>/conf 目录.
内容如下:

```
mount {
	cpuset	= /cgroup/cpuset;
	cpu	= /cgroup/storm_resources;
	cpuacct	= /cgroup/cpuacct;
	memory	= /cgroup/storm_resources;
	devices	= /cgroup/devices;
	freezer	= /cgroup/freezer;
	net_cls	= /cgroup/net_cls;
	blkio	= /cgroup/blkio;
}

group storm {
       perm {
               task {
                      uid = 500;
                      gid = 500;
               }
               admin {
                      uid = 500;
                      gid = 500;
               }
       }
       cpu {
       }
}
```

有关 cgconfig.conf 文件的格式和配置的更详细说明, 请访问:

https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Resource_Management_Guide/ch-Using_Control_Groups.html#The_cgconfig.conf_File

# 与 Storm 中的 CGroup 相关的设置

| 设置 | 功能 |
| --- | --- |
| storm.cgroup.enable                | 此配置用于设置是否使用 cgroup.设置 "true" 以启用 cgroups 的使用.设置 "false" 不使用 cgroups.当该配置设置为 false 时, 将跳过与 cgroup 相关的单元测试.默认设置为 "false" |
| storm.cgroup.hierarchy.dir   | 到风暴将使用的 cgroup 层次结构的路径.默认设置为  "/cgroup/storm_resources" |
| storm.cgroup.resources       | 将由 CGroups 监管的子系统列表.默认设置为 cpu 和内存.目前只支持 cpu 和内存 |
| storm.supervisor.cgroup.rootdir     | supervisor 使用的根 cgroup.cgroup 的路径将是 \<storm.cgroup.hierarchy.dir>/\<storm.supervisor.cgroup.rootdir>.默认设置为 "storm" |
| storm.cgroup.cgexec.cmd            | 用于在 cgroup 中启动工作的 cgexec 命令的绝对路径.默认设置为 "/bin/cgexec" |
| storm.worker.cgroup.memory.mb.limit | 每个 worker 的内存限制为MB.这可以基于每个 supervisor 节点设置.该配置用于设置 cgroup config memory.limit_in_bytes.有关 memory.limit_in_bytes 的更多详细信息, 请访问：https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Resource_Management_Guide/sec-memory.html.请注意, 如果您使用资源感知计划程序, 请不要设置此配置, 因为此配置将覆盖资源意识计划程序计算的值 |
| storm.worker.cgroup.cpu.limit       | 每个 worker 的cpu份额.这可以基于每个 supervisor 节点设置.此配置用于设置 cgroup config cpu.share.有关 cpu.share 的更多详细信息, 请访问：https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Resource_Management_Guide/sec-cpu.html.请注意, 如果您使用资源感知调度程序, 请不要设置此配置, 因为此配置将覆盖资源意识计划程序计算的值 |

由于通过 cpu.shares 限制 CPU 使用率仅限制进程的 CPU 占用比例, 以限制 supervisor 节点上所有 worker 进程的CPU使用量, 请设置config supervisor.cpu.capacity.
其中每个增量代表核心的1％, 因此如果用户设置 supervisor.cpu.capacity: 200, 则用户指示使用2个内核.

## 与资源意识调度程序集成

CGroup 可以与资源意识调度程序一起使用.
然后, CGroup 将强制资源意识调度程序分配的 worker 的资源使用情况.
要将资源感知计划程序使用 cgroup, 只需启用 cgroups, 并确保不要设置 storm.worker.cgroup.memory.mb.limit 和 storm.worker.cgroup.cpu.limit 配置.
