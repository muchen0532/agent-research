# Cohen's Kappa 笔记

> 用途：多标注员一致性评估（inter-annotator agreement），常用于 taxonomy 构建 / failure mode 标注类研究（如 MAST）。

## 1. 为什么不能直接用"一致率"（Percent Agreement）

假设两个标注员对100条样本做二分类标注（是否失效 / 非失效），两人有80条判断一致，直觉上会说"一致率80%，还不错"。

但这个数字有个致命问题：**没有扣除"纯靠猜也能碰上"的部分**。如果失效类别在数据里占比很高（比如90%都是失效），两个标注员就算全部瞎猜"失效"，一致率也能轻松到80%+，这跟标注质量毫无关系。

Cohen's Kappa 解决的就是这个问题：**扣除随机一致的部分，衡量真正超出随机水平的一致程度**。

## 2. 公式

$$
\kappa = \frac{p_o - p_e}{1 - p_e}
$$

- $p_o$（observed agreement）：观测到的实际一致率
- $p_e$（expected agreement）：假设两人完全独立随机打标，理论上会产生的一致率

$p_e$ 的计算方式（以二分类为例）：

$$
p_e = P(A=\text{正}) \times P(B=\text{正}) + P(A=\text{负}) \times P(B=\text{负})
$$

即：两人各自标"正类"的边际概率相乘，加上两人各自标"负类"的边际概率相乘。

## 3. 手算示例

两个标注员对100条trace判断"是否存在Read-before-Write失效"：

|          | B: 是 | B: 否 | 合计 |
|----------|------|------|------|
| A: 是    | 40   | 10   | 50   |
| A: 否    | 5    | 45   | 50   |
| 合计     | 45   | 55   | 100  |

- p_o = (40 + 45) / 100 = 0.85
- P(A=是) = 0.50，P(B=是) = 0.45
- P(A=否) = 0.50，P(B=否) = 0.55
- p_e = 0.50 × 0.45 + 0.50 × 0.55 = 0.225 + 0.275 = 0.50
- κ = (0.85 − 0.50) / (1 − 0.50) = 0.35 / 0.50 = 0.70

## 4. 数值解读标准（Landis & Koch, 1977，最常引用的一版）

| Kappa 区间 | 一致性水平 |
|-----------|-----------|
| < 0.00    | 差于随机（Poor） |
| 0.00–0.20 | 极轻微（Slight） |
| 0.21–0.40 | 一般（Fair） |
| 0.41–0.60 | 中等（Moderate） |
| 0.61–0.80 | 相当（Substantial） |
| 0.81–1.00 | 几乎完全一致（Almost Perfect） |

**参照系**：MAST（Cemri et al., 2025）报告的 κ = 0.88，属于 Almost Perfect 区间，这是它标注质量能立住的关键数据支撑之一。一般论文里 κ ≥ 0.6 才算"可信的人工标注"，低于0.4通常会被审稿人质疑分类标准模糊。

## 5. 常见坑

- **类别不平衡会压低Kappa**：即使标注员实际判断很一致，如果某一类占比极端（比如99% vs 1%），Kappa 反而可能显得偏低（"Kappa悖论"）。这种情况下需要在论文里额外说明，或者补充报告 $p_o$ 本身。
- **多分类 / 多标注员**要用扩展版本：
  - 3人以上：用 **Fleiss' Kappa**（不是Cohen's Kappa，公式不同，注意别用混）
  - 需要考虑类别间"距离"权重（比如失效严重程度分级，1级和5级的分歧比1级和2级更严重）：用 **Weighted Kappa**
- **Kappa 高不代表分类标准客观**，只代表两个标注员按同一套（哪怕是错的）标准判断得很稳定。分类框架本身是否合理，仍需要额外论证（比如MAST用grounded theory迭代打磨分类定义）。

## 6. Python 计算方式

```python
from sklearn.metrics import cohen_kappa_score

annotator_a = [1, 1, 0, 1, 0, 1, 0, 0, 1, 0]  # 1=失效, 0=非失效
annotator_b = [1, 0, 0, 1, 0, 1, 0, 0, 1, 1]

kappa = cohen_kappa_score(annotator_a, annotator_b)
print(f"Cohen's Kappa: {kappa:.3f}")
```

多分类（比如14类失效模式）同样适用，`cohen_kappa_score` 默认支持多类别标签，无需额外处理。

需要加权版本（比如分类之间有严重程度顺序）：

```python
kappa_weighted = cohen_kappa_score(annotator_a, annotator_b, weights='quadratic')
# weights 可选 'linear' 或 'quadratic'
```
