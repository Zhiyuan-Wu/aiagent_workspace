# 多模型集成预测效果实验计划

## 1. 背景与动机

### 1.1 问题定义

在量化投资策略研究中，单一模型的预测能力往往存在局限性。多模型集成（Ensemble）通过组合多个基模型的预测结果，有望获得更稳定、更准确的预测信号。然而，集成策略的设计需要考虑多个维度：

- **模型多样性**：不同架构、不同损失函数、不同训练策略的模型
- **预测目标多样性**：不同预测周期的模型（短/中/长期）
- **学习范式多样性**：Pointwise回归 vs Pairwise排序学习
- **集成方法多样性**：简单平均、加权平均、Stacking、Rank-based集成等

### 1.2 Alpha Mining 项目架构特点

**策略插件架构**（位于 `backend/scripts/strategies/`）：

```
BaseStrategy (抽象基类)
├── PointwiseStrategy (LGBM + MyAlpha数据处理器)
└── PairwiseRankStrategy (LGBM/MLP/MLP-Residual/Twin-Tower + 锦标赛排序)
```

**Pointwise 特点**：
- 直接回归预测股票未来收益率
- 使用 LGBModel（Qlib原生）
- 输出：每只股票的预测得分（score）
- 推理模式：`pointwise_direct`（直接排序）

**Pairwise 特点**：
- 通过股票对比较学习排序关系
- 支持多种模型架构：LGBM、MLP、MLP-Residual、Twin-Tower
- 支持多种排序方法：single_elim、double_elim、random_matching、hybrid、adaptive
- 输出：经过锦标赛排序的最终得分
- 推理模式：`pairwise_tournament`（锦标赛排序）

### 1.3 实验目标

系统性地研究多模型集成对预测效果的改善，具体目标：

1. **验证集成价值**：量化集成策略相对单一模型的提升幅度
2. **探索最优集成配置**：找到针对本项目架构的最佳集成方法
3. **理解集成机制**：分析不同维度多样性的贡献度
4. **评估鲁棒性**：测试集成策略在不同市场环境下的稳定性
5. **形成生产实践指南**：产出可直接应用于模拟盘/实盘的集成配置推荐

---

## 2. 实验设计原则

### 2.1 基线策略

**单一基线**：
- Pointwise LGBM（当前项目默认配置）
- 使用默认模型参数和交易方法配置

**评估维度**：
- IC（Information Coefficient）：预测与真实收益的相关性
- Rank IC：排序相关性（集成的主要优化目标）
- 年化收益率（Annual Return）
- 夏普比率（Sharpe Ratio）
- 最大回撤（Max Drawdown）
- ICIR（IC Information Ratio）

### 2.2 实验分层设计

```
Layer 1: 基础集成可行性验证
    └── 实验 1-2：同架构多随机种子集成

Layer 2: 模型多样性集成
    ├── 实验 3-4：不同损失函数集成
    ├── 实验 5-6：不同模型架构集成（Pairwise内部）
    └── 实验 7-8：Pointwise + Pairwise 混合集成

Layer 3: 预测目标多样性集成
    ├── 实验 9-10：多预测周期集成
    └── 实验 11-12：多股票池集成

Layer 4: 集成方法优化
    ├── 实验 13-15：权重学习策略
    ├── 实验 16-18：高级集成方法
    └── 实验 19-20：动态集成策略

Layer 5: 综合最优集成
    └── 实验 21：最终配置验证
```

### 2.3 模型复用策略

为避免重复训练，采用预训练模型库：

```
data/models/ensemble/
├── pointwise/
│   ├── pw_seed42_lgbm.pkl
│   ├── pw_seed123_lgbm.pkl
│   └── pw_seed456_lgbm.pkl
├── pairwise_lgbm/
│   ├── pwlgbm_seed42_single.pkl
│   ├── pwlgbm_seed42_adaptive.pkl
│   └── ...
├── pairwise_mlp/
│   └── ...
└── multi_horizon/
    ├── horizon_2d_lgbm.pkl
    ├── horizon_5d_lgbm.pkl
    ├── horizon_15d_lgbm.pkl
    └── horizon_30d_lgbm.pkl
```

**训练阶段**：
```bash
# 批量训练基模型
./.venv/bin/python backend/scripts/train_ensemble_models.py \
    --config configs/ensemble/base_models.yaml
```

**集成阶段**：
```bash
# 加载预训练模型并测试集成策略
./.venv/bin/python backend/scripts/backtest_service.py \
    --task_id ensemble_exp01 \
    --config configs/ensemble/exp01_simple_avg.yaml
```

### 2.4 评估协议

**时间划分**：
- 训练集：2008-01-01 至 2014-12-31
- 验证集：2015-01-01 至 2016-12-31（用于权重学习）
- 测试集：2017-01-01 至 2020-08-01（用于最终评估）

**统计显著性检验**：
- 使用 Diebold-Mariano 检验比较集成策略与基线的显著性
- 计算改进的95%置信区间

**可复现性**：
- 固定随机种子：数据划分、模型初始化、锦标赛排序
- 记录完整配置：模型参数、集成参数、交易方法参数
- 版本控制：代码、依赖库版本

---

## 3. 实验列表

### 实验 0: 基线建立与预训练模型库构建

