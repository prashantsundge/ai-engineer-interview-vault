## Transformer Architecture

Here is the complete guide reformatted with clean spacing, distinct section blocks, and isolated LaTeX equations.

**Word (.docx) Formatting Tip:** Modern versions of Microsoft Word natively support LaTeX formulas. If a formula looks like raw code after pasting, simply click on it, press Alt + = (the Word Equation shortcut), ensure the equation type is set to LaTeX in the top menu ribbon, and press Enter or Space to instantly render it into a beautiful mathematical layout.

## Mastering the Transformer Architecture: An Elite Interview Guide

To crack an advanced AI/LLM engineering interview, you cannot treat the Transformer as a black box. Interviewers will test your intuition on why components exist, the mathematical mechanics stabilizing training, and the computational bottlenecks of the system.

Below is a structured, production-grade technical breakdown designed to serve as your comprehensive interview preparation framework.

## Part 1: The Interview Vocabulary Matrix (Cheat Sheet)

If asked to outline the Transformer architecture, these are the core landmarks you must mention in chronological sequence:

- **Tokenization & Input Embeddings:** Mapping continuous text tokens into low-dimensional dense continuous spaces ($\mathbb{R}^{d_{model}}$).
- **Positional Encodings:** Injecting token order awareness into a completely permutation-invariant architecture using static or learned spatial frequencies.
- **Linear Projections ($Q, K, V$):** Mapping raw word representations into functional semantic roles: Queries, Keys, and Values.
- **Scaled Dot-Product Attention:** Calculating global conceptual alignment matrices and applying variance scaling to prevent softmax saturation.
- **Multi-Head Attention (MHA):** Splitting representations into parallel linear subspaces to track distinct semantic and syntactic relationships concurrently.
- **Residual Connections & Layer Normalization:** Combating vanishing/exploding gradients and enforcing hidden-state statistical stability across deep blocks (Pre-LN vs. Post-LN).
- **Position-wise Feed-Forward Networks (FFN):** Introducing structural non-linearities separately at every single sequence index to process attention-gathered properties.
- **Linear Projection & Softmax Out Head:** Mapping processing tensors back up into a wide vocabulary dimension to yield localized token probability distributions.

## Part 2: Architectural Deep Dive (The Step-by-Step Pipeline)

### Step 1: Input Representation (Embeddings & Positional Encoding)

**The Problem**

The self-attention mechanism processes input tokens simultaneously across a set, meaning it is fundamentally permutation-invariant. If you shuffle the words in a sentence, the standard attention matrix outputs the exact same vectors, losing all structural and grammatical meaning.

**The Architecture**

To resolve this, we construct a compound input vector.

- **Token Embedding:** A raw sequence of text tokens is projected via a weight lookup matrix into continuous arrays:

  $$
  \mathbf{X}_{embed} \in \mathbb{R}^{T \times d_{model}}
  $$

  Where $T$ is the sequence length and $d_{model}$ is the internal hidden size.

- **Positional Encoding ($\mathbf{PE}$):** We superimpose fixed or learned deterministic vectors of identical dimensions directly onto the embedding layer. The original paper utilizes static sinusoidal geometric waves:

  $$
  \mathbf{PE}_{(pos, 2i)} = \sin\left(\frac{pos}{10000^{\frac{2i}{d_{model}}}}\right)
  $$

  $$
  \mathbf{PE}_{(pos, 2i+1)} = \cos\left(\frac{pos}{10000^{\frac{2i}{d_{model}}}}\right)
  $$

**Interview Insight:** Why sines and cosines? Because for any fixed offset $k$, $\mathbf{PE}_{pos+k}$ can be represented as a linear function of $\mathbf{PE}_{pos}$. This allows the model to easily learn to attend to relative positions.

The final composite tensor entering the first encoder block is:

$$
\mathbf{X} = \mathbf{X}_{embed} + \mathbf{PE}
$$

### Step 2: The Core Engine — Scaled Dot-Product Attention

**The Architecture**

The hidden state $\mathbf{X}$ is transformed into three specific matrices using learned projection weights $\mathbf{W}^Q, \mathbf{W}^K, \mathbf{W}^V \in \mathbb{R}^{d_{model} \times d_k}$:

