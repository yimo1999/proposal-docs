## 前端状态机

```mermaid
stateDiagram-v2
    [*] --> 初始态
    初始态 --> 排队中: 用户发起测试请求
    排队中 --> 运行中: CI-JOB成功下发
    排队中 --> 测试终止: 用户终止测试/被动终止
    运行中 --> 测试通过: 测试成功
    运行中 --> 测试失败: 测试失败
    运行中 --> 测试终止: 用户终止测试/被动终止
    测试终止 --> 初始态: 用户重新发起测试
    测试通过 --> 初始态: 用户重新发起测试
    测试失败 --> 初始态: 用户重新发起测试
```

## flex-compute 单个job状态机

### 推理失败原因

- 模型等文件下载失败

- 对比测试
  - 推理脚本或模型问题导致的单次前向推理失败
  - 虽然单次前向推理成功，但是与标杆对比后低于评估标准
- 基线测试
  - 推理脚本或模型问题导致的单次前向推理失败

```mermaid
stateDiagram-v2
    [*] --> 初始态
    初始态 --> pending: ci-adapter通过sdk创建job
    pending --> downloading: job成功下发，拉起下载用容器
    downloading --> downloadFailed: 下载失败
    downloadFailed --> 初始态: 用户重新发起测试
    downloading --> downloadSuccess: 下载成功
    downloadSuccess --> running: 拉起推理用容器，开始推理
    running --> runningFailed: 推理失败
    running --> runningSuccess: 推理成功
    runningFailed --> 初始态: 用户重新发起测试
    runningSuccess --> 初始态: 用户重新发起测试
```





## 结构体新增/变更

### Kafka消息接口结构体

**原有的模型可用性认证消息命名由 model_ci_created, model_ci_stopped修改为usability_test_created, usability_test_stopped**

#### accuracy_test_created消息字段

| 字段名           | 类型    | 描述                                         | 示例值                                                       | 是否必选 |
| ---------------- | ------- | -------------------------------------------- | ------------------------------------------------------------ | :------: |
| id               | string  | 该精度测试对应的唯一标识符，数据库自增主键   |                                                              |    是    |
| TestType         | string  | 区分测试类型（如可用性测试、精度测试）       | "accuracy_test" \|"usability_test"                           |    是    |
| ModelTestType    | string  | 区分基础测试类型（推理或训练）               | "inference"  \|"training"                                    |    是    |
| ModelTemplate    | string  | 精度测试执行所需的模型模板                   | 见约束对应章节                                               |    否    |
| DatasetID        | string  | 训练精度测试关联的数据集信息，用于下载数据集 | "dataset_abc"                                                |    否    |
| BaselineID       | Int64   | 训练关联的基线id                             |                                                              |    是    |
| BaselineName     | string  | 训练关联的基线名称                           | "baseline_1"                                                 |    否    |
| Metric           | float64 | 使用的评估指标                               | BLEU、绝对误差                                               |    否    |
| Threshold        | float64 | 通过测试的阈值                               | 0.85                                                         |    否    |
| Operator         | string  | 用于比较的操作符                             | >=, <=                                                       |    是    |
| ModelID          | int64   | 测试所需要的模型对应的唯一ID                 |                                                              |    是    |
| Owner            | string  | 模型Owner                                    |                                                              |    是    |
| ModelName        | string  | 模型名称                                     |                                                              |    是    |
| ImageName        | string  | 测试所需要的镜像名称                         | openeuler-python3.10-cann8.0.rc2.beta1-pytorch2.1.0-openmind0.9.0 |    是    |
| GitURL           | string  | 模型对应的git url                            | https://modelers.cn/ModelOwner/ModelName.git                 |    是    |
| CommitID         | string  | 模型对应的某一commit id                      |                                                              |    是    |
| HardwareVersion  | string  | 硬件信息                                     | NPU                                                          |    是    |
| Framework        | string  | 框架信息                                     | pytorch                                                      |    是    |
| FrameworkVersion | string  | 框架版本                                     | 2.1.0                                                        |    是    |
| CannVersion      | string  | Cann版本                                     | 8.0.rc1.beta1                                                |    是    |
| IsPubToPri       | bool    | 判断Repo是否由公转私                         | True/False                                                   |    是    |
| IsRepoDeleted    | bool    | 判断Repo是否被删除的标志                     | True/False                                                   |    是    |
| NumComputeCards  | int     | 精度测试所需卡数                             | 1，2，4，8                                                   |    是    |



