# Artificial Intelligence / 人工智能

## 概念→原理→公式→代码→应用 结构化参考

---

## 1. 数学基础

### 线性代数

| 概念 | 公式 | 意义 |
|------|------|------|
| 矩阵乘法 | C = AB, Cᵢⱼ = Σₖ AᵢₖBₖⱼ | 线性变换复合 |
| 特征分解 | Av = λv | 主成分、稳定性 |
| SVD | A = UΣVᵀ | 降维、推荐、压缩 |
| 范数 | ‖x‖ₚ = (Σ|xᵢ|ᵖ)^(1/p) | 正则化、距离 |

### 概率统计

```
贝叶斯定理：P(A|B) = P(B|A)P(A) / P(B)

常见分布：高斯 N(μ,σ²)、伯努利 Bernoulli(p)、泊松 Poisson(λ)、多项式
假设检验：p-value < α → 拒绝 H₀
```

### 优化理论

```
梯度下降：θ ← θ - η∇L(θ)
SGD：θ ← θ - η∇Lᵢ(θ)        （随机采样单个样本）
Adam：一阶矩 m + 二阶矩 v 的自适应学习率
  m_t = β₁m_{t-1} + (1-β₁)g_t
  v_t = β₂v_{t-1} + (1-β₂)g_t²
  θ ← θ - η · m̂_t / (√v̂_t + ε)
L-BFGS：拟牛顿法，近似 Hessian，适合小批量参数
```

### 信息论

```
熵：H(X) = -Σ P(x)log P(x)
交叉熵：H(p,q) = -Σ p(x)log q(x)
KL 散度：D_KL(p‖q) = Σ p(x)log(p(x)/q(x))
```

---

## 2. 机器学习

### 监督学习

| 算法 | 原理 | 损失函数 |
|------|------|---------|
| 线性回归 | y = wᵀx + b | MSE = 1/n Σ(yᵢ - ŷᵢ)² |
| 逻辑回归 | P(y=1) = σ(wᵀx+b) | 交叉熵 |
| SVM | 最大化间隔超平面 | Hinge loss + 正则化 |
| 决策树 | 信息增益/基尼系数分裂 | 不纯度 |
| 随机森林 | Bagging + 随机特征子集 | 多树投票/平均 |
| GBDT | 串行拟合负梯度（残差） | 可微损失 |
| XGBoost | 正则化 GBDT + 列采样 | 二阶泰勒展开近似 |
| LightGBM | GOSS + EFB 加速 | 直方图加速 |

```python
# XGBoost 示例
import xgboost as xgb
dtrain = xgb.DMatrix(X_train, label=y_train)
params = {"objective": "binary:logistic", "max_depth": 6, "eta": 0.1}
model = xgb.train(params, dtrain, num_boost_round=100)
preds = model.predict(xgb.DMatrix(X_test))
```

### 无监督学习

| 算法 | 原理 | 应用 |
|------|------|------|
| K-Means | 最小化 Σ‖xᵢ - μ_c(i)‖² | 客户分群 |
| DBSCAN | 密度可达聚类 | 异常检测、任意形状 |
| GMM | 高斯混合 + EM 算法 | 软聚类 |
| PCA | 最大方差方向投影 SVD | 降维、可视化 |
| t-SNE | 条件概率保持的嵌入 | 高维可视化 |
| UMAP | 拓扑结构保持 | 更快的可视化 |

### 模型评估

```
混淆矩阵 → Precision / Recall / F1
ROC 曲线 → AUC（面积）
交叉验证：K-fold，分层 K-fold
过拟合诊断：训练误差↓ 验证误差↑ → 正则化 / 增加数据 / 早停
```

### 正则化

- L1 (Lasso)：稀疏解，特征选择。损失 + λ‖w‖₁
- L2 (Ridge)：权重衰减。损失 + λ‖w‖²₂
- Dropout：训练时随机丢弃神经元，p=0.5
- Early Stopping：验证集损失不再下降时停止

### 特征工程

- 编码：One-Hot / Label Encoding / Target Encoding
- 缩放：StandardScaler (z-score) / MinMaxScaler
- 特征选择：方差阈值 / 互信息 / L1 / 树特征重要性
- 降维：PCA / SVD

---

## 3. 深度学习

### 神经网络基础

