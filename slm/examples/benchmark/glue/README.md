# GLUE Benchmark

[GLUE](https://gluebenchmark.com/)是当今使用最为普遍的自然语言理解评测基准数据集，评测数据涵盖新闻、电影、百科等许多领域，其中有简单的句子，也有困难的句子。其目的是通过公开的得分榜，促进自然语言理解系统的发展。详细可参考 [GLUE 论文](https://openreview.net/pdf?id=rJ4km2R5t7)

本项目是 GLUE 评测任务 在 Paddle 2.0上的开源实现。

本项目支持 BERT, ELECTRA,ERNIE,ALBERT,RoBERTa 模型，可在 model_type 中进行指定。

## 快速开始

### 启动 GLUE 任务
以 GLUE/SST-2 任务为例，启动 GLUE 任务进行 Fine-tuning 的方式如下：

#### 单卡训练
```shell
export CUDA_VISIBLE_DEVICES=0
export TASK_NAME=SST-2

python -u ./run_glue.py \
    --model_name_or_path bert-base-uncased \
    --tokenizer_name_or_path bert-base-uncased \
    --task_name $TASK_NAME \
    --max_seq_length 128 \
    --batch_size 32   \
    --learning_rate 3e-5 \
    --num_train_epochs 3 \
    --logging_steps 1 \
    --save_steps 100 \
    --output_dir ./tmp/$TASK_NAME/ \
    --device gpu

```

#### 多卡训练
```shell
unset CUDA_VISIBLE_DEVICES
export TASK_NAME=SST-2

python -m paddle.distributed.launch --gpus "0,1" run_glue.py \
    --model_name_or_path bert-base-uncased \
    --tokenizer_name_or_path bert-base-uncased \
    --task_name $TASK_NAME \
    --max_seq_length 128 \
    --batch_size 32   \
    --learning_rate 3e-5 \
    --num_train_epochs 3 \
    --logging_steps 1 \
    --save_steps 100 \
    --output_dir ./tmp/$TASK_NAME/ \
    --device gpu

```
其中参数释义如下：
- `model_name_or_path` 指示了 Fine-tuning 使用的具体预训练模型，可以是 PaddleNLP 提供的预训练模型 或者 本地的预训练模型。如果使用本地的预训练模型，可以配置本地模型的目录地址，例如: /home/xx_model/，目录中需包含 paddle 预训练模型 model_state.pdparams。
如果使用 PaddleNLP 提供的预训练模型，可以选择`model_type`在[Transformer 预训练模型汇总](https://paddlenlp.readthedocs.io/zh/latest/model_zoo/index.html#transformer) 中相对应的英文预训练权重。注意这里选择的模型权重要和上面配置的模型类型匹配，例如 model_type 配置的是 bert，则 model_name_or_path 只能选择 bert 相关的模型。另，glue 任务应选择英文预训练权重。
- `tokenizer_name_or_path` 指示了 Fine-tuning 使用的具体 tokenizer，一般保持和 model_name_or_path 一致，也可以单独指定
- `task_name` 表示 Fine-tuning 的任务，当前支持 CoLA、SST-2、MRPC、STS-B、QQP、MNLI、QNLI、RTE。
- `max_seq_length` 表示最大句子长度，超过该长度将被截断。
- `batch_size` 表示每次迭代**每张卡**上的样本数目。
- `learning_rate` 表示基础学习率大小，将于 learning rate scheduler 产生的值相乘作为当前学习率。
- `num_train_epochs` 表示训练轮数。
- `logging_steps` 表示日志打印间隔。
- `save_steps` 表示模型保存及评估间隔。
- `output_dir` 表示模型保存路径。
- `device` 表示训练使用的设备, 'gpu'表示使用 GPU, 'xpu'表示使用百度昆仑卡, 'cpu'表示使用 CPU。

Fine-tuning 过程将按照 `logging_steps` 和 `save_steps` 的设置打印如下日志：

```
global step 6310/6315, epoch: 2, batch: 2099, rank_id: 0, loss: 0.035772, lr: 0.0000000880, speed: 3.1527 step/s
global step 6311/6315, epoch: 2, batch: 2100, rank_id: 0, loss: 0.056789, lr: 0.0000000704, speed: 3.4201 step/s
global step 6312/6315, epoch: 2, batch: 2101, rank_id: 0, loss: 0.096717, lr: 0.0000000528, speed: 3.4694 step/s
global step 6313/6315, epoch: 2, batch: 2102, rank_id: 0, loss: 0.044982, lr: 0.0000000352, speed: 3.4513 step/s
global step 6314/6315, epoch: 2, batch: 2103, rank_id: 0, loss: 0.139579, lr: 0.0000000176, speed: 3.4566 step/s
global step 6315/6315, epoch: 2, batch: 2104, rank_id: 0, loss: 0.046043, lr: 0.0000000000, speed: 3.4590 step/s
eval loss: 0.549763, acc: 0.9151376146788991, eval done total : 1.8206987380981445 s
```

使用各种预训练模型进行 Fine-tuning ，在 GLUE 验证集上有如下结果：

| Model GLUE Score   | CoLA  | SST-2  | MRPC   | STS-B  | QQP    | MNLI   | QNLI   | RTE    |
|--------------------|-------|--------|--------|--------|--------|--------|--------|--------|
| electra-small      | 58.22 | 91.85  | 88.24  | 87.24  | 88.83  | 82.45  | 88.61  | 66.78  |
| ernie-2.0-large-en | 65.4  | 96.0   | 88.7   | 92.3   | 92.5   | 89.1   | 94.3   | 85.2   |

关于 GLUE Score 的说明：
1. 因 Fine-tuning 过程中有 dropout 等随机因素影响，同样预训练模型每次运行的 GLUE Score 会有较小差异，上表中的 GLUE Score 是运行多次取 eval 最好值的得分。
2. 不同 GLUE 任务判定得分所使用的评价指标有些差异，简单如下表，详细说明可参考[GLUE 论文](https://openreview.net/pdf?id=rJ4km2R5t7)。

| GLUE Task  | Metric                       |
|------------|------------------------------|
| CoLA       | Matthews corr                |
| SST-2      | acc.                         |
| MRPC       | acc./F1                      |
| STS-B      | Pearson/Spearman corr        |
| QQP        | acc./F1                      |
| MNLI       | matched acc./mismatched acc. |
| QNLI       | acc.                         |
| RTE        | acc.                         |

#### trainer 版本

```shell
export task_name=mnli
export learning_rate=5e-5

python run_glue_trainer.py \
--model_name_or_path roberta-large \
--task_name $task_name \
--do_train \
--do_eval \
--max_seq_length 512 \
--per_device_train_batch_size 16 \
--per_device_eval_batch_size 64 \
--learning_rate $learning_rate \
--num_train_epochs 10 \
--output_dir ./checkpoints/$task_name/ft \
--overwrite_output_dir \
--logging_steps 10 \
--evaluation_strategy epoch \
--save_strategy epoch \
--warmup_ratio 0.06 \
--seed 0 \
--weight_decay 0.1 \
--disable_tqdm True
```
