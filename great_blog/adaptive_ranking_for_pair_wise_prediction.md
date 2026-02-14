# 自适应边界聚焦排序方法实验计划

## 1. 背景与动机

### 1.1 问题定义
在 pair-wise 排序学习框架中，需要将 N 只股票进行排序以构建投资组合。传统方法（如单淘汰赛、双淘汰赛、随机匹配）对所有股票对进行均匀比较，存在以下问题：

- **计算效率低**：对于 N 只股票，需要进行 O(N²) 次比较
- **聚焦不足**：对所有股票一视同仁，未关注对投资决策最关键的边界区域
- **资源浪费**：在明显优秀的股票和明显较差的股票上花费过多比较预算

### 1.2 自适应边界聚焦方法

**核心思想**：将有限的比较预算动态分配给当前最不确定的边界区域股票对。

**算法流程**（位于 `sorting_algorithms.py:run_adaptive_boundary_focusing`）：

```
阶段1: 初始探索（exploration_ratio × budget）
    - 随机配对，确保每只股票都有初始比较
    - 获得粗糙的排序估计

阶段2: 自适应边界聚焦（(1 - exploration_ratio) × budget）
    每轮迭代：
        1. 计算当前得分（使用贝叶斯平均平滑）
        2. 确定边界区域：[topk - boundary_bandwidth, topk + boundary_bandwidth]
        3. 使用UCB算法选择信息增益最大的配对
           - 优先考虑边界区域内部的配对
           - 考虑不确定性高的配对
        4. 批量推理（BATCH_SIZE=5），更新胜负记录
        5. 更新得分，重新排序

最终输出：使用贝叶斯平均平滑的最终得分
```

**关键参数**：
- `budget`: 总比较次数预算
- `topk`: 目标识别的top-k数量（默认50）
- `boundary_bandwidth`: 边界带宽
- `exploration_ratio`: 探索比例

**UCB 计算公式**（代码行 546-558）：
```python
ucb_value = (
    (score_a + score_b) / 2 * 0.3 +           # 平均得分
    uncertainty * 0.4 * boundary_bonus * 2 +    # 不确定性加成
    proximity * 0.3                             # 得分接近度
)
```

### 1.3 实验目标

寻找自适应边界聚焦方法的最优参数配置与变种，以达到：
- **更高的 IC/Rank IC**：更准确的股票排序
- **更好的回测收益**：更高的年化收益率和夏普比率
- **更高的计算效率**：更少的模型推理次数

---

## 2. 实验设计原则

### 2.1 模型复用策略（关键）

为避免每次参数搜索实验都重新采样数据与训练模型，采用以下策略：

**阶段0：基准模型训练**
```bash
# 训练并保存基准模型（仅需一次）
python myportfolio.py \
    --model lgbm \
    --tournament adaptive \
    --k-pairs-per-stock 3 \
    --save-model-path data/models/baseline_lgbm_model.pkl
```

**后续实验：加载模型**
```bash
# 加载模型并测试不同排序参数
python myportfolio.py \
    --model lgbm \
    --load-model-path data/models/baseline_lgbm_model.pkl \
    --tournament adaptive \
    --budget-params ...
```

**模型保存路径规范**：
```
data/models/
├── baseline_lgbm_model.pkl         # LGBM基准模型
├── baseline_mlp_model.pth          # MLP基准模型
└── baseline_twin_tower_model.pth   # Twin-Tower基准模型
```

### 2.2 贪心搜索策略

- 每个实验基于迄今为止**回测年化收益率最高**的配置
- 每次仅改变一个维度的设置
- 清晰评估该修改对投资绩效的影响

### 2.3 评估指标

**主要指标**（用于选择最优配置）：
- 年化收益率 (Annual Return)

**辅助指标**（用于综合判断）：
- IC (Information Coefficient)
- Rank IC
- 夏普比率 (Sharpe Ratio)
- 最大回撤 (Max Drawdown)
- Calmar 比率
- ICIR

### 2.4 可复现性

- 所有实验使用相同的随机种子（数据划分、模型初始化、PyTorch/Numpy）
- 所有模型在完全相同的回测框架下评估
- 数据范围：2017-01-01 至 2020-08-01（测试集）

---

## 3. 实验列表

### 实验 0: baseline 模型训练（先决条件）

**目标**：训练并保存基准模型，后续实验直接加载使用。