```
感知机：y = f(wᵀx + b)
激活函数：
  ReLU: max(0, x)          — 最常用
  GELU: x·Φ(x)             — Transformer 常用
  SiLU/Swish: x·σ(x)       — LLaMA 等使用
  Sigmoid: 1/(1+e^(-x))    — 二分类输出
  Tanh: (e^x - e^(-x))/(e^x + e^(-x))

反向传播：链式法则 ∂L/∂w = ∂L/∂y · ∂y/∂z · ∂z/∂w
```

### CNN

```
卷积：output_size = (input_size - kernel_size + 2×padding) / stride + 1
池化：MaxPool / AvgPool 降低空间维度
残差连接：y = F(x) + x  → 解决梯度消失，训练极深网络
  ResNet: 50/101/152 层
  EfficientNet: 复合缩放（深度×宽度×分辨率）
```

### RNN / LSTM / GRU

```
RNN: h_t = tanh(W_h h_{t-1} + W_x x_t + b)  → 梯度消失/爆炸
LSTM: 遗忘门 f、输入门 i、输出门 o、细胞状态 C
  f_t = σ(W_f·[h_{t-1}, x_t] + b_f)
  C_t = f_t ⊙ C_{t-1} + i_t ⊙ tanh(W_C·[h_{t-1}, x_t])
  h_t = o_t ⊙ tanh(C_t)
GRU: 简化版 LSTM，合并门控，参数更少
```

### Transformer

```
Self-Attention:
  Q = XW_Q,  K = XW_K,  V = XW_V
  Attention(Q,K,V) = softmax(QKᵀ / √d_k) V

Multi-Head: 多组 Q,K,V 并行，拼接后线性投影

位置编码：
  PE(pos, 2i) = sin(pos / 10000^(2i/d))
  PE(pos, 2i+1) = cos(pos / 10000^(2i/d))
  RoPE（旋转位置编码）：LLaMA/GLM 使用
```

```python
import torch, torch.nn as nn, math

class SelfAttention(nn.Module):
    def __init__(self, d_model, n_heads):
        super().__init__()
        self.n_heads = n_heads
        self.d_k = d_model // n_heads
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)

    def forward(self, x, mask=None):
        B, T, C = x.shape
        q = self.W_q(x).view(B, T, self.n_heads, self.d_k).transpose(1, 2)
        k = self.W_k(x).view(B, T, self.n_heads, self.d_k).transpose(1, 2)
        v = self.W_v(x).view(B, T, self.n_heads, self.d_k).transpose(1, 2)
        attn = (q @ k.transpose(-2, -1)) / math.sqrt(self.d_k)
        if mask is not None:
            attn = attn.masked_fill(mask == 0, float('-inf'))
        attn = torch.softmax(attn, dim=-1)
        out = (attn @ v).transpose(1, 2).contiguous().view(B, T, C)
        return self.W_o(out)
```

### 预训练模型

| 模型 | 架构 | 特点 |
|------|------|------|
| BERT | Encoder-only | MLM + NSP 预训练，理解任务 |
| GPT | Decoder-only | 自回归生成 |
| LLaMA | Decoder-only | 开源，RoPE + RMSNorm + SwiGLU |
| GLM | Prefix-Decoder | 通义/智谱，双语 |

### 扩散模型

```
前向（加噪）：q(x_t | x_{t-1}) = N(x_t; √(1-β_t)x_{t-1}, β_t I)
反向（去噪）：学习 ε_θ 预测噪声
  损失：L = E[‖ε - ε_θ(x_t, t)‖²]

DDPM → Stable Diffusion（latent space）→ FLUX（rectified flow）
```

### GAN

```
min_G max_D V(D,G) = E_x[log D(x)] + E_z[log(1-D(G(z)))]

生成器 G 欺骗判别器 D，D 区分真假
应用：图像生成、风格迁移、超分辨率
```

### 归一化

| 方法 | 公式/特点 |
|------|---------|
| BatchNorm | 沿 batch 维归一化，训练/推理行为不同 |
| LayerNorm | 沿特征维归一化，Transformer 标准 |
| RMSNorm | x / RMS(x) · γ，LLaMA 使用，更高效 |

### 损失函数

