---
layout: post
title: "Linear Regression"
date: 2025-09-06
tags: [Generalization, Gradient-Decent ]
math: true
---

### Linear Regression - Supervised Learning - Generlization
--------------------

suppose want to predict the price of houses
    Size(sqft) --- price($$)

Training Set -> Learning Algo -> prediction function

size -> prediction function h(x)  -> price

how to represent h(x) = $\theta_{0} + \theta_{1}x$

now you have two features
- x1 = size
- x2 = bedrooms

Linear Regression - h(x1, x2) = $\theta_{0} + \theta_{1}x1 + \theta_{2}x2$

## Notations

hypothesis = $h(x) = \sum_{j=0}^{2} \theta_j * x_j$ where $x_0$ is 1

$\theta$ = $[\theta_0, \theta_1, \theta_2]$
features = x = $[x_0, x_1, x_2]$
h(x) choose the correct \theta value

$\theta$ -> parameters

m -> number of training examples -> number of rows

x -> inputs / features

y -> output

(x,y) -> one training example

$(x^i, y^i)$ - ith training sample

n -> number of fearures

--------------

### How to choose parameter $\theta$ ?

choose θ such that hθ​(x)≈y for training example $h_\theta(x) = h(x)$

$h_\theta(x)$ means it depends on both x and θ 

**we want to minimize square diff** $(h_\theta(x)-y)^2$ **For all training examples (minimize error)** -
$
\min_{\theta} \; \sum_{i=1}^{m} \big( h_\theta(x^{(i)}) - y^{(i)} \big)^2
$

**cost function** -
$
J(\theta) = \frac{1}{2} \sum_{i=1}^{m} \Big(h_\theta(x^{(i)}) - y^{(i)}\Big)^2
$

**Why the $\tfrac{1}{2}$ in front?**
-------------------
It simplifies gradient descent math.

When you take derivative of the squared term:

$
\frac{d}{d\theta}\Big( (h_\theta(x) - y)^2 \Big)
= 2 \cdot (h_\theta(x) - y) \cdot \frac{d}{d\theta}h_\theta(x)
$

That **2 cancels out** the $\tfrac{1}{2}$ in front, leaving a clean expression.

It doesn’t change the minimizer — it just makes the calculus cleaner.

-----------------------

### How to minimize $J(\theta)$ ?

- One such algorithm is **Gradient Decent**

- Start with some $\theta$ (say $\theta = \vec{0}$)
- Keep changeing θ to reduce J(θ)

Gradient decent for linear regression will not be local optimum


derivetives of a function defines direction of steepest decent

**gradient descent update** - $\theta_j := \theta_j - α\frac{\partial J(\theta)}{\partial \theta_j}$

- α -> learning rate
- $\theta_j$ -> the jth parameter (weight) of your model.
- := -> means “update/assign” (replace the old $θ_j$ with the new value).
- $
\frac{\partial J(\theta)}{\partial \theta_{j}}$ -> the derivative (slope) of the cost function w.r.t. parameter $\theta_{j}$

**Intuition 🚀**
- You compute the slope (gradient) of the cost function at the current  $\theta_{j}$
- Multiply it by the learning rate α.
- Subtract it from the current parameter → move downhill on the cost function.

**cost function** - $
J(\theta)
= \frac{1}{2} \sum_{i=1}^m \Big( h_\theta(x^{(i)}) - y^{(i)} \Big)^2
$

taking for one single training example , which means m = 1

$
\frac{\partial J(\theta)}{\partial \theta_{j}}
=  \frac{\partial}{\partial \theta_j} (\frac{1}{2} \sum_{i=1}^1 \Big( h_\theta(x^{(i)}) - y^{(i)} \Big)^2)
$

= $2 ⋅ \frac{1}{2} ⋅ \Big (h_\theta(x) - y \Big) ⋅ \frac{\partial}{\partial \theta_j}\Big (h_\theta(x) - y \Big)$

= $\Big (h_\theta(x) - y \Big) ⋅ \frac{\partial}{\partial \theta_j} \Big (h_\theta(x) - y \Big)$

= $\Big (h_\theta(x) - y \Big) ⋅ \frac{\partial}{\partial \theta_j} \Big (\theta_{0}x_{0} + \theta_{1}x_{1} + .... + \theta_{n}x_{n} - y \Big)$

now think derivites of $\Big (\theta_{0}x_{0} + \theta_{1}x_{1} + .... + \theta_{n}x_{n} - y \Big)$ in respectice of $\theta_j $ is (0 + 0 + 0 + .... + $ \Big (\frac{\partial}{\partial \theta_j}  \theta_j x_j \Big)$ + 0 + 0 + ...) = $x_j$

