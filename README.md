# 钙钛矿埋入界面：分子指纹因果发现（EQE）

基于 **LiNGAM**（Linear Non-Gaussian Acyclic Model）从观测数据学习分子结构指纹与 **外量子效率 EQE (%)** 之间的有向因果结构，识别对 EQE 具有**直接**线性影响的结构片段（区别于 SHAP 等以关联/预测为导向的解释）。

本仓库当前聚焦于因果发现流程；分析代码与数据均位于 `neuralNetworks/` 目录。

---

## 目录结构

```
neuralNetworks/
├── causal_discovery.ipynb    # 主分析 notebook（按单元格顺序运行）
├── result.csv                # 输入：SMILES + EQE + 分子指纹位
└── result_eqe_*.csv          # 输出：LiNGAM 因果分析结果（运行后生成/更新）
```

运行 notebook 后还会在同一目录生成图表（PNG）及 Origin 导出用 CSV，见下文「输出文件」。

---

## 环境依赖

建议使用 Python 3.9+，安装以下包：

```bash
pip install numpy pandas scipy scikit-learn matplotlib networkx lingam
```

| 包 | 用途 |
|---|---|
| `lingam` | DirectLiNGAM 因果结构学习 |
| `scikit-learn` | 特征标准化 |
| `scipy` | Shapiro-Wilk、Pearson/Spearman 检验 |
| `networkx` | 因果 DAG 可视化 |
| `pandas` / `numpy` / `matplotlib` | 数据处理与绘图 |

---

## 快速开始

1. 进入工作目录：

   ```bash
   cd neuralNetworks
   ```

2. 确认 `result.csv` 存在且包含目标列 `EQE (%)`。

3. 打开并**自上而下**运行 `causal_discovery.ipynb` 全部单元格。

4. 在 `neuralNetworks/` 下查看生成的 `result_eqe_*.csv` 与 PNG 图。

---

## 输入数据 `result.csv`

| 列 | 说明 |
|---|---|
| `SMILES` | 分子 SMILES（仅用于追溯，**不参与** LiNGAM 拟合） |
| `EQE (%)` | 目标变量：外量子效率 |
| `Bit *` | MACCS 等指纹位（0/1） |
| `Morgan Bit *` | Morgan 指纹位（0/1） |

- 除 `SMILES` 与 `EQE (%)` 外，其余数值列自动识别为特征。
- 无法转为数值的列会被丢弃；缺失指纹位按 `0` 填充。
- 当前数据集约 **113** 条样本（不含表头）。

更新数据时，直接替换 `result.csv` 后重新运行 notebook 即可。

---

## 分析流程概览

```
result.csv
    │
    ├─ 数据清洗（去 NaN/inf、零方差列）
    ├─ 降维（按与 EQE 的 |相关| 取 Top-K，K ≤ min(300, n-2)）
    ├─ 去共线（|r| > 0.999 的冗余特征）
    ├─ 标准化 → Z_df
    │
    ├─ 假设检验
    │   ├─ Shapiro-Wilk：非高斯性（LiNGAM 识别性假设）
    │   └─ Pearson vs Spearman：线性 vs 单调非线性风险
    │
    ├─ DirectLiNGAM 拟合 → 邻接矩阵 B
    ├─ 阈值化构图（|B| > 0.05）→ DAG
    ├─ Bootstrap 稳定性（默认 50 次重采样）
    ├─ 可视化（热力图、条形图、散点图）
    └─ 结构方程拟合质量导出（Origin 用）
```

---

## 输出文件

### CSV（因果结构）

| 文件 | 内容 |
|---|---|
| `result_eqe_linearity_check_pearson_vs_spearman.csv` | 各特征与 EQE 的 Pearson / Spearman 及非线性风险标记 |
| `result_eqe_causal_adjacency_lingam.csv` | 完整邻接矩阵 **B**（`B[i,j]`：j → i 的系数） |
| `result_eqe_causal_parents_of_target.csv` | 直接指向 `EQE (%)` 的父节点列表 |
| `result_eqe_causal_edges_thresholded.csv` | 阈值化后的有向边（cause, effect, weight） |
| `result_eqe_top20_features_abs_direct_to_target.csv` | \|直接系数\| 最大的 Top 20 特征 |
| `result_eqe_top20_features_positive_direct_to_target.csv` | 直接系数 > 0 的 Top 20（可能不足 20 行） |
| `result_eqe_bootstrap_direct_edges_to_target.csv` | Bootstrap 边权频率与均值 |
| `result_eqe_bootstrap_top20_abs_direct_to_target.csv` | Bootstrap \|均值系数\| Top 20 |
| `result_eqe_bootstrap_top20_positive_direct_to_target.csv` | Bootstrap 正向 Top 20 |

### CSV / PNG（运行可视化单元后生成）

| 文件 | 内容 |
|---|---|
| `result_eqe_adjacency_heatmap_topN.png` | 指向 EQE 的 Top 节点邻接热力图 |
| `result_eqe_top30_direct_effects_bar.png` | 直接效应 Top 30 条形图 |
| `result_eqe_direct_vs_marginal_scatter.png` | 直接系数 vs 边际相关 |
| `result_eqe_bootstrap_stability_scatter.png` | Bootstrap 频率 vs \|均值效应\| |
| `result_eqe_bootstrap_top20_bar.png` | Bootstrap Top 20 条形图 |
| `origin_eqe_structural_fit_scatter.csv` | 结构方程：真实 vs 预测（标准化尺度） |
| `origin_structural_R2_all_variables.csv` | 各变量结构方程 R² |
| `origin_like_eqe_true_vs_pred.png` | 真实-预测散点图 |
| `origin_like_eqe_residual_hist.png` | 残差直方图 |
| `origin_like_eqe_residual_qq.png` | 残差 Q-Q 图 |

### Top 20 表字段说明

`result_eqe_top20_features_abs_direct_to_target.csv` 示例列：

- **weight**：LiNGAM **直接**效应（控制 DAG 中排在 EQE 之前的变量）
- **marginal_corr**：与 EQE 的边际 Pearson 相关

> **注意**：直接系数符号与边际相关可以不一致（混淆、链式结构）。解读时请同时对照 `weight` 与 `marginal_corr`，勿仅凭 `weight` 判断「全是负向影响」。

---

## 方法要点与局限

**LiNGAM 假设**

- 线性结构方程 + 非高斯独立噪声（用于识别因果方向）
- 无潜在混杂、无反馈环（DAG）

**本流程中的工程设置**

- 特征数上限：`min(300, n_samples - 2)`，避免高维下 DirectLiNGAM 不稳定
- 边阈值：`|B| > 0.05` 才保留于稀疏 DAG
- Bootstrap：默认 50 次（后台 nbconvert 时自动降为 15 次）

**结果解读建议**

- 因果发现基于**观测数据**，结论为「与假设相容的因果假设」，不能替代干预实验。
- 指纹位高度稀疏且共线，Top-K 筛选会改变进入模型的特征集合。
- 样本量有限时，Bootstrap 频率宜作为**稳定性参考**，而非显著性检验。

---

## 引用

若使用 LiNGAM，可引用：

> Shimizu, S., Hoyer, P. O., Hyvärinen, A., & Kerminen, A. (2006). A linear non-Gaussian acyclic model for causal discovery. *Journal of Machine Learning Research*, 7, 2003–2030.

---