**目标**：建立性能基线，构建预训练模型库。

**基线配置**：
```python
baseline_config = {
    "strategy_type": "pointwise",
    "stock_pool": "csi300",
    "benchmark": "SH000300",
    "data_range": {"start_date": "2008-01-01", "end_date": "2020-08-01"},
    "train_segments": {
        "train_start": "2008-01-01", "train_end": "2014-12-31",
        "valid_start": "2015-01-01", "valid_end": "2016-12-31",
        "test_start": "2017-01-01", "test_end": "2020-08-01",
    },
    "model_params": {
        "loss": "mse", "learning_rate": 0.2, "num_leaves": 210,
        "max_depth": 8, "subsample": 0.8789, "colsample_bytree": 0.8879,
        "lambda_l1": 205.6999, "lambda_l2": 580.9768, "num_threads": 8,
    },
    "trading_method": {"type": "equal_weight", "params": {"topk": 50, "n_drop": 5}},
}
```

**预训练模型库**：

| 模型ID | 策略类型 | 模型类型 | 排序方法 | 损失函数 | 随机种子 |
|--------|----------|----------|----------|----------|----------|
| PW-01 | pointwise | lgbm | direct | mse | 42 |
| PW-02 | pointwise | lgbm | direct | mse | 123 |
| PW-03 | pointwise | lgbm | direct | mae | 42 |
| PW-04 | pointwise | lgbm | direct | huber | 42 |
| PWL-01 | pairwise_rank | lgbm | single_elim | bce | 42 |
| PWP-05 | pairwise_rank | lgbm | adaptive | bce | 42 |
| PWM-01 | pairwise_rank | mlp | adaptive | bce | 42 |
| PWM-02 | pairwise_rank | mlp_residual | adaptive | bce | 42 |
| PWM-03 | pairwise_rank | twin_tower | adaptive | bce | 42 |
| H-2D | pointwise | lgbm | direct | mse | 42 |
| H-5D | pointwise | lgbm | direct | mse | 42 |
| H-15D | pointwise | lgbm | direct | mse | 42 |
| H-30D | pointwise | lgbm | direct | mse | 42 |

**输出**：
- 基线性能报告
- 预训练模型库（`data/models/ensemble/`）
- 各模型的预测结果文件（`data/predictions/ensemble/`）

---

### 实验 1: 同架构多随机种子简单平均集成

**目标**：验证最基本的集成策略是否有效。

**集成配置**：
```python
ensemble_config = {
    "name": "exp01_simple_avg_same_seeds",
    "base_models": ["PW-01", "PW-02", "PW-03"],  # 3个不同种子的Pointwise LGBM
    "method": "simple_average",  # 简单平均
    "weight_learning": None,  # 不学习权重
}
```

**预期**：
- 预期有轻微提升（降低方差）
- Rank IC 应有所改善
- 年化收益率可能有小幅提升

**评估**：
- 相对基线的改进百分比
- 改进的统计显著性（Diebold-Mariano test）

---

### 实验 2: 同架构多随机种子加权平均集成

**目标**：探索是否可以通过学习权重改善集成效果。

**集成配置**：
```python
ensemble_config = {
    "name": "exp02_weighted_avg_same_seeds",
    "base_models": ["PW-01", "PW-02", "PW-03"],
    "method": "weighted_average",
    "weight_learning": {
        "method": "ic_based",  # 基于验证集IC加权
        "normalization": "softmax",  # softmax归一化
        "temperature": 1.0,  # 温度参数
    },
}
```

**权重计算方法**：

| 方法 | 公式 | 说明 |
|------|------|------|
| equal | $w_i = 1/N$ | 等权（实验1） |
| ic_based | $w_i \propto IC_i$ | 按验证集IC加权 |
| rank_ic_based | $w_i \propto RankIC_i$ | 按验证集Rank IC加权 |
| sharpe_based | $w_i \propto Sharpe_i$ | 按验证集夏普比率加权 |
| exp_ic | $w_i \propto \exp(IC_i / \tau)$ | IC的softmax，温度$\tau$可调 |

**预期**：
- IC_based 加权应优于简单平均
- 权重分布可能不均衡（性能好的模型获得更高权重）

---

### 实验 3: 不同损失函数集成（Pointwise）

**目标**：探索不同损失函数学习到的互补性。

**集成配置**：
```python
ensemble_config = {
    "name": "exp03_loss_function_ensemble",
    "base_models": [
        "PW-01",  # MSE
        "PW-03",  # MAE
        "PW-04",  # Huber
    ],
    "method": "weighted_average",
    "weight_learning": {"method": "ic_based"},
}
```

**损失函数特点分析**：

| 损失函数 | 优化目标 | 预期特性 |
|----------|----------|----------|
| MSE | 均方误差 | 对异常值敏感，拟合均值 |
| MAE | 平均绝对误差 | 对异常值鲁棒，拟合中位数 |
| Huber | 混合损失 | 兼顾鲁棒性与效率 |
| Quantile | 分位数回归 | 直接优化特定分位数 |

**预期**：
- MSE + MAE 组合可能有互补性（稳健均值 + 鲁棒中位数）
- Huber 可能起到平衡作用

---

### 实验 4: 不同排序方法集成（Pairwise LGBM）