**设置**：
- 模型类型：LGBM（训练快速，适合参数搜索）
- 排序方法：adaptive（使用默认参数）
- k_pairs_per_stock: 3
- quantile_threshold: 0.0

**执行命令**：
```bash
python -m pair_wise_ranking.workflow \
    --model lgbm \
    --tournament adaptive \
    --k-pairs-per-stock 3 \
    --n-tournaments 10
```

**输出**：
- 训练好的模型：`data/models/baseline_lgbm_model.pkl`
- 基准性能指标

---

### 实验 1: 比较预算 (budget) 扫描

**目标**：确定最优的比较预算规模。

**变量**：`budget` 参数

**设置**：
```python
budget_configs = [
    ("低预算", len(stocks) * 5),        # 5轮锦标赛规模
    ("中低预算", len(stocks) * 10),     # 10轮锦标赛规模（默认）
    ("中高预算", len(stocks) * 20),     # 20轮锦标赛规模
    ("高预算", len(stocks) * 50),       # 50轮锦标赛规模
]
```

**预期**：
- 预算过低：排序不稳定，IC较低
- 预算适中：性能与效率平衡
- 预算过高：边际收益递减，计算成本增加

**评估**：绘制 年化收益率 vs budget 曲线，寻找拐点

---

### 实验 2: 边界带宽 (boundary_bandwidth) 调优

**目标**：确定最优的边界聚焦范围。

**变量**：`boundary_bandwidth` 参数

**设置**（基于实验1的最佳 budget）：
```python
boundary_bandwidth_configs = [
    ("窄聚焦", 5),      # [topk-5, topk+5]
    ("默认聚焦", 10),    # [topk-10, topk+10]
    ("宽聚焦", 20),      # [topk-20, topk+20]
    ("超宽聚焦", 30),    # [topk-30, topk+30]
]
```

**预期**：
- 带宽过窄：可能遗漏边界附近的股票
- 带宽适中：有效聚焦关键区域
- 带宽过宽：退化为全局探索，失去聚焦优势

**评估**：绘制 年化收益率 vs boundary_bandwidth 曲线

---

### 实验 3: 探索比例 (exploration_ratio) 优化

**目标**：确定初始探索与自适应聚焦的最优比例。

**变量**：`exploration_ratio` 参数

**设置**（基于实验1-2的最佳配置）：
```python
exploration_ratio_configs = [
    ("低探索", 0.1),    # 10% 探索，90% 聚焦
    ("默认探索", 0.2),  # 20% 探索，80% 聚焦
    ("高探索", 0.3),    # 30% 探索，70% 聚焦
    ("平衡探索", 0.5),   # 50% 探索，50% 聚焦
]
```

**预期**：
- 探索过低：初始排序粗糙，可能陷入局部最优
- 探索适中：平衡全局认知与局部优化
- 探索过高：浪费预算在非关键比较上

**评估**：绘制 年化收益率 vs exploration_ratio 曲线

---

### 实验 4: UCB 权重组合调优

**目标**：优化 UCB（Upper Confidence Bound）公式的权重组合。

**变量**：UCB 计算中的三个权重系数

**当前实现**（`sorting_algorithms.py:554-558`）：
```python
ucb_value = (
    (score_a + score_b) / 2 * 0.3 +              # 平均得分权重
    uncertainty * 0.4 * boundary_bonus * 2 +     # 不确定性权重
    proximity * 0.3                               # 得分接近度权重
)
```

**设置**（基于实验1-3的最佳配置）：
```python
ucb_weight_configs = [
    ("得分优先", [0.5, 0.3, 0.2]),      # 优先选择平均得分高的配对
    ("不确定性优先", [0.2, 0.6, 0.2]),  # 优先选择不确定性高的配对
    ("接近度优先", [0.2, 0.3, 0.5]),    # 优先选择得分接近的配对
    ("均衡配置", [0.33, 0.34, 0.33]),   # 三者均衡
]
```

**修改代码位置**：`sorting_algorithms.py:554-558`

**预期**：
- 不确定性优先：更适合早期探索阶段
- 接近度优先：更适合边界区域精排序
- 均衡配置：整体性能更稳定

---

### 实验 5: 批量大小 (BATCH_SIZE) 调优

**目标**：确定批量推理的最优大小。

**变量**：`BATCH_SIZE` 常量（当前固定为5）

**设置**（基于实验1-4的最佳配置）：
```python
batch_size_configs = [1, 3, 5, 10, 20]
```

