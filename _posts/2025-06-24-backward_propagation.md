---
layout: post
title: "How Neural Networks Learn: A Deep Dive into Backpropagation"
date: 2025-06-24
tags: [neural-networks, deep-learning, backpropagation, softmax, cross-entropy]
math: true
---
Ever since nonlinear recursive functions (i.e., artificial neural networks) were introduced to the world of machine learning, the range of applications — from chatbots to image generators — has exploded.

But behind every smart AI model is a training process. And at the heart of that training lies one magic trick:

> **Backpropagation.**

Despite being fundamental, it remains one of the most intimidating topics for newcomers.

---

## 🧠 What *Is* Backpropagation?

Backpropagation (short for *backward propagation of errors*) is the process by which a neural network learns from its mistakes.

Think of it this way:

1. You ask the model a question.  
2. It gives you an answer.  
3. You check how wrong it was.  
4. You go backward through the network and adjust all the knobs (weights) so it can do better next time.

This “adjustment” is what backpropagation does using **calculus** — particularly the **chain rule**.


Training neural networks requires adjusting weights to minimize the error between predicted and true labels. This is done using:

- 🔗 **Chain Rule** from calculus  
- 🔥 **Softmax Activation** for multiclass classification  
- 📉 **Cross-Entropy Loss** to measure prediction error  
- 🔄 **Backpropagation** to update weights

This post walks through the math behind these concepts — step by step.

---

## 🚀 Let’s Jump to the Basics

Before diving into math, let’s walk through a full forward-and-backward cycle with a tiny neural network.

We’ll show:
- How the model makes a prediction  
- How we measure its mistake  
- How it updates itself using gradients  

### 🧠 1. Softmax Activation Function

Given logits $z = [z_1, z_2, \dots, z_K]$ from the last layer of the model, the softmax function converts them to probabilities:

$$
\hat{y}_i = \frac{e^{z_i}}{\sum_{j=1}^{K} e^{z_j}}
$$

Properties:
- $\hat{y}_i \in (0, 1)$
- $\sum_{i=1}^K \hat{y}_i = 1$


### 🏷️ 2. One-Hot Encoding & Cross-Entropy Loss

Assume the true label is one-hot encoded:

- For class $c$, $y = [0, 0, \dots, 1, \dots, 0]$ where only $y_c = 1$

Then the cross-entropy loss is:

$$
L = -\sum_{i=1}^{K} y_i \log(\hat{y}_i)
$$

Since only $y_c = 1$, this simplifies to:

$$
L = -\log(\hat{y}_c)
$$

**Example**:  
If predicted output is $\hat{y} = [0.1, 0.2, 0.6, 0.1]$ and the correct class is index 3:

$$
L = -\log(0.6) \approx 0.51
$$


### 🔄 3. Backpropagation: Using the Chain Rule

We aim to compute the gradient:

$$
\frac{dL}{dx} = \frac{dL}{d\hat{y}} \cdot \frac{d\hat{y}}{dz} \cdot \frac{dz}{dx}
$$

Let:
- $z_i = \text{logit for class } i$
- $$
\hat{y}_i = \frac{e^{z_i}}{\sum_{j=1}^n e^{z_j}}
$$ — softmax output
- $L = -\log(\hat{y}_c)$ — cross-entropy loss for class $c$
- $x \in \mathbb{R}^d$, input vector

## 🧮 The Math Behind It All: Gradients Step-by-Step

At this point, we’ve seen *what* backpropagation does and the functions involved for the process

Now let’s see *how* it does that — by computing gradients layer by layer using calculus. Specifically, we’ll break down how the **cross-entropy loss** and **softmax** work together during backpropagation.

This section walks through:
- How to compute the gradient of the loss with respect to the softmax output  
- How the softmax output changes with logits  
- And how everything connects using the **chain rule**

These formulas form the backbone of how your model “learns” to push probabilities in the right direction.

---

### Step 1️⃣ — Derivative of Loss w.r.t. Softmax Output

We start by calculating how the loss changes with respect to each predicted probability $\hat{y}_i$ from the softmax layer.

We will begin with the **cross-entropy loss** for a single example, where the correct class is $c$:

$$
L = -\log(\hat{y}_c)
$$

Now, recall that softmax converts logits $z_k$ into predicted probabilities $\hat{y}_k$:

