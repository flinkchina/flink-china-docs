---
title: "JobManager 高可用(HA)"
nav-title: JobManager 高可用(HA)
nav-parent_id: ops
nav-pos: 2
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

JobManager协调每一次Flink部署。 它负责*调度*和*资源管理*。

默认情况下，每个Flink群集都有一个JobManager实例。 这会产生*单点故障single point of failure*（SPOF）：如果JobManager崩溃，则无法提交新程序并且运行程序失败。

使用JobManager高可用性，您可以从JobManager故障中恢复，从而消除*SPOF *。 您可以为**standalone独立**和**YARN clusters**配置高可用性。


* Toc
{:toc}

## Standalone Cluster HA

对于独立集群来说，JobManager高可用性的一般思想是，在任何时候都有一个**单领导JobManager**和**多个备用JobManager**来接管领导层主节点，以防leader失败。这就保证了**没有单点故障**，并且只要有一个备用JobManager领导，程序就可以取得进展。备用和主JobManager实例之间没有明确的区别。 每个JobManager都可以充当主服务器或备用服务器。



例如，考虑以下三个JobManager实例的设置：

<img src="{{ site.baseurl }}/fig/jobmanager_ha_overview.png" class="center" />

### 配置

要启用JobManager高可用性，您必须将**高可用性模式 high-availability mode**设置为*zookeeper*，配置**ZooKeeper quorum**并设置**masters file主文件**以及所有JobManagers主机及其Web UI端口。

