# 多元 — Green公式与路径无关精讲

[[04-多元函数微积分/多元微积分总览|← 返回总览]]

---

## Green公式

$$\oint_L P\,dx+Q\,dy=\iint_D\left(\frac{\partial Q}{\partial x}-\frac{\partial P}{\partial y}\right)dA$$

**使用条件**：
- $L$ 为分段光滑的**封闭**曲线，并按正向（逆时针）取向；
- $D$ 是 $L$ 围成的平面区域；
- $P,Q$ 在包含 $D$ 的开区域内具有连续一阶偏导数；
- 若区域内有奇点，必须挖去小圆或改用等价曲线，不能直接套公式。

---

## 类型一：直接用Green公式

### 例1 ⭐⭐

计算 $\oint_L (x^2+y^2)dx+(x+y)^2dy$，$L$ 为正向单位圆 $x^2+y^2=1$

**解**：$P=x^2+y^2$，$Q=(x+y)^2$

$$\frac{\partial Q}{\partial x}-\frac{\partial P}{\partial y}=2(x+y)-2y=2x$$

$$=\iint_{x^2+y^2\leq 1}2x\,dA=0\quad(\text{奇函数对称区域积分为0})$$

**答案**：$\boxed{0}$

---

### 例2 ⭐⭐⭐（路径无关）

计算

$$I=\int_L (ye^x+2x)\,dx+e^x\,dy,$$

其中 $L$ 是从 $A(0,1)$ 到 $B(\pi,0)$ 的任意光滑曲线。

**解**：令

$$P(x,y)=ye^x+2x,\qquad Q(x,y)=e^x.$$

有

$$P_y=e^x,\qquad Q_x=e^x,$$

且定义域 $\mathbb R^2$ 单连通，因此积分与路径无关。

寻找势函数 $u$：

$$u_x=P=ye^x+2x.$$

对 $x$ 积分：

$$u(x,y)=ye^x+x^2+\varphi(y).$$

再由 $u_y=Q=e^x$ 得 $\varphi'(y)=0$，故可取

$$u(x,y)=ye^x+x^2.$$

因此

$$I=u(\pi,0)-u(0,1)=\pi^2-1.$$

**答案**：

$$\boxed{\pi^2-1}$$

> [!important] 判断路径无关不仅要验证 $P_y=Q_x$，还要确认定义域单连通且无奇点。

## 类型二：路径无关的充要条件

### 例3 ⭐⭐⭐（竞赛常考）

设 $P=\dfrac{-y}{x^2+y^2}$，$Q=\dfrac{x}{x^2+y^2}$，计算 $\oint_L P\,dx+Q\,dy$

**分析**：$Q_x=\dfrac{(x^2+y^2)-x\cdot 2x}{(x^2+y^2)^2}=\dfrac{y^2-x^2}{(x^2+y^2)^2}$

$P_y=\dfrac{-(x^2+y^2)-(-y)\cdot 2y}{(x^2+y^2)^2}=\dfrac{y^2-x^2}{(x^2+y^2)^2}$

$Q_x=P_y$ ✓，但在原点 $(0,0)$ 处无定义！

**情况1**：若 $L$ 不包围原点 → $\oint=0$（Green公式可用）

**情况2**：若 $L$ 包围原点一次（逆时针）：

取单位圆 $x=\cos\theta$，$y=\sin\theta$，$\theta:0\to 2\pi$：

$$\oint=\int_0^{2\pi}\frac{-\sin\theta}{\cos^2\theta+\sin^2\theta}(-\sin\theta)\,d\theta+\frac{\cos\theta}{\cos^2\theta+\sin^2\theta}\cos\theta\,d\theta=\int_0^{2\pi}(\sin^2\theta+\cos^2\theta)\,d\theta=2\pi$$

**结论**：$\oint_L P\,dx+Q\,dy=\begin{cases}0 & \text{$L$不含原点}\\2\pi k & \text{$L$绕原点$k$次（逆时针正）}\end{cases}$

---

## 类型三：Green公式计算面积

$$S=\frac{1}{2}\left|\oint_L x\,dy-y\,dx\right|$$

### 例4 ⭐⭐

用Green公式求椭圆 $\dfrac{x^2}{a^2}+\dfrac{y^2}{b^2}=1$ 的面积

**解**：$L: x=a\cos t$，$y=b\sin t$，$t:0\to 2\pi$（逆时针）

$$S=\frac{1}{2}\oint_L x\,dy-y\,dx=\frac{1}{2}\int_0^{2\pi}(a\cos t\cdot b\cos t+b\sin t\cdot a\sin t)\,dt=\frac{ab}{2}\cdot 2\pi=\pi ab$$

---

## ⚡ 关键总结

| 情况 | 结论 |
|------|------|
| $L$ 封闭，$D$ 内 $Q_x=P_y$ | $\oint=0$ |
| $L$ 封闭，$D$ 内奇点 | 换等价曲线计算 |
| 路径不封闭，$Q_x=P_y$ | 找原函数 $u$，$I=u(终)-u(始)$ |
| 路径不封闭，$Q_x\neq P_y$ | 参数化直接计算，或补线用Green |


---

## 🧭 第二类曲线积分决策树

```text
∫L P dx + Q dy
├─ L 封闭
│  ├─ 区域内无奇点 → Green公式
│  └─ 有奇点 → 挖洞 / 换同伦曲线 / 参数化
└─ L 不封闭
   ├─ P_y = Q_x 且区域单连通 → 找势函数
   ├─ 方便补线 → 补线后用Green
   └─ 否则 → 参数化直接积分
```

## ✅ 势函数检查

求得 $u(x,y)$ 后必须同时验证

$$u_x=P,\qquad u_y=Q.$$

只验证其中一个偏导数，不能说明原函数正确。
