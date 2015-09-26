---
layout: katex
title: 近似计算天空颜色
---

## {{ page.title }}

**求解观察者在任意位置和任意观察方向上所获得的光：**

我只讨论一次散射的情形，即阳光在大气层发生且仅发生一次散射的情形，因为一次散射的光是组成天空颜色的绝大部分光源。

分析太空中任意一束阳光经过一次散射并在观察角度上到达观察者：

大气层内：

![](/images/2015/atmosphere_scattering_inside.png)

大气层外：

![](/images/2015/atmosphere_scattering_outside.png)

对上图中的某一束阳光过程概括如下：

一束阳光进入大气层 -> 阳光衰减 $$s'$$ 距离 -> 阳光到达散射点 $$p$$，并在与观察者呈 $$\theta$$ 角度的方向上发生散射 -> 阳光继续衰减 $$s$$ 距离，终于到达观察者 $$P_{v}$$。
那么该束阳光到达观察者时的量 $$I_{p}(\lambda)$$ 可以用下面的函数概括：

$$
I_{p}(\lambda) = I_{sun}(\lambda)\cdot e^{-t(s',\lambda)}\cdot F(\lambda,p,\theta)\cdot e^{-t(s,\lambda)}
$$

其中：

* $$I_{sun}(\lambda)$$：太空中阳光在指定波长 $$\lambda$$ 下的强度

* $$F(\lambda,p,\theta)$$：光在 $$\lambda$$ 波长、$$p$$ 点、与入射阳光 $$\theta$$ 角度上的散射比例。该散射是由瑞利（Rayleigh）散射和 Mie 散射组成，且有：

  $$
  F(\lambda,p,\theta) = F_{R}(\lambda,p,\theta) + F_{M}(\lambda,p,\theta)
  $$

  其中瑞利散射部分：

    $$
    F_{R}(\lambda,p,\theta) = \frac{K\rho_{R}(p)F_{R}(\theta)}{\lambda^4} = \frac{\pi^2(n^2-1)^2(1+\cos^2(\theta))}{2N_{S}\lambda^4}e^{-\frac{h}{H_{R}}}
    $$

    * $$K = \frac{2\pi^2(n^2-1)^2}{3N_{S}}$$

      表示海平面的空气原子密度常数，其中：

      * $$n$$：标准大气折射率

      * $$N_{S}$$：标准大气原子密度

    * $$\rho_{R}(p) = e^{-\frac{h}{H_{R}}}$$

      $$h$$ 高度上空气原子密度较海平面的比率，体现散射程度与空气原子密度的关系，其中：

      * $$H_{R}$$：7994 米

    * $$F_{R}(\theta) = \frac{3}{4}(1+\cos^2(\theta))$$

      散射相位函数，体现散射程度与散射角度的关系

  Mie 散射部分：

  * //TODO

* $$e^{-t(s',\lambda)}$$ 和 $$e^{-t(s,\lambda)}$$：阳光经过 $$s'$$ 和 $$s$$ 的轨迹的衰减比例，且有：

  $$
  t(s,\lambda) = \beta_{R}(\lambda)\int_0^s\rho_{R}(h)\text{d}h+\beta_{M}(\lambda)\int_0^s\rho_{M}(h)\text{d}h
  $$

  * $$\beta_{R}(\lambda) = \int_{0}^{4\pi}\frac{F_{R}(\lambda,p,\theta)}{\rho_{R}(p)}\text{d}\theta = \int_{0}^{4\pi}\frac{KF_{R}(\theta)}{\lambda^4}\text{d}\theta = \frac{8\pi^3(n^2-1)^2}{3N_{S}\lambda^4}$$

  * $$\beta_{M}(\lambda)$$：//TODO

散射点 $$p$$ 在观察者的观察方向的射线中处于大气层内的线段上，即 $$P_{v}$$ 与 $$P_{a}$$ 之间，所以要计算观察者所获得的光 $$I(\lambda)$$，需要计算 $$I_{p}(\lambda)$$ 在从 $$P_{v}$$ 到 $$P_{a}$$ 之间的积分：

$$
I(\lambda) = \int_{P_{v}}^{P_{a}}I_{p}(\lambda)\text{d}p = \int_{P_{v}}^{P_{a}}I_{sun}(\lambda)\cdot F(\lambda,p,\theta)\cdot e^{(-t(s,\lambda)-t(s',\lambda))}\text{d}p
$$


//先缓一缓 = =