### 内存数据结构体字段修改

**用于模型可用性认证的内存相关的结构体 ModelInfo 修改为 UsabilityModelInfo**

新增用于精度测试的内存相关结构体 AccuracyModelInfo

#### AccuracyModelInfo结构体参数

| 字段名           | 类型             | 描述                                   | 示例值                                                       | 是否必选 |
| ---------------- | ---------------- | -------------------------------------- | ------------------------------------------------------------ | -------- |
| ID               | int64            | 唯一任务ID                             |                                                              | 是       |
| Owner            | string           | 模型Owner                              | ModelOwner                                                   | 是       |
| ModelName        | string           | 模型名字                               | ModelName-7B                                                 | 是       |
| ModelID          | Int64            | 模型ID                                 |                                                              | 是       |
| ImageName        | string           | 精度测试使用的基础镜像                 | openeuler-python3.10-cann8.0.rc2.beta1-pytorch2.1.0-openmind0.9.0 | 是       |
| GitURL           | string           | 模型的URL                              | https://modelers.cn/ModelOwner/ModelName.git                 | 是       |
| TestStatus       | string           | 测试状态                               | running, pending, etc...                                     | 是       |
| StatusConfirmed  | bool             |                                        |                                                              | 是       |
| LastUpdated      | string           |                                        |                                                              |          |
| HardwareVersion  | string           | 硬件信息                               | NPU                                                          | 是       |
| Framework        | string           | 框架信息                               | pytorch                                                      | 是       |
| FrameworkVersion | string           | 框架版本                               | 2.1.0                                                        | 是       |
| CannVersion      | string           | Cann版本                               | 8.0.rc1.beta1                                                | 是       |
| CommitID         | string           | 模型仓的commit ID                      | 239f776                                                      | 是       |
| IsRepoDeleted    | bool             | 判断Repo是否被删除的标志               | True/False                                                   | 是       |
| CITask           | CITaskInfo       |                                        |                                                              | 是       |
| NumComputeCards  | int              | 精度测试所需卡数                       | 1，2，4，8                                                   | 是       |
| TestType         | string           | 区分测试类型（如可用性测试、精度测试） | "accuracy_test" \|"usability_test"                           | 是       |
| ModelTestType    | string           | 区分基础测试类型（推理或训练）         | "inference"  \|"training"                                    | 是       |
| RunAsBaseLine    | bool             | 当前任务是否作为基线运行的标志         | True/False                                                   | 是       |
| ModelTemplate    | string           | 模型Template                           | qwen, internal, etc...                                       | 否       |
| DatasetName      | string           | 精度训练所用的数据集名称               | alpace-demo                                                  | 否       |
| Metric           | string           | 使用的评估指标                         | BLEU、绝对误差                                               | 否       |
| Threshold        | float            | 通过测试的阈值                         | 0.85                                                         | 否       |
| Operator         | string           | 用于比较的操作符                       | >=, <=                                                       | 是       |
| BaseLineName     | string           | 用于对比的基线名称                     | qwen2-0.5b-baseline2                                         | 否       |
| CiTask           | AccuracyTaskInfo | 精度测试相关的信息                     |                                                              | 是       |



#### AccuracyTaskInfo结构体参数

| 字段名         | 类型   | 描述                           | 示例值                     | 是否必选 |
| -------------- | ------ | ------------------------------ | -------------------------- | -------- |
| Status         | string | 获取模型在flex-compute上的状态 | running, success, etc...   | 否       |
| JobID          | string | K8s上对应的JobID               |                            | 是       |
| ReportURL      | string | 日志文件路径                   |                            | 否       |
| NewBaseLineURL | string | 基线URL                        | obs URL                    | 否       |
| TestResultURL  | string | test_result.json文件URL        | obs URL                    | 否       |
| OutputURL      | string | 推理输出文件URL                | obs URL                    | 否       |
| MetricValue    | float  | 使用对应评估输出的得分         | 0.85                       | 否       |
| Passed         | bool   | 用于判断对比测试是否通过的标志 | True/False                 | 否       |
| Stage          | string | 测试处在的阶段                 | pending/download/inference | 否       |