**目标**：评估Pairwise内部不同排序方法的集成价值。

**集成配置**：
```python
ensemble_config = {
    "name": "exp04_sorting_method_ensemble",
    "base_models": [
        "PWL-01",  # single_elimination
        "PWL-02",  # double_elimination
        "PWL-03",  # random_matching
        "PWL-04",  # hybrid
        "PWP-05",  # adaptive（优化后）
    ],
    "method": "weighted_average",
    "weight_learning": {"method": "rank_ic_based"},
}
```

**排序方法特点**：

| 方法 | 特点 | 计算成本 |
|------|------|----------|
| single_elim | 快速，单次比较 | 低 |
| double_elim | 更稳定，双次确认 | 中 |
| random_matching | 随机覆盖均匀 | 中 |
| hybrid | 结合多种方法 | 中高 |
| adaptive | 自适应边界聚焦 | 高 |

**预期**：
- 不同方法捕捉不同的排序关系
- Adaptive 应该获得较高权重（如果调优得当）

---

### 实验 5: 不同模型架构集成（Pairwise）

**目标**：探索深度学习模型与GBDT模型的互补性。

**集成配置**：
```python
ensemble_config = {
    "name": "exp05_architecture_ensemble",
    "base_models": [
        "PWP-05",  # LGBM + adaptive
        "PWM-01",  # MLP + adaptive
        "PWM-02",  # MLP-Residual + adaptive
        "PWM-03",  # Twin-Tower + adaptive
    ],
    "method": "weighted_average",
    "weight_learning": {"method": "ic_based"},
}
```

**模型架构对比**：

| 架构 | 表达能力 | 训练速度 | 过拟合风险 |
|------|----------|----------|------------|
| LGBM | 中（树集成） | 快 | 低 |
| MLP | 高（非线性） | 中 | 中 |
| MLP-Residual | 很高（残差连接） | 中 | 中高 |
| Twin-Tower | 很高（双塔交互） | 慢 | 高 |

**预期**：
- 树模型与神经网络可能有互补性
- LGBM可能提供稳定的基线，神经网络可能提供提升空间

---

### 实验 6: Pointwise + Pairwise 混合集成

**目标**：评估不同学习范式的集成价值。

**集成配置**：
```python
ensemble_config = {
    "name": "exp06_pointwise_pairwise_ensemble",
    "base_models": [
        "PW-01",   # Pointwise LGBM
        "PWP-05",  # Pairwise LGBM + adaptive
    ],
    "method": "weighted_average",
    "weight_learning": {"method": "ic_based"},
}
```

**学习范式对比**：

| 范式 | 优化目标 | 预测特性 |
|------|----------|----------|
| Pointwise | 回归误差（MSE/MAE） | 预测绝对收益值 |
| Pairwise | 排序误差（BCE） | 预测相对排序 |

**理论分析**：
- Pointwise 关注预测值准确性
- Pairwise 关注排序准确性（更接近投资目标）
- 两者可能有互补性

**预期**：
- Pairwise 应该在 Rank IC 上占优
- Pointwise 可能在绝对收益预测上有优势
- 集成可能结合两者优点

---

### 实验 7: 全模型大集成

**目标**：探索包含所有维度多样性的大规模集成。

**集成配置**：
```python
ensemble_config = {
    "name": "exp07_full_ensemble",
    "base_models": [
        # Pointwise族
        "PW-01", "PW-02", "PW-03", "PW-04",
        # Pairwise LGBM族
        "PWL-01", "PWL-02", "PWL-03", "PWL-04", "PWP-05",
        # Pairwise深度学习族
        "PWM-01", "PWM-02", "PWM-03",
    ],
    "method": "weighted_average",
    "weight_learning": {"method": "ic_based", "temperature": 0.5},
}
```

**预期**：
- 可能达到最佳性能
- 但计算成本和复杂度显著增加
- 需要评估性价比（性能提升 vs 复杂度增加）

---

### 实验 8: 稀疏集成（正则化权重）

**目标**：探索是否可以通过稀疏正则化简化集成。

**集成配置**：
```python
ensemble_config = {
    "name": "exp08_sparse_ensemble",
    "base_models": [与实验7相同],
    "method": "weighted_average",
    "weight_learning": {
        "method": "ic_based",
        "sparsity": "l1",  # L1正则化
        "lambda": 0.1,     # 正则化强度
    },
}
```

**稀疏化方法**：

| 方法 | 实现 | 效果 |
|------|------|------|
| top_k | 仅保留前k个权重 | 硬截断 |
| l1 | L1正则化 | 软稀疏 |
| threshold | 截断低于阈值的权重 | 硬截断 |

**预期**：
- 可能识别出最重要的几个子模型
- 在性能和复杂度之间取得平衡

---

### 实验 9: 多预测周期集成（固定权重）

**目标**：探索不同预测周期模型的集成价值。

**预测周期配置**：
```python
horizon_configs = {
    "2d": {"label_shift": 2, "model_id": "H-2D"},
    "5d": {"label_shift": 5, "model_id": "H-5D"},
    "15d": {"label_shift": 15, "model_id": "H-15D"},
    "30d": {"label_shift": 30, "model_id": "H-30D"},
}
```

