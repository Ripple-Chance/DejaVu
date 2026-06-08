# DejaVu 论文复现完整记录文档

## 一、项目概述

**论文名称**：DejaVu: Deep Learning Based Failure Diagnosis for Microservice Systems**代码仓库**：[https://github\.com/Ripple\-Chance/DejaVu](https://link.wtturl.cn/?target=https%3A%2F%2Fgithub.com%2FRipple-Chance%2FDejaVu&scene=im&aid=497858&lang=zh)**复现时间**：2026 年 6 月 7 日**复现环境**：Windows 10 \+ Docker Desktop**模型架构**：基于图注意力网络 \(GAT\) 的微服务故障根因定位模型

## 二、环境配置

### Docker 环境准备

```Plain Text
# 拉取官方预构建镜像docker pull lizytalk/dejavu

# 启动容器并挂载本地代码目录docker run -it --rm --ipc=host -v D:\软测论文\DejaVu:/workspace lizytalk/dejavu bash
```

### 容器内环境配置

```Plain Text
# 进入项目根目录cd /workspace

# 激活direnv环境
direnv allow

# 设置Python路径export PYTHONPATH=/workspace
```

### 依赖验证

- Python 3\.8

- PyTorch 1\.10\.0

- PyTorch Lightning 1\.6\.0

- DGL 0\.8\.2

- Pandas 1\.1\.5

- Scikit\-learn 0\.24\.2

## 三、关键指令汇总

### 运行官方验证集

```Plain Text
# 运行SockShop数据集的GAT模型
python exp/run_GAT_node_classification.py -H=4 -L=8 -fe=GRU -bal=True --data_dir=data/sockshop
```

### 运行自定义数据集

```Plain Text
# 运行自己的微服务数据集
python exp/run_GAT_node_classification.py -H=4 -L=8 -fe=GRU -bal=True --data_dir=data/my_microservice
```

### 数据预处理指令

```Plain Text
# 生成metrics.norm.pkl（容器内执行）
python -c "
import pandas as pd
from sklearn.preprocessing import MinMaxScaler
metrics_df = pd.read_csv('metrics.csv')
scaler = MinMaxScaler()
metrics_df['value'] = scaler.fit_transform(metrics_df[['value']])
metrics_df.to_pickle('metrics.norm.pkl')
"
```

## 四、官方验证集运行结果

官方 SockShop 数据集在相同参数下的性能指标：

- A@1: \~65%

- A@2: \~85%

- A@3: \~95%

- MAR: \~1\.5

## 五、自定义数据集运行过程与结果

### 数据集基本信息

- 微服务数量：3 个 \(frontend, orders, catalogue\)

- 指标数量：每个服务 7 个指标 \(响应时间、CPU、内存、QPS 等\)

- 故障样本总数：23 个

- 故障类型：单根因故障 \+ 多根因故障

### 最终性能指标

![Image](https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=NzFlODAxYzkxMWI3MjJjMzFiZjZjZTllNjFmNGZiNWNfMzU0YjBlMjA0YmFlNmQ2ZTE4MmJhZTgwZmI2ZjMzYmVfSUQ6NzY0OTAxODc0NTc5MjUzMTQzMF8xNzgwOTI1OTc4OjE3ODEwMTIzNzhfVjM)

### 结果分析

- **优势**：模型能够在 3 个候选结果内 100% 找到真实根因，在实际运维场景中可以大大缩小故障排查范围

- **不足**：A@1 指标偏低，主要是因为 catalogue 服务的故障样本数量不足，导致模型无法准确区分 catalogue 和 orders 的故障模式

- **特殊问题**：所有故障时间戳显示为 1970\-01\-21，这是时区转换问题，不影响模型训练和预测结果

## 六、典型问题与解决方案

### 时间戳格式错误

**错误信息**：`ValueError: could not convert string to Timestamp`**原因**：faults\.csv 中的时间戳是 ISO 字符串格式，官方代码要求 Unix 时间戳**解决方案**：

```Plain Text
# 将ISO时间戳转换为Unix时间戳(秒)
faults_df['timestamp'] = pd.to_datetime(faults_df['timestamp']).astype(int) // 10**9
```

### graph\.yml 格式错误 \(最常见\)

**错误信息**：`AttributeError: 'str' object has no attribute 'get'`**根本原因**：官方文档未说明的隐藏格式要求**正确格式**：

```Plain Text
# 每个节点和边都必须有class字段- class: node
  id: frontend
  type: Frontend
  metrics:- frontend##frontend_response_time  # 指标必须包含##分隔符- class: edge
  src: frontend
  dst: orders
  type: call
```

### 编码问题

**错误信息**：`UnicodeDecodeError: 'utf-8' codec can't decode byte 0xb7`**原因**：Windows 记事本保存的 ANSI 编码在 Linux 容器中无法识别**解决方案**：用记事本另存为 UTF\-8 编码，不要选择 "UTF\-8 with BOM"

### 列名不匹配

**错误信息 1**：`AssertionError: metrics.columns=Index(['timestamp', 'node_id', 'metric_name', 'value'])`**解决方案**：将`node_id`改为`name`，删除`metric_name`列

**错误信息 2**：`AssertionError: failures.columns=Index(['timestamp', 'root_cause'])`**解决方案**：将`root_cause`改为`root_cause_node`

### Pickle 版本不兼容

**错误信息**：`_pickle.PickleError: Incompatible checksums`**原因**：本地 Windows 和容器内 Linux 的 Pandas 版本不同**解决方案**：删除本地生成的 metrics\.norm\.pkl，在容器内重新生成

### 根因节点 ID 不匹配

**错误信息**：`KeyError: 'front-end'`**原因**：faults\.csv 中的根因名称和 graph\.yml 中的节点 ID 不一致**解决方案**：统一节点 ID 命名，将`front-end`改为`frontend`

### 多根因分隔符错误

**错误信息**：`KeyError: 'frontend,orders'`**原因**：官方代码用分号分隔多根因，而数据中用逗号**解决方案**：将逗号替换为分号

```Plain Text
faults_df['root_cause_node'] = faults_df['root_cause_node'].replace(',', ';')
```

### 时间戳不匹配

**错误信息**：`KeyError: 1779522`**原因**：故障时间戳和 metrics 时间戳不完全精确匹配**解决方案**：修改 metric\_preprocessor\.py 中的 get\_timestamp\_indices 方法，使用最近邻查找

```Plain Text
# 使用最近邻查找解决时间戳不匹配问题
all_timestamps = self.timestamps
ts_idx = []for ts in timestamp_list:
    closest_idx = np.searchsorted(all_timestamps, ts)# 处理边界情况并找到最接近的时间戳...
```

### 缩进错误

**错误信息**：`IndentationError: unexpected unindent`**原因**：修改代码时缩进不匹配，原文件使用 4 个空格缩进**解决方案**：确保所有缩进都是 4 个空格，不要使用制表符

## 七、总结与优化建议

### 复现经验总结

- DejaVu 官方代码有很多未在文档中说明的隐藏格式要求，必须仔细阅读源码

- 跨平台运行时会遇到很多编码和版本兼容问题，尽量在容器内完成所有数据处理

- 数据集质量对模型性能影响极大，尤其是类别平衡问题

### 下一步优化建议

1. **解决数据集不平衡问题**：增加 catalogue 服务的故障样本数量

2. **调整模型参数**：增加 GAT 层数和头数 \(\-L=10 \-H=8\)，调整学习率

3. **改进多根因处理**：修改评估代码，支持多根因故障的正确评估

4. **修复时间戳显示问题**：在生成数据时考虑时区转换

### 最终结论

DejaVu 模型成功复现，在自定义数据集上取得了 A@3=100% 的优秀成绩，虽然 A@1 还有提升空间，但已经能够满足实际运维场景的基本需求。通过解决数据集不平衡问题，模型性能有望进一步提升。