### merlin server模型状态变更通知

#### UpdateAccuracyCI结构体

| 字段名                   | 类型   | 描述                                          | 示例值 | 是否必选 |
| ------------------------ | ------ | --------------------------------------------- | ------ | -------- |
| Id                       | int64  | 任务id                                        |        | 是       |
| ModelId                  | int64  | 数据库内模型自增id                            |        | 是       |
| CommitId                 | string | commit id                                     |        | 是       |
| CiStatus                 | string | ci状态                                        |        | 是       |
| ResultURL                | string | 日志URL（模型可用性测试）                     |        | 否       |
| AccuracyExternalArgsInfo | string | merlin server端需要的数据被整合到这个结构体中 |        | 是       |



#### AccuracyExternalArgsInfo字段

| 字段名          | 类型    | 描述                                   | 示例值                                                       | 是否必选 |
| --------------- | ------- | -------------------------------------- | ------------------------------------------------------------ | -------- |
| ImageName       | string  | 镜像名称                               | openeuler-python3.10-cann8.0.rc2.beta1-pytorch2.1.0-openmind0.9.0 | 是       |
| JobID           | string  | job id                                 |                                                              | 是       |
| Status          | string  | ci状态                                 | running/success/failed                                       | 是       |
| ReportURL       | string  | 日志URL（模型可用性测试）              |                                                              | 否       |
| StatusConfirmed | bool    | 状态确认位                             | True/False                                                   | 是       |
| Stage           | string  | 测试所处stage                          | downloading/infer/train                                      | 是       |
| OutputFileURL   | string  | 输出文件URL                            |                                                              | 否       |
| LogFileURL      | string  | 日志文件URL                            |                                                              | 否       |
| Passed          | bool    | 用于告知测试是否通过的判断标志         | True/False                                                   | 是       |
| MetricValue     | float64 | 对比得到的得分                         | 0.75                                                         | 否       |
| Metric          | string  | 精度对比所用的方法                     | BLEU                                                         | 是       |
| Threshold       | float64 | 阈值                                   | 0.85                                                         | 否       |
| BaseLineID      | int64   | 基线唯一ID                             |                                                              | 否       |
| BaseLineName    | string  | 基线名称                               |                                                              | 否       |
| TestType        | string  | 区分测试类型（如可用性测试、精度测试） | "accuracy_test" \|"usability_test"                           | 是       |



## 推理精度测试发起

前端/merlin server/kafka/ci-adapter/flex-compute之间的交互

### 精度测试发起流程概述

1. 用户发起测试的处理请求
2. 前端向merlin server发起精度测试请求
3. merlin通过Kafka发送消息
4. ci-adapter消费Kafka消息，调用flex-compute暴露的job create接口创建job，创建成功返回值包含job id。若失败，直接更新merlin中对应模型的精度测试状态为failed
5. ci-adapter周期性查询内存中保存的job的状态，得到job状态后，通知merlin状态变更
6. merlin返回结果给前端



```mermaid
sequenceDiagram
	participant user as 用户
  participant frontEnd as 前端
	participant merlin as merlin server
	participant Kafka as Kafka消息队列
	participant ci-adapter as ci-adapter
	participant flex-compute as flex-compute
	
	
	user ->> frontEnd: 发起“对比测试”处理请求
	frontEnd ->> merlin: 发送精度测试请求
	merlin ->> Kafka: 创建Kafka消息体放至usability_test_created
	Kafka ->> ci-adapter: 消费消息，调用flex-compute提供的接口创建job
	ci-adapter ->> flex-compute: job创建成功
	flex-compute ->> flex-compute: job status修改为running
	flex-compute -->> ci-adapter: 返回job id
	alt 创建失败
	  ci-adapter ->> ci-adapter: job创建失败
		ci-adapter ->> merlin: 更新merlin server中对应模型的精度测试状态为failed
	end
	
	loop 周期性查询Job状态
		flex-compute ->> flex-compute: flex-compute内创建pod执行相应任务
		ci-adapter ->> flex-compute: 查询job状态
		flex-compute -->> ci-adapter: 返回job状态
		ci-adapter ->> merlin: 通知状态变更
	end
	
	merlin -->> frontEnd: 返回结果
	
```