**集成配置**：
```python
ensemble_config = {
    "name": "exp09_multi_horizon_fixed",
    "base_models": ["H-2D", "H-5D", "H-15D", "H-30D"],
    "method": "weighted_average",
    "weight_learning": {"method": "ic_based"},
}
```

**理论分析**：
- 短期模型（2d/5d）：捕捉短期动量/反转
- 中期模型（15d）：捕捉波段趋势
- 长期模型（30d）：捕捉长期基本面

**预期**：
- 不同周期可能捕捉不同时间尺度的alpha
- 短期模型可能更适合高频调仓
- 长期模型可能提供稳定信号

---

### 实验 10: 多预测周期集成（动态权重）

**目标**：探索权重是否应该随市场状态动态调整。

**市场状态划分**：
```python
market_regimes = {
    "bull": "price_ma20 > price_ma60 and volume_ratio > 1.2",
    "bear": "price_ma20 < price_ma60 and volume_ratio > 1.2",
    "neutral": "otherwise",
}
```

**集成配置**：
```python
ensemble_config = {
    "name": "exp10_multi_horizon_dynamic",
    "base_models": ["H-2D", "H-5D", "H-15D", "H-30D"],
    "method": "weighted_average",
    "weight_learning": {
        "method": "conditional",
        "condition": "market_regime",
        "weights": {
            "bull": [0.1, 0.3, 0.4, 0.2],    # 牛市：偏中长期
            "bear": [0.4, 0.3, 0.2, 0.1],    # 熊市：偏短期
            "neutral": [0.25, 0.25, 0.25, 0.25],  # 震荡：均衡
        },
    },
}
```

**预期**：
- 动态权重可能适应市场环境变化
- 牛市可能更关注中长期趋势
- 熊市可能更需要快速调整

---

### 实验 11: Rank-based集成（排序级别融合）

**目标**：探索基于排序的集成方法。

**集成配置**：
```python
ensemble_config = {
    "name": "exp11_rank_ensemble",
    "base_models": ["PW-01", "PWP-05", "PWM-01"],
    "method": "rank_fusion",
    "rank_method": "borda_count",  # Borda计数法
}
```

**Rank融合方法**：

| 方法 | 公式 | 说明 |
|------|------|------|
| borda_count | $score_i = \sum_{m} (N - rank_i^m)$ | 累加反向排名 |
| condorcet | 两两比较投票 | 寻找孔塞塞赢家 |
| rank_avg | $score_i = \text{mean}(rank_i^m)$ | 平均排名 |
| rank_weighted | $score_i = \sum_{m} w_m \cdot rank_i^m$ | 加权排名 |

**预期**：
- Rank融合可能对异常值更鲁棒
- 可能改善 Rank IC

---

### 实验 12: Stacking集成（元学习器）

**目标**：探索使用元学习器自动学习集成权重。

**架构**：
```
Base Models (Level 0)
    ├── PW-01: pred_1
    ├── PWP-05: pred_2
    └── PWM-01: pred_3
         ↓
Meta Features (Level 1)
    ├── pred_1, pred_2, pred_3
    ├── pred_1^2, pred_2^2, pred_3^2
    ├── pred_1 * pred_2, ...
    └── meta_features (market_regime, volatility, etc.)
         ↓
Meta Learner (Level 2)
    └── Linear Regression / Ridge / LightGBM
         ↓
Final Prediction
```

**集成配置**：
```python
ensemble_config = {
    "name": "exp12_stacking",
    "base_models": ["PW-01", "PWP-05", "PWM-01"],
    "method": "stacking",
    "meta_learner": {
        "type": "linear",  # linear / ridge / lgbm
        "feature_engineering": True,  # 是否包含交互特征
    },
}
```

**元学习器选择**：

| 类型 | 优点 | 缺点 |
|------|------|------|
| Linear | 可解释性强 | 线性假设限制 |
| Ridge | 防止过拟合 | 需要调参 |
| LGBM | 捕捉非线性 | 过拟合风险 |
| Neural | 表达能力强 | 数据需求大 |

**预期**：
- Stacking可能自动学习复杂的集成模式
- 需要谨慎防止过拟合

---

### 实验 13: Bagging集成（Bootstrap采样）

**目标**：探索Bootstrap采样对集成效果的影响。

**集成配置**：
```python
ensemble_config = {
    "name": "exp13_bagging",
    "base_models": [],  # 动态生成
    "method": "bagging",
    "bagging_config": {
        "n_models": 10,  # 生成10个Bootstrap模型
        "base_model": "pointwise_lgbm",
        "sample_ratio": 0.8,  # 每次采样80%的训练数据
        "feature_ratio": 0.8,  # 每次采样80%的特征
    },
}
```

**预期**：
- Bootstrap采样增加模型多样性
- 可能改善泛化性能

---

### 实验 14: Boosting集成（顺序优化）

**目标**：探索顺序优化的集成策略。

**架构**：
```python
# 模型1预测残差 → 模型2拟合残差 → 模型3拟合残差2 → ...
pred_1 = Model_1(X)
pred_2 = Model_2(X, residual_1)  # residual_1 = y - pred_1
pred_3 = Model_3(X, residual_2)  # residual_2 = residual_1 - pred_2
final_pred = pred_1 + alpha_1 * pred_2 + alpha_2 * pred_3
```

