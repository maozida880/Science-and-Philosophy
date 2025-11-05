# README.md

## bdp\_rights\_coupon\_rec\_project (停车权益优惠券推荐系统)

### 1\. 项目简介

本项目是一个完整的端到端机器学习推荐系统，旨在为用户推荐最合适的停车权益优惠券。

项目流程覆盖了从数据处理、样本生成、特征工程、模型训练、模型预测到最终结果写入HBase以供线上服务调用的全过程。

  - **作者**: Li Rongmao
  - **版本**: 2.1.0

### 2\. 技术栈

  - **数据仓库与存储**: Hive, HDFS
  - **实时服务数据库**: HBase (采用A/B表零停机更新机制)
  - **数据处理与模型**: Spark, PySpark, XGBoost (`sparkxgb`)
  - **任务调度**: Airflow
  - **运行环境**: Shell, Python

### 3\. 核心工作流

本项目由Airflow DAG `bdp_rights_coupon_rec_project` 统一调度，执行以下核心步骤：

1.  **负样本生成 (55服务器)**:

      * 从埋点日志中提取曝光数据，生成负样本。
      * 脚本: `insert_negtive_sample_user_plan.sql`

2.  **正样本生成 (55服务器)**:

      * 从订单表中提取购买(转化)数据，生成正样本。
      * 脚本: `insert_positive_sample_user_plan.sql`

3.  **训练/验证集合成 (55服务器)**:

      * 合并正负样本，并关联用户特征与方案特征，生成完整的训练集和验证集。
      * 脚本: `train_dataset_generation.sql`

4.  **模型训练 (133服务器)**:

      * 使用PySpark和XGBoost训练模型，启用早停机制。
      * 模型和特征重要性等产出物保存到HDFS。
      * 脚本: `bdp_rights_coupon_rec_project_train.sh`, `bdp_rights_coupon_rec_project_train.py`

5.  **预测集合成 (55服务器)**:

      * 获取全量用户与全量方案进行笛卡尔积，并关联所有特征，生成预测集。
      * 脚本: `prediction_dataset_generation.sql`

6.  **模型预测 (133服务器)**:

      * 加载HDFS上的模型，对预测集进行批量打分。
      * 预测结果（Top N）格式化为JSON，存回Hive表。
      * 脚本: `bdp_rights_coupon_rec_project_predict.sh`, `bdp_rights_coupon_rec_project_predict.py`

7.  **结果写入HBase (55服务器)**:

      * 从Hive中读取JSON格式的预测结果。
      * 执行A/B表切换逻辑，将最新结果写入非活动HBase表，然后切换元数据，实现零停机更新。
      * 脚本: `hbase_export.sql`

### 4\. 项目结构

所有脚本和配置文件都应部署在项目根目录 `/opt/bdp/project/bdp_rights_coupon_rec_project` 下。

```
/opt/bdp/project/bdp_rights_coupon_rec_project
├── HbaseService/
│   ├── hbase_ddl.sql             # HBase映射表和服务元数据表 (初始化用)
│   └── hbase_export.sql          # 导出结果到HBase (Airflow调度)
├── bin/
│   ├── ad_exec_day_sql.sh        # SQL执行脚本 (55服务器)
│   ├── bdp_rights_coupon_rec_project_predict.sh # 预测脚本 (133服务器)
│   └── bdp_rights_coupon_rec_project_train.sh   # 训练脚本 (133服务器)
├── python/
│   ├── predict/
│   │   ├── bdp_rights_coupon_rec_project_predict.py # 预测PySpark代码
│   │   ├── coupon_encode_config.yaml          # 预测-编码配置
│   │   └── coupon_feature_config.yaml         # 预测-特征配置
│   └── train/
│       ├── bdp_rights_coupon_rec_project_train.py   # 训练PySpark代码
│       ├── coupon_encode_config.yaml          # 训练-编码配置
│       └── coupon_feature_config.yaml         # 训练-特征配置
└── sample/
    ├── data_set/
    │   └── train_dataset_generation.sql   # 训练/验证集生成SQL
    ├── negtive_sample/
    │   ├── init_negtive_sample_user_plan.sql # 负样本表初始化SQL
    │   └── insert_negtive_sample_user_plan.sql # 负样本插入SQL
    ├── positive_sample/
    │   ├── init_positive_sample_user_plan.sql  # 正样本表初始化SQL
    │   └── insert_positive_sample_user_plan.sql  # 正样本插入SQL
    └── predict_data_set/
        └── prediction_dataset_generation.sql # 预测集生成SQL
```

### 5\. 初始化部署

在Airflow DAG首次运行前，需要手动执行一次初始化脚本，以创建HBase表和Hive分区表结构：

1.  **HBase及映射表 (55服务器)**:

      * 执行 `HbaseService/hbase_ddl.sql`

2.  **Hive样本表 (55服务器)**:

      * 执行 `sample/negtive_sample/init_negtive_sample_user_plan.sql`
      * 执行 `sample/positive_sample/init_positive_sample_user_plan.sql`

-----

### 运维部署指南 (致运维同事)

请将项目文件按以下结构部署到指定服务器。

