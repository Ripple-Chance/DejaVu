# DejaVu 论文复现完整记录文档

# DejaVu 论文复现（微服务故障根因定位）

本仓库是 **DejaVu 论文的个人复现版本**，基于原项目 Fork 后完成完整复现，包含环境配置、运行命令、实验结果、问题解决记录。

原论文：*DejaVu: Deep Learning Based Failure Diagnosis for Microservice Systems*原仓库：[https://github\.com/NetManAIOps/DejaVu](https://link.wtturl.cn/?target=https%3A%2F%2Fgithub.com%2FNetManAIOps%2FDejaVu&scene=im&aid=497858&lang=zh)本复现仓库：[https://github\.com/Ripple\-Chance/DejaVu](https://link.wtturl.cn/?target=https%3A%2F%2Fgithub.com%2FRipple-Chance%2FDejaVu&scene=im&aid=497858&lang=zh)

---

# 目录

1. 项目介绍

2. 复现环境

3. 快速运行命令

4. 数据集说明

5. 实验结果

6. 常见错误与解决方法

7. 总结与优化建议

8. 引用

---

# 1\. 项目介绍

DejaVu 是一种 \*\* 基于图注意力网络（GAT）\*\* 的微服务系统故障根因定位模型。它通过服务调用关系图 \+ 监控指标，自动定位微服务的故障根因节点。

本复现工作：

- 在 Windows 10 \+ Docker 环境下完整复现模型

- 运行官方数据集（SockShop）

- 使用**自定义微服务数据集**（3 个服务：frontend、orders、catalogue）测试

- 记录所有报错、解决方案、运行结果

完整复现笔记：**DejaVu 论文复现完整记录文档\.md**

---

# 2\. 复现环境

系统：Windows 10 \+ Docker Desktop容器内环境：

- Python 3\.8

- PyTorch 1\.10\.0

- PyTorch Lightning 1\.6\.0

- DGL 0\.8\.2

- Pandas 1\.1\.5

- Scikit\-learn 0\.24\.2

---

# 3\. 快速运行命令

## 拉取镜像

```Plain Text
docker pull lizytalk/dejavu
```

## 启动容器（Windows 路径）

```Plain Text
docker run -it --rm --ipc=host -v D:\软测论文\DejaVu:/workspace lizytalk/dejavu bash
```

## 容器内初始化

```Plain Text
cd /workspace
direnv allow
export PYTHONPATH=/workspace
```

## 运行模型（官方数据集）

```Plain Text
python exp/run_GAT_node_classification.py -H=4 -L=8 -fe=GRU -bal=True --data_dir=data/sockshop
```

## 运行自定义数据集

```Plain Text
python exp/run_GAT_node_classification.py -H=4 -L=8 -fe=GRU -bal=True --data_dir=data/my_microservice
```

## 指标归一化预处理

```Plain Text
python -c "
import pandas as pd
from sklearn.preprocessing import MinMaxScaler
metrics_df = pd.read_csv('metrics.csv')
scaler = MinMaxScaler()
metrics_df['value'] = scaler.fit_transform(metrics_df[['value']])
metrics_df.to_pickle('metrics.norm.pkl')
"
```

---

# 4\. 数据集说明

## 官方数据集

提供 SockShop 等微服务故障数据集，包含：

- graph\.yml：服务调用图

- metrics\.csv：监控指标

- faults\.csv：故障标签

## 自定义数据集

- 服务数量：3 个（frontend、orders、catalogue）

- 每个服务指标：7 个（响应时间、CPU、内存、QPS 等）

- 故障样本：23 个

- 故障类型：单根因 \+ 多根因故障

---

# 5\. 实验结果

## 官方数据集 SockShop

- A@1：≈65%

- A@2：≈85%

- A@3：≈95%

- MAR：≈1\.5

## 自定义数据集（最重要）

- **A@3 = 100%**模型能在前 3 个候选结果里 100% 找到真实根因

- A@1 偏低原因：catalogue 服务故障样本太少

- 时间戳显示 1970 年属于时区问题，**不影响模型效果**

---

# 6\. 常见错误与解决方法

## 1\. 时间戳格式错误

错误：`ValueError: could not convert string to Timestamp`解决：把 ISO 时间转为 Unix 时间戳

## 2\. graph\.yml 格式错误

错误：`AttributeError: 'str' object has no attribute 'get'`解决：必须写 class 字段，指标必须带 \#\# 分隔符

## 3\. 编码错误

错误：`UnicodeDecodeError`解决：文件保存为 UTF\-8，不要用 ANSI

## 4\. 列名不匹配

metrics\.csv 必须用：timestamp, name, valuefaults\.csv 必须用：timestamp, root\_cause\_node

## 5\. Pickle 版本不兼容

解决：删除本地 pkl，在容器内重新生成

## 6\. 节点名称不匹配

faults\.csv 里的名字必须和 graph\.yml 完全一致

## 7\. 多根因分隔符错误

必须用 **分号；** 分隔多个根因，不能用逗号

## 8\. 时间戳不匹配

修改 metric\_preprocessor\.py 使用最近邻匹配

## 9\. 缩进错误

统一使用 4 个空格缩进

---

# 7\. 总结与优化建议

## 复现总结

- 官方代码有很多隐藏格式要求，必须看源码

- 跨平台（Windows \+ Docker）容易出现编码、路径、时间问题

- 数据集越均衡，模型效果越好

## 优化建议

1. 增加 catalogue 服务的故障样本

2. 调参：\-L=10 \-H=8

3. 完善多根因故障评估逻辑

4. 修复时区导致的时间戳显示问题

## 结论

DejaVu 模型复现成功，在自定义数据集上 **A@3=100%**，可用于实际微服务故障定位场景。

---

# 8\. 引用

plaintext

```Plain Text
@inproceedings{li2022actionable,
  title={Actionable and Interpretable Fault Localization for Recurring Failures in Online Service Systems},
  author={Li, Zeyan and others},
  booktitle={ESEC/FSE 2022},
  year={2022}
}
```