**集成配置**：
```python
ensemble_config = {
    "name": "exp14_boosting",
    "base_models": [],  # 动态生成
    "method": "boosting",
    "boosting_config": {
        "n_rounds": 3,  # 3轮boosting
        "base_model": "pointwise_lgbm",
        "learning_rate": 0.5,  # 每轮的贡献权重
    },
}
```

**预期**：
- Boosting可能逐步减小系统偏差
- 后续模型专注于修正前期模型的错误

---

### 实验 15: 交叉验证集成（CV Ensemble）

**目标**：探索基于交叉验证的稳定集成策略。

**流程**：
```
Train Data (2008-2016)
    ↓ 5-Fold CV
    ├── Fold 1: Train(80%) → Val(20%) → Model_1
    ├── Fold 2: Train(80%) → Val(20%) → Model_2
    ├── ...
    └── Fold 5: Train(80%) → Val(20%) → Model_5
         ↓
Test Data (2017-2020)
    └── Average(Model_1.predict, Model_2.predict, ...)
```

**集成配置**：
```python
ensemble_config = {
    "name": "exp15_cv_ensemble",
    "method": "cv_average",
    "cv_config": {
        "n_folds": 5,
        "base_model": "pointwise_lgbm",
        "stratified": True,  # 按时间分层
    },
}
```

**预期**：
- CV集成可能更稳定
- 减少单次训练的随机性

---

### 实验 16: 集成剪枝（Ensemble Pruning）

**目标**：从大规模集成中选择最优子集。

**方法**：
```python
ensemble_config = {
    "name": "exp16_ensemble_pruning",
    "base_models": [实验7的全部模型],  # 14个模型
    "method": "pruning",
    "pruning_config": {
        "target_size": 5,  # 剪枝到5个模型
        "selection_method": "forward_selection",  # 前向选择
        "selection_metric": "ic",  # 基于IC选择
    },
}
```

**剪枝方法**：

| 方法 | 策略 | 复杂度 |
|------|------|--------|
| forward_selection | 从空集逐步添加 | O(M^2) |
| backward_elimination | 从全集逐步删除 | O(M^2) |
| genetic_algorithm | 遗传算法搜索 | 可配置 |

**预期**：
- 可能找到性能接近但更精简的子集
- 评估性能 vs 复杂度的权衡

---

### 实验 17: 不确定性加权集成

**目标**：根据模型预测不确定性动态调整权重。

**不确定性估计**：
```python
# 对于LGBM：基于叶子节点样本数的方差估计
uncertainty_lgbm = estimate_prediction_variance(model, X)

# 对于神经网络：MC Dropout
uncertainty_nn = mc_dropout_uncertainty(model, X, n_samples=100)
```

**集成配置**：
```python
ensemble_config = {
    "name": "exp17_uncertainty_weighted",
    "base_models": ["PW-01", "PWP-05", "PWM-01"],
    "method": "uncertainty_weighted",
    "weighting": {
        "formula": "inverse_variance",  # 权重 ∝ 1/方差
        "min_weight": 0.1,  # 最小权重防止除零
    },
}
```

**预期**：
- 不确定性低的模型获得更高权重
- 可能提升预测稳定性

---

### 实验 18: 时间衰减加权集成

**目标**：给予近期表现更好的模型更高权重。

**权重计算**：
```python
# 基于滚动窗口的IC计算
window = 60  # 60天滚动窗口
ic_t = rolling_ic(pred_t, label_t, window=60)

# 时间衰减权重
weight = ic_t * exp(-lambda * days_ago)
```

**集成配置**：
```python
ensemble_config = {
    "name": "exp18_time_decay",
    "base_models": ["PW-01", "PWP-05", "PWM-01"],
    "method": "time_decay_weighted",
    "decay_config": {
        "window": 60,  # 60天IC窗口
        "lambda": 0.05,  # 衰减系数
    },
}
```

**预期**：
- 能够适应模型的性能衰减
- 可能更适应市场环境变化

---

### 实验 19: 多股票池集成

**目标**：探索在不同股票池上训练的模型的泛化性。

**股票池配置**：
```python
pool_configs = {
    "csi300": {"instruments": "csi300", "model_id": "PW-300"},
    "csi500": {"instruments": "csi500", "model_id": "PW-500"},
    "csi800": {"instruments": "csi800", "model_id": "PW-800"},
}
```

**集成配置**：
```python
ensemble_config = {
    "name": "exp19_multi_pool",
    "base_models": ["PW-300", "PW-500", "PW-800"],
    "method": "weighted_average",
    "weight_learning": {"method": "ic_based"},
    "target_pool": "csi300",  # 目标股票池
}
```

**预期**：
- 大股票池（CSI800）的模型可能对子股票池有效
- 小股票池的模型可能更专注于该池的特定模式

---

### 实验 20: 集成分歧度分析

**目标**：量化分析模型间的分歧度，理解集成机制。

**分歧度指标**：
```python
# 1. 预测相关性矩阵
correlation_matrix = np.corrcoef([pred_1, pred_2, pred_3])

# 2. 排序一致性（Kendall's Tau）
kendall_tau = kendalltau(rank_1, rank_2)

# 3. Top-K重叠度
overlap_k = len(set(topk_1) & set(topk_2)) / k

# 4. 集成分歧度（Diversity Score）
diversity = 1 - np.mean(correlation_matrix)
```