$$
\hat{y}_k = \frac{e^{z_k}}{\sum_{j=1}^{K} e^{z_j}}
$$

Substituting the softmax expression for $\hat{y}_c$ into the loss:

$$
L = -\log\left(\frac{e^{z_c}}{\sum_{j=1}^{K} e^{z_j}}\right)
$$

Apply the log rule:

$$
L = -z_c + \log\left(\sum_{j=1}^{K} e^{z_j}\right)
$$

Now we differentiate the loss $L$ with respect to each logit $z_k$.

We do this by taking the derivative:

$$
\frac{dL}{dz_k} = \frac{d}{dz_k} \left[ -z_c + \log\left(\sum_{j=1}^{K} e^{z_j} \right) \right]
$$

There are two cases to consider:

---

#### Case 1: $k = c$ : We are computing the gradient with respect to the logit corresponding to the correct class.

- First term derivative: $\frac{d(-z_c)}{dz_c} = -1$
- Second term derivative:
  
  $$
  \frac{d}{dz_c} \log\left(\sum_{j=1}^{K} e^{z_j}\right) = \frac{e^{z_c}}{\sum_{j=1}^{K} e^{z_j}} = \hat{y}_c
  $$

So:

$$
\frac{dL}{dz_c} = -1 + \hat{y}_c = \hat{y}_c - 1
$$

---

#### Case 2: $k \ne c$ : We are computing the gradient with respect to the logit corresponding to a class other than the correct one.

- First term derivative: 0  
- Second term derivative:

  $$
  \frac{d}{dz_k} \log\left(\sum_{j=1}^{K} e^{z_j}\right) = \frac{e^{z_k}}{\sum_{j=1}^{K} e^{z_j}} = \hat{y}_k
  $$

So:

$$
\frac{dL}{dz_k} = \hat{y}_k
$$

### 🚨 Why This Matters (Case 2: $k \ne c$)

When your model assigns **high probability to a wrong class**, it’s not just incorrect — it’s *confidently wrong*. That’s what the gradient helps fix.

We compute the gradient of the loss with respect to the logit $z_k$ for an incorrect class ($k \ne c$):

$$
\frac{dL}{dz_k} = \hat{y}_k
$$

This value is always **positive**, since $\hat{y}_k \in (0, 1)$.


#### 🔎 What This Tells Us:

- If $\hat{y}_k$ is **large**, the model is confidently wrong.  
  → The gradient is large  
  → **Push the logit $z_k$ down** to reduce its influence

- If $\hat{y}_k$ is **small**, the model is unsure about class $k$.  
  → The gradient is small  
  → **Don’t change $z_k$ much**

---

### 🔁 Interpreting Positive vs Negative Gradients

Think of gradients as instructions for moving toward lower loss:

- **Positive gradient**:  
  $\frac{dL}{dz_k} > 0$  
  → Increasing $z_k$ increases loss  
  → So, we should **decrease $z_k$**

- **Negative gradient**:  
  $\frac{dL}{dz_k} < 0$  
  → Increasing $z_k$ decreases loss  
  → So, we should **increase $z_k$**

---

### 🧠 Intuition:

| Class Type        | Gradient               | Action During Training               |
|-------------------|------------------------|--------------------------------------|
| Correct ($k = c$) | $\hat{y}_c - 1$ (negative if $\hat{y}_c < 1$) | Boost this logit's value to improve prediction |
| Incorrect ($k \ne c$) | $\hat{y}_k$ (always positive)         | Lower this logit’s value to reduce confusion     |

This elegant gradient behavior is why softmax + cross-entropy is the go-to combo for classification problems in deep learning.

### ✅ Final Combined Result:

We can now write this as a single equation for all $k$:

$$
\frac{dL}{dz_k} = \hat{y}_k - y_k
$$

Where $y_k = 1$ if $k = c$, else $0$ (i.e., one-hot encoded target).

This result is **elegant, efficient**, and used widely in deep learning frameworks for training classification models.

---

### Step 2️⃣ — Derivative of Softmax Output w.r.t. Logit

Next, we ask: *how does changing a logit $z_k$ affect the predicted probabilities $\hat{y}_i$?*

This gives us the Jacobian of softmax:

