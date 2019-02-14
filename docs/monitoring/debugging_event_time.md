---
title: "调试Windows & Event Time"
nav-parent_id: monitoring
nav-pos: 13
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

* ToC
{:toc}

## Monitoring Current Event Time 监控当前事件时间

Flink的[event time事件时间]({{ site.baseurl }}/dev/event_time.html)和watermark水印支持是处理无序事件的强大特性。然而，由于时间的进度是在系统中跟踪的，因此很难准确地理解发生了什么。

每个任务的低水印可以通过Flink web接口或[metrics system]({{ site.baseurl }}/monitoring/metrics.html).。
可以通过Flink Web界面或[metrics systems]({{ site.baseurl }}/monitoring/metrics.html)访问每个任务的Low watermarks(低水印???)

Flink中的每个任务都公开一个名为`currentInputWatermark`的度量，它表示该任务接收到的最低水印(lowest watermark )。这个长值表示"current event time 当前事件时间"。
该值是通过取上游运营商接收到的所有水印的最小值来计算的。这意味着使用水印跟踪的事件时间总是由最后面(furthest-behind)的源控制。

使用**using the web interface**，通过在度量标签选项中选择任务，然后选择`<taskNr>.currentInputWatermark`度量，可以访问低水印度量标准。 在新框中，您现在可以看到任务的当前低水印。


获取指标的另一种方法是使用一个**metric reporters 指标报告器**，如[metrics system]({{ site.baseurl }}/monitoring/metrics.html)。
对于本地设置，我们建议使用JMX度量报告器和[VisualVM](https://visualvm.github.io/)这样的工具。


## Handling Event Time Stragglers 处理事件时间延迟

  - 方法1: Watermark stays late (indicated completeness), windows fire early 水印保持延迟（指示完整性），窗口提前启动。


  - 方法2: Watermark heuristic with maximum lateness, windows accept late data
  水印启发式最大延迟，Windows接受延迟数据



{% top %}
