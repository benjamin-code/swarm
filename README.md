# swarm

#Swarm 简介
Docker 自诞生以来，其容器特性以及镜像特性给 DevOps 爱好者带来了诸多方便。然而在很长的一段时间内，Docker 只能在单 host 上运 行，其跨 host 的部署、运行与管理能力颇受外界诟病。跨 host 能力的薄弱，直接导致 Docker 容器与 host 的紧耦合，这种情况下，Docker 容器的灵活性很难令人满意，容器的迁移、分组等都成为很难实现的功能点。
Swarm 是 Docker 公司在 2014 年 12 月初新发布的容器管理工具。和 Swarm 一起发布的 Docker 管理工具还有 Machine 以及 Compose。
Swarm 是一套较为简单的工具，它可以用来管理 Docker 集群。通过 Swarm 的管理，对外完全可以把这个集群看成一个单一的物理机或者虚拟机。同时，Swarm 使用标准的 Docker API 接口作为其访问入口。换言之，您可以以标准的 Docker 命令来管理您的集群，比如 Docker ps 等等。Swarm 几乎全部用 Go 语言来完成开发，目前的版本是 0.30。从 Github 上可以看到，Swarm 的发展非常迅速，几乎每隔几天就有代码的更新。功能和特性的变更迭代还非常频繁。根据 Docker 官方文档来看，目前处于 beta 版本，一切都可能会变化，所以 Docker 官方不推荐用在生产环境中（Note: Swarm is currently in beta, so things are likely to change. We don't recommend you use it in production yet. ）。
Swarm 从设计之初就和 Docker 的其它项目一样，遵循"batteries included but removable"原则。也就是说，Swarm 能够通过简单的部署就能让用户很方便的来管理 Docker 集群。同时，它也是一个可插拔的模块，用户在任何时候去掉这个模块并不会影响已有 Docker 容器的使用。

#Swarm 架构介绍
Swarm 是用来被用来管理 Docker 集群的，所以单个 Docker host 是整个集群的基础。Swarm 自身可以有两种安装方式，一种是当成普通的 Docker 容器来安装，一种是当成一个简单的应用被安装在一台虚拟机或者物理机上。
所有的 Docker node 都会被当成一个调度候选对象。类似于 OpenStack 中的 compute node.

#Swarm 调度策略
Swarm 目前有多种调度机制来对目前的 Docker host 做排序。用户可以选择一种策略来决定 Swarm 节点的排序。当您运行一个新的容器的时候，Swarm 通过用户的指定的策略，经过计算后会选择最佳的节点，最终将容器放到这个节点上。目前 Swarm 支持调度策略包括：Spread、binpack、random,如果用户不指定策略，Swarm 会指定一个默认的策略：spread 来调度计算资源。
Spread、binpack 策略均会根据当前 Cluster 中的所有 host 的 CPU、RAM,以及容器的数量来决定最终把用户新的 container 放在哪个节点上。Random 策略比较简单，它不做任何计算，不关心 CPU 和 RAM。仅仅是随机的来选择一个节点来将新的 container 放到节点上。所以 random 策略主要被用来做 debug，简单来说，它不太适合生产环境。
Spread 和 binpack 也是有一定区别的。spread 策略用最少运行中的容器数量来决定是否将容器放在该节点上。Binpack 策略则刚好相反。所以 spread 策略和 binpack 各有优势。当 Cluster 中的一个节点出现了异常，那么使用 spread 策略能使您仅仅丢失少量的容器。Binpack 的优势在于，它能使比较大的容器能放在没有使用的节点上。这样的好处在于能有效避免碎片化。

#服务发现
当节点越来越多的时候，服务发现就显得尤为重要。简单点说，Swarm 作为控制器来管理整个 Cluster，它需要有自己的发现节点策略。目前说来，Swarm 的发现策略比较多。
第一种策略，我把它称之为节点主动注册策略。Swarm 创建好一个 Cluster 后，会返回一个 Cluster id，其它的 Swarm 节点都可以主动以命令行的形式来加入该 Cluster，在实战部分我会以这种策略来做详细的操作。
第二种策略，我把它称之为静态文件描述策略。在所有的 Swarm 节点上做一个配置文件。配置文件很简单，如例：
<pre>
	$ echo node_ip1:2375 >> /tmp/my_cluster.