$$
\frac{\partial \hat{y}_i}{\partial z_k} =
\begin{cases}
\hat{y}_i(1 - \hat{y}_i) & \text{if } i = k \\
-\hat{y}_i \hat{y}_k & \text{if } i \ne k
\end{cases}
$$

This captures the interaction between logits — softmax isn’t independent across dimensions!

---

### Step 3️⃣ — Combine Using Chain Rule

Now, we apply the **chain rule** to connect the loss to the logits:

$$
\frac{dL}{dz_k} = \sum_i \frac{dL}{d\hat{y}_i} \cdot \frac{d\hat{y}_i}{dz_k}
$$

Let’s compute both cases:

#### Case 1: $k = c$ (correct class)

$$
\frac{dL}{dz_c} = -\frac{1}{\hat{y}_c} \cdot \hat{y}_c (1 - \hat{y}_c) = \hat{y}_c - 1
$$

#### Case 2: $k \ne c$

$$
\frac{dL}{dz_k} = -\frac{1}{\hat{y}_c} \cdot (-\hat{y}_c \hat{y}_k) = \hat{y}_k
$$

---

### ✅ Final Result:

The gradient of the loss with respect to each logit $z_k$ is simply:

$$
\frac{dL}{dz_k} = \hat{y}_k - y_k
$$

This **elegant result** is why the softmax + cross-entropy combo is so popular — the gradient simplifies beautifully!

---

## 🧾 4. Derivative w.r.t. Input: $\frac{dL}{dx}$

Lastly, we connect the gradient all the way back to the input layer.

If each logit $z_k$ was computed as:

$$
z_k = \mathbf{w}_k^\top \mathbf{x} + b_k
$$

Then:

$$
\frac{dz_k}{dx} = \mathbf{w}_k
$$

So the full gradient of the loss with respect to the input vector $x$ is:

$$
\frac{dL}{dx} = \sum_{k=1}^{K} \frac{dL}{dz_k} \cdot \frac{dz_k}{dx} = \sum_{k=1}^{K} (\hat{y}_k - y_k) \cdot \mathbf{w}_k
$$

Or, more compactly in matrix form:

$$
\boxed{
\frac{dL}{dx} = W^\top (\hat{\mathbf{y}} - \mathbf{y})
}
$$

Where:
- $W \in \mathbb{R}^{K \times d}$ is the weight matrix from input to logits  
- $(\hat{y} - y)$ is the prediction error vector

---

## 📌 Summary

| Concept            | Formula |
|--------------------|---------|
| Softmax            | $\hat{y}_i = \frac{e^{z_i}}{\sum_j e^{z_j}}$ |
| Cross-Entropy Loss | $L = -\log(\hat{y}_{\text{true}})$ |
| Gradient w.r.t. $z_k$ | $\frac{dL}{dz_k} = \hat{y}_k - y_k$ |
| Gradient w.r.t. $x$ | $\frac{dL}{dx} = W^\top (\hat{\mathbf{y}} - \mathbf{y})$ |

---

## 💡 Intuition

- Softmax transforms logits to probabilities
- Cross-entropy penalizes confident wrong predictions
- The chain rule enables gradient flow through all layers
- The combined gradient simplifies beautifully to:  $$\boxed{\hat{y}_k - y_k}$$

This efficient gradient powers the entire backpropagation algorithm in modern deep learning frameworks.

---
## 🧪 Example: “The Capital of India Is…”

Let’s solidify everything with a concrete example.

Suppose we’re training a neural network to answer the question:

> ❓ **“The capital of India is…”**

The output layer has 3 neurons (classes):

1. 🏙️ Delhi  
2. 🌆 Bangalore  
3. 🌇 Mumbai

Let’s say the final layer outputs raw scores (logits):

```
z = [2.0, 1.0, 0.1]
```

We apply the **softmax**:

$$
\hat{y}_i = \frac{e^{z_i}}{\sum_j e^{z_j}}
$$

Softmax converts $z$ into probabilities:

```
softmax(z) = [0.659, 0.242, 0.099]
```

So the model is **most confident** that the answer is **Delhi**.

Let’s assume:
- The **correct answer is Delhi** (index 0)  
- The **true label** (one-hot encoded): `[1, 0, 0]`

---

### 🧮 Step 1: Cross-Entropy Loss

