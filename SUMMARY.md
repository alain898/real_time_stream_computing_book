# 目录

* [前言](preface.md)
    * [实时流计算](rts_preface/real_time_computing.md)
    * [本书的目的](rts_preface/target.md)
    * [本书的组织](rts_preface/orgnization.md)
    * [更多的信息](rts_preface/more_info.md)
    * [致谢](rts_preface/thanks.md)

* [实时流计算介绍](rts_introduction/introduction.md)
    * [实时流计算使用场景](rts_introduction/use_case.md)
    * [实时流数据特点](rts_introduction/streaming_data_feature.md)
    * [实时流计算系统架构](rts_introduction/common_architecture.md)

* [数据采集](data_receiver/data_receiver.md)
    * [风控系统简介](data_receiver/risk_management_system.md)
    * [Spring Boot](data_receiver/spring_boot.md)
    * [BIO和NIO](data_receiver/bio_and_nio.md)
    * [NIO和异步](data_receiver/nio_and_async.md)
    * [Netty](data_receiver/netty.md)
    * [流和异步的关系](data_receiver/stream_and_async.md)
    
* [单节点流计算框架](rts_impl_single_node_rts_app/rts_impl_node_rts_application.md)
    * [构造自己的流计算框架](rts_impl_single_node_rts_app/implement_a_single_node_rts.md)
    * [CompletableFuture原理](rts_impl_single_node_rts_app/mechanism_of_completable_future.md)
    * [基于CompletableFuture实现流计算框架](rts_impl_single_node_rts_app/implement_using_completable_future.md)
    * [流计算的性能调优](rts_impl_single_node_rts_app/performance_optimized.md)

* [数据处理](rts_computing/computing.md)
    * [数据操作](rts_computing/data_operation.md)
    * [指标统计之时间序列](rts_computing/statistics_on_time_sequence.md)
    * [指标统计之关联图谱](rts_computing/graph.md)
    * [模式匹配](rts_computing/cep.md)
    * [模型学习和预测](rts_computing/model_train_and_prediction.md)

* [流的状态管理](rts_from_single_node_to_cluster/state_management_of_streaming_data.md)
    * [流的状态](rts_from_single_node_to_cluster/state_of_streaming_data.md)
    * [采用redis](rts_from_single_node_to_cluster/use_redis_design.md)
    * [采用ignite](rts_from_single_node_to_cluster/use_ignite_design.md)
    * [扩展为集群](rts_from_single_node_to_cluster/expand_to_cluster.md)

* [开源分布式流计算框架](rts_opensource/opensource.md)
    * [Apache Storm](rts_opensource/storm.md)
    * [Spark Streaming](rts_opensource/spark_streaming.md)
    * [Apache Flink](rts_opensource/flink.md)
    * [如何面对琳琅满目的流计算框架](rts_opensource/how_to_kinds_of_treat_rts_opensource_framework.md)

* [当做不到实时时](rts_real_time_unavailable/real_time_unavailable.md)
    * [做不到实时的原因](rts_real_time_unavailable/why_real_time_unavailable.md)
    * [Lambda架构](rts_real_time_unavailable/use_lambda.md)

* [数据传输](rts_data_transfer/data_transfer.md)
    * [消息中间件](rts_data_transfer/message_oriented_middleware.md)
    * [Kafka](rts_data_transfer/kafka.md)

* [数据存储](rts_data_storage/data_storage.md)
    * [存储的设计应该根据应用类型选择](rts_data_storage/chose_storage_type_by_usage_case.md)
    * [点查询](rts_data_storage/point_query.md)
    * [adhoc查询](rts_data_storage/adhoc_query.md)
    * [离线分析](rts_data_storage/offline_analysis.md)

* [服务治理和配置管理](rts_config_management/service_and_config_management.md)
    * [服务治理](rts_config_management/service_orgnization.md)
    * [面向配置编程](rts_config_management/config_oriented_programming.md)
    * [动态配置](rts_config_management/dynamic_config.md)
    * [将前端配置与后端服务配置隔离开](rts_config_management/separate_ui_config_and_service_config.md)