</pre>
所有的节点做相同的操作。最后在 Swarm 上运行如下命令即可以来管理整个 Cluster。如例：
<pre>
	$ swarm manage -H tcp://<swarm_ip:swarm_port> file:///tmp/my_cluster
</pre>
第三种策略是静态 IP 列表策略。直接在 Swarm 上运行如下命令即可
<pre>
	$ swarm manage -H <swarm_ip:swarm_port> <node_ip1:2375>,<node_ip2:2375>
</pre>
第四种策略，我把它称之为匹配 IP 化静态文件描述策略。它其实是第二种策略的升级版。只需要描述清楚 Swarm 节点的 IP 范围，就可以直接管理该 Cluster。 其它策略包括 etcd、consul、zookeeper,均是依赖第三方工具来实现。我把它归为一类。他们的具体使用也比较简单，请参考 https://docs.Docker.com/Swarm/discovery/
用户同样可以自定义自己的发现策略。您只需要实现以下接口即可：
<pre>
	type DiscoveryService interface {
	Initialize(string, int) error
	Fetch() ([]string, error)
	Watch(WatchCallback)
	Register(string) error
	}
</pre>
Swarm 同样支持 TLS，这能增强整个 Cluster 的安全性，限于篇幅，本文不做介绍，感兴趣的朋友可以参考 Swarm 官方文档。

#Swarm 实战
本节我会以一个实际的例子来给大家展示如何构建一个完整的基于 Swarm 的 Cluster 环境。
实战准备：一台 Swarm 虚拟机，2 台 Swarm node。Swarm 虚拟机主要用来控制整个 Cluster，这在前面已经讲过。2 台 node 分别代表用来做计算资源。由于资源有限，我直接将某一台机器既当做 Swarm node，也当作 Swarm 控制器。注意，虚拟机内核最好是 3.10 以上版本，这个版本以上的内核对于 Docker 来说运行比较稳定。同时，我以第一种策略来管理整个 Cluster，即节点主动注册策略。
两台机器的 ip 分别为 9.114.170.173 和 9.114.170.183。
### 第一步，分别在每个 Swarm node 上安装 Docker。这一步比较简单，我就不再赘述。
### 第二步，在 9.114.170.173 上安装 Swarm。先启动 Docker，以如下方式启动：
<pre>
	[root@c582f1-n36-vm3 ~]# docker -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock -d&
	[root@c582f1-n36-vm3 ~]# docker pull swarm
</pre>
### 第三步，启动 Swarm，并创建 Cluster id。
<pre>
	[root@c582f1-n36-vm3 ~]# docker run --rm swarm create
	2a826df6282e2ef24eaa1d1403b022fa
</pre>
上面那一串字符串是您的 Cluster id，再接下来的过程中您将会用到。
### 第四步，将各个 Swarm node 添加到 Swarm Cluster 里面来。将 9.114.170.173 加入到 Cluster 中：
<pre>
	[root@c582f1-n36-vm3 ~]# docker run -d swarm join --addr=9.114.170.173:2375 
	token://2a826df6282e2ef24eaa1d1403b022fa
</pre>
将 9.114.170.183 加入到 Cluster 中。
<pre>
	[root@c582f1-n34-vm3 ~]# docker run -d swarm join --addr=9.114.170.183:2375 
	token://2a826df6282e2ef24eaa1d1403b022fa
</pre>
### 第五步，启动 Swarm 管理器，在 9.114.170.173 中做如下操作：
<pre>
	[root@c582f1-n36-vm3 ~]# docker run -d -p 8888:2375 swarm manage 
	token://2a826df6282e2ef24eaa1d1403b022fa
</pre>
经过以上步骤，整个 Cluster 就会正常的运转起来。我们可以在 Swarm 控制器即 9.114.170.173 上来查看一下现在的 Swarm node:
<pre>
	[root@c582f1-n36-vm3 ~]# docker -H tcp://9.114.170.173:8888 info
</pre>
至此，您可以像操作单个 Docker 主机一样来操作 Swarm 了。比如 docker ps、docker ps -a、docker images