## flex-compute/obs/社区之间的交互

### 整体流程

1. 容器启动

   - 获取环境变量:
     - MODEL_OWNER: 模型拥有者
     - MODEL_NAME: 模型名称
     - COMMIT_ID: commit ID
     - MODEL_TEMPLATE: 模型模版
     - TEST_TYPE: 测试类型（可用性或精度测试）
     - MODEL_TEST_TYPE: 基础测试类型（推理或训练）
     - RUN_AS_BASELINE: 是否作为基线运行
     - BASELINE_NAME: 基线名称
     - BASELINE_ID: 基线对应的唯一ID
     - METRIC: 精度对比策略（BLEU，Accuracy...）
     - THRESHOLD: 阈值大小
     - OPERATOR: 与阈值对比符号（>=, <=）
     - BUCKET_NAME: obs对应的bucket name
     - ENDPOINT: obs对应的endpoint
     - ACCURACY_TEST_ID: 精度测试对应的唯一ID
     - INPUT_FILE_NAME: 输入文本文件名称
     - INPUT_FILE_PATH: 输入文本文件路径
     - OUTPUT_FILE_NAME: 输出文本文件名称
     - OUTPUT_FILE_PATH: 输出文本文件路径
2. 下载模型

   - 使用openmind-hub sdk，根据MODEL_OWNER，MODEL_NAME和COMMIT_ID下载模型到本地路径
3. 执行推理

   - 对比测试

     - 读取环境变量中的INPUT_FILE_PATH，将文件内容作为输入

     - 读取OUTPUT_FILE_PATH文件，作为标杆

     - 使用openmind-cli执行单次前向推理

     - 计算推理输出文本与基线输出文本的BLEU值，并与阈值比较判断测试是否通过

     - 将输出文本存入OBS对应的基线目录

   - 基线测试

     - 读取环境变量中的INPUT_FILE_PATH，将文件内容作为输入

     - 使用openmind-cli执行单次前向推理
4. 日志处理

   - 推理过程中的打屏日志重定向到文件
   - 生成test_results.json文件
   - 将日志文件和test_result.json文件上传至OBS



```pseudocode
# test_result.json sample
# comparison_datails字段当"is_comparison_test": true 时才会存在
{
  "baseline_id": "baseline_123",
  "is_comparison_test": true,
  "test_type": "inference",
  "stage": "test",
  "output": "output.txt",
  "log": "execution.log",
  "passed": true,
  "comparison_details": {
    "metric": "BLEU",
    "metric_value": 0.85,
    "threshold": 0.80,
    "operator": ">="
  }
}
```





#### 对比测试流程图

```mermaid
sequenceDiagram
	participant modelers as 魔乐社区
	participant flex-compute as flex-compute
	participant container as 推理容器
	participant obs as 华为云OBS
	

	flex-compute ->> flex-compute: job status修改为running
	modelers ->> container: 从社区镜像中心拉取对应镜像
	flex-compute ->> container: 创建用于模型下载的pod
	container -->> flex-compute: 下载成功
	
	alt 下载失败
		container ->> flex-compute: job status修改为failed
	end
	
	flex-compute ->> container: 创建用于模型推理的pod
	
		obs ->> container: 读取标杆的输入和输出文件
		container ->> container: 执行单次前向推理
		container ->> obs: 输出文件存储到OBS
		
		alt 推理成功
			container ->> container: 计算BLEU值
			
			alt BLEU值 >= threshold
			  container ->> flex-compute: job status修改为success
			end
			
			alt BLEU值 <= threshold
			  container ->> flex-compute: job status修改为failed
			end
			
		end
		
		alt 推理失败
			container ->> flex-compute: job status修改为failed
		end
	
	container ->> container: 日志重定向到日志文件
	container ->> obs: 日志文件存储到OBS
	container ->> obs: 生成test_results.json，存储到OBS
	
	
	
```