**分析内容**：
- 不同维度（架构/损失/排序方法）的模型间分歧度
- 分歧度与集成效果的关系
- 识别冗余模型（高相关性、低分歧度）

**预期**：
- 高分歧度的模型组合可能带来更大集成收益
- 识别出真正互补的模型组合

---

### 实验 21: 最优集成配置综合验证

**目标**：综合前述实验结果，形成最优配置推荐。

**配置选择**：
```python
optimal_config = {
    "name": "exp21_optimal",
    # 从实验1-20中选择最优子集
    "base_models": ["根据实验结果选择"],
    "method": "根据实验结果选择",
    "weight_learning": "根据实验结果选择",
    # 其他优化参数...
}
```

**验证维度**：
1. **样本内验证**（2015-2016）：验证集上的IC/收益
2. **样本外验证**（2017-2020）：测试集上的泛化性能
3. **细分时段分析**：牛市/熊市/震荡市分别表现
4. **鲁棒性测试**：参数扰动敏感性

**输出**：
- 最优集成配置完整参数
- 性能提升详细报告
- 与单一基线的全面对比
- 生产部署建议

---

## 4. 实验执行与评估流程

### 4.1 实验依赖关系

```
Phase 0: 基础准备
    ├── 实验0: 基线+预训练模型库
    └── 输出: 14+个预训练模型

Phase 1: 基础集成验证
    ├── 实验1: 简单平均
    └── 实验2: 加权平均

Phase 2: 模型多样性探索
    ├── 实验3: 损失函数多样性
    ├── 实验4: 排序方法多样性
    ├── 实验5: 架构多样性
    ├── 实验6: 学习范式多样性
    └── 实验7: 全模型大集成

Phase 3: 预测目标多样性
    ├── 实验9: 多周期固定权重
    ├── 实验10: 多周期动态权重
    └── 实验19: 多股票池

Phase 4: 集成方法优化
    ├── 实验8: 稀疏化
    ├── 实验11: Rank融合
    ├── 实验12: Stacking
    ├── 实验13: Bagging
    ├── 实验14: Boosting
    ├── 实验15: CV集成
    ├── 实验16: 剪枝
    ├── 实验17: 不确定性加权
    └── 实验18: 时间衰减

Phase 5: 分析与总结
    ├── 实验20: 分歧度分析
    └── 实验21: 最优配置验证
```

### 4.2 结果记录规范

**每个实验记录**：
```json
{
    "experiment_id": "exp01",
    "name": "simple_average_same_seeds",
    "config": {...},
    "base_models": ["PW-01", "PW-02", "PW-03"],
    "ensemble_method": "simple_average",
    "metrics": {
        "ic": {...},
        "rank_ic": {...},
        "performance": {...}
    },
    "baseline_comparison": {
        "ic_improvement": 0.05,
        "rank_ic_improvement": 0.08,
        "return_improvement": 0.03,
        "sharpe_improvement": 0.12
    },
    "statistical_test": {
        "diebold_mariano": {
            "statistic": 2.34,
            "p_value": 0.019,
            "significant": true
        }
    },
    "computational_cost": {
        "training_time": 0,  // 使用预训练模型
        "inference_time": 1.5,  // 相对基线倍数
        "memory_usage": 3.0  // 相对基线倍数
    },
    "timestamp": "2025-01-XX",
    "reproducibility": {
        "random_seed": 42,
        "code_version": "git_commit_hash",
        "dependencies": {...}
    }
}
```

**结果存储路径**：
```
data/experiments/ensembling/
├── baseline/
│   └── exp00_baseline_results.json
├── basic_ensemble/
│   ├── exp01_simple_avg/
│   │   ├── config.json
│   │   ├── results.json
│   │   └── predictions.parquet
│   └── exp02_weighted_avg/
├── model_diversity/
│   ├── exp03_loss_function/
│   ├── exp04_sorting_method/
│   ├── exp05_architecture/
│   ├── exp06_pointwise_pairwise/
│   └── exp07_full_ensemble/
├── prediction_diversity/
│   ├── exp09_multi_horizon_fixed/
│   ├── exp10_multi_horizon_dynamic/
│   └── exp19_multi_pool/
├── method_optimization/
│   ├── exp08_sparse/
│   ├── exp11_rank_fusion/
│   ├── exp12_stacking/
│   ├── ...
│   └── exp18_time_decay/
├── analysis/
│   ├── exp20_diversity_analysis/
│   └── exp21_optimal_config/
└── summary/
    ├── all_experiments_summary.csv
    ├── performance_comparison.json
    └── final_report.md
```

### 4.3 可视化分析

**关键可视化**：

1. **性能对比雷达图**：各实验的多维指标对比
2. **改进百分比条形图**：相对基线的改进
3. **权重分布热力图**：不同集成方法的权重配置
4. **分歧度相关性散点图**：分歧度 vs 集成收益
5. **时间序列对比图**：基线 vs 各集成策略的累计收益
6. **滚动IC对比图**：不同时段的IC稳定性
7. **模型相关性矩阵**：模型间预测相关性热力图
8. **计算成本 vs 性能散点图**：性价比分析
9. **参数敏感性曲线**：集成超参数的影响
10. **市场环境细分图**：牛市/熊市/震荡市的差异化表现