**修改代码位置**：`sorting_algorithms.py:433`

**预期**：
- 批量过小：推理开销大，速度慢
- 批量适中：平衡推理效率与排序精度
- 批量过大：单轮更新步长过大，可能不稳定

**评估**：同时考虑 年化收益率 与 总推理时间

---

### 实验 6: 候选配对生成策略

**目标**：优化候选配对的生成策略。

**变量**：候选配对的采样方式

**当前实现**（`sorting_algorithms.py:505-525`）：
```python
# 优先考虑边界区域内部的配对（50% 概率）
if len(boundary_region) >= 2 and random.random() < 0.5:
    i, j = random.sample(range(len(boundary_region)), 2)
    inst_a, inst_b = boundary_region[i], boundary_region[j]
else:
    # 考虑边界区域与外部区域的配对（50% 概率）
    ...
```

**设置**（基于实验1-5的最佳配置）：
```python
candidate_strategy_configs = [
    ("边界内部优先", 0.7),    # 70% 边界内部，30% 跨边界
    ("均衡采样", 0.5),        # 50% 边界内部，50% 跨边界（默认）
    ("跨边界优先", 0.3),      # 30% 边界内部，70% 跨边界
]
```

**修改代码位置**：`sorting_algorithms.py:514`

**预期**：
- 边界内部优先：更精确的边界排序，但可能遗漏外部股票
- 均衡采样：平衡边界精度与全局稳定性
- 跨边界优先：更稳定的全局排序

---

### 实验 7: 贝叶斯平滑参数调优

**目标**：优化最终得分计算的贝叶斯平滑先验参数。

**变量**：`prior_wins` 和 `prior_losses` 参数

**当前实现**（`sorting_algorithms.py:588-594`）：
```python
prior_wins = 2   # 先验胜场
prior_losses = 2  # 先验负场
final_score = (wins + prior_wins) / (wins + prior_wins + losses + prior_losses)
```

**设置**（基于实验1-6的最佳配置）：
```python
bayesian_prior_configs = [
    ("弱先验", 1),      # prior_wins=1, prior_losses=1
    ("默认先验", 2),    # prior_wins=2, prior_losses=2（默认）
    ("强先验", 5),      # prior_wins=5, prior_losses=5
]
```

**修改代码位置**：`sorting_algorithms.py:588-589`

**预期**：
- 弱先验：更信任观察数据，可能过拟合
- 强先验：更保守，平滑效果更强，可能欠拟合

---

### 实验 8: 自适应轮数动态调整

**目标**：探索自适应阶段的迭代次数动态调整策略。

**变量**：自适应阶段的终止条件

**当前实现**（`sorting_algorithms.py:476-585`）：
```python
n_adaptive_rounds = max(1, remaining_budget // BATCH_SIZE)  # 固定轮数
for round_idx in range(n_adaptive_rounds):
    # ...
```

**设置**（基于实验1-7的最佳配置）：
```python
adaptive_strategy_configs = [
    ("固定轮数", "fixed"),           # 使用当前固定轮数策略
    ("早停策略", "early_stopping"), # 边界区域连续3轮无变化则停止
    ("收敛检测", "convergence"),    # 得分变化 < 阈值则停止
]
```

**新增代码**：实现早停与收敛检测逻辑

**预期**：
- 早停策略：节省计算资源，避免过度迭代
- 收敛检测：自适应终止，更灵活

---

### 实验 9: 多阶段自适应策略

**目标**：探索多阶段自适应策略（渐进式聚焦）。

**变量**：边界带宽的动态调整策略

**设置**（基于实验1-8的最佳配置）：
```python
multi_stage_configs = [
    ("单阶段", "single"),              # 使用固定的 boundary_bandwidth
    ("两阶段", "two_stage"),           # 初始宽带宽 → 后续窄带宽
    ("三阶段", "three_stage"),         # 宽 → 中 → 窄
]
```

**新增代码**：实现多阶段边界调整逻辑

**示例**（两阶段）：
```python
if round_idx < n_adaptive_rounds // 2:
    current_boundary_bandwidth = boundary_bandwidth * 2  # 前半段：宽聚焦
else:
    current_boundary_bandwidth = boundary_bandwidth      # 后半段：窄聚焦
```

**预期**：
- 多阶段策略：先粗后精，更高效的预算分配

---

### 实验 10: 不同模型架构下的泛化性验证

