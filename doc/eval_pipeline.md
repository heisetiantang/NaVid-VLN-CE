# Uni-NaVid 评测链路文档

> 本文档描述从训练完成到多实验横向对比的完整评测流程。

---

## 目录

- [整体链路概览](#整体链路概览)
- [第一步：训练完成，保存 checkpoint](#第一步训练完成保存-checkpoint)
- [第二步：运行评测脚本](#第二步运行评测脚本)
- [第三步：分析单个实验结果](#第三步分析单个实验结果)
- [第四步：多实验横向对比](#第四步多实验横向对比)
- [命名约定](#命名约定)
- [常见问题](#常见问题)

---

## 整体链路概览

```
Uni-NaVid 训练
      ↓
  checkpoint 保存至 model_zoo/xxx
      ↓
  eval_uninavid_vlnce.sh
  （指定 MODEL_PATH + SAVE_PATH，多卡并行推理）
      ↓
  tmp/xxx/log/*.json
  （每条轨迹对应一个 json，含 success / spl / distance_to_goal 等字段）
      ↓
  analyze_results.py --path tmp/xxx        ← 单个实验
  compare_results.py --tmp-dir tmp         ← 多实验对比（见第四步）
      ↓
  终端打印 SR / OSR / SPL / DTG / PL 指标表
```

---

## 第一步：训练完成，保存 checkpoint

训练完成后，checkpoint 默认保存在 Uni-NaVid 仓库的 `output/` 目录下。
将需要评测的 checkpoint 复制（或软链接）到本目录的 `model_zoo/` 下，并使用描述性名称：

```bash
# 示例
cp -r ~/Uni-NaVid/output/uninavid_improve1_llama31  model_zoo/

# 或使用软链接（节省磁盘）
ln -s ~/Uni-NaVid/output/uninavid_improve1_llama31  model_zoo/uninavid_improve1_llama31
```

### 推荐命名格式

```
model_zoo/
  uninavid_baseline                 ← 原始 Vicuna-7B 模型
  uninavid_improve1_llama31         ← 改进 1：LLaMA-3.1-8B
  uninavid_improve5_phi35           ← 改进 5：Phi-3.5-mini
  uninavid_improve7_qwen25          ← 改进 7：Qwen2.5-7B
  uninavid_improve2_dpo             ← 改进 2：SFT + DPO
  uninavid_improve8_siglip          ← 改进 8：SigLIP 视觉编码器
  ...
```

---

## 第二步：运行评测脚本

编辑 `eval_uninavid_vlnce.sh`，修改以下三个变量：

```bash
MODEL_PATH="model_zoo/uninavid_improve1_llama31"   # checkpoint 路径
SAVE_PATH="tmp/results_improve1_llama31_r2r"        # 结果保存路径
CONFIG_PATH="VLN_CE/vlnce_baselines/config/r2r_baselines/uninavid_r2r.yaml"  # R2R 或 RxR
```

然后运行：

```bash
bash eval_uninavid_vlnce.sh
```

脚本会按 `CHUNKS` 数量将数据集切分，每个切片分配一块 GPU 并行推理，全部完成后退出。

### 参数说明

| 参数 | 说明 |
|---|---|
| `CHUNKS` | 并行 GPU 数量，默认 2，按实际 GPU 数量调整 |
| `MODEL_PATH` | checkpoint 目录（含 `config.json` 和权重文件） |
| `SAVE_PATH` | 推理结果保存目录，自动创建 `log/` 子目录 |
| `CONFIG_PATH` | 评测配置文件，R2R 和 RxR 分别对应不同 yaml |
| `EXP_SAVE` | `"video-data"` 使用视频帧；`"data"` 加速但无视频 |
| `MODEL_NAME` | `"uni-navid"`（Uni-NaVid）或 `"navid"`（NaVid） |

### 结果目录结构

```
tmp/results_improve1_llama31_r2r/
  log/
    episode_0001.json
    episode_0002.json
    ...
```

每个 json 含字段：`success`, `spl`, `distance_to_goal`, `oracle_success`, `path_length`。

---

## 第三步：分析单个实验结果

```bash
python analyze_results.py --path tmp/results_improve1_llama31_r2r
```

输出示例：

```
Success rate: 516/783 (0.659)
Oracle success rate: 581/783 (0.742)
SPL: 472.1/783 (0.603)
Distance to goal: 2.08
Path length: 10.23
```

### 指标说明

| 指标 | 全称 | 含义 | 越高/低越好 |
|---|---|---|---|
| SR | Success Rate | 成功到达目标的比例 | 越高越好 |
| OSR | Oracle Success Rate | 轨迹中任意时刻到达过目标 | 越高越好 |
| SPL | Success weighted by Path Length | 兼顾成功率与路径效率 | 越高越好 |
| DTG | Distance to Goal | 终点距目标的平均距离（米） | 越低越好 |
| PL | Path Length | 平均路径长度（米） | 参考值 |

---

## 第四步：多实验横向对比

使用 `compare_results.py` 一次性对比所有实验：

```bash
# 方式 1：手动指定要对比的目录
python compare_results.py \
  --paths tmp/results_baseline_r2r \
          tmp/results_improve1_llama31_r2r \
          tmp/results_improve5_phi35_r2r \
          tmp/results_improve7_qwen25_r2r

# 方式 2：自动扫描 tmp/ 下所有含 log/ 的目录
python compare_results.py --tmp-dir tmp

# 方式 3：指定排序字段（默认 SR 降序）
python compare_results.py --tmp-dir tmp --sort spl
```

输出示例：

```
--------------------------------------------------------------------------------
实验名称                              N      SR    Δ SR     OSR     SPL   Δ SPL     DTG       PL
--------------------------------------------------------------------------------
results_baseline_r2r (base)         783   0.621    —      0.714   0.572    —      2.31    10.45
results_improve1_llama31_r2r        783   0.659  +0.038   0.742   0.603  +0.031   2.08    10.23
results_improve5_phi35_r2r          783   0.635  +0.014   0.723   0.581  +0.009   2.24    10.37
results_improve7_qwen25_r2r         783   0.648  +0.027   0.735   0.594  +0.022   2.15    10.31
--------------------------------------------------------------------------------
排序依据：SR（降序）
Δ 列相对于第一行（基准）计算。
```

> `compare_results.py` 尚未创建，待确认需求后统一实现。

---

## 命名约定

为保持实验可追溯，建议 checkpoint 目录名与结果目录名保持一致的描述性后缀：

```
改进编号_改进描述_benchmark
```

示例：

| checkpoint | 结果目录 |
|---|---|
| `model_zoo/uninavid_baseline` | `tmp/results_baseline_r2r` |
| `model_zoo/uninavid_improve1_llama31` | `tmp/results_improve1_llama31_r2r` |
| `model_zoo/uninavid_improve2_dpo` | `tmp/results_improve2_dpo_r2r` |
| `model_zoo/uninavid_improve5_phi35` | `tmp/results_improve5_phi35_r2r` |
| `model_zoo/uninavid_improve7_qwen25` | `tmp/results_improve7_qwen25_r2r` |
| `model_zoo/uninavid_improve8_siglip` | `tmp/results_improve8_siglip_r2r` |

RxR benchmark 将后缀 `_r2r` 替换为 `_rxr`。

---

## 常见问题

**Q：CHUNKS 设置多少合适？**
建议等于可用 GPU 数量。脚本内 `CUDA_VISIBLE_DEVICES=$(( IDX % 8 ))` 自动轮换 GPU 0~7，超过 8 卡需手动调整。

**Q：评测中途中断了怎么办？**
已生成的 `log/*.json` 不会被覆盖，可以直接重新运行脚本，脚本会跳过（实际上会重复写入），或先手动清空 `SAVE_PATH/log/` 再重跑。

**Q：analyze_results.py 打印的 SPL 第一个数字是什么？**
是所有轨迹的 SPL 之和（非平均），括号内才是平均值。`compare_results.py` 统一使用平均值。

**Q：R2R 和 RxR 结果可以放在同一张对比表里吗？**
不建议混放，两个 benchmark 的路线数量和难度不同，Δ 没有意义。应分别跑 `compare_results.py`。