| 损失 | 公式 | 应用 |
|------|------|------|
| CrossEntropy | -Σ yᵢ log ŷᵢ | 分类 |
| MSE | Σ(yᵢ - ŷᵢ)² | 回归 |
| Focal Loss | -α(1-p)^γ log p | 类别不平衡 |
| Contrastive Loss | max(0, d_ap - d_an + margin) | 度量学习 |

### 训练技巧

- **学习率调度**：Cosine Annealing / Warmup + Decay
- **混合精度**：FP16 前向 + FP32 主权重，加速 2x
- **梯度裁剪**：`torch.nn.utils.clip_grad_norm_(params, max_norm)`
- **EMA**：权重指数移动平均，提升泛化

---

## 4. 自然语言处理

### Tokenization

| 方法 | 原理 |
|------|------|
| BPE | 频率合并字节对，GPT 系列 |
| SentencePiece | 语言无关子词切分，LLaMA |
| WordPiece | 类似 BPE，BERT |

### 词嵌入

Word2Vec（CBOW/Skip-gram）→ GloVe → FastText（子词）→ 上下文嵌入（BERT）

### 大语言模型技术

```
Prompt Engineering: Zero-shot / Few-shot / CoT (Chain of Thought)
RAG: 检索增强生成
  Query → 检索相关文档 → 拼接到 Prompt → LLM 生成
  框架：LangChain / LlamaIndex
RLHF: 人类反馈强化学习
  SFT → Reward Model → PPO 对齐
RLAIF: AI 反馈替代人类反馈
```

```python
# RAG 简化示例
from langchain.vectorstores import FAISS
from langchain.embeddings import OpenAIEmbeddings

db = FAISS.from_texts(documents, OpenAIEmbeddings())
docs = db.similarity_search(query, k=3)
context = "\n".join([d.page_content for d in docs])
prompt = f"基于以下内容回答：\n{context}\n\n问题：{query}"
```

---

## 5. 计算机视觉

| 任务 | 代表模型 | 原理 |
|------|---------|------|
| 分类 | ResNet/EfficientNet/ViT | CNN/Transformer 特征提取 |
| 检测 | YOLOv8/Faster R-CNN/DETR | 锚框/Anchor-free/端到端 |
| 语义分割 | U-Net/DeepLab | 编码器-解码器 |
| 实例分割 | Mask R-CNN/SAM | ROI + mask |
| 生成 | Stable Diffusion/FLUX | 扩散模型 |
| OCR | PaddleOCR/TrOCR | 检测 + 识别 |

---

## 6. 强化学习

```
MDP: (S, A, P, R, γ)
值函数：V(s) = E[Σ γᵗ rₜ | s₀=s]
Q 函数：Q(s,a) = E[Σ γᵗ rₜ | s₀=s, a₀=a]

Q-Learning: Q(s,a) ← Q(s,a) + α[r + γ max Q(s',a') - Q(s,a)]
DQN: 用神经网络近似 Q(s,a)，经验回放 + 目标网络
PPO: Clipped surrogate objective，稳定策略梯度
  L = E[min(r_t(θ)Â_t, clip(r_t(θ), 1-ε, 1+ε)Â_t)]
A3C: 异步多线程并行训练
多智能体：QMIX/MADDPG，协作/竞争场景
```

---

## 7. 工具框架

| 工具 | 定位 | 特点 |
|------|------|------|
| PyTorch | 研究+生产 | 动态图，社区活跃 |
| TensorFlow | 生产 | 静态图优化，TF Serving |
| JAX | 研究 | 函数式，自动向量化，TPU 原生 |
| Hugging Face | 模型/数据/NLP | Transformers/Diffusers/PEFT |
| scikit-learn | 经典 ML | 开箱即用，API 一致 |
| ONNX | 互操作 | 跨框架模型格式 |
| TensorRT | 部署优化 | NVIDIA GPU 推理加速 |

```python
# Hugging Face 微调示例
from transformers import AutoModelForSequenceClassification, Trainer, TrainingArguments

model = AutoModelForSequenceClassification.from_pretrained("bert-base-chinese", num_labels=2)
args = TrainingArguments(
    output_dir="./results", num_train_epochs=3,
    per_device_train_batch_size=16, learning_rate=2e-5,
    evaluation_strategy="epoch"
)
trainer = Trainer(model=model, args=args, train_dataset=train_ds, eval_dataset=eval_ds)
trainer.train()
```