### 4.4 统计显著性检验

**Diebold-Mariano检验**：
- H0：集成策略与基线预测精度相同
- H1：集成策略与基线预测精度不同
- 显著性水平：α = 0.05

**实现**：
```python
from scipy import stats

def diebold_mariano_test(actual, pred1, pred2):
    """
    Diebold-Mariano检验
    :param actual: 真实收益
    :param pred1: 策略1预测
    :param pred2: 策略2预测
    """
    loss_diff = (actual - pred1)**2 - (actual - pred2)**2
    # 计算自调整标准误
    var = np.var(loss_diff) / len(loss_diff)
    dm_stat = np.mean(loss_diff) / np.sqrt(var)
    p_value = 2 * (1 - stats.norm.cdf(abs(dm_stat)))
    return {
        "statistic": dm_stat,
        "p_value": p_value,
        "significant": p_value < 0.05
    }
```

---

## 5. 集成策略实现架构

### 5.1 Ensemble Strategy 插件

**新建策略**：`backend/scripts/strategies/ensemble_strategy.py`

```python
class EnsembleStrategy(BaseStrategy):
    """
    多模型集成策略

    支持的集成方法：
    - simple_average: 简单平均
    - weighted_average: 加权平均（支持多种权重学习）
    - rank_fusion: 排序融合
    - stacking: Stacking元学习
    - bagging: Bagging集成
    - uncertainty_weighted: 不确定性加权
    - time_decay: 时间衰减加权
    """

    def get_schema(self) -> Dict[str, Any]:
        return {
            "type": "object",
            "properties": {
                "base_models": {
                    "type": "array",
                    "items": {"type": "string"},
                    "description": "基模型ID列表"
                },
                "ensemble_method": {
                    "type": "string",
                    "enum": [
                        "simple_average",
                        "weighted_average",
                        "rank_fusion",
                        "stacking",
                        "bagging",
                        "uncertainty_weighted",
                        "time_decay",
                    ],
                    "default": "weighted_average"
                },
                "weight_learning": {
                    "type": "object",
                    "properties": {
                        "method": {
                            "type": "string",
                            "enum": ["ic_based", "rank_ic_based", "sharpe_based", "exp_ic"]
                        },
                        "temperature": {"type": "number"},
                        "sparsity": {"type": "string", "enum": ["none", "l1", "top_k"]},
                        "lambda": {"type": "number"},
                    }
                },
                "meta_learner": {
                    "type": "object",
                    "properties": {
                        "type": {"type": "string", "enum": ["linear", "ridge", "lgbm"]},
                        "feature_engineering": {"type": "boolean"},
                    }
                },
                # 其他参数...
            },
        }

    def train(self, workdir, parameters):
        """
        加载预训练的基模型
        （训练阶段实际上不做训练，只是验证和准备）
        """
        # 1. 验证所有基模型存在
        # 2. 加载模型配置
        # 3. 如果需要元学习器，在验证集上训练
        pass

    def predict(self, workdir, parameters):
        """
        执行集成预测
        """
        # 1. 加载所有基模型
        # 2. 获取各模型的预测
        # 3. 应用集成方法
        # 4. 返回集成预测结果
        pass

    def backtest(self, workdir, parameters):
        """
        执行集成回测
        """
        # 1. 获取测试集所有基模型预测
        # 2. 应用集成方法
        # 3. 计算集成回测指标
        pass
```

### 5.2 权重学习模块

**新建模块**：`backend/scripts/strategies/ensemble_weights.py`

```python
def learn_weights(predictions_dict, labels, method="ic_based", **kwargs):
    """
    学习集成权重

    :param predictions_dict: {model_id: pd.Series} 预测结果
    :param labels: 真实标签
    :param method: 权重学习方法
    :return: {model_id: weight}
    """
    if method == "ic_based":
        # 基于IC加权
        weights = {m: ic(pred, labels) for m, pred in predictions_dict.items()}
    elif method == "rank_ic_based":
        # 基于Rank IC加权
        weights = {m: rank_ic(pred, labels) for m, pred in predictions_dict.items()}
    elif method == "sharpe_based":
        # 基于夏普比率加权
        weights = {m: sharpe(pred, labels) for m, pred in predictions_dict.items()}
    elif method == "exp_ic":
        # IC的softmax
        temperature = kwargs.get("temperature", 1.0)
        raw_weights = {m: ic(pred, labels) / temperature for m, pred in predictions_dict.items()}
        weights = softmax(raw_weights)

    # 应用稀疏化
    if kwargs.get("sparsity") == "l1":
        weights = apply_l1_sparsity(weights, kwargs.get("lambda", 0.1))
    elif kwargs.get("sparsity") == "top_k":
        weights = apply_top_k_sparsity(weights, kwargs.get("k", 5))

    return weights
```

### 5.3 实验管理脚本

**新建脚本**：`backend/scripts/run_ensemble_experiments.py`

