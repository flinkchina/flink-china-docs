---
title: "数据源和接收器的容错保证"
nav-title: 容错保障
nav-parent_id: connectors
nav-pos: 0
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

Flink的容错机制在出现故障时恢复程序并继续执行它们。这些故障包括机器硬件故障、网络故障、暂态程序故障等。

Flink只在源参与快照机制时才能保证对用户定义的状态进行一次精确的exactly-once状态更新。下表列出了与绑定连接器耦合的Flink的状态更新保证。

请阅读每个连接器的文档，以了解容错保证的细节。

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 25%">Source</th>
      <th class="text-left" style="width: 25%">Guarantees</th>
      <th class="text-left">Notes</th>
    </tr>
   </thead>
   <tbody>
        <tr>
            <td>Apache Kafka</td>
            <td>exactly once</td>
            <td>Use the appropriate Kafka connector for your version</td>
        </tr>
        <tr>
            <td>AWS Kinesis Streams</td>
            <td>exactly once</td>
            <td></td>
        </tr>
        <tr>
            <td>RabbitMQ</td>
            <td>at most once (v 0.10) / exactly once (v 1.0) </td>
            <td></td>
        </tr>
        <tr>
            <td>Twitter Streaming API</td>
            <td>at most once</td>
            <td></td>
        </tr>
        <tr>
            <td>Collections</td>
            <td>exactly once</td>
            <td></td>
        </tr>
        <tr>
            <td>Files</td>
            <td>exactly once</td>
            <td></td>
        </tr>
        <tr>
            <td>Sockets</td>
            <td>at most once</td>
            <td></td>
        </tr>
  </tbody>
</table>

为了保证端到端精确一次的记录传递(除了精确一次exactly-once的状态语义)，数据接收器需要参与检查点机制。下表列出了与绑定接收器耦合的Flink的交付保证(假设只进行一次状态更新exactly-once):

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 25%">Sink</th>
      <th class="text-left" style="width: 25%">Guarantees</th>
      <th class="text-left">Notes</th>
    </tr>
  </thead>
  <tbody>
    <tr>
        <td>HDFS rolling sink</td>
        <td>exactly once</td>
        <td>Implementation depends on Hadoop version</td>
    </tr>
    <tr>
        <td>Elasticsearch</td>
        <td>at least once</td>
        <td></td>
    </tr>
    <tr>
        <td>Kafka producer</td>
        <td>at least once</td>
        <td></td>
    </tr>
    <tr>
        <td>Cassandra sink</td>
        <td>at least once / exactly once</td>
        <td>exactly once only for idempotent updates</td>
    </tr>
    <tr>
        <td>AWS Kinesis Streams</td>
        <td>at least once</td>
        <td></td>
    </tr>
    <tr>
        <td>File sinks</td>
        <td>at least once</td>
        <td></td>
    </tr>
    <tr>
        <td>Socket sinks</td>
        <td>at least once</td>
        <td></td>
    </tr>
    <tr>
        <td>Standard output</td>
        <td>at least once</td>
        <td></td>
    </tr>
    <tr>
        <td>Redis sink</td>
        <td>at least once</td>
        <td></td>
    </tr>
  </tbody>
</table>

{% top %}