$$
\mathbf{Q} = \mathbf{X}\mathbf{W}^Q \quad (\text{Queries})
$$

$$
\mathbf{K} = \mathbf{X}\mathbf{W}^K \quad (\text{Keys})
$$

$$
\mathbf{V} = \mathbf{X}\mathbf{W}^V \quad (\text{Values})
$$

- **Queries ($\mathbf{Q}$):** What a given token is looking for.
- **Keys ($\mathbf{K}$):** What a given token offers to match against.
- **Values ($\mathbf{V}$):** The actual semantic content of the token to be aggregated.

The mathematical calculation of attention is defined as:

$$
\text{Attention}(\mathbf{Q}, \mathbf{K}, \mathbf{V}) = \text{softmax}\left(\frac{\mathbf{Q}\mathbf{K}^T}{\sqrt{d_k}}\right)\mathbf{V}
$$

**The Mathematical Guardrail:** Why divide by $\sqrt{d_k}$?

This is a favorite interview question. Assume the components of $\mathbf{Q}$ and $\mathbf{K}$ are independent random variables with a mean of 0 and a variance of 1. Their vector dot product yields a mean of 0 but a variance of $d_k$.

For large values of $d_k$, the dot products grow extremely large in magnitude. This pushes the softmax function into regions with near-zero gradients (the saturation region), causing the notorious vanishing gradient problem during backpropagation. Dividing the product by $\sqrt{d_k}$ scales the variance back down to 1, stabilizing the softmax distribution.

### Step 3: Expansion to Multi-Head Attention (MHA)

**The Problem**

Single-headed self-attention averages out all relationship vectors across the text sequence. If a word depends on both a grammatical verb nearby and a pronoun reference further away, a single attention head struggles to capture both relationships simultaneously.

**The Architecture**

Instead of computing attention once over $d_{model}$, Multi-Head Attention projects $\mathbf{Q}$, $\mathbf{K}$, and $\mathbf{V}$ into $h$ separate, lower-dimensional linear subspaces ($d_k = d_{model} / h$) running concurrently.

$$
\text{MultiHead}(\mathbf{Q}, \mathbf{K}, \mathbf{V}) = \text{Concat}(\text{head}_1, \dots, \text{head}_h)\mathbf{W}^O
$$

$$
\text{where} \quad \text{head}_i = \text{Attention}(\mathbf{Q}\mathbf{W}_i^Q, \mathbf{K}\mathbf{W}_i^K, \mathbf{V}\mathbf{W}_i^V)
$$

Where the learned parameter projections match the dimensions:

$$
\mathbf{W}_i^Q \in \mathbb{R}^{d_{model} \times d_k}
$$

$$
\mathbf{W}_i^K \in \mathbb{R}^{d_{model} \times d_k}
$$

$$
\mathbf{W}_i^V \in \mathbb{R}^{d_{model} \times d_k}
$$

$$
\mathbf{W}^O \in \mathbb{R}^{h \cdot d_k \times d_{model}}
$$

### Step 4: Stabilization Layers (Residuals & Layer Normalization)

**The Architecture**

The output of the Multi-Head Attention layer is processed through a residual skip connection followed by a Layer Normalization step:

$$
\text{Output}_{MHA} = \text{LayerNorm}(\mathbf{X} + \text{MultiHead}(\mathbf{X}))
$$

**Key Interview Pivot:** Post-LN vs. Pre-LN

- **Post-LN (Original Paper):** Normalization is applied after the residual addition.

  $$
  \mathbf{X}_{l+1} = \text{LayerNorm}(\mathbf{X}_l + \text{SubLayer}(\mathbf{X}_l))
  $$

  **Problem:** Gradients near the output layer are large, but they decay exponentially as they pass back through the residual blocks. This requires strict learning-rate warm-up schedules to avoid divergence.

