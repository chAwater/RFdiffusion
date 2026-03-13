# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

RFdiffusion (v1.1.0) 是 Rosetta Commons 开发的蛋白质结构生成工具，基于 SE(3) 等变扩散概率模型。支持无条件生成、motif scaffolding、binder 设计、对称寡聚体生成、部分扩散等多种蛋白质设计任务。

## 安装与环境

```bash
# 创建 conda 环境并安装 SE(3)-Transformer
conda env create -f env/SE3nv.yml
conda activate SE3nv
cd env/SE3Transformer && pip install --no-cache-dir -r requirements.txt && python setup.py install
cd ../..
pip install -e .

# 下载模型权重到 models/ 目录
mkdir models && cd models
wget http://files.ipd.uw.edu/pub/RFdiffusion/6f5902ac237024bdd0c176cb93063dc4/Base_ckpt.pt
wget http://files.ipd.uw.edu/pub/RFdiffusion/e29311f6f1bf1af907f9ef9f44b8328b/Complex_base_ckpt.pt
wget http://files.ipd.uw.edu/pub/RFdiffusion/60f09a193fb5e5ccdc4980417708dbab/Complex_Fold_base_ckpt.pt
wget http://files.ipd.uw.edu/pub/RFdiffusion/74f51cfb8b440f50d70878e05361d8f0/InpaintSeq_ckpt.pt
wget http://files.ipd.uw.edu/pub/RFdiffusion/76d00716416567174cdb7ca96e208296/InpaintSeq_Fold_ckpt.pt
wget http://files.ipd.uw.edu/pub/RFdiffusion/5532d2e1f3a4738decd58b19d633b3c3/ActiveSite_ckpt.pt
wget http://files.ipd.uw.edu/pub/RFdiffusion/12fc204edeae5b57713c5ad7dcb97d39/Base_epoch8_ckpt.pt
```

## 常用命令

```bash
# 推理（所有设计任务的入口）
./scripts/run_inference.py 'contigmap.contigs=[150-150]' inference.output_prefix=test_outputs/test inference.num_designs=10

# Motif scaffolding 示例
./scripts/run_inference.py inference.output_prefix=test_outputs/scaffolding inference.input_pdb=examples/input_pdbs/5TPN.pdb 'contigmap.contigs=[10-40/A163-181/10-40]'

# Binder 设计
./scripts/run_inference.py inference.output_prefix=test_outputs/binder inference.input_pdb=examples/input_pdbs/insulin_target.pdb 'contigmap.contigs=[A1-150/0 70-100]' 'ppi.hotspot_res=[A59,A83,A91]'

# 运行测试（需要 A100 GPU 以匹配参考输出）
python -m pytest tests/test_diffusion.py -v

# 设计示例脚本（examples/ 目录下有 27 个 .sh）
bash examples/design_unconditional.sh
```

Hydra 配置支持 CLI 覆盖，如 `diffuser.T=30`、`inference.num_designs=5`。

## 架构概览

### 推理流程

```
run_inference.py (入口)
  → Sampler.initialize()          # 加载权重、构建模型
  → for each design:
      → sample_init()             # 解析 PDB、ContigMap 映射、前向扩散到 t=T
      → for t in T→1:
          → sample_step()         # 预处理 → RoseTTAFoldModule → 去噪采样
      → writepdb()                # 保存设计 PDB
```

### 核心模块

