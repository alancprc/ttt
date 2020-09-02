---
layout:     post
title:      measure capacitor on ATE
categories: ate
---

* 目录
{:toc}

## 说明
ATE(Automatic Test Equipment)，主要用于各种芯片的量产测试。

笔者最近在工作中碰到几次需要对测试电容的情况，用到了几种不同类型的板卡，通过这篇文章进行一个总结。

电容既可以在充电时测量，也可以在放电时测量，因为客户的测试方案中指定了在放电时测量，所以如下测试都先充电到预设电压，在放电时测量电容值。

## 使用VI源+Digitizer
### 原理
```
C = Q / U = ΔQ / ΔU = I * Δt / ΔU
```
通过施加一个固定电流I，并测量在一段时间内的电容电压变化，即可通过以上公式计算出电容值。

### 步骤
1. 使用电压源模式，先Force 0.5V，等待一段时间，确保电容充满。需要注意电源的电流钳制设置，过小的话充电耗时太长。
2. 设置VI源对电容电压进行采样，以采样周期为samplePeriod，采样点数为numSamples
3. 切换VI源为电流源模式，施加一个反向电流（从电容流回到VI源）Idischarge，开始放电。注意需要设置好电压钳位，因为放电到0V之后，VI源会开始对电容进行反向充电，如果不设置电压钳位，反向充电电压过大，有损坏电容的风险。
4. 等待采样完成，或超出一定时间后，切换VI源为电压源模式，Force 0V，恢复电容状态。
5. 从采样数据中选取一段线性区间，计算两个点的电压差ΔU，以及对应的Δt，则电容值为 C = Idischarge * Δt / ΔU

### 特点
测试大电容的情况下，如果VI源的电流能力通常比较大，测试时间会比较短。

## 使用电压源+固定电阻+Digitizer
### 电路图
使用到的板卡结构大概如下：
```
                            电压测试点
电压源--------- 电阻 ---------------------------------- 电容 -- GND
          1KOhm / 500KOhm       |
                                |
                             Digitizer
```
板卡spec大概如下：
* 每个通道的电压可选 `[-10V, 10V]` 或者 `[0V, 20V]`
* 每个通道的电阻，可以在`1KOhm` 和 `500 KOhm`中间切换
* 最大的电流能力只有`2mA = 20V / 1KOhm`，所以并不是典型的电压源
* 每个通道都有一个digitizer，最小采样周期8us
这个板卡原本是为测试Flat Panel Display driver设计的，但是用来测量不是很大的电容效果很好。

### 原理
固定电压时，电容的充放电公式：
```
Vt = V0 + (Ve - V0) * (1-e^(-t/RC))
```
其中，
* `Ve` 是施加的固定电压
* `V0` 是施加`Ve`之前电容两端的电压
* `t` 是从`Ve`施加后经过的时间
* `R` 是电路中的电阻
* `C` 是电容的容值
* `e` 为自然底数

如果`Ve` 为0V，则上述公式可以简化成
```
Vt = V0 * e^(-t/RC)
```
电容C的值为
```
C = t / (  R  * ln(V0 / Vt) )
```
如果我们能找到一个点，使得 Vt = V0 / e，上述公式可以进一步简化为
```
C = t / R
```
### 步骤
1. 根据电容大小，选择1KOhm或者500KOhm
2. Force 0.5V，等待一段时间，确保电容充满。
2. 设置Digitizer对电容电压进行采样，以采样周期为samplePeriod，采样点数为numSamples
3. Force 0.5V，开始放电
4. 等待采样完成
5. 选取合适的电压区间，比如从 [0.4V, 0.4V * 1/e]，从采样数据中搜索第一个小于0.4V的点A，第一个小于 0.4V * 1/e = 0.147V的点B。点A和点B的间隔为
```
Δt = AB 之间的距离 * samplePeriod
```
则电容值为 C = Δt / R

