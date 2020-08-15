<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [如何编译构建 TiDB 集群，并运行起最小示例](#如何编译构建-tidb-集群并运行起最小示例)
  - [步骤拆解](#步骤拆解)
  - [执行](#执行)
    - [环境 Hello World](#环境-hello-world)
    - [编译组件](#编译组件)
      - [PD server 编译](#pd-server-编译)
      - [TiKV server 编译](#tikv-server-编译)
      - [TiDB server 编译](#tidb-server-编译)
    - [修改事务声明代码](#修改事务声明代码)
  - [后验总结](#后验总结)
    - [分布式事务相关](#分布式事务相关)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# 如何编译构建 TiDB 集群，并运行起最小示例

任务话题很有趣，作为一个有着一定开发经验的开发人员，如何在一次都没玩过 TiDB 的基础上能够快速编译一个 TiDB 集群，并使得 TiDB 运行事务时输出一个 "hello transaction" 的日志.


## 步骤拆解
TiDB 的生态和文档建设已经十分丰富，相信像是部署这种文档是肯定完备的。秉承着从简单开始的原则，我们的第一步是在单机搭建起一个 TiDB 集群。

从介绍材料内我们可以了解跑起一个最简单的 TiDB 集群需要三个组件：TiDB，PD 和 TiKV，那么从用户角度来讲，部署文档肯定会位于最上层组件，则去 TiDB 的 git 内找寻文档效率应该是最高的。

从任务描述来看，要完成我们做的事情：
1. 编译 tidb，tikv，pd 模块
2. 跑起三个模块，排除过程中遇到的困难，熟悉三个模块各自在工作时候的日志模式，以及观察 CPU，内存等占用情况
3. 进一步搞清楚什么叫 "TiDB 运行事务时"，最直白的理解，需要起一个事务 sql 语句请求 tidb，观察各个组件日志情况
4. 在请求入口处插入日志列即可

我们还拥有一个利器：[问答社区](https://asktug.com/c/developer/developer-qa)

## 执行
### 环境 Hello World
在 TiDB 内，我们从首页被引导到 [quick start](https://docs.pingcap.com/zh/tidb/stable/quick-start-with-tidb)，可以在其中发现 TiUP 这个工具可以帮助我们快速体验日志。

在快速启动中，引导只有 TiUP 和容器部署两种工具，没有手动部署，这点不利于我们自己动手，因为 TiUP 一看就是下载已经编译好的东西，第一步我们可以尝试，使用 TiUP 部署后，观察启动的进程位置等情况，来决定下一步能替换哪些东西可以快速的自己替换库的版本。

根据官方说明，使用 TiUP 启动环境后，会有一个后台进程显示启动成功，以及对应的监控地址，无奈的是一个是 grafana 监控和 dashboard 监控全都因为密码问题无法登陆，可以回头参与 tidb 内优化一下。

使用 htop 观察进程，可以看到如下的进程结构，得到想要的信息
```shell
/Users/liuyi/.tiup/components/pd/v4.0.4/pd-server --name=pd-0 --data-dir=/Users/liuyi/.tiup/data/S7SS4aM/pd-0/data --peer-urls=http://127.0.0.1:2380 --advertise-peer-urls=http://127.0.0.1:2380 --client-urls=http://127.0.0.1:2379 --advertise-client-urls=http://127.0.0.1:2379 --log-file=/Users/liuyi/.tiup/data/S7SS4aM/pd-0/pd.log --initial-cluster=pd-0=http://127.0.0.1:2380
```
从官方的 tiup 启动的环境有 pd-server, grafana-server 和 prometheus，似乎并没有我们想要的 tidb 和 tikv 组件。使用 mysql 连接不通也能印证我们的思路。
```
mysql --host 127.0.0.1 --port 4000 -u root
ERROR 2003 (HY000): Can't connect to MySQL server on '127.0.0.1' (61)
```
我们从进程信息了解到所有的数据都被放在了 ~/.tiup 下，所以可以去目录观察一下，从 components 目录下可以找到组件，观察后发现启动不了的原因：v4.0.4

换用 v4.0.0 的镜像后所有服务正常启动，花一点心思就可以找到对应的目录位置：本机的 data 目录，或者使用 [Dashboard Search Logs](http://127.0.0.1:2379/dashboard/#/search_logs/detail/1)

我们期望在 tidb 层加入日志，则可以多注意观察 tidb 的日志和 tidb 编译目录内容.
通过观察日志，我们在整个 .tiup 搜索 tidb 的配置项，发现没有匹配位置，所以我们可以推测，对应的配置是由 tiup **自动生成**的，如果需要配置，也是由 tiup 参数等控制了，从这里了解不到更多编译的细节，但是我们得到了一个能快速启动的脚本，以及知道了所有组件的二进制项目位置，那么我们就可以自己编译然后替换二进制了，顺便熟悉一下 tiup 的[文档](https://docs.pingcap.com/zh/tidb/stable/tiup-documentation-guide)。

快速启动 tidb 集群的命令行
```
tiup playground v4.0.0 --db 1 --pd 1 --kv 3 --monitor
```

### 编译组件

组件启动顺序为 pd->tikv->tidb, 非常好理解, 先启动 manager 组件掌控全局, 再启动实际存储把工人准备好, 最后启动 sql server 开始接客.

所以我们可以按照顺序替换自己的编译版本, 每次都用 tiup 重启集群确认情况.
为了学习的稳定，我们使用所有的分支为 4.0.0 的 tag 拉出来的分支.

#### PD server 编译
下载 pd 源代码后即可正常编译，执行命令。
```
git branch -b 4.0 origin/release-4.0
git checkout 4.0
make -j8
```

#### TiKV server 编译
下载 tikv 源代码后正常编译，执行命令。
```
git checkout tags/v4.0.0 -b 4.0
make -j8
```
注意因为 tikv 目录下有一个 rust-toolchain 文件，所以第一次执行的时候会使用指定 rust 版本，因为一般情况下机器上没有安装，所以会先安装一次，但是因为日志被接管，所以现象是 make -j8 一段时间会一直没有任何输出，耐心等待第一次就好。
细节可以参考 rustup 的文档关于 [rust-toolchain](https://github.com/rust-lang/rustup#the-toolchain-file) 文件的相关内容。

#### TiDB server 编译
tidb 的文档首页没有直接编译的引导，需要自己扒拉一下 docs 目录找一下资料, 在[快速开始](https://github.com/pingcap/tidb/blob/master/docs/QUICKSTART.md) 内可以找到编译的指令.。
在我的机器上遇到的问题:
```
# command-line-arguments
flag provided but not defined: -L/usr/local/opt/llvm/lib
usage: link [options] main.o
CGO_ENABLED=1 GO111MODULE=on go build  -tags codes  -ldflags '-L/usr/local/opt/llvm/lib -X "github.com/pingcap/parser/mysql.TiDBReleaseVersion=v4.0.0-beta.2-948-g148f2d456" -X "github.com/pingcap/tidb/util/versioninfo.TiDBBuildTS=2020-08-15 01:43:52" -X "github.com/pingcap/tidb/util/versioninfo.TiDBGitHash=148f2d456bdc531bb212028f8499ece141e20401" -X "github.com/pingcap/tidb/util/versioninfo.TiDBGitBranch=master" -X "github.com/pingcap/tidb/util/versioninfo.TiDBEdition=Community" ' -o bin/tidb-server tidb-server/main.go
```

简单确认了 Makefile 内没有加入这个 flags，后确认是我原来的配置自带了，删除这个配置即可正确编译。

执行编译命令
```
git checkout tags/v4.0.0 -b 4.0
make -j8
```

### 修改事务声明代码
从理论上分析，TiKV 本身是一个分布式一致性 db，并且有 MVCC 和事务抽象，则向上应该提供了
简单的事务提交接口供无状态的 sql server 调用，我们只需要找到那个位置即可，在这一步应该忽略具体事务的实现，先专注找到调用事务的位置，集中位置应该是 tidb 项目目录内，同时我们也应该意识到，我们每次为了验证维护稳定，只需要 tiup 内替换 tidb-server 即可。


## 后验总结

### 分布式事务相关
在分析修改代码位置的时候，我们不可避免的要从逻辑上想明白要改的代码位于什么位置，并针对性的搜索材料，因为 tidb 的文档和社区教程做的很不错，我们可以找到很多文档辅助我们更快完成工作，罗列一下在执行过程中参考的资料

[总体架构概述](https://pingcap.com/blog-cn/how-do-we-build-tidb/)

[计算模型概述](https://pingcap.com/blog-cn/tidb-internal-2/)