**目标**：验证自适应方法在深度学习模型上的泛化性。

**变量**：模型类型

**设置**（基于实验1-9的最佳自适应参数）：
```python
model_type_configs = [
    ("LGBM", "lgbm"),
    ("MLP", "mlp"),
    ("MLP + LayerNorm", "mlp_residual"),
    ("Twin-Tower", "twin_tower"),
]
```

**执行**：
```bash
# LGBM
python myportfolio.py --model lgbm --load-model-path data/models/baseline_lgbm_model.pkl ...

# MLP（需训练）
python myportfolio.py --model mlp --hidden-layers 512 256 128 ...

# Twin-Tower（需训练）
python myportfolio.py --model twin_tower ...
```

**预期**：
- 自适应方法对不同模型架构均有效
- 深度学习模型可能需要不同的最优参数配置

---

### 实验 11: 与其他排序方法的对比

**目标**：验证自适应方法相对于其他排序方法的优越性。

**设置**（所有方法使用相同的比较预算）：
```python
sorting_method_configs = [
    ("随机匹配", "random_matching"),
    ("单淘汰赛", "single_elim"),
    ("双淘汰赛", "double_elim"),
    ("混合排序", "hybrid"),
    ("自适应边界聚焦", "adaptive"),
]
```

**控制变量**：
- 所有方法的比较次数 = 实验1-9确定的最优 budget
- 使用相同的基准模型

**评估指标对比**：
- IC / Rank IC
- 年化收益率
- 夏普比率
- 计算时间

**预期**：
- 自适应方法在相同预算下达到更高的 IC 和收益率
- 自适应方法计算效率更高

---

### 实验 12: 最终最优配置综合测试

**目标**：综合所有实验结果，形成最优配置推荐。

**设置**：
- 模型：实验10中表现最佳的模型
- 排序方法：adaptive
- 参数：实验1-9确定的最优参数组合

**执行**：
```bash
python -m pair_wise_ranking.workflow \
    --model <best_model> \
    --tournament adaptive \
    --budget <optimal_budget> \
    --boundary-bandwidth <optimal_bw> \
    --exploration-ratio <optimal_er> \
    ...
```

**输出**：
- 最优配置的性能报告
- 与 baseline（实验0）的对比
- 参数敏感性分析

---

## 4. 实验执行与评估流程

### 4.1 顺序依赖

```
实验0（baseline训练）
    ↓
实验1（budget扫描）→ 确定 best_budget
    ↓
实验2（boundary_bandwidth）→ 确定 best_boundary_bandwidth
    ↓
实验3（exploration_ratio）→ 确定 best_exploration_ratio
    ↓
实验4（UCB权重）→ 确定 best_ucb_weights
    ↓
实验5（BATCH_SIZE）→ 确定 best_batch_size
    ↓
实验6（候选策略）→ 确定 best_candidate_strategy
    ↓
实验7（贝叶斯先验）→ 确定 best_bayesian_prior
    ↓
实验8（早停策略）→ 确定 best_adaptive_strategy
    ↓
实验9（多阶段）→ 确定 best_stage_strategy
    ↓
实验10（模型泛化性）→ 确定 best_model
    ↓
实验11（方法对比）→ 验证优越性
    ↓
实验12（最优配置）→ 最终报告
```

### 4.2 结果记录

**每个实验记录**：
- 参数配置
- IC, Rank IC, ICIR
- 年化收益率、夏普比率、最大回撤、Calmar 比率
- 计算时间
- 相对于 baseline 的提升百分比

**结果存储路径**：
```
data/experiments/adaptive_ranking/
├── exp01_budget_scan/
│   ├── config.json
│   ├── results.csv
│   └── plots/
├── exp02_boundary_bandwidth/
├── ...
└── exp12_optimal_config/
    └── final_report.md
```

### 4.3 可视化分析

**关键可视化**：
1. **参数敏感性曲线**：年化收益率 vs 各参数值
2. **热力图**：两参数组合的年化收益率热力图（如 budget × boundary_bandwidth）
3. **收敛曲线**：自适应阶段的得分变化曲线
4. **边界演化**：边界区域股票随迭代的变化
5. **方法对比雷达图**：不同排序方法的多维指标对比

---

## 5. 预期贡献

### 5.1 学术贡献

- **算法创新**：自适应边界聚焦方法的理论分析与实验验证
- **参数敏感性**：系统性研究各参数对性能的影响
- **泛化性验证**：在不同模型架构上的有效性