#### 基准测试流程图

```mermaid
sequenceDiagram
	participant modelers as 魔乐社区
	participant flex-compute as flex-compute
	participant container as 推理容器
	participant obs as 华为云OBS
	

	flex-compute ->> flex-compute: job status修改为running
	modelers ->> container: 从社区镜像中心拉取对应镜像
	flex-compute ->> container: 创建用于模型下载的pod
	container -->> flex-compute: 下载成功
	
	alt 下载失败
		container ->> flex-compute: job status修改为failed
	end
	
	flex-compute ->> container: 创建用于模型推理的pod
	
		obs ->> container: 读取输入文件
		container ->> container: 使用输入文件内的文本执行单次前向推理
		
		alt 推理成功
			container ->> flex-compute: job status修改为success
			container ->> obs: 输出文件存储到OBS基线目录
			
		end
		
		alt 推理失败
			container ->> flex-compute: job status修改为failed
		end
	
	container ->> container: 日志重定向到日志文件
	container ->> obs: 日志文件存储到OBS
	container ->> obs: 生成test_results.json，存储到OBS
	
	
	
```



## 终止推理精度测试（涵盖发起终止请求/模型仓删除/转私有/文件更改）

### 整体流程

1. 用户主动发起终止精度测试的处理请求/被动发起终止的请求
2. 前端向merlin server发起终止请求
3. merlin通过Kafka发送消息
4. ci-adapter消费Kafka消息
   - 主动终止
     - 调用flex-compute暴露的job create接口根据job id终止对应的job
   - 被动终止
     - 调用相关接口终止和该模型有关的job

5. ci-adapter周期性查询内存中保存的job的状态，得到job状态后，通知merlin状态变更
6. merlin返回结果给前端



```mermaid
sequenceDiagram
	participant user as 用户
  participant frontEnd as 前端
	participant merlin as merlin server
	participant Kafka as Kafka消息队列
	participant ci-adapter as ci-adapter
	participant flex-compute as flex-compute
	
	
	user ->> frontEnd: 执行模型仓删除/转私有/发起终止请求/文件更改等操作
	frontEnd ->> merlin: 发送对应请求
	merlin ->> Kafka: 创建Kafka消息体放至usability_test_stopped

	Kafka ->> ci-adapter: 消费消息
	alt 被动终止
		ci-adapter ->> flex-compute: 调用felx-compute提供的terminateTaskByModelID终止和该模型有关的job
	end
	ci-adapter ->> flex-compute: 调用flex-compute提供的terminateTaskByID接口终止对应的job
	flex-compute ->> flex-compute: job终止成功，job status修改为stopped
	
	loop 周期性查询Job状态
		flex-compute ->> flex-compute: flex-compute内创建pod执行相应任务
		ci-adapter ->> flex-compute: 查询job状态
		flex-compute -->> ci-adapter: 返回job状态
		ci-adapter ->> flex-compute: delete 状态为 stopped/job_not_exist 的job
		ci-adapter ->> merlin: 通知状态变更
	end
	
	merlin -->> frontEnd: 返回结果
	
```









## 精度测试执行侧伪代码



### 伪代码

输入：OWNER, MODEL_NAME,  BASELINE_NAME, NEW_BASELINE_NAME, METRIC, THRESHOLD, OPERATOR, TEST_TYPE, MODEL_TEST_TYPE, RUN_AS_BASELINE, INPUT_FILE_PATH, TEST_ID



