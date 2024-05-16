---
title: openlineage学习（一）
tags:
  - openlineage
  - design
  - data lineage
date: 2024-05-16 14:05:10
---

# OpenLineage学习（一）

# 简介

OpenLineage是一个数据血缘收集和分析的开放框架，其核心是一个系统之间用于交换血缘数据的可扩展的规范。在核心之外，OpenLineage提供了Spark、Flink、Airflow等常用的计算引擎、调度组件的集成实现。

OpenLineage只规范的数据血缘信息交换的格式，未对数据血缘的存储和查询进行定义，因此需要配合一个数据血缘存储和查询的实现，如 **[Marquez](https://github.com/MarquezProject/marquez)** 。Marquez是一个开源的元数据收集、聚合和可视化服务，是OpenLineage规范的一个实现。

# 设计

## Run State Update（执行信息）

OpenLineage通过作业的执行信息（Run State Update）来收集和表达数据以及血缘关系数据，一个执行信息包含数据集-任务-作业三个方面，如下图所示：

![OpenLineage数据血缘模型](object-model.svg)

为了能够准确的识别唯一的数据集、任务、作业信息，OpenLineage通过id进行标识，各个标识均有自己的生成方式和规则，具体请查看对应章节。对于一个系统来说，id的生成规则需要在多个系统内保持一致，或由一个统一的元数据管理系统提供，以确保在从计算引擎、存储组件等采集数据血缘的过程中，能够的到统一的标识。

Run State Update是OpenLineage中收集信息的载体，可以在一个作业执行的多个阶段产生，OpenLineage中定义了6个阶段，分别为：
* START： 表明一个作业的开始
* RUNNING： 表示一个正在进行中的作业，一般用于流式作业
* COMPLETE： 表明一个作业完成，一般用于批处理作业
* ABORT：表明一个因异常而退出
* FAILED： 表明一个作业失败
* OTHER： 表明其它为定义的情况

各个阶段的转换关系如下：

![Run State Update阶段转换](run-life-cycle.svg)

这是一个通用的阶段转换关系，在实际过程中，需要根据批/流作业的类型以及调度系统的功能特点，来确定具体使用哪些阶段。

## Dataset（数据集）

Dataset是OpenLineage中数据集的抽象，可用于多种数据结构，例如：数据库中的表、对象存储中的桶、文件系统中的路径等。分为inputs和outputs两个类型。

每一个Dataset都会根据其组件定义的命名规则生成唯一的名称来标识这个Dataset；名称包含Namespace、Name两个部分（如何根据Namespace、Name生成唯一表示的规则可以由实现系统自身设计）；例如：

|组件名称|Namespace|Name|Format|
|---|---|---|---|
|MySQL|Host,Port|Database,Table|mysql://{host}:{port}/{database}.{table}
|Hive|Host,Port|Database,Table|hive://{host}:{port}/{database}/{table}
|Kafka|Host,Port|Topic|kafka://{host}:{port}/{topic}

## Job（任务）

Job是一个消费或产生Dataset的处理过程。

Job是一个抽象，在不用的运行环境下可以指代不同的内容。例如：在调度系统中可以指代一个workflow，在集成系统中可以指代一个集成任务，在数据共享服务中可以指代一个数据接口等。

每一个Job由一个包含Namespace的唯一名称，Namespace通常指代Job所处的运行环境，其后会携带在运行环境中的唯一名称。例如：

|运行环境|格式|示例|
|---|---|---|
|Airflow|namespace + DAG + task|airflow-staging.orders_etl.count_orders|
|SQL|namespace + name|mysql-instance.aggregate_orders|

## Run（作业）

Run是Job的执行实例，作业的id使用uuid进行表示以确保唯一性。在数据血缘采集的过程中，需要确保同一个Run在不同状态之间变换时的id一致。

## 数据格式

Run State Update以及其子项Job、Run、Dataset的input和output的具体定义可以查看官方文档，链接：[OpenLineage Facts](https://openlineage.io/docs/spec/facets/)

OpenLineage使用json格式描述元数据和血缘关系数据，使用json-schema定义所包含的信息，具体请查看[OpenLineage JSON Schema](https://github.com/OpenLineage/OpenLineage/blob/main/spec/OpenLineage.json)。