so now,

$\Big (h_\theta(x) - y \Big) ⋅ \frac{\partial}{\partial \theta_j} \Big (\theta_{0}x_{0} + \theta_{1}x_{1} + .... + \theta_{n}x_{n} - y \Big)$

= $\Big (h_\theta(x) - y \Big) ⋅ x_j$


now, as we explained before in **gradient descent update**
$\theta_j := \theta_j - α\frac{\partial J(\theta)}{\partial \theta_j}$

= $\theta_j - α ⋅ \Big (h_\theta(x) - y \Big) ⋅ x_j$


taking for m training example = $\theta_j - α ⋅ \Big (\sum_{i=1}^{m}  h_\theta(x^i) - y^i \Big) ⋅ x_j^i$

$x^i$ -> ith trainig example

$y^i$ -> target label

We **repeat the update step until convergence**.  
For each iteration of Gradient Descent, we update all parameters $\theta_j$ simultaneously:

$$
\theta_j := \theta_j - \alpha \cdot \frac{1}{m} \sum_{i=1}^{m} \big( h_\theta(x^{(i)}) - y^{(i)} \big) \cdot x_j^{(i)}, \quad j = 0, 1, \dots, n
$$

Where:
- $n$ = number of features
- $\alpha$ = learning rate

---

### Choosing the Learning Rate $\alpha$

- If **$\alpha$ is too large** → gradient descent may **overshoot** and run past the minimum.
- If **$\alpha$ is too small** → convergence will be **very slow**, requiring more computation.

A common approach is to **start with $\alpha = 0.01$** and calibrate this value by experimentation.


This is also known as **Batch Gradient Desecnt**

the problem with this approach is in order to make one update to parameter (in order to even take a single step of gradient descent) we need to calculate the sum - $\sum_{i=1}^{m}$

If m is huge, mean a huge dataset, we need scan through entire dataset , we have to calculate it for all the m - which is going to be pretty slow and expensive

---------

### Stochastic Gradient Decent

Instead of scanning through all the m for each step

```
Repeat {
    For i = 1 to m {
        𝜃𝑗 := 𝜃𝑗 - α⋅(ℎ𝜃(𝑥)−𝑦)⋅𝑥𝑗
        // α (ℎ𝜃(𝑥)−𝑦)⋅𝑥𝑗 is taking only one training example
    }
}
```

if you take the contour, after completing one step it will not go steep downhill, rather a noisy path but on avg it is headed to global minimum. Better is decreasing learning rate + Stochastic Decent

This is faster when you have large dataset

-------

### Normal Equation - Vectorized form - only works for linear regression


**Cost function:**

$$
J(\theta) = \frac{1}{2} \; \lVert X\theta - y \rVert^2
$$

$$
\nabla_\theta J(\theta) = \frac{1}{2} X^\top \big( X\theta - y \big)
$$


$$
X =
\begin{bmatrix}
x_1^{(1)} & x_2^{(1)} \\
x_1^{(2)} & x_2^{(2)} \\
x_1^{(3)} & x_2^{(3)}
\end{bmatrix},
\quad
\theta =
\begin{bmatrix}
\theta_1 \\
\theta_2
\end{bmatrix}
$$

$$
X\theta =
\begin{bmatrix}
x_1^{(1)} & x_2^{(1)} \\
x_1^{(2)} & x_2^{(2)} \\
x_1^{(3)} & x_2^{(3)}
\end{bmatrix}
\begin{bmatrix}
\theta_1 \\
\theta_2
\end{bmatrix}
=
\begin{bmatrix}
\theta_1 x_1^{(1)} + \theta_2 x_2^{(1)} \\
\theta_1 x_1^{(2)} + \theta_2 x_2^{(2)} \\
\theta_1 x_1^{(3)} + \theta_2 x_2^{(3)}
\end{bmatrix}
$$


---

## Q&A

**Q. Is it possible to start with Stochastic Gradient Descent (SGD) and switch to Batch Gradient Descent?**  

**A.** Yes, it is. In practice this is called **Mini-Batch Gradient Descent**, where you process small batches of data instead of one example at a time (SGD) or the entire dataset (Batch GD). This combines the **speed of SGD** with the **stability of Batch GD**.  

---

**Q. When to stop the Gradient Descent?**  

**A.** Monitor the cost function $J(\theta)$. When it stops decreasing significantly, that’s the point to stop.  

Since **Linear Regression has a convex cost function** (no local optima), you are less likely to run into convergence debugging issues. The algorithm will reliably approach the global minimum.

![](/assets/img/gradient_decent.png)