Flink在所有正在运行的JobManager实例之间利用**[ZooKeeper](http://zookeeper.apache.org)**进行分布式协调*。 ZooKeeper是Flink的独立服务，通过领导者选举和轻量级一致状态存储提供高度可靠的分布式协调。 有关ZooKeeper的更多信息，请查看[ZooKeeper的入门指南](http://zookeeper.apache.org/doc/current/zookeeperStarted.html)。 Flink包含[引导简单的ZooKeeper bootstrap a simple ZooKeeper](#bootstrap-zookeeper)安装的脚本。

#### Masters文件(masters)

要启动HA-cluster，请在`conf/masters`中配置*masters*文件：

- **masters file**: *masters文件*包含启动JobManagers的所有主机以及Web用户界面绑定的端口。

  <pre>
jobManagerAddress1:webUIPort1
[...]
jobManagerAddressX:webUIPortX
  </pre>

默认情况下，作业管理器将为进程间通信选择*随机端口*。 您可以通过**`high-availability.jobmanager.port`**键更改此设置。 该key接受单个端口（例如`50010`），范围（`50000-50025`）或两者的组合（`50010,50011,50020-50025,50050-50075`）。

#### Config文件 (flink-conf.yaml)

要启动HA群集，请将以下配置键添加到`conf/flink-conf.yaml`：

- **high-availability mode** (必须的): *高可用性模式*必须在`conf/flink-conf.yaml`中设置为*zookeeper*才能启用高可用性模式。或者，此选项可以设置为Flink应用于创建HighAvailabilityServices实例的工厂类的FQN。

  <pre>high-availability: zookeeper</pre>

- **ZooKeeper quorum** (必须的): *ZooKeeper quorum*是ZooKeeper服务器的复制组，它提供分布式协调服务。


  <pre>high-availability.zookeeper.quorum: address1:2181[,...],addressX:2181</pre>

 每个*addressX:port*指的是一个ZooKeeper服务器，Flink可以在给定的地址和端口访问它。


- **ZooKeeper root** (建议的): *root ZooKeeper node*，在其下放置所有集群节点。

  <pre>high-availability.zookeeper.path.root: /flink

- **ZooKeeper cluster-id** (建议的): *cluster-id ZooKeeper node*，在该节点下放置集群的所有必需协调数据。


  <pre>high-availability.cluster-id: /default_ns # important: customize per cluster</pre>

  **重要的**: 在运行一个YARN集群、每个作业的纱线会话或在另一个集群管理器上运行时，不应手动设置此值。在这些情况下，将根据应用程序ID自动生成集群ID(cluster-id)。手动设置集群ID将覆盖yarn中的这一行为。反过来，使用-z cli选项指定集群ID将覆盖手动配置。如果您在裸机上运行多个Flink HA集群，则必须为每个集群手动配置单独的集群ID。

- **Storage directory** (必须的): JobManager元数据保存在文件系统*storagedir*中，只有一个指向该状态的指针存储在ZooKeeper中

    <pre>
high-availability.storageDir: hdfs:///flink/recovery
    </pre>

    `storageDir`存储恢复JobManager故障所需的所有元数据。

在配置了主服务器和ZooKeeper quorum之后，您可以像往常一样使用提供的集群启动脚本。 他们将启动HA群集。 请记住，当您调用脚本时，**ZooKeeper quorum必须运行**并确保为您正在启动的每个HA群集**配置单独的ZooKeeper根路径**。

#### 示例: 拥有两个JobManager的Standalone集群

1. **配置高可用模式和ZooKeeper quorum** in `conf/flink-conf.yaml`:

   <pre>
high-availability: zookeeper
high-availability.zookeeper.quorum: localhost:2181
high-availability.zookeeper.path.root: /flink
high-availability.cluster-id: /cluster_one # important: customize per cluster
high-availability.storageDir: hdfs:///flink/recovery</pre>

2. **配置masters** in `conf/masters`:

   <pre>
localhost:8081
localhost:8082</pre>

3. **配置ZooKeeper服务器** in `conf/zoo.cfg` (currently it's only possible to run a single ZooKeeper server per machine):

   <pre>server.0=localhost:2888:3888</pre>

4. **启动 ZooKeeper quorum**:

   <pre>
$ bin/start-zookeeper-quorum.sh
Starting zookeeper daemon on host localhost.</pre>

5. **启动HA-cluster集群**:

   <pre>
$ bin/start-cluster.sh
Starting HA cluster with 2 masters and 1 peers in ZooKeeper quorum.
Starting jobmanager daemon on host localhost.
Starting jobmanager daemon on host localhost.
Starting taskmanager daemon on host localhost.</pre>

6. **停止ZooKeeper quorum 和 flink集群**:

   <pre>
$ bin/stop-cluster.sh
Stopping taskmanager daemon (pid: 7647) on localhost.
Stopping jobmanager daemon (pid: 7495) on host localhost.
Stopping jobmanager daemon (pid: 7349) on host localhost.
$ bin/stop-zookeeper-quorum.sh
Stopping zookeeper daemon (pid: 7101) on host localhost.</pre>

## Flink on Yarn集群高可用

运行高可用性YARN集群时，**我们不运行多个JobManager（ApplicationMaster）实例**，而只运行一个实例，由YARN在失败时重新启动。 确切的行为取决于您使用的特定YARN版本。

### 配置

#### Application Master最大尝试次数 (yarn-site.xml)


您必须在`yarn-site.xml`中为**您的** YARN配置应用程序管理器(application masters)的最大尝试次数：

{% highlight xml %}
<property>
  <name>yarn.resourcemanager.am.max-attempts</name>
  <value>4</value>
  <description>
    The maximum number of application master execution attempts.
  </description>
</property>
{% endhighlight %}

当前YARN版本的默认值是2(这意味着可以容忍单个JobManager失败)。

#### Application尝试次数 (flink-conf.yaml)

除了HA配置([见上面](#configuration))之外，您还必须在`conf/flink-conf.yaml`中配置最大尝试次数:
<pre>yarn.application-attempts: 10</pre>

这意味着在YARN应用失败之前，应用程序可以重新启动9次失败的尝试（9次重试+1次初始尝试）。如果YARN操作运维需要，可以通过YARN执行额外的重新启动：抢占、节点硬件故障或重新启动，或节点管理器(NodeManager)重新同步。这些重启不计入`yarn.application-attempts`（yarn.application attempts），请参见<a href="http://johnjianfang.blogspot.de/2015/04/the number of maximum attempts of yarn.html">jian fang's blog post</a>。需要注意的是，`yarn.resourcemanager.am.max-attempts`是应用程序重新启动的上限。因此，在Flink中设置的应用尝试次数不能超过启动YARN的YARN集群设置。

#### Container Shutdown Behaviour

- **YARN 2.3.0 < version < 2.4.0**. All containers are restarted if the application master fails.
- **YARN 2.4.0 < version < 2.6.0**. TaskManager containers are kept alive across application master failures. This has the advantage that the startup time is faster and that the user does not have to wait for obtaining the container resources again.
- **YARN 2.6.0 <= version**: Sets the attempt failure validity interval to the Flinks' Akka timeout value. The attempt failure validity interval says that an application is only killed after the system has seen the maximum number of application attempts during one interval. This avoids that a long lasting job will deplete it's application attempts.

<p style="border-radius: 5px; padding: 5px" class="bg-danger"><b>Note</b>: Hadoop YARN 2.4.0 has a major bug (fixed in 2.5.0) preventing container restarts from a restarted Application Master/Job Manager container. See <a href="https://issues.apache.org/jira/browse/FLINK-4142">FLINK-4142</a> for details. We recommend using at least Hadoop 2.5.0 for high availability setups on YARN.</p>

#### 示例: Highly Available YARN Session

1. **Configure HA mode and ZooKeeper quorum** in `conf/flink-conf.yaml`:

   <pre>
high-availability: zookeeper
high-availability.zookeeper.quorum: localhost:2181
high-availability.storageDir: hdfs:///flink/recovery
high-availability.zookeeper.path.root: /flink
yarn.application-attempts: 10</pre>

3. **Configure ZooKeeper server** in `conf/zoo.cfg` (currently it's only possible to run a single ZooKeeper server per machine):

   <pre>server.0=localhost:2888:3888</pre>

4. **Start ZooKeeper quorum**:

   <pre>
$ bin/start-zookeeper-quorum.sh
Starting zookeeper daemon on host localhost.</pre>

5. **Start an HA-cluster**:

   <pre>
$ bin/yarn-session.sh -n 2</pre>

## Configuring for Zookeeper Security

If ZooKeeper is running in secure mode with Kerberos, you can override the following configurations in `flink-conf.yaml` as necessary:

<pre>
zookeeper.sasl.service-name: zookeeper     # default is "zookeeper". If the ZooKeeper quorum is configured
                                           # with a different service name then it can be supplied here.
zookeeper.sasl.login-context-name: Client  # default is "Client". The value needs to match one of the values
                                           # configured in "security.kerberos.login.contexts".
</pre>

For more information on Flink configuration for Kerberos security, please see [here]({{ site.baseurl}}/ops/config.html).
You can also find [here]({{ site.baseurl}}/ops/security-kerberos.html) further details on how Flink internally setups Kerberos-based security.

## Bootstrap ZooKeeper

If you don't have a running ZooKeeper installation, you can use the helper scripts, which ship with Flink.

There is a ZooKeeper configuration template in `conf/zoo.cfg`. You can configure the hosts to run ZooKeeper on with the `server.X` entries, where X is a unique ID of each server:

<pre>
server.X=addressX:peerPort:leaderPort
[...]
server.Y=addressY:peerPort:leaderPort
</pre>

The script `bin/start-zookeeper-quorum.sh` will start a ZooKeeper server on each of the configured hosts. The started processes start ZooKeeper servers via a Flink wrapper, which reads the configuration from `conf/zoo.cfg` and makes sure to set some required configuration values for convenience. In production setups, it is recommended to manage your own ZooKeeper installation.

{% top %}
