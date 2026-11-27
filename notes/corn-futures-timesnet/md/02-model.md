---
title: Model — TimesNet with a classification head
order: 2
---

# Model: TimesNet with a classification head

## 1. Input embedding

A sample is a window $X \in \mathbb{R}^{T \times C}$ with $T = 60$ days and $C = 7$ standardized features, plus calendar marks (month, day of month, weekday, hour). The embedding is the sum of three terms:

$$E = \underbrace{\mathrm{Conv1d}_{3}(X)}_{\text{value}} + \underbrace{\mathrm{PE}}_{\text{positional}} + \underbrace{\sum_{u \in \{\text{month, day, weekday, hour}\}} W_u[\text{mark}_u]}_{\text{temporal}} \in \mathbb{R}^{T \times d},$$

with $d = 64$. The value embedding is a 1-D convolution with kernel size 3 over time, padded causally: two copies of the first day are added on the left and none on the right, so that

$$\mathrm{Conv1d}_{3}(X)_t = \sum_{j=0}^{2} W_j\, x_{\max(t-2+j,\,1)}, \qquad t = 1, \dots, T,$$

and each embedded day depends only on itself and the two days before it. The code behind the paper's results padded circularly, so the first embedded day also mixed in the newest day of the window and the last one the oldest (note 5, section 1.4). The positional embedding is the fixed sinusoidal encoding of Vaswani et al. (2017). With `embed = learned`, each calendar mark is looked up in a trainable table $W_u$. So "learned" refers to the **calendar** embedding; the positional embedding stays fixed. With daily data the hour index is always 0.

A linear map over the time axis extends the sequence to the forecast horizon $H = 10$:

$$Z^{(0)} = \left(W_{\text{pred}} E^\top\right)^\top \in \mathbb{R}^{(T+H) \times d}, \qquad W_{\text{pred}} \in \mathbb{R}^{(T+H) \times T}.$$

## 2. TimesBlock

TimesNet's premise is that a time series is a superposition of periodic components, and that a 1-D series folded along one of its periods becomes a 2-D array whose rows carry *inter-period* variation and whose columns carry *intra-period* variation. A 2-D convolution can then model both at once. Each TimesBlock $\ell$ maps $Z^{(\ell-1)} \in \mathbb{R}^{L \times d}$, with $L = T + H = 70$, as follows.

**Period detection.** Take the real FFT along time and average the amplitudes over the batch and the channels:

$$A_f = \frac{1}{B\,d} \sum_{b=1}^{B} \sum_{c=1}^{d} \left| \mathrm{FFT}\big(Z^{(\ell-1)}_{b,\cdot,c}\big)_f \right|, \qquad f = 1, \dots, \lfloor L/2 \rfloor ,$$

with the zero frequency excluded. Keep the $k = 5$ frequencies with the largest $A_f$, $\{f_1, \dots, f_k\}$, and set the periods $p_i = \lfloor L / f_i \rfloor$.

**Fold, convolve, unfold.** For each $p_i$, zero-pad the sequence to a multiple of $p_i$, reshape it to $\lceil L/p_i \rceil \times p_i \times d$, apply an Inception block (parallel 2-D convolutions with kernel sizes $1, 3, 5, 7, 9, 11$, averaged), a GELU, and a second Inception block back to $d$ channels, then reshape to length $L$:

$$\hat Z^{(i)} = \mathrm{Unfold}\Big(\mathrm{Incep}_{d_{ff} \to d}\big(\mathrm{GELU}(\mathrm{Incep}_{d \to d_{ff}}(\mathrm{Fold}_{p_i}(Z^{(\ell-1)})))\big)\Big), \qquad d_{ff} = 256 .$$

**Aggregate.** Weight the $k$ branches by a softmax over each sample's own amplitudes at the selected frequencies, then add a residual connection and layer normalization:

$$Z^{(\ell)} = \mathrm{LayerNorm}\left(Z^{(\ell-1)} + \sum_{i=1}^{k} \frac{e^{A_{b,f_i}}}{\sum_j e^{A_{b,f_j}}}\, \hat Z^{(i)}\right).$$

Two TimesBlocks are stacked.

## 3. Classification head

The upstream forecasting head projects each of the $L$ time steps back to the $C$ channels and is trained with mean squared error. This project replaces it with a classifier over the $K = 8$ return buckets:

$$z = W_{\text{out}}\, \mathrm{vec}\Big(\mathrm{Dropout}\big(\mathrm{GELU}(Z^{(2)})\big)\Big) + b, \qquad W_{\text{out}} \in \mathbb{R}^{K \times L d}, \quad z \in \mathbb{R}^{K},$$

and the predicted distribution is $q = \mathrm{softmax}(z / T_{\text{cal}})$, where $T_{\text{cal}}$ is the calibration temperature (note 3). The head flattens all $L = 70$ positions, including the 10 positions produced by the horizon extension $W_{\text{pred}}$.

## 4. Training

Training minimizes the cross-entropy between $q$ and the observed bucket $y$:

$$\mathcal{L}(\theta) = -\frac{1}{N} \sum_{n=1}^{N} \log q_{y_n}(x_n;\theta).$$

| Setting | Value |
|---|---|
| Optimizer | Adam, initial learning rate $10^{-2}$, gradient-norm clipping at 4 |
| Learning-rate schedule | halved after every epoch from the second on, $\eta_e = 10^{-2} \cdot 0.5^{\max(e-2,\,0)}$ in epoch $e$ (`lradj = type1`, applied after every epoch as upstream); the paper run applied it only every 5 epochs, a single drop to $1/16$ after epoch 5 (note 5, section 1.5) |
| Batch size | 32, shuffled, last incomplete batch dropped |
| Regularization | dropout 0.2, early stopping on validation loss with patience 3 |
| Epochs | at most 10; the best checkpoint was at epoch 6 |

## 5. Changes from the upstream TimesNet

| Component | Upstream (long-term forecasting) | This project |
|---|---|---|
| Output | $L \times C$ values | $K = 8$ logits from the flattened sequence |
| Loss | MSE | cross-entropy |
| Value-embedding padding | circular | causal: the first day repeated on the left (section 1) |
| Per-window normalization | each window centered and scaled per channel, and de-normalized at the output (from the Non-stationary Transformer) | removed; global standardization with training statistics only |
| Calibration | none | temperature scaling fitted on the validation set |

The TimesBlock itself (FFT period selection, Inception blocks, aggregation) is unchanged.

## 6. Size

| Part | Parameters |
|---|---|
| Two TimesBlocks (Inception kernels) | 18,747,136 |
| Classification head | 35,848 |
| Embedding | 6,208 |
| Horizon extension $W_{\text{pred}}$ | 4,270 |
| Layer normalization | 128 |
| **Total** | **18,793,590** |

Almost all of the capacity sits in the Inception blocks: each one holds $\sum_{s \in \{1,3,5,7,9,11\}} s^2 = 286$ spatial weights per input/output channel pair. Note 4 compares this with the amount of independent information in the data.

## References

- Wu, H. et al. (2023). TimesNet: Temporal 2D-Variation Modeling for General Time Series Analysis. ICLR.
- Liu, Y., Wu, H., Wang, J. and Long, M. (2022). Non-stationary Transformers: Exploring the Stationarity in Time Series Forecasting. NeurIPS.
- Szegedy, C. et al. (2015). Going Deeper with Convolutions. CVPR.
- Vaswani, A. et al. (2017). Attention Is All You Need. NeurIPS.