### 5.2 实践贡献

- **最优参数配置**：提供可直接用于生产环境的参数推荐
- **计算效率提升**：在相同或更高性能下减少计算成本
- **开源实现**：完整的实验代码和结果复现指南

### 5.3 潜在扩展

- **动态 topk**：根据市场状态动态调整 topk 值
- **多目标边界**：同时考虑收益、风险、流动性等多个维度的边界
- **在线学习**：根据最新市场数据实时更新排序
- **跨市场验证**：在其他市场（如美股、港股）上的验证

---

## 6. 附录：代码修改清单

### 6.1 模型保存与加载功能

**新增函数**（`models.py`）：
```python
def save_model(self, path: str) -> None:
    """保存模型到指定路径"""

def load_model(self, path: str) -> None:
    """从指定路径加载模型"""
```

### 6.2 参数化接口

**修改 `run_adaptive_boundary_focusing` 函数签名**（`sorting_algorithms.py:391-427`）：
```python
def run_adaptive_boundary_focusing(
    date: pd.Timestamp,
    stocks: List[str],
    features: Dict[str, np.ndarray],
    model_predict_fn: Callable[[List[Tuple[str, str, np.ndarray]]], np.ndarray],
    budget: int = 100,
    topk: int = 50,
    boundary_bandwidth: int = 10,
    exploration_ratio: float = 0.2,
    batch_size: int = 5,                    # 新增
    ucb_weights: tuple = (0.3, 0.4, 0.3),  # 新增
    candidate_strategy_prob: float = 0.5,   # 新增
    bayesian_prior: int = 2,               # 新增
    adaptive_strategy: str = "fixed",       # 新增
    random_seed: Optional[int] = None
) -> Dict[str, float]:
```

### 6.3 配置文件支持

**新增配置文件**（`config/adaptive_ranking_config.yaml`）：
```yaml
adaptive:
  budget: 3000
  topk: 50
  boundary_bandwidth: 10
  exploration_ratio: 0.2
  batch_size: 5
  ucb_weights: [0.3, 0.4, 0.3]
  candidate_strategy_prob: 0.5
  bayesian_prior: 2
  adaptive_strategy: "fixed"
```

---

## 7. 时间估算

| 实验编号 | 实验名称 | 预计耗时（小时） |
|---------|---------|----------------|
| 0 | baseline 模型训练 | 2-3 |
| 1 | budget 扫描（4个配置） | 4-6 |
| 2 | boundary_bandwidth（4个配置） | 4-6 |
| 3 | exploration_ratio（4个配置） | 4-6 |
| 4 | UCB 权重（4个配置） | 4-6 |
| 5 | BATCH_SIZE（5个配置） | 5-7 |
| 6 | 候选策略（3个配置） | 3-5 |
| 7 | 贝叶斯先验（3个配置） | 3-5 |
| 8 | 早停策略（3个配置） | 4-6 |
| 9 | 多阶段（3个配置） | 4-6 |
| 10 | 模型泛化性（4个模型） | 8-12 |
| 11 | 方法对比（5个方法） | 6-8 |
| 12 | 最优配置测试 | 2-3 |
| **总计** | | **53-85 小时** |

**建议执行周期**：2-3 周（每周20-30小时计算时间）

---

## 8. 风险与缓解策略

### 8.1 计算资源风险

**风险**：实验数量多，总计算时间长

**缓解**：
- 使用模型复用策略，避免重复训练
- 并行运行独立实验（不同参数配置）
- 使用云服务器加速计算

### 8.2 过拟合风险

**风险**：参数搜索可能在测试集上过拟合

**缓解**：
- 保留一部分数据作为最终验证集（不参与参数搜索）
- 使用时间序列交叉验证
- 报告参数敏感性分析，而非单一最优值

### 8.3 实现复杂度风险

**风险**：部分实验需要修改核心代码

**缓解**：
- 采用配置文件方式，避免硬编码
- 充分测试代码修改
- 使用 git 分支管理，保持主分支稳定

---

## 9. 成功标准

**实验计划成功的标志**：
1. ✓ 找到相对于 baseline 有显著提升（年化收益率提升 > 2%）的配置
2. ✓ 理解各参数对性能的影响规律
3. ✓ 验证自适应方法在不同模型架构上的泛化性
4. ✓ 形成可复现的最优配置推荐
5. ✓ 完成实验报告与可视化分析