- **Pre-LN (Modern Standard - GPT, LLaMA):** Normalization is applied directly to the inputs of the sub-layer before the computation.

  $$
  \mathbf{X}_{l+1} = \mathbf{X}_l + \text{SubLayer}(\text{LayerNorm}(\mathbf{X}_l))
  $$

  **Advantage:** Gradients can flow directly through the skip connections from the final layer down to the initial embedding layer. This makes training much more stable and allows you to drop complex warm-up routines.

### Step 5: Position-wise Feed-Forward Networks (FFN)

**The Architecture**

After aggregating context using MHA, the representation passes through an isolated Feed-Forward Network applied to each token index independently and identically. It consists of two linear transformations with an activation function (originally ReLU, modernly GeLU or SwiGLU) in between:

$$
\text{FFN}(\mathbf{Z}) = \max(0, \mathbf{Z}\mathbf{W}_1 + \mathbf{b}_1)\mathbf{W}_2 + \mathbf{b}_2
$$

Where $\mathbf{W}_1 \in \mathbb{R}^{d_{model} \times d_{ff}}$ and $\mathbf{W}_2 \in \mathbb{R}^{d_{ff} \times d_{model}}$. Crucially, the internal dimension $d_{ff}$ is typically configured to be much larger than $d_{model}$ (usually $4 \times d_{model}$).

**Interview Insight:** Why do we need this layer? MHA is purely an aggregation operation—it computes linear combinations of existing value vectors. The FFN layer introduces non-linear feature abstractions and allows the model to process the information gathered by the attention heads.

### Step 6: The Final Prediction Head

At the final output of the decoder stacked block, our processed vector $\mathbf{H} \in \mathbb{R}^{T \times d_{model}}$ needs to be converted back into text tokens.

- **Linear Projection:** A dense linear layer projects $\mathbf{H}$ up to the vocabulary dimension size $|V|$:

  $$
  \mathbf{L} = \mathbf{H}\mathbf{W}_U \quad (\mathbf{W}_U \in \mathbb{R}^{d_{model} \times |V|})
  $$

- **Softmax Output:** This converts these raw logit arrays into structured probabilities:

  $$
  P(y_t | y_{<t}) = \text{softmax}(\mathbf{L}_t)
  $$

## Part 3: High-Frequency Interview "Gotchas"

Be ready for these common conceptual follow-ups:

### Q1: What is the exact Time and Space Complexity of Self-Attention, and why?

- **Time Complexity:** $\mathcal{O}(T^2 \cdot d_{model})$

- **Space Complexity:** $\mathcal{O}(T^2 \cdot h)$ (to store the attention weight matrices for backpropagation).

**The "Why":** To compute the attention matrix, you must perform a dot product between every query ($T$ rows) and every key ($T$ rows). Multiplying two matrices of shapes $(T \times d_{model})$ and $(d_{model} \times T)$ takes quadratic time relative to the sequence length $T$. This quadratic scaling is the primary bottleneck driving the research into modern linear attention mechanisms and flash attention optimization routines.

### Q2: How does the Decoder avoid looking into the future during training?

During training, the entire target sentence is fed into the decoder simultaneously. To prevent the model from cheating by looking ahead at upcoming tokens, we apply Causal Masking.

Before calculating the softmax in the dot-product attention step, an upper-triangular matrix filled with $-\infty$ values is added to the raw score matrix:

$$
\mathbf{M}_{i,j} = \begin{cases} 0 & \text{if } j \le i \\ -\infty & \text{if } j > i \end{cases}
$$

When passed through the softmax function, these $-\infty$ entries drop to exactly 0, zeroing out any attention weights pointing to future token positions.

### Q3: What is the structural difference between Encoder-Only, Decoder-Only, and Encoder-Decoder architectures?

- **Encoder-Only (BERT):** Uses bidirectional attention (every token can look at every other token). Ideal for extraction tasks, classification, and embeddings.

- **Decoder-Only (GPT, LLaMA):** Uses causal masking to ensure tokens can only attend to past positions. Designed for autoregressive text generation.

- **Encoder-Decoder (T5, BART):** The encoder processes a source sequence using bidirectional attention, and the decoder generates a target sequence using causal masking over cross-attention layers. Ideal for translation, summarization, and sequence-to-sequence problems.



