---
title: my TA learning path
date: 2024-12-16 14:27:37
tags:
---
I am gonna to become a TA after several months!! Happy__y!

# Chapter 1: Wind
## 1. Navier-Stokes Wind Simulation
### 1.1 Intro


$$\frac{\partial v}{\partial t} = \mathbf{F} + \frac{\mu}{\rho} \nabla^2 V_i - (V_i \cdot \nabla)V_i - \frac{1}{\rho}P$$

- $ \frac{\partial \mathbf{v}}{\partial t}$ 是速度随时间的变化。
- $ (\mathbf{v} \cdot \nabla) \mathbf{v} $ 是速度对流项。
- $ -\frac{1}{\rho} \nabla P $ 是压力梯度项。
- $ \nu \nabla^2 \mathbf{v} $ 是粘性扩散项。

#### 1.1.1 $\mathbf{F}$ 
外力。加在速度场中间的外力。

#### 1.1.2 $\frac{\mu}{\rho} \nabla^2 V_i$

扩散项。扩散项主要是描述场是如何在空间内平滑/扩散的。$\mu$ 是动力粘度，$\rho$ 是流体密度。$V_i$ 是拉普拉斯算子，描述的是空间二维导数。高粘性物体(蜂蜜)，流动趋近于平滑。欧拉视角(网格)下扩散想可以理解为，当前网格于周边6格(3D)交换。如果某网格速度高，周围网格速度低，扩散项会让它们的速度逐渐接近通过‘交换’。

推导

1. 原公式：  
   $x_0 = x - a \cdot (\text{sum}_c - \text{num} \cdot x) \cdot \Delta t$

2. 展开括号：  
   $x_0 = x - a \cdot \text{sum}_c \cdot \Delta t + a \cdot \text{num} \cdot x \cdot \Delta t$

3. 整理项：  
   $(1 + a \cdot \text{num} \cdot \Delta t) \cdot x = x_0 + a \cdot \text{sum}_c \cdot \Delta t$

4. 解出 $x$：  
   $x = \frac{x_0 + a \cdot \text{sum}_c \cdot \Delta t}{1 + a \cdot \text{num} \cdot \Delta t}$



* x0 先前状
* x当前状态
* a扩散速率
* $\mathbf{num}_p$和$\mathbf{num}_c$分别表示相邻单元格前一时刻和后一时刻值的总和。


### 1.2.化简
#### Step 1: (origin formula) 
$$
\frac{\partial \mathbf{v}}{\partial t} + (\mathbf{v} \cdot \nabla) \mathbf{v} = -\frac{1}{\rho} \nabla P + \nu \nabla^2 \mathbf{v} + \mathbf{F}
$$



#### Step 2: (differentiation velocity)

$$
\frac{\partial v_i}{\partial t} + (\mathbf{v} \cdot \nabla) v_i = \mathbf{F}_i - \frac{1}{\rho} \frac{\partial P}{\partial x_i} + \nu \nabla^2 v_i
$$

#### Step 3: (Expanding the Convection Term)
$$
(\mathbf{v} \cdot \nabla)\mathbf{v} =
\begin{bmatrix}
v_1 \frac{\partial v_1}{\partial x} + v_2 \frac{\partial v_1}{\partial y} + v_3 \frac{\partial v_1}{\partial z} \\
v_1 \frac{\partial v_2}{\partial x} + v_2 \frac{\partial v_2}{\partial y} + v_3 \frac{\partial v_2}{\partial z} \\
v_1 \frac{\partial v_3}{\partial x} + v_2 \frac{\partial v_3}{\partial y} + v_3 \frac{\partial v_3}{\partial z}
\end{bmatrix}
$$

$$
(\mathbf{v} \cdot \nabla)v_i = v_1 \frac{\partial v_i}{\partial x_1} + v_2 \frac{\partial v_i}{\partial x_2} + v_3 \frac{\partial v_i}{\partial x_3}
$$

Put it in the origin formula we got: 
$$
\frac{\partial v_i}{\partial t} + v_j \frac{\partial v_i}{\partial x_j} = \textbf{F}_i- \frac{1}{\rho} \frac{\partial P}{\partial x_i} + \frac{u}{\rho} \nabla^2 v_i
$$


#### Step 4: 

表示二阶空间导数，是衡量某个点周围速度变化的一个标量值,速度分量$v_i$表示速度在$v_1,v_2,v_3$方向上的累加。
$$\nabla^2 v_i = \frac{\partial^2 v_i}{\partial x_1^2} + \frac{\partial^2 v_i}{\partial x_2^2} + \frac{\partial^2 v_i}{\partial x_3^2}$$

粘粘项:

$$
\frac{u}{\rho} \nabla^2 v_i
$$

equal to 

$$
\frac{u}{\rho} \left( \frac{\partial^2 v_i}{\partial x_1^2} + \frac{\partial^2 v_i}{\partial x_2^2} + \frac{\partial^2 v_i}{\partial x_3^2} \right)
$$

put it in the origin formula we got: 
$$
\rho \frac{\partial v_i}{\partial t} + \rho v_j \frac{\partial v_i}{\partial x_j} = \rho f_i - \frac{\partial P}{\partial x_i} + u \left( \frac{\partial^2 v_i}{\partial x_1^2} + \frac{\partial^2 v_i}{\partial x_2^2} + \frac{\partial^2 v_i}{\partial x_3^2} \right)
$$x

#### Step 5: (final formula)

$$\rho \frac{\partial v_i}{\partial t} + \rho v_j \frac{\partial v_i}{\partial x_j} = \rho f_i - \frac{\partial P}{\partial x_i} + u \frac{\partial^2 v_i}{\partial x_j^2}$$


1. **左边：流体的惯性项**
   - $\rho \frac{\partial v_i}{\partial t}$：速度分量 $v_i$ 的时间变化部分，描述流体的加速度。
   - $\rho v_j \frac{\partial v_i}{\partial x_j}$：对流部分，描述流体由于自身速度引起的变化。

2. **右边：流体的外力、压力梯度和黏性力**
   - $\rho f_i$：外力，例如重力。
   - $-\frac{\partial P}{\partial x_i}$：压力梯度的影响。
   - $u \frac{\partial^2 v_i}{\partial x_j^2}$：黏性力，用二阶导数表示。

这就是 **Navier-Stokes 方程的分量形式**，描述了流体在受力平衡下的运动状态，涵盖了时间变化、空间对流、外力、压力和黏性效应。
# Appendix 

## $\nabla$(gradient operator)

$\nabla$ is a vector differential operator

$$
\nabla = \left(\frac{\partial}{\partial x}, \, \frac{\partial}{\partial y}, \, \frac{\partial}{\partial z}\right).
$$

$$
\nabla \times \mathbf{v} \;=\;
\begin{vmatrix}
\mathbf{i} & \mathbf{j} & \mathbf{k} \\
\frac{\partial}{\partial x} & \frac{\partial}{\partial y} & \frac{\partial}{\partial z} \\
v_x & v_y & v_z
\end{vmatrix}.
$$
i,j,k: unit vector