**项目根路径**: `/opt/bdp/project/bdp_rights_coupon_rec_project`

#### 1\. 部署到 55 服务器 (ssh\_conn\_id='ssh\_208\_55')

请将以下文件部署到 `55` 服务器的对应路径：

  - `sys/55/bin/ad_exec_day_sql.sh`
    \-\> `/opt/bdp/project/bdp_rights_coupon_rec_project/bin/ad_exec_day_sql.sh`

  - `sys/55/HbaseService/hbase_ddl.sql`
    \-\> `/opt/bdp/project/bdp_rights_coupon_rec_project/HbaseService/hbase_ddl.sql`

  - `sys/55/HbaseService/hbase_export.sql`
    \-\> `/opt/bdp/project/bdp_rights_coupon_rec_project/HbaseService/hbase_export.sql`

  - `sys/55/sample/data_set/train_dataset_generation.sql`
    \-\> `/opt/bdp/project/bdp_rights_coupon_rec_project/sample/data_set/train_dataset_generation.sql`

  - `sys/55/sample/negtive_sample/init_negtive_sample_user_plan.sql`
    \-\> `/opt/bdp/project/bdp_rights_coupon_rec_project/sample/negtive_sample/init_negtive_sample_user_plan.sql`

  - `sys/55/sample/negtive_sample/insert_negtive_sample_user_plan.sql`
    \-\> `/opt/bdp/project/bdp_rights_coupon_rec_project/sample/negtive_sample/insert_negtive_sample_user_plan.sql`

  - `sys/55/sample/positive_sample/init_positive_sample_user_plan.sql`
    \-\> `/opt/bdp/project/bdp_rights_coupon_rec_project/sample/positive_sample/init_positive_sample_user_plan.sql`

  - `sys/55/sample/positive_sample/insert_positive_sample_user_plan.sql`
    \-\> `/opt/bdp/project/bdp_rights_coupon_rec_project/sample/positive_sample/insert_positive_sample_user_plan.sql`

  - `sys/55/sample/predict_data_set/prediction_dataset_generation.sql`
    \-\> `/opt/bdp/project/bdp_rights_coupon_rec_project/sample/predict_data_set/prediction_dataset_generation.sql`

#### 2\. 部署到 133 服务器 (ssh\_conn\_id='ssh\_model\_133')

请将以下文件部署到 `133` 服务器的对应路径：

  - `sys/133/bin/bdp_rights_coupon_rec_project_predict.sh`
    \-\> `/opt/bdp/project/bdp_rights_coupon_rec_project/bin/bdp_rights_coupon_rec_project_predict.sh`

  - `sys/133/bin/bdp_rights_coupon_rec_project_train.sh`
    \-\> `/opt/bdp/project/bdp_rights_coupon_rec_project/bin/bdp_rights_coupon_rec_project_train.sh`

  - `sys/133/python/predict/bdp_rights_coupon_rec_project_predict.py`
    \-\> `/opt/bdp/project/bdp_rights_coupon_rec_project/python/predict/bdp_rights_coupon_rec_project_predict.py`

  - `sys/133/python/predict/coupon_encode_config.yaml`
    \-\> `/opt/bdp/project/bdp_rights_coupon_rec_project/python/predict/coupon_encode_config.yaml`

  - `sys/133/python/predict/coupon_feature_config.yaml`
    \-\> `/opt/bdp/project/bdp_rights_coupon_rec_project/python/predict/coupon_feature_config.yaml`

  - `sys/133/python/train/bdp_rights_coupon_rec_project_train.py`
    \-\> `/opt/bdp/project/bdp_rights_coupon_rec_project/python/train/bdp_rights_coupon_rec_project_train.py`

  - `sys/133/python/train/coupon_encode_config.yaml`
    \-\> `/opt/bdp/project/bdp_rights_coupon_rec_project/python/train/coupon_encode_config.yaml`

  - `sys/133/python/train/coupon_feature_config.yaml`
    \-\> `/opt/bdp/project/bdp_rights_coupon_rec_project/python/train/coupon_feature_config.yaml`

#### 3\. 部署到 Airflow 服务器

请将上一步生成的 Airflow DAG 脚本 `bdp_rights_coupon_rec_project.py` 部署到 Airflow 的 `dags` 目录下。

#### 4\. 初始化操作 (重要)

在Airflow DAG**首次运行前**，请在 `55` 服务器上手动执行以下SQL文件，以确保HBase和Hive中的目标表已创建：

1.  `hive -f /opt/bdp/project/bdp_rights_coupon_rec_project/HbaseService/hbase_ddl.sql`
2.  `hive --hiveconf project_name=rights_coupon_rec_project -f /opt/bdp/project/bdp_rights_coupon_rec_project/sample/negtive_sample/init_negtive_sample_user_plan.sql`
3.  `hive --hiveconf project_name=rights_coupon_rec_project -f /opt/bdp/project/bdp_rights_coupon_rec_project/sample/positive_sample/init_positive_sample_user_plan.sql`

部署和初始化完成后，即可在Airflow中启动 `bdp_rights_coupon_rec_project` DAG。