

## 修复myportfolio_v2的数据不匹配错误

### 问题
- 错误信息：`LightGBMError: Length of labels differs from length of #data`
- 原因：特征数据是pair-level（~460万pairs），但标签数据是stock-level（~51万labels）
- 导致LightGBM无法训练

### 解决方案
- 创建了修复版本：`myportfolio_v2_fixed.py`
- 修复标签生成逻辑：改为pair-wise标签
  - 如果target胜opponent，label=1
  - 如果opponent胜target，label=0
- 确保pair_labels和pair_features的维度一致
- 保持max_pairs_per_stock=5（降低内存使用）

### 执行状态
- ✅ 代码已修复
- 🔄 准备启动实验

**Javis Feedback (2026-02-03):** Fixed a critical bug in `PairwiseRankModel.__init__` — the `max_pairs_per_stock` attribute was missing, causing an AttributeError in the `predict` method at line 429. The fix adds `self.max_pairs_per_stock = max_pairs_per_stock` to properly store this parameter for use in prediction. The experiment can now be rerun successfully.

