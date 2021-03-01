---
title: Activiti流程引擎学习
date: 2021-03-01 11:25:12
tags: 
    - Activiti
    - Java
    - BPMN
---

## 什么是BPMN（Business Process Model and Notation）

## 什么是Activiti

## 什么是Flowable

## 什么是Activiti Cloud

## 什么是Jeesite

## Activiti应用场景

## Activiti使用

### 数据库表设计

<img src="activiti.png" width="100%" height="100%">

|表名|用途|备注|
|:---|:---|:---|
|ACT_EVT_LOG|事件处理日志|-|
|ACT_GE_BYTEARRAY|二进制数据表|存储流程定义相关的部署信息。即流程定义文档的存放地。每部署一次就会增加两条记录，一条是关于BPMN规则文件的，一条是图片的（如果部署时只指定了BPMN一个文件，Activiti会在部署时解析BPMN文件内容自动生成流程图）。两个文件不是很大，都是以二进制形式存储在数据库中。|
|ACT_GE_PROPERTY|主键生成表|主张表将生成下次流程部署的主键ID。|
|ACT_HI_ACTINST|历史节点表|只记录usertask内容,某一次流程的执行一共经历了多少个活动|
|ACT_HI_ATTACHMENT|历史附件表|-|
|ACT_HI_COMMENT|历史意见表|-|
|ACT_HI_DETAIL|历史详情表，提供历史变量的查询|流程中产生的变量详细，包括控制流程流转的变量等|
|ACT_HI_IDENTITYLINK|历史流程人员表|-|
|ACT_HI_PROCINST|历史流程实例表|-|
|ACT_HI_TASKINST|历史任务实例表|一次流程的执行一共经历了多少个任务|
|ACT_HI_VARINST|历史变量表|-|
|ACT_PROCDEF_INFO||-|
|ACT_RE_DEPLOYMENT|部署信息表|存放流程定义的显示名和部署时间，每部署一次增加一条记录|
|ACT_RE_MODEL|流程设计模型部署表|流程设计器设计流程后，保存数据到该表|
|ACT_RE_PROCDEF|流程定义数据表|存放流程定义的属性信息，部署每个新的流程定义都会在这张表中增加一条记录。注意：当流程定义的key相同的情况下，使用的是版本升级|
|ACT_RU_DEADLETTER_JOB|-|-|
|ACT_RU_EVENT_SUBSCR|throwEvent，catchEvent时间监听信息表|-|
|ACT_RU_EXECUTION|运行时流程执行实例表|历史流程变量|
|ACT_RU_IDENTITYLINK|运行时流程人员表|主要存储任务节点与参与者的相关信息|
|ACT_RU_INTEGRATION|-|-|
|ACT_RU_JOB|运行时定时任务数据表|-|
|ACT_RU_SUSPENDED_JOB|-|-|
|ACT_RU_TASK|运行时任务节点表|-|
|ACT_RU_TIMER_JOB|-|-|
|ACT_RU_VARIABLE|运行时流程变量数据表|通过JavaBean设置的流程变量，在act_ru_variable中存储的类型为serializable，变量真正存储的地方在act_ge_bytearray中。|

## 总结

## 扩展

## 遇到的问题

### Spring boot 初始化表格报错

  如果同一个数据库地址下面有多个库，当其中某个库已经生成过表时。别的库生成数据库的时候会报错。在数据库URL后面加上参数`nullCatalogMeansCurrent=true`  
例如：`jdbc:mysql://127.0.0.1:3306/activiti?nullCatalogMeansCurrent=true`  
> [深入分析mysql 6.0.6 和 activiti 6.0.0自动创建表失败的问题](https://blog.csdn.net/jiaoshaoping/article/details/80748065)

### 扩展知识



---

## 参考

- [BPMN前端库](https://bpmn.io/toolkit/bpmn-js/)
- [Activiti7 Document](https://activiti.gitbook.io/activiti-7-developers-guide/)
- [Activiti6 Document](https://www.activiti.org/userguide/)
- [Spring Activiti](https://www.baeldung.com/spring-activiti)
- [Spring Activiti Github](https://github.com/eugenp/tutorials/tree/master/spring-activiti)
- [BPMN Wiki](https://zh.wikipedia.org/wiki/%E4%B8%9A%E5%8A%A1%E6%B5%81%E7%A8%8B%E6%A8%A1%E5%9E%8B%E5%92%8C%E6%A0%87%E8%AE%B0%E6%B3%95)
- [Activiti7.X结合SpringBoot2.1、Mybatis](https://dinghuang.github.io/2020/03/14/Activiti7.X%E7%BB%93%E5%90%88SpringBoot2.1%E3%80%81Mybatis/)
- [Ruoyi Vue Activiti](https://gitee.com/smell2/ruoyi-vue-activiti)
- [Activiti6 Java Docs](https://www.activiti.org/javadocs/)
- [Jeesite官网](https://jeesite.com/)