```python
def run_experiment(exp_config, workdir):
    """
    运行单个集成实验
    """
    # 1. 加载实验配置
    # 2. 初始化EnsembleStrategy
    # 3. 执行回测
    # 4. 记录结果
    pass

def run_all_experiments(experiment_list):
    """
    批量运行实验列表
    """
    results = {}
    for exp_id in experiment_list:
        config = load_experiment_config(exp_id)
        results[exp_id] = run_experiment(config, workdir/f"exp_{exp_id}")
    return results

def generate_report(results):
    """
    生成实验报告
    """
    # 1. 汇总所有实验结果
    # 2. 生成对比表格和图表
    # 3. 输出markdown报告
    pass
```

---

## 6. 成功标准

### 6.1 实验完成标准

- [ ] 完成实验0：基线建立与预训练模型库
- [ ] 完成实验1-7：模型多样性集成
- [ ] 完成实验8-10、19：预测目标多样性集成
- [ ] 完成实验11-18：集成方法优化
- [ ] 完成实验20：分歧度分析
- [ ] 完成实验21：最优配置验证

### 6.2 性能提升标准

**最低要求**：
- Rank IC 提升率 > 5%
- 年化收益率提升率 > 3%
- 相对基线的改进具有统计显著性（p < 0.05）

**理想目标**：
- Rank IC 提升率 > 10%
- 年化收益率提升率 > 8%
- 夏普比率提升率 > 15%

### 6.3 产出物标准

**代码产出**：
- [ ] `EnsembleStrategy` 策略插件实现
- [ ] `ensemble_weights.py` 权重学习模块
- [ ] `run_ensemble_experiments.py` 实验管理脚本
- [ ] 单元测试覆盖率 > 80%

**文档产出**：
- [ ] 实验配置文件（`configs/ensemble/`）
- [ ] 实验结果汇总（`data/experiments/ensembling/summary/`）
- [ ] 完整实验报告（markdown）
- [ ] 最优配置部署指南

**可视化产出**：
- [ ] 性能对比图（10+种）
- [ ] 权重分析图
- [ ] 分歧度分析图
- [ ] 时间序列对比图

---

## 7. 后续扩展方向

### 7.1 深度学习集成方法

- **模型蒸馏**：将大型集成蒸馏为单一高效模型
- **神经架构搜索**：自动搜索最优集成架构
- **图神经网络集成**：捕捉股票间关系的图模型集成

### 7.2 在线学习集成

- **Bandit集成**：多臂老虎机动态选择模型
- **增量学习**：模型随新数据动态更新
- **概念漂移检测**：自动识别市场环境变化

### 7.3 可解释性增强

- **SHAP集成解释**：解释集成预测的贡献度
- **注意力机制集成**：学习动态权重的时间注意力
- **因果推断集成**：结合因果关系的集成策略

### 7.4 实盘部署优化

- **推理加速**：模型并行、批处理优化
- **缓存策略**：智能缓存减少重复计算
- **A/B测试框架**：在线验证集成策略

---

## 8. 风险与挑战

### 8.1 技术风险

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| 过拟合 | 集成在测试集表现好但实盘差 | 严格的时间划分、早停、正则化 |
| 计算复杂度 | 推理时间过长影响实盘 | 模型剪枝、并行化、缓存优化 |
| 数据泄露 | 验证集信息泄漏到测试集 | 严格的时间切分、Walk-forward验证 |
| 复杂度增加 | 维护成本高 | 模块化设计、充分文档化 |

### 8.2 实施风险

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| 实验周期长 | 占用大量计算资源 | 并行执行、使用预训练模型 |
| 参数搜索空间大 | 难以找到全局最优 | 分层搜索、贪心策略 |
| 结果不稳定 | 随机性影响结论 | 固定种子、多次重复实验 |

---

## 9. 时间规划

**总工期**：约4-6周

```
Week 1: Phase 0-1
├── 实验0: 基线+预训练模型库 (3天)
├── 实验1-2: 基础集成验证 (2天)
└── 阶段性分析报告

Week 2: Phase 2
├── 实验3-7: 模型多样性探索 (5天)
└── 阶段性分析报告

Week 3: Phase 3
├── 实验9-10、19: 预测目标多样性 (3天)
├── 实验8、11-15: 集成方法优化（第一部分）(2天)
└── 阶段性分析报告

Week 4: Phase 4
├── 实验16-18: 集成方法优化（第二部分）(3天)
├── 实验20: 分歧度分析 (2天)
└── 阶段性分析报告

Week 5-6: Phase 5
├── 实验21: 最优配置验证 (1周)
├── 完整实验报告撰写 (1周)
└── 生产部署指南
```

---

## 10. 参考文献

1. **Ensemble Methods**：
   - Zhou, Z. H. (2012). Ensemble Methods: Foundations and Algorithms.
   - Kaggle Ensembling Guide. (https://mlwave.com/kaggle-ensembling-guide/)

2. **Ranking Ensemble**：
   - Cormack, G. V., et al. (2009). Recombination and rank-based multiple classifiers.
   - Aslam, J. A., et al. (2001). The learning power of voting ensembles.

3. **Financial Applications**：
   - Batchelor, R. A. (1998). Forecasting combining and encompassing.
   - Guillén, M. T. (2021). Machine learning ensemble forecasting.

4. **Pairwise Learning**：
   - Burges, C. (2010). From RankNet to LambdaRank to LambdaMART.
   - Herbrich, R., et al. (2000). Large margin rank boundaries for ordinal regression.