| 模块 | 文件 | 职责 |
|------|------|------|
| **RoseTTAFoldModule** | `RoseTTAFoldModel.py` | 三轨道网络（MSA + Pair + Structure），使用 SE(3)-Transformer |
| **采样器** | `model_runners.py` | `Sampler` → `SelfConditioning` → `ScaffoldedSampler` 继承链 |
| **扩散过程** | `diffusion.py` | 欧氏扩散（CA 坐标）+ SO(3) 扩散（旋转帧） |
| **IGSO(3)** | `igso3.py` | SO(3) 上的各向同性高斯分布：密度、评分函数、CDF 采样 |
| **去噪器** | `inference/utils.py` | `Denoise` 类：反向采样、motif 对齐、坐标重建 |
| **Contig 映射** | `contigs.py` | `ContigMap`：解析 contig 字符串，管理 ref/hal/rf 索引映射 |
| **电势引导** | `potentials/` | `PotentialManager` + 可插拔电势（ROG、contacts 等），梯度引导去噪 |
| **对称性** | `symmetry.py` | `SymGen`：循环/二面体/四面体对称坐标复制与旋转 |
| **Track 模块** | `Track_module.py` | `IterativeSimulator`：MSA、Pair、结构三轨道迭代更新 |
| **工具函数** | `util.py` | 坐标转换（torsion↔xyz）、RMSD、化学常量 |

### 三轨道网络 (RoseTTAFoldModule)

- **MSA 轨道**：序列 one-hot + 位置编码 → `MSA_emb` → `MSAPairStr2MSA` 更新
- **Pair 轨道**：残基对关系 → 结构偏置 → `PairStr2Pair` 三角形更新
- **Structure 轨道**：SE(3) 等变网络处理 3D 坐标（`SE3TransformerWrapper`）
- 预测头：`c6d_pred`（距离）、`aa_pred`（氨基酸）、`lddt_pred`（pLDDT 置信度）

### 模型权重自动选择逻辑

```
scaffoldguided=True           → Complex_Fold_base_ckpt.pt
hotspot_res 非空               → Complex_base_ckpt.pt
inpaint_seq + scaffoldguided  → InpaintSeq_Fold_ckpt.pt
inpaint_seq                   → InpaintSeq_ckpt.pt
active_site 小 motif           → ActiveSite_ckpt.pt
默认                           → Base_ckpt.pt
```

### 配置系统 (Hydra)

主配置文件 `config/inference/base.yaml`，但实际推理值来自检查点（`assemble_config_from_chk()` 合并）。关键配置组：

- `inference.*` — 输入 PDB、输出路径、设计数量、模型选择
- `contigmap.*` — contig 字符串、inpaint_seq/str 掩蔽
- `diffuser.*` — 时间步 T、噪声表 b_0/b_T、schedule 类型
- `denoiser.*` — 噪声缩放
- `ppi.*` — hotspot 残基
- `potentials.*` — 电势类型和权重
- `scaffoldguided.*` — Fold 条件化、二级结构/邻接矩阵

### Contig 字符串语法

```
"A10-25/30-40"     — 保留 A 链 10-25 残基，hallucinate 30-40 个新残基
"A1-150/0 70-100"  — A 链固定 + 链间断 + 新链 70-100 残基（binder）
"[150-150]"        — 无条件生成 150 残基
```

`/0` 表示链断裂，`/` 分隔不同片段。

### 扩散核心

- **前向扩散**：对 CA 原子施加高斯噪声（欧氏），对残基帧施加 IGSO(3) 旋转噪声
- **反向去噪**：模型预测 x₀，通过 `get_next_pose()` 计算后验均值采样 x_{t-1}
- `diffusion_mask`：True = 固定（motif 不扩散），False = 设计区域
- **自条件化**：前一步预测 px₀ 作为下一步结构模板输入

### 电势系统

可插拔设计：任何电势实现 `compute(xyz) → tensor([value])`，通过 `PotentialManager` 加权求和。反向传播梯度引导去噪方向（`xyz.requires_grad_(True)`）。

## 关键依赖

- `torch` — 深度学习框架
- `se3-transformer` — NVIDIA SE(3)-Transformer（需从 `env/SE3Transformer` 源码安装）
- `hydra-core` / `omegaconf` — 配置管理
- `scipy` — 旋转操作、线性代数
- `opt-einsum` — 张量缩约优化
