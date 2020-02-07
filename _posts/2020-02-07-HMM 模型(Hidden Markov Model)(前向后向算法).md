## HMM 模型(Hidden Markov Model)(前向后向算法)

HMM的问题一是:已知HMM模型的参数$\lambda = (A, B, \Pi)$。其中𝐴是隐藏状态转移概率的矩阵，𝐵是观测状态生成概率的矩阵， $\Pi$是隐藏状态的初始概率分布。同时我们也已经得到了观测序列$O =\{o_1,o_2,...o_T\}$。现在我们要求观测序列𝑂在模型λ下出现的条件概率𝑃(𝑂|𝜆)。

### 暴力求解

根据HMM的第二个假设，我们知道观测序列肯定是与$I$相关的，我们需要在原始公式$𝑃(𝑂|𝜆)$中加入变量$I$,根据联合概率公式
$$
P(O|\lambda) = \sum\limits_{I}P(O,I|\lambda)  =\sum\limits_{I}P(O,I|\lambda) = \sum\limits_{I}P(I|\lambda)P(O|I, \lambda)
$$
其中
$$
P(I|\lambda) = p(i_T|i_{T-1},i_{T-2},...i_1|\lambda)*p(i_{T-1},i_{T-2},...i_1|\lambda)
$$
根据HMM的第一个假设其中$p(i_T|i_{T-1},i_{T-2},...i_1|\lambda) = p(i_T|i_{t-1})$ ，因此
$$
P(I|\lambda) = \pi_{i_1} a_{i_1i_2} a_{i_2i_3}... a_{i_{T-1}\;\;i_T}
$$
同理,使用HMM的第二个假设可得：
$$
P(O|I, \lambda) = b_{i_1}(o_1)b_{i_2}(o_2)...b_{i_T}(o_T)
$$
因此：
$$
P(O|\lambda) = \sum\limits_{i_1,i_2,...i_T}\pi_{i_1}b_{i_1}(o_1)a_{i_1i_2}b_{i_2}(o_2)...a_{i_{T-1}\;\;i_T}b_{i_T}(o_T)
$$
观察到我们序列的长度为T，每个时间点取值的可能性为N，因此直接穷举的话时间复杂度为$O(N^T)$。

### 前向算法

前向算法本质上属于动态规划的算法，也就是我们要通过找到局部状态递推的公式，这样一步步的从子问题的最优解拓展到整个问题的最优解。

在前向算法中，通过定义"前向概率“来定义动态规划的这个局部转改。什么是前向概率呢，其实定义很简单：定义时刻t时刻的隐藏状态为$q_c$,观测状态的序列为$o_1,o_2,...o_t$ 的概率为前向概率，记为：
$$
\alpha_t(c) = P(o_1,o_2,...o_t, i_t =q_c | \lambda)
$$
既然是动态规划，我们就要递推了，现在我们假设我们已经找到了在时刻𝑡t时各个隐藏状态的前向概率，现在我们需要递推出时刻t+1时各个隐藏状态的前向概率。其中：
$$
a_{t+1}(d) = P(o_1,o_2,...o_t,o_{t+1}, i_{t+1}=q_d | \lambda)
$$
现在开始找$a_{t+1}$和 $a_t$的关系。
$$
\begin{align} a_{t+1}(d)&= \sum\limits_{c=1}^N P(o_1,o_2,...o_t,o_{t+1}, i_t =q_c,i_{t+1}=q_d | \lambda) \\
& = \sum\limits_{c=1}^N P(o_{t+1}|o_1,o_2,...o_t, i_t =q_c,i_{t+1}=q_d )P(o_1,o_2,...o_t,, i_t =q_c,i_{t+1}=q_d | \lambda) \end{align}
$$
根据HMM的假设：
$$
P(o_{t+1}|o_1,o_2,...o_t, i_t =q_c,i_{t+1}=q_d ) = b_d(o_{t+1})
$$

$$
\begin{align}
P(o_1,o_2,...o_t,, i_t =q_c,i_{t+1}=q_d | \lambda)  
= P(i_{t+1}=q_d|o_1,o_2,...o_t,i_t =q_c | \lambda) * a_{t+1}(d) = a_{cd} a_{t+1}(d)
\end{align}
$$

因此$a_{t+1}$ 和 $a_t$ 的递推关系可以表示为：
$$
\alpha_{t+1}(d) = \Big[\sum\limits_{c=1}^N\alpha_t(c)a_{cd}\Big]b_d(o_{t+1})
$$
我们的动态规划从时刻1开始，到时刻𝑇结束，由于$a_T(i)$表示在时刻𝑇T观测序列为$o_1,o_2,...o_T$，并且时刻𝑇T隐藏状态$q_i$的概率，我们只要将所有隐藏状态对应的概率相加，即$\sum\limits_{i=1}^N\alpha_T(i)$就得到了在时刻𝑇T观测序列为$o_1,o_2,...o_T$的概率。

### 后向算法

后向算法和前向算法非常类似，都是用的动态规划，唯一的区别是选择的局部状态不同，后向算法用的是“后向概率”，定义时刻t时隐藏状态为$ q_c $, 从时刻t+1到最后时刻𝑇的观测状态的序列为$o_{t+1},o_{t+2},...o_T$的概率为后向概率。记为：
$$
\beta_t(c) = P(o_{t+1},o_{t+2},...o_T| i_t =q_c , \lambda)
$$
后向概率的动态规划递推公式和前向概率是相反的。现在我们假设我们已经找到了在时刻t+1时各个隐藏状态的后向概率$\beta_{t+1}(j)$
$$
\beta_{t+1}(d) = P(o_{t+2},o_{t+3},...o_T| i_{t+1} =q_d , \lambda)
$$
现在开始找$\beta_t$ 和$\beta_{t+1}$ 的关系:
$$
\begin{align}
\beta_t(c) &= \sum_{d=1}^NP(o_{t+1},o_{t+2},...o_T,i_{t+1}=q_d| i_t =q_c,\lambda)\\
&=\sum_{d=1}^NP(o_{t+1},o_{t+2},...o_T| i_{t+1}=q_d,i_t =q_c,\lambda)P(i_{t+1}=q_d|i_t =q_c,\lambda) \\
&=\sum_{d=1}^NP(o_{t+1},o_{t+2},...o_T| i_{t+1}=q_d,\lambda)P(i_{t+1}=q_d|i_t =q_c,\lambda)) \\
&=\sum_{d=1}^NP(o_{t+1}|o_{t+2},...o_T|i_{t+1}=q_d,\lambda)P(o_{t+2},o_{t+3},...o_T| i_{t+1} =q_d , \lambda)a_{cd} \\
& =\sum_{d=1}^N b_j(O_{t+1})a_{ij}\beta_{t+1}(j)

\end{align}
$$
找到递推关系式后，我们看$P(O|\lambda)$ 与 $\beta$ 的关系
$$
\begin{align}
P(O|\lambda) &= P(o_1,...,o_T|\lambda) \\
& =\sum_{i=1}^N P(o_1,...,o_T,i_1 = q_c|\lambda)\\
& = \sum_{i=1}^N P(o_1,...,o_T|i_1 = q_c,\lambda)P(i_1=q_c|\lambda) \\
& = \sum_{i=1}^NP(O_1|O_2,...O_T|i_1 = q_c,\lambda)P(O_2,...O_T|i_1 = q_c,\lambda)P(i_1=q_c|\lambda) \\
& =  \sum_{i=1}^N b_c(O_1) \beta_1(c)\pi_c
\end{align}
$$