## Part 1: Long Short-Term Memory Networks (LSTM)

### 1. The LSTM Vocabulary Matrix (Cheat Sheet)

If asked to trace the execution of an LSTM, you must explicitly outline these parameters in logical sequence:

- **Vanishing Gradient Bottleneck**: The structural failure of standard Recurrent Neural Networks (RNNs) where long-range temporal dependencies decay exponentially during backpropagation through time (BPTT).
- **Cell State ($c_t$)**: The internal conveyor belt running linearly down the sequence, acting as a long-term memory buffer protected by linear updates.
- **Hidden State ($h_t$)**: The short-term memory vector that is output by the cell at each step, serving as the localized sequence representation.
- **Forget Gate ($f_t$)**: A sigmoid-activated network component determining what percentage of historical information to drop from the cell state.
- **Input Gate ($i_t$)**: A sigmoid layer controlling which newly arrived token attributes are permitted to write into the cell state.
- **Candidate Cell State ($\tilde{c}_t$)**: A hyperbolic tangent ($\tanh$) activation layer that models the raw vector of newly extracted features.
- **Output Gate ($o_t$)**: A control gate determining what proportion of the updated cell state is exposed to the subsequent hidden state.

### 2. Architectural Deep Dive & Gating Mechanics

#### The Problem

Standard Recurrent Neural Networks multiply hidden states across sequence time-steps using a shared weight matrix $W_{hh}$. If the largest eigenvalue of $W_{hh}$ is less than 1, multiplying this matrix repeatedly across a sequence of length $T$ causes the gradient to vanish exponentially ($\mathcal{O}(\gamma^T)$). This renders the network incapable of learning long-range context.

#### The Architecture

The LSTM introduces a dedicated linear channel called the Cell State ($c_t$) alongside specific multiplicative gating structures to control information flow.

##### Step 1: The Forget Gate

The current input vector $x_t$ and the previous hidden state $h_{t-1}$ are evaluated to determine what historical context to discard:

$$
f_t = \sigma(W_f \cdot [h_{t-1}, x_t] + b_f)
$$

Where $\sigma(z) = \frac{1}{1 + e^{-z}}$, compressing values strictly between 0 (completely drop) and 1 (completely preserve).

##### Step 2: The Input Gate & Candidate State

The network next identifies what new information should be added to the cell state. First, the input gate determines which dimensions to update:

$$
i_t = \sigma(W_i \cdot [h_{t-1}, x_t] + b_i)
$$

Concurrently, a candidate state creates a vector of new prospective values using a $\tanh$ activation function to map data between -1 and 1:

$$
\tilde{c}_t = \tanh(W_c \cdot [h_{t-1}, x_t] + b_c)
$$

##### Step 3: Updating the Cell State

The old cell state $c_{t-1}$ is updated into the new cell state $c_t$ via direct linear operations. This design choice prevents gradients from vanishing during backpropagation:

$$
c_t = f_t \odot c_{t-1} + i_t \odot \tilde{c}_t
$$

Where $\odot$ represents the element-wise Hadamard product.

##### Step 4: The Output Gate & Hidden State

Finally, the output gate regulates what information from the updated cell state is exposed as the new hidden state $h_t$:

$$
o_t = \sigma(W_o \cdot [h_{t-1}, x_t] + b_o)
$$

$$
h_t = o_t \odot \tanh(c_t)
$$

### 3. High-Frequency LSTM "Gotchas"

#### Q1: Why do LSTMs use $\tanh$ for the candidate state/hidden update but $\sigma$ for the gates?

- **Gates ($\sigma$)**: Sigmoid outputs values strictly between 0 and 1. This math matches the physical property of a valve or switch—0 means "completely block," and 1 means "completely pass."
- **State Updates ($\tanh$)**: Hyperbolic tangent outputs values between -1 and 1. This range is essential for normalization, preventing the values in the cell state from driving towards infinity during repeated additions. It also allows the model to actively pull values down (negative updates).

#### Q2: What is the fundamental operational bottleneck of an LSTM compared to a Transformer?