$$
L = -\log(\hat{y}_{\text{true}}) = -\log(0.659) \approx 0.417
$$

---

### 🧮 Step 2: Gradients w.r.t. Logits

We apply:

$$
\frac{dL}{dz_k} = \hat{y}_k - y_k
$$

Compute for each class:

- $k = 0$ (Delhi, correct):  
  $\hat{y}_0 - y_0 = 0.659 - 1 = -0.341$

- $k = 1$ (Bangalore):  
  $\hat{y}_1 - y_1 = 0.242 - 0 = 0.242$

- $k = 2$ (Mumbai):  
  $\hat{y}_2 - y_2 = 0.099 - 0 = 0.099$

---

### 💡 What This Means

- Negative gradient for **Delhi**:  
  → Increase its logit to become even more confident

- Positive gradients for **Bangalore** and **Mumbai**:  
  → Decrease their logits to reduce confusion

---

### 🔄 Final Backprop Step: Input Gradient

Assume weights:

```
W = [
  [0.2, 0.5],    # weights for Delhi
  [-0.3, 0.8],   # weights for Bangalore
  [0.1, -0.6]    # weights for Mumbai
]
```

Then:

$$
\frac{dL}{dx} = W^\top (\hat{y} - y)
$$

Compute:

```
dz = [ -0.341, 0.242, 0.099 ]

Wᵗ @ dz =
= [
    (0.2)(-0.341) + (-0.3)(0.242) + (0.1)(0.099),
    (0.5)(-0.341) + (0.8)(0.242) + (-0.6)(0.099)
  ]
= [ -0.068, -0.014 ]
```

This tells us how to nudge the **input layer** to improve the prediction.

---

## ✅ PyTorch Code: Softmax + Cross-Entropy + Manual Backward

```python
import torch
import torch.nn.functional as F

# 1. Inputs and weights (from blog example)
x = torch.tensor([1.0, 1.0], requires_grad=True)  # Input vector
W = torch.tensor([
    [0.2, 0.5],    # Delhi
    [-0.3, 0.8],   # Bangalore
    [0.1, -0.6]    # Mumbai
], requires_grad=True)
b = torch.tensor([0.0, 0.0, 0.0], requires_grad=True)  # Bias

# 2. Logits: z = Wx + b
z = W @ x + b  # shape: [3]
print("Logits (z):", z)

# 3. Softmax
y_hat = F.softmax(z, dim=0)
print("Predicted probabilities (softmax):", [round(p.item(), 3) for p in y_hat])

# 4. True label (class index) – Delhi
y_true_idx = torch.tensor(0)

# 5. Cross-entropy loss manually
loss_manual = -torch.log(y_hat[y_true_idx])
print("Manual cross-entropy loss:", round(loss_manual.item(), 3))

# 6. Built-in CrossEntropyLoss
criterion = torch.nn.CrossEntropyLoss()
loss_builtin = criterion(z.unsqueeze(0), y_true_idx.unsqueeze(0))
print("Built-in CrossEntropyLoss:", round(loss_builtin.item(), 3))

# 7. Manual gradient: dL/dz = y_hat - y_true
y_true_one_hot = torch.zeros_like(y_hat)
y_true_one_hot[y_true_idx] = 1.0
dz = y_hat - y_true_one_hot
print("Manual dL/dz:", [round(d.item(), 3) for d in dz])

# 8. Backprop to input: dL/dx = W.T @ dz
dx = W.T @ dz
print("Manual dL/dx:", [round(v.item(), 3) for v in dx])

# 9. Autograd verification
loss_builtin.backward()
print("Autograd dL/dx:", [round(g.item(), 3) for g in x.grad])

```

---
## 🧠 Conclusion

Backpropagation is the heart of how neural networks learn.

In this post, we:

- Introduced **Softmax** to turn raw scores into probabilities  
- Explained how **Cross-Entropy Loss** penalizes wrong predictions  
- Used the **Chain Rule** to compute gradients step-by-step  
- Showed how the final update boils down to this elegant result:

$$
\frac{dL}{dz_k} = \hat{y}_k - y_k
$$

- Walked through a real-world example (answering “The capital of India is…”)  
- Verified it all in **PyTorch** — both manually and with autograd

This gradient — simple yet powerful — is what enables neural nets to tweak weights, minimize error, and eventually make smarter predictions.