```python
#推理代码   
def inference_for_accuracy():
  model_path = f"/home/openmind/data/.cache/{OWNER/MODEL_NAME}/snapshot/commit-id"
  obs_root_path = f"/root_path/results"
  # 执行推理
  pipe = openmind.pipeline(model_path)
  benchmark_input=file_reader("input_benchmark.txt")
  # 确定随机性种子，使多次推理输出能保持一致
  output = pipe(benchmark_input)
  file_writer("output.txt", output)
  benchmark_output = file_reader("output_benchmark.txt")
  # 通过环境变量METRIC判断使用的评估方法，对基线输出和pipeline的到的输出进行对比输出得分
  metric_val = compute_metric(metric, output, benchmark_output)
  # 根据metric_val, threshold, operator等判断精度测试是否成功，并生成test_results文件需要的对应json格式字符串
  test_results = generate_result_file(metric, metric_val, threshold, operator)
  file_writer("test_results.json", test_results)
  
  upload_files_to_obs("output.txt", "log.log", "test_results.json")
  
```



### OBS文件存放路径

```apl
/{root_path}/
├──results/
│  ├── {accuracy_test_id}/
│  │  ├── test_results.json
│  │  ├── output.txt
│  │  └── log.log
└──baselines/
   ├── {model_id}/
    	├── inference/
    	│	 ├── {baseline_id}/
    	│	 │  ├── input.txt
    	│	 │  └── output.txt
      └── training/
         ├── {baseline_id}/
         │   └── loss.jsonl
```





## ci-adapter判断精度测试结果

### 内存状态位与执行测状态位对照表

内存内通过flex-compute sdk返回的任务状态对是否通过精度测试进行判断

| ci-adapter内存处状态位 | flex-compute job状态位       |
| ---------------------- | ---------------------------- |
| success                | runningSuccess               |
| failed                 | downloadFailed/runningFailed |
| running                | running                      |
| stopped                | stopped                      |
|                        |                              |



### 测试真实结果判断逻辑

由于flex-compute sdk返回的job运行完的状态只有runningSuccess和runningFailed，没有更加细粒度的状态划分。因此当前设计为ci-adapter中查询到job状态为runningSuccess后，即表示流程跑通+与标杆比较成功

- flex-compute的runningSuccess意味着流程跑通+与标杆比较成功，与精度测试是否合格无关。
- flex-compute返回runningFailed意味着3个流程中有一个失败（下载失败，推理失败，对比测试失败）
- flex-compute返回为runningFailed时：
  - 下载失败/推理失败：
    - comparison_details = nil
    - output = nil
    - passed = nil
    - log = "/path/to/your/log"
  - 对比失败：
    - comparison_detals  不为空
    - passed = false
    - output = "/path/to/your/output"
    - Log = "/path/to/your/log"



```go
"{
  "baseline_id": "baseline_123",
  "is_comparison_test": true,
  "test_type": "inference",
  "stage": "test",
  "output": "output.txt",
  "log": "execution.log",
  "passed": true,
  "comparison_details": {
    "metric": "BLEU",
    "metric_value": 0.85,
    "threshold": 0.80,
    "operator": ">="
  }
}"


// 当flex-compute返回job状态为success时，调用该方
func (ami *AccuracyModelInfo) accuracy_test_inspector() {
  // 内存中的status只有 success, failed(runningFailed and downloadFailed included), running, stopped
  test_result = download(/obs_endpoint/accuracy_test_id/test_result.json)	

  ami.TaskInfo{
    ReportURL: test_result.log
    OutputURL: test_result.output
    MetricValue: test_result.comparison_details.metric_value
    Passed: test_result.passed
  }
}


// 当flex-compute返回job状态为failed时，调用该方法
func (ami *AccuracyModelInfo) IsRunningFailure() {
  test_result = download(/obs_endpoint/accuracy_test_id/test_result.json)	
  // 如果test_result.json存在，说明执行侧全流程运行成功，为精度对比失败
  if isFileExisted() {
    ami.TaskInfo{
      OutputURL: test_result.output
      MetricValue: test_result.comparison_details.metric_value
      Passed: test_result.passed
    }
  } else {
    // outputURL, metricValue, Passed等置空，只返回log URL
    ami.TaskInfo{
      reportURL: reportURL
    }
  }
 }
}
```





