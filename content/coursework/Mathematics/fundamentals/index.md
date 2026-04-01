---
title: Mathematics for AI
date: 2026-03-03
tags:
  - UNIVERSITY
  - 선형대수
featured: true
weight: 10
summary_line: Core concepts of reinforcement learning from MDP to modern policy optimization methods.
summary: 선형대수, ML basics
---

{{< katex >}}

## Overview

Reinforcement Learning (RL) is a framework for learning from interaction with an environment. The agent learns to make decisions by receiving rewards or penalties, aiming to maximize cumulative reward over time.

---

## 1. Markov Decision Process (MDP)

**Definition:** An MDP is a mathematical framework for modeling decision-making problems where outcomes are partly random and partly under the control of a decision-maker.

**Components (S, A, P, R, γ):**
- **S**: State space (all possible states)
- **A**: Action space (all possible actions)
- **P**: Transition probability $P(s'|s, a)$
- **R**: Reward function $R(s, a, s')$
- **γ**: Discount factor (0 ≤ γ ≤ 1)

**Markov Property:** The future depends only on the current state, not on the sequence of events that preceded it.

$$P(s_{t+1} | s_t, a_t, s_{t-1}, a_{t-1}, ...) = P(s_{t+1} | s_t, a_t)$$

**Why it matters:** The Markov assumption makes RL tractable by allowing us to model environments without tracking the entire history.

---

## 2. Value Functions

**State Value Function V(s):** Expected return starting from state s and following policy π.

$$V_π(s) = E_π[G_t | s_t = s] = E_π\left[\sum_{k=0}^\infty γ^k R_{t+k+1} | s_t = s\right]$$

**Action Value Function Q(s, a):** Expected return starting from state s, taking action a, then following policy π.

$$Q_π(s, a) = E_π[G_t | s_t = s, a_t = a]$$

**Bellman Equations:** Recursive relationships that decompose value into immediate reward and future value.

$$V_π(s) = \sum_a π(a|s) \sum_{s'} P(s'|s,a) [R(s,a,s') + γV_π(s')]$$

$$Q_π(s,a) = \sum_{s'} P(s'|s,a) [R(s,a,s') + γ\sum_{a'} π(a'|s') Q_π(s',a')]$$

**Why it matters:** Bellman equations enable dynamic programming and form the basis for most RL algorithms.

---

## 3. Policy

**Definition:** A mapping from states to action probabilities.

**Types:**
- **Deterministic:** π: S → A (always same action for given state)
- **Stochastic:** π: S × A → [0,1] (probability distribution over actions)

**Goal:** Find optimal policy π* that maximizes expected return.

$$π^* = \arg\max_π E_π[G_t]$$

---

## 4. Q-Learning

**Type:** Off-policy value-based method

**Update Rule:**
$$Q(s,a) ← Q(s,a) + α[R + γ\max_{a'} Q(s',a') - Q(s,a)]$$

**Key Properties:**
- Learns optimal Q-function independently of the policy being followed
- Uses TD (Temporal Difference) learning
- Model-free (doesn't require knowing P and R)

**Limitations:**
- Works only for discrete action spaces
- Can be unstable with function approximation
- Tends to overestimate Q-values

---

## 5. Policy Gradient Methods

**Type:** On-policy policy-based methods

**Idea:** Directly optimize the policy parameters θ by gradient ascent on expected return.

**Policy Gradient Theorem:**
$$\nabla_θ J(θ) = E_π[\nabla_θ \log π_θ(a|s) Q_π(s,a)]$$

**REINFORCE Algorithm:**
```
for each episode:
    Generate trajectory (s₀, a₀, r₀, ..., s_T)
    for t = 0 to T-1:
        G = r_t + γr_{t+1} + ... + γ^{T-t-1}r_{T-1}
        ∇θ J(θ) ≈ ∇θ log π_θ(a_t|s_t) * G
        θ ← θ + α * ∇θ J(θ)
```

**Advantages:**
- Works with continuous action spaces
- Can find stochastic policies
- More sample-efficient than value-based methods in some cases

**Disadvantages:**
- High variance in gradient estimates
- Can be slow to converge
- May get stuck in local optima

---

## 6. Actor-Critic Methods

**Idea:** Combine value-based and policy-based approaches.

**Components:**
- **Actor:** Policy network π_θ(a|s) that selects actions
- **Critic:** Value network V_φ(s) or Q_φ(s,a) that evaluates states/actions

**Update:**
- Critic learns to predict values (minimize TD error)
- Actor uses critic's evaluation to update policy

**Advantages:**
- Lower variance than pure policy gradient
- More sample-efficient than pure value-based methods
- Can handle continuous action spaces

---

## 7. Proximal Policy Optimization (PPO)

**Type:** Modern actor-critic algorithm with clipped surrogate objective

**Motivation:** Balance between sample efficiency and stability. Avoid large policy updates that can destroy learned behavior.

**Clipped Surrogate Objective:**
$$L^{CLIP}(θ) = E_t[min(r_t(θ)\hat{A}_t, clip(r_t(θ), 1-ε, 1+ε)\hat{A}_t)]$$

Where:
- $r_t(θ) = \frac{π_θ(a_t|s_t)}{π_{θ_{old}}(a_t|s_t)}$ (probability ratio)
- $\hat{A}_t$ is the advantage estimate
- ε is a hyperparameter (typically 0.1-0.3)

**Key Features:**
- **Clipping:** Prevents large policy updates
- **Multiple epochs:** Reuse samples for efficiency
- **Generalized Advantage Estimation (GAE):** Reduces variance in advantage estimates

**Why PPO is popular:**
- Simple and robust
- Works well across many domains
- Good balance between performance and stability
- Fewer hyperparameters to tune compared to TRPO

---

## 8. Deep Q-Network (DQN)

**Type:** Combines Q-learning with deep neural networks

**Key Innovations:**
1. **Experience Replay:** Store transitions in replay buffer, sample randomly for training
2. **Target Network:** Separate network for computing target Q-values, updated periodically

**Loss Function:**
$$L(θ) = E[(R + γ\max_{a'} Q_{θ̄}(s',a') - Q_θ(s,a))^2]$$

**Advantages:**
- Can learn from high-dimensional inputs (e.g., images)
- More stable training than vanilla Q-learning with neural nets
- Sample efficient due to experience replay

---

## Comparison Table

| Algorithm | Type | Action Space | Stability | Sample Efficiency |
|-----------|------|--------------|-----------|-------------------|
| Q-Learning | Value-based | Discrete | Medium | Low |
| REINFORCE | Policy-based | Continuous | Low | Medium |
| Actor-Critic | Hybrid | Continuous | Medium | High |
| PPO | Actor-Critic | Both | High | High |
| DQN | Value-based | Discrete | High | Medium |

---

## When to Use What

**Q-Learning / DQN:**
- Discrete action spaces
- When you need optimal policy
- Simpler environments

**Policy Gradient / Actor-Critic:**
- Continuous action spaces
- When stochastic policies are beneficial
- Complex, high-dimensional environments

**PPO:**
- General-purpose choice
- When stability is important
- When you want good performance with minimal tuning

---

## Key Takeaways

1. **MDP** provides the mathematical foundation for RL
2. **Value functions** enable decomposition of long-term objectives
3. **Policy gradients** directly optimize policies, handling continuous actions
4. **Actor-Critic** combines the best of value-based and policy-based methods
5. **PPO** is currently the most popular choice due to its stability and performance
6. **Function approximation** (neural networks) enables scaling to complex environments

---

## References

- Sutton, R. S., & Barto, A. G. (2018). *Reinforcement Learning: An Introduction*. MIT Press.
- Schulman, J., et al. (2017). Proximal Policy Optimization Algorithms. arXiv:1707.06347.
- Mnih, V., et al. (2015). Human-level control through deep reinforcement learning. Nature.