The "Why": LSTMs are sequentially constrained. To compute the state at time-step $t$, you must first compute the hidden state at $t-1$. This creates an unbreakable serial dependency across the sequence length $T$.

As a result, training cannot be parallelized across time-steps, making it impossible to scale LSTMs effectively on modern GPU clusters. Transformers eliminate this requirement by processing all tokens simultaneously via self-attention matrices.

## Part 2: Bidirectional Encoder Representations from Transformers (BERT)

### 1. The BERT Vocabulary Matrix (Cheat Sheet)

If asked to differentiate BERT from generative models, focus on these defining landmarks:

- **Encoder-Only Stack**: A multi-layer architecture built entirely from standard Transformer encoder blocks, omitting causal masking to enable native, multi-directional contextualization.
- **Masked Language Modeling (MLM)**: The primary fill-in-the-blank pre-training task where a percentage of tokens are obscured, forcing the model to infer semantics from both left and right contexts simultaneously.
- **Next Sentence Prediction (NSP)**: A binary classification pre-training head that evaluates whether Sentence B naturally follows Sentence A.
- **Special Tokens ([CLS], [SEP], [MASK])**: Non-text markers injected into the sequence to handle classification tasks, designate sequence boundaries, and act as prediction targets.
- **Segment Embeddings**: A learned dense representation layer added directly to token embeddings to indicate sequence identity during multi-sentence processing.

### 2. Deep Dive: Input Fusion & Pre-Training Mechanics

#### The Problem

Standard autoregressive language models (like GPT) apply a triangular causal mask to prevent tokens from looking ahead. While this is necessary for generation, it creates a limitation for extraction or classification tasks because a token can only build representations based on its preceding context.

#### The Architecture

BERT removes the causal mask completely. Every single token in the sequence can attend to every other token simultaneously across all layers.

To maintain a structured layout when processing pairs of sentences, the final input representation $\mathbf{X}$ is constructed by summing three distinct embedding layers:

$$
\mathbf{X} = \mathbf{X}_{Token} + \mathbf{X}_{Segment} + \mathbf{X}_{Position}
$$

- **Token Embeddings**: WordPiece token identifiers mapping down to dense vectors.
- **Segment Embeddings**: A learned embedding vector indicating whether a token belongs to sentence $A$ or sentence $B$.
- **Position Embeddings**: Learned absolute position vectors (unlike the fixed sinusoids of the original Transformer).

#### The Masking Strategy: The 80-10-10 Mixing Rule

To train a bidirectional model without it simply copying the answer, BERT uses Masked Language Modeling. It randomly selects 15% of the input tokens as optimization targets. To prevent a mismatch between pre-training and downstream fine-tuning (where the [MASK] token does not exist), those selected tokens undergo a split modification:

- 80% of the time: Replaced with the literal token [MASK].
- 10% of the time: Replaced with a completely random token from the vocabulary.
- 10% of the time: Kept exactly as the original token.

The objective function optimizes the cross-entropy loss over the masked positions:

$$
\mathcal{L}_{MLM} = -\sum_{i \in \text{Masked}} \log P(x_i | \mathbf{X}_{\setminus i})
$$

### 3. High-Frequency BERT "Gotchas"

#### Q1: Why does BERT use the 10% random and 10% unchanged rule in MLM?

If BERT only saw the token [MASK] at the target positions, the encoder would learn to generate high-quality contextual representations only for the [MASK] token. By keeping 10% unchanged, the model learns to maintain accurate representations for normal words. By including 10% random words, it forces the model to remain critical of its inputs and rely on the surrounding context to resolve anomalies, which improves overall semantic understanding.

#### Q2: Why is BERT poorly suited for real-time text generation?

BERT is a non-causal bidirectional model. Because it lacks a causal mask, it does not have an efficient mechanism for autoregressive generation (predicting token $t+1$ given tokens $1$ to $t$).

Generating a sentence with BERT requires an iterative, computationally heavy process: you must pass a template string containing a [MASK] token through the model, sample the highest probability word, replace the [MASK] with that word, append a new [MASK] token, and re-run the entire sequence through the network. This makes it highly inefficient for generation compared to decoder-only models.
