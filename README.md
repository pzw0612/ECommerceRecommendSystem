# ECommerceRecommendSystem
基于spark的商品推荐系统

# 项目体系架构设计

## 1.1 项目系统架构

项目以推荐系统建设领域知名的经过修改过的中文亚马逊电商数据集作为依托，以某电商网站真实业务数据架构为基础，构建了符合教学体系的一体化的电商推荐系统，包含了离线推荐与实时推荐体系，综合利用了协同过滤算法以及基于内容的推荐方法来提供混合推荐。提供了从前端应用、后台服务、算法设计实现、平台部署等多方位的闭环的业务实现。

![img](G:\project\hadoop\ECommerceRecommendSystem\doc\wps3.jpg) 

**用户可视化**：主要负责实现和用户的交互以及业务数据的展示，主体采用AngularJS2进行实现，部署在Apache服务上。
**综合业务服务**：主要实现JavaEE层面整体的业务逻辑，通过Spring进行构建，对接业务需求。部署在Tomcat上。

**【数据存储部分】**
**业务数据库**：项目采用广泛应用的文档数据库MongDB作为主数据库，主要负责平台业务逻辑数据的存储。
**缓存数据库**：项目采用Redis作为缓存数据库，主要用来支撑实时推荐系统部分对于数据的高速获取需求。

**【离线推荐部分】**
**离线统计服务**：批处理统计性业务采用Spark Core + Spark SQL进行实现，实现对指标类数据的统计任务。
**离线推荐服务**：离线推荐业务采用Spark Core + Spark MLlib进行实现，采用ALS算法进行实现。

**【实时推荐部分】**
**日志采集服务**：通过利用Flume-ng对业务平台中用户对于商品的一次评分行为进行采集，实时发送到Kafka集群。
**消息缓冲服务**：项目采用Kafka作为流式数据的缓存组件，接受来自Flume的数据采集请求。并将数据推送到项目的实时推荐系统部分。
**实时推荐服务**：项目采用Spark Streaming作为实时推荐系统，通过接收Kafka中缓存的数据，通过设计的推荐算法实现对实时推荐的数据处理，并将结构合并更新到MongoDB数据库。

## 1.2 项目数据流程

![img](G:\project\hadoop\ECommerceRecommendSystem\doc\wps4.jpg) 

【系统初始化部分】

0. 通过Spark SQL将系统初始化数据加载到MongoDB中。

【离线推荐部分】

1. 可以通过Azkaban实现对于离线统计服务以离线推荐服务的调度，通过设定的运行时间完成对任务的触发执行。

2. 离线统计服务从MongoDB中加载数据，将【商品平均评分统计】、【商品评分个数统计】、【最近商品评分个数统计】三个统计算法进行运行实现，并将计算结果回写到MongoDB中；离线推荐服务从MongoDB中加载数据，通过ALS算法分别将【用户推荐结果矩阵】、【影片相似度矩阵】回写到MongoDB中。

【实时推荐部分】

3. Flume从综合业务服务的运行日志中读取日志更新，并将更新的日志实时推送到Kafka中；Kafka在收到这些日志之后，通过kafkaStream程序对获取的日志信息进行过滤处理，获取用户评分数据流【UID|MID|SCORE|TIMESTAMP】，并发送到另外一个Kafka队列；Spark Streaming监听Kafka队列，实时获取Kafka过滤出来的用户评分数据流，融合存储在Redis中的用户最近评分队列数据，提交给实时推荐算法，完成对用户新的推荐结果计算；计算完成之后，将新的推荐结构和MongDB数据库中的推荐结果进行合并。

【业务系统部分】

4. 推荐结果展示部分，从MongoDB中将离线推荐结果、实时推荐结果、内容推荐结果进行混合，综合给出相对应的数据。

5. 商品信息查询服务通过对接MongoDB实现对商品信息的查询操作。

6. 商品评分部分，获取用户通过UI给出的评分动作，后台服务进行数据库记录后，一方面将数据推动到Redis群中，另一方面，通过预设的日志框架输出到Tomcat中的日志中。

7. 商品标签部分，项目提供用户对商品打标签服务。

## 1.3 数据模型

1. Product【商品数据表】

| 字段名     | 字段类型 | 字段描述      | 字段备注         |
| ---------- | -------- | ------------- | ---------------- |
| productId  | Int      | 商品的ID      |                  |
| name       | String   | 商品的名称    |                  |
| categories | String   | 商品所属类别  | 每一项用“\|”分割 |
| imageUrl   | String   | 商品图片的URL |                  |
| tags       | String   | 商品的UGC标签 | 每一项用“\|”分割 |

 

2. Rating【用户评分表】

| 字段名    | 字段类型 | 字段描述   | 字段备注 |
| --------- | -------- | ---------- | -------- |
| userId    | Int      | 用户的ID   |          |
| productId | Int      | 商品的ID   |          |
| score     | Double   | 商品的分值 |          |
| timestamp | Long     | 评分的时间 |          |

 

3. Tag【商品标签表】

| 字段名    | 字段类型 | 字段描述   | 字段备注 |
| --------- | -------- | ---------- | -------- |
| userId    | Int      | 用户的ID   |          |
| productId | Int      | 商品的ID   |          |
| tag       | String   | 商品的标签 |          |
| timestamp | Long     | 评分的时间 |          |

 

 

4. User【用户表】

| 字段名    | 字段类型 | 字段描述       | 字段备注 |
| --------- | -------- | -------------- | -------- |
| userId    | Int      | 用户的ID       |          |
| username  | String   | 用户名         |          |
| password  | String   | 用户密码       |          |
| timestamp | Lon0067  | 用户创建的时间 |          |

 

5. RateMoreProductsRecently【最近商品评分个数统计表】

| 字段名    | 字段类型 | 字段描述     | 字段备注 |
| --------- | -------- | ------------ | -------- |
| productId | Int      | 商品的ID     |          |
| count     | Int      | 商品的评分数 |          |
| yearmonth | String   | 评分的时段   | yyyymm   |

 

6. RateMoreProducts【商品评分个数统计表】

| 字段名    | 字段类型 | 字段描述     | 字段备注 |
| --------- | -------- | ------------ | -------- |
| productId | Int      | 商品的ID     |          |
| count     | Int      | 商品的评分数 |          |

 

7. AverageProductsScore【商品平均评分表】

| 字段名    | 字段类型 | 字段描述       | 字段备注 |
| --------- | -------- | -------------- | -------- |
| productId | Int      | 商品的ID       |          |
| avg       | Double   | 商品的平均评分 |          |

 

8. ProductRecs【商品相似性矩阵】

| 字段名    | 字段类型                            | 字段描述               | 字段备注 |
| --------- | ----------------------------------- | ---------------------- | -------- |
| productId | Int                                 | 商品的ID               |          |
| recs      | Array[(productId:Int,score:Double)] | 该商品最相似的商品集合 |          |

 

9. UserRecs【用户商品推荐矩阵】

| 字段名 | 字段类型                            | 字段描述               | 字段备注 |
| ------ | ----------------------------------- | ---------------------- | -------- |
| userId | Int                                 | 用户的ID               |          |
| recs   | Array[(productId:Int,score:Double)] | 推荐给该用户的商品集合 |          |

 

10. StreamRecs【用户实时商品推荐矩阵】

| 字段名 | 字段类型                            | 字段描述                   | 字段备注 |
| ------ | ----------------------------------- | -------------------------- | -------- |
| userId | Int                                 | 用户的ID                   |          |
| recs   | Array[(productId:Int,score:Double)] | 实时推荐给该用户的商品集合 |          |