## 使用Digital Pin(Resistor Load)
该款数字板卡带有电阻负载，在Drive off时，可以经由`RLoad`到`Vref`的负载，其中`RLoad`和`Vref`均可在一定范围内编程，如下所示
```
               --------- Drive
               |
DUT pin -------+-------- RLoad ---------- Vref
               |
               --------- Comparator
```
所以类似的，可以考虑设置`Vref = 0V`，按照上述电压源+电阻的方式进行测量。
这里的问题在于，上面用的电压源可以用Digitizer对输出进行持续的采样，可是数字通道可没有Digitizer，得换种方式。

我们再看看利用电压源+固定电阻+Digitizer的方案中，所需要的信息究竟有哪些：
1. `V0` 和 `Vt`，且`Vt = V0 / e`
2. `V0` 和 `Vt` 之间的时间差

怎么用数字通道测量两个电压值中间的时间差？

也许我们可以分两次测量，一次设置`VOH`为`V0`，执行pattern，pattern中比较`H`，看第一个fail所在的`cycle number1`，然后把`VOH`设置为`V0/e`，看第一个fail所在的`fail cycle2`，则电容值为`C = (cycle number2 - cycle number1) * pattern period / RLoad`。

可是，有没有更好的，一次测量的方法？

没错，我们知道数字通道带有两个`Comparator`，一个对应`VOH`(Voltage Output High)，一个是`VOL`(Voltage Output Low)。如果
1. 把`VOH`设置成`V0`，`VOH`设置成`V0/e`
2. 在pattern中比较`V`（电压高于`VOH`，或者低于`VOL`时，都pass）

那么fail cycle的数量，对应的就是电容放电时，从`V0`到`V0/e`的时间，即
```
C = number of fail cycle * pattern period / RLoad
```

![GX1-100MHz-4.2Ohm-waveform]({{ site.baseurl }}/pic/clkin-discharge-waveform-freq-100MHz-with-vref-0-rlv-4.2kohm.png)
下图为测量数字通道自身电容值的波形，参数如下
* pattern period为10ns
* RLoad = 4.2 KOhm，Vref = 0V
* VOH = 2.7V, VOL = 0.994V

测得fail cycle数为23，电容值 C = 23 * 10ns / 4.2 KOhm = 55pF，接近数字通道60pF的spec

### 特点
* 和Digitizer相比，Comparator的`VOH` `VOL`的电压精度通常较差，如果设置得电压值较小，可能导致较大的相对误差
* 需要根据电容值，选择合适的`pattern period`和`RLoad`，使fail cycle数量不会太小。比如fail cycle数量大于20时，粗略估计测量误差可以控制在10%以内
* 由于pattern系统的timing比较固定，所以测量的稳定性很好
* 因为数字板卡的电流能力通常不高，测量大电容耗时较长。比如`C = 50uF`，`R = 680 Ohm`，则`RC = 34ms`
* 因为pattern 和 Pin Electronic都是Per Pin的，如果电容值相近时，可以并行测量，效率很高

## 使用Digital Pin(Current Load)
这种测量方式，可以用于带有current load的数字板卡，原理实际上是 *使用VI源+Digitizer* 和 *使用Digital Pin(Resistor Load)* 这两种方式的结合，在此仅贴一个公式
```
C =  IOL * number of fail cycle * period / (VOH - VOL)
```

## 后记
上述用电源板卡测量电容的两种方案是美国同事的方案，用在了实际的项目中，稳定性很好。比较大的问题，反而是被测的电容，放在socket一晚上之后，电容值就变了，而在tray盘中放了一晚上的就没啥事。一开始客户说是空气湿度，因为烘干之后结果会变回来，好在实际测试时芯片不会在socket中待太久。

至于用数字板卡测量电容，源于另一个同事，当时他在尝试用`RLoad`放电，用`TMU`测量`VOH`到`VOL`的下降时间。但反复实验都说明，该板卡使用`RLoad`时`TMU`无法测量。我受此启发，想到在pattern data中写的`V`，利用fail cycle的数量进行电容放电时间的测量，于是就有了这篇文章。
