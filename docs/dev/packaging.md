---
title: "打包程序和分发执行"
nav-title: 打包程序
nav-parent_id: execution
nav-pos: 20
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


如前所述，Flink程序可以通过使用`远程环境`在集群上执行。或者，可以将程序打包到JAR文件(Java archive)中执行。打包程序是通过[命令行接口]({{ site.baseurl }}/ops/cli.html)。
### 打包程序

要支持通过命令行或web接口从打包JAR文件执行，程序必须使用`StreamExecutionEnvironment.getExecutionEnvironment()`获得的环境。当JAR被提交到命令行或web界面时，这个环境将充当集群的环境。如果Flink程序的调用方式与通过这些接口调用的方式不同，那么环境的行为将类似于本地环境。

要打包该程序，只需将所有涉及的类导出为JAR文件。JAR文件的清单必须指向包含程序的*入口点*的类(具有公共`main`方法的类)。最简单的方法是将*main-class*条目放入清单中(例如`main-class: org.apache.flinkexample.MyProgram`)。*main-class*属性与Java虚拟机在执行JAR时用于查找main方法的属性相同文件通过命令`java -jar pathToTheJarFile`。大多数IDE在导出JAR文件时自动包含该属性。


### 通过计划打包程序

此外，我们支持包装程序*计划*。 计划打包不是在main方法中定义程序并在环境中调用`execute（）`，而是返回* Program Plan *，它是程序数据流的描述。 为此，程序必须实现`org.apache.flink.api.common.Program`接口，定义`getPlan（String ...）`方法。 传递给该方法的字符串是命令行参数。 可以通过`ExecutionEnvironment#createProgramPlan()`方法从环境创建程序的计划。 打包程序的计划时，JAR清单必须指向实现`org.apache.flink.api.common.Program` 接口的类，而不是使用main方法的类。

### 总结

调用打包程序的总体过程如下:

1. JAR的清单将搜索一个*main-class*或*program-class*属性。如果找到这两个属性，则*program-class*属性优先于*main-class*属性。命令行和web接口都支持一个参数，以便在JAR清单不包含任何属性的情况下手动传递入口点类名。

2. 如果入口点类实现了`org.apache.flink.api.common.Program`，那么系统调用“getPlan(String…)”方法来获得要执行的程序计划。

3. 如果入口点类没有实现`org.apache.flink.api.common.Program` 接口，系统将调用该类的主方法。


{% top %}
