# DTC 器件 CZ 门校准方案总结

> 新手请先读 [CZ 校准入门手册](CZ校准入门手册.md)。本文保留论文研读细节；所有器件数值都是论文案例，不是本项目的实测配置。核对记录见 [教程勘误](教程勘误.md)。

> **原始文献**：Rui Li, Kentaro Kubo, Yinghao Ho, Zhiguang Yan, Yasunobu Nakamura, Hayato Goto,
> *Realization of High-Fidelity CZ Gate Based on a Double-Transmon Coupler*,
> Phys. Rev. X **14**, 041050 (2024). DOI: [10.1103/PhysRevX.14.041050](https://doi.org/10.1103/PhysRevX.14.041050)
> **原文 PDF**：`resources/PhysRevX.14.041050.pdf`
> **配套图解网页**：`docs/cz-calibration.html`（同目录，浏览器直接打开）
> **本文性质**：对原文"校准/标定方案"的提炼与工程化整理，供 `CZCali` 项目复现与参考。
> 公式编号沿用原文（正文 Eq.(1)–(15)，附录 Eq.(H1)、(I1)–(I5)、(D15)–(D16)、(K1)–(K26)、(L1)–(L17)），便于回查。

---

## 目录

- [0. 阅读指引与术语速查](#0-阅读指引与术语速查)
- [1. 一句话结论](#1-一句话结论)
- [2. 背景：DTC 方案与被校准对象](#2-背景dtc-方案与被校准对象)
- [3. 校准流程总览](#3-校准流程总览)
- [4. 分步校准细节](#4-分步校准细节)
  - [4.0 测量链路要点](#40-测量链路要点硬件前提) · [4.1 器件参数与能谱](#41-器件参数与能谱表征前置设计) ·
    [4.2 idle 点](#42-step-1idle-bias-point-定标zz-最小点) · [4.3 单比特门](#43-step-2单比特门精标drag--rb) ·
    [4.4 Z 脉冲失真](#44-step-3z-脉冲失真标定与预畸变) · [4.5 CZ 相位](#45-step-4cz-相位θcz--π-优化) ·
    [4.6 VZ 相位](#46-step-5vzvirtual-z-相位优化) · [4.7 QPT](#47-step-6qpt--ptm-复核) ·
    [4.8 评精度](#48-step-7srb--irb--lrb-正式评精度) · [4.9 误差预算](#49-step-8误差预算)
- [5. 关键公式速查](#5-关键公式速查)
- [6. 实验结果与关键参数汇总](#6-实验结果与关键参数汇总)
- [7. 对本项目（CZCali）的落地清单](#7-对本项目czcali的落地清单)
- [8. 结论与展望](#8-结论与展望原文观点)
- [附录 A · 原文附录结构导航（A–L）](#附录-a--原文附录结构导航al)
- [附录 B · 术语中英对照](#附录-b--术语中英对照)

---

## 0. 阅读指引与术语速查

**三种读法**

| 目标 | 建议路径 |
|---|---|
| 只想知道"怎么把 CZ 校出来" | §3 流程 → §4.4/4.5/4.6 三个核心步骤 → §7 清单 |
| 想复现数据处理与误差分析 | §4.8/4.9 → §5 公式速查 → 附录 A 的 K/L |
| 想做器件设计（同方案新芯片） | §2.2 → §4.1 → 附录 A 的 C/D |

**一句话术语速查**（完整对照见[附录 B](#附录-b--术语中英对照)）

| 缩写 | 含义 |
|---|---|
| **DTC** | double-transmon coupler，双 transmon 耦合器（本文的耦合方案） |
| **STC** | single-transmon coupler，单 transmon 可调耦合器（对照方案） |
| **JAZZ** | joint amplification of ZZ interaction，ZZ 相互作用联合放大测量法 |
| **LRB / SRB / IRB** | 泄漏 RB / 标准 RB / 交错 RB |
| **CPHASE** | 受控相位门，`diag(e^{iθ0},e^{iθ1},e^{iθ2},e^{iθ3})` |
| **VZ** | virtual-Z，零时长虚拟 Z 门（只改帧相位） |
| **PTM / QPT** | Pauli 转移矩阵 / 量子过程层析 |
| **PPO** | proximal policy optimization，近端策略优化（本文用的 RL 算法） |
| **DRAG** | derivative removal by adiabatic gate，单比特脉冲整形方法 |
| **SPAM** | state preparation and measurement，态制备与测量误差 |

---

## 1. 一句话结论

用 **双 transmon 耦合器（Double-Transmon Coupler, DTC）** 做耦合开关：超导线圈把磁通偏置固定在 **idle bias point**（ZZ 残余相互作用 ≈ 0）做单比特实验；通过快速磁通线向 DTC 环路打 **预畸变的 Slepian Z 脉冲 + RL(PPO) 优化出的脉冲形状**，在该脉冲期间制造 −82.5 MHz 量级的 ZZ 相互作用实现 CPHASE，再用两个 **virtual-Z (VZ)** 门补偿单比特附加相位，即得 CZ 门。

**最终指标**：CZ 门 48 ns，保真度 **99.90 ± 0.01%**（12 小时内稳定）；单比特门 **≥ 99.98%**；ZZ on/off 比 **1.3 × 10⁴**。

---

## 2. 背景：DTC 方案与被校准对象

### 2.1 器件构成
- 4 个 transmon：数据比特 **Q1**（4.314 GHz）、**Q2**（4.778 GHz），耦合器 transmon **C3**、**C4**。
- 频率失谐 Δ₂₁/2π = 464 MHz > 二者的非谐性（−212 / −199 MHz），因此**处于 straddling regime 之外**（这是 DTC 相对单 transmon 耦合器 STC 的核心优势：放宽数据比特失谐限制，但仍需检查耦合器及高能级的避免交叉）。
- C3–Q1、C4–Q2 电容直接耦合；JJ3、JJ4 与额外结点 JJ5 构成超导环，外磁通 φ<sub>ex</sub> 调控耦合。
- 读出：Q1、Q2 各一个 λ/4 谐振器，C3/C4 共用一个谐振器（只能区分 |00⟩ 与 |xx⟩）。**注意：耦合器谐振器对 CZ 校准并非必需**，仅用于泄漏误差表征。

### 2.2 物理图像（玩具模型）
DTC 可等效为两个模式与两个数据比特作用：
- **p 模式**（定频 transmon P，5.495 GHz）：提供"固定"耦合项；
- **m 模式**（CSFQ，5.373 GHz）：频率随 φ<sub>ex</sub> 可调，提供"可变"耦合项，且与 Q2 的耦合为**负**（等效负电容）。

色散区有效耦合（Eq. 4）：

$$g_{\text{eff}} = \frac{g_{1p}g_{2p}}{2}\left(\frac{1}{\Delta_{1p}}+\frac{1}{\Delta_{2p}}\right) - \frac{g_{1m}g_{2m}}{2}\left(\frac{1}{\Delta_{1m}}+\frac{1}{\Delta_{2m}}\right)$$

扫 φ<sub>ex</sub> 使 m 模式频率扫过 p 模式频率，g<sub>eff</sub> 由负变正并过零 → 得到最小 ZZ；继续加大磁通利用强耦合区的能级排斥得到最大 ZZ。

**idle 点的解析估计**（附录 D）：环路电势最小条件给出

$$2\varphi_m + \varphi_{ex} = \frac{\pi}{2} + k\pi,\qquad \sin\varphi_m = \pm\alpha \;(\alpha \equiv E_{J5}/E_{J3})$$

代入 α = 0.216 得 φ<sub>ex</sub>/2π ≈ **0.319**，与实测 0.309 接近（偏差来自工艺误差与寄生电容）。

### 2.3 三个必须精确标定的对象
1. **idle bias point 的磁通**（超导线圈 dc 电流）；
2. **Z 脉冲的传输失真**（AWG → 快速磁通线 → 低温 → 环路 的瞬态）；
3. **CZ 相位（θ<sub>CZ</sub> = π）与两个 VZ 相位**（二者强耦合，必须分开优化）。

> DTC 方案带来的便利：数据比特在强耦合区被耦合器能级排斥，**比特自身频率对磁通敏感**，因此可以用"比特 + 读出谐振器"当传感器标定 Z 脉冲失真，**不需要耦合器读出谐振器**。

---

## 3. 校准流程总览

```
[硬件层]  先做好：DRAG 单比特脉冲、readout 谐振器频率/幅度、k-means 单发分类

Step 0  线路与电子学准备
        ├─ AWG 预补偿（real-time precompensation）先大致补偿
        └─ 超导线圈 dc 源粗扫磁通到 idle 区
                ↓
Step 1  idle bias point 定标（JAZZ）
        └─ 得到 φex/2π = 0.309（模型估计 0.319），残余 ZZ = 2π×(−6.3) kHz
                ↓
Step 2  单比特门精标（DRAG + 单独/同时 RB）  ← CZ 优化前必须完成，否则相位引入错误
                ↓
Step 3  Z 脉冲失真标定（Ramsey + 相位层析，短期/长期）→ 预畸变参数 {a_k, τ_k}
                ↓
Step 4  CZ 相位优化
        4a. JAZZ-N 扫 Slepian 脉冲幅度 → θCZ 初值 ≈ π
        4b. JAZZ2-N 作 reward → RL(PPO) 优化脉冲"形状"（20 点 → 2 GSa/s 三次样条插值）
                ↓
Step 5  VZ 相位优化
        5a. Ramsey 型序列单独测 Q1/Q2 累积相位（初值 139.33° / −102.05°）
        5b. 两比特 Clifford RB 的保真度作 reward → RL(PPO) 微调 VZ（Z 脉冲固定不动）
                ↓
Step 6  QPT/PTM 复核（验证"确实是 CZ"，受 SPAM 低估，仅作定性/矩阵确认）
                ↓
Step 7  SRB / IRB / LRB 正式评精度：r_CZ = 0.090%, L_CZ = 0.027%, F̄ = 99.90%
                ↓
Step 8  误差预算 + 门长扫描（找泄漏/相干误差平衡点，本文选 48 ns）
```

**依赖顺序要点**
- Step 2 一定在 Step 4/5 之前：单比特门的相位误差会污染 VZ 相位标定。
- Step 3（失真标定）在 Step 4 之前，否则 RL 会花大量容量去补偿未知的线路瞬态。
- Step 4b 与 Step 5 必须**解耦**：VZ 相位对 Z 脉冲形状"高度敏感"（数据比特被耦合器能级排斥，Z 脉冲期间频率动态变化），联合优化会非常难收敛。

---

## 4. 分步校准细节

### 4.0 测量链路要点（硬件前提）

| 设备 | 用途 |
|---|---|
| Zurich SHFQC | 单机直接产生 ≤8.5 GHz 信号，省混频；多频同时解调 3 个谐振器做复用读出 |
| Zurich HDAWG8 | 产生 DTC 的 Z 脉冲，**带实时预补偿（precompensation）** |
| Yokogawa GS200 | 给超导线圈供恒定 dc，维持 idle bias point |
| SRS FS725 铷钟 | 全系统同步 |
| BlueFors LD400（8 mK）+ IMPA + HEMT | 读出链路（IMPA 为自制阻抗匹配参放） |

快速磁通/耦合器 XY 线路上串了 diplexer（QMC 0218LNM，XY+Z 合路）、低通 VLF-2250+；独立的直流线圈链路使用 4 K 处自制 <1.6 kHz 低通与室温 <1 MHz π 滤波器抑制高频噪声；样品用铜盒 + 两层 μ 金属屏蔽。
> **启示**：dc 磁通链路的低通滤波必须足够强，否则 idle point 抖动会直接抬升残余 ZZ 与相干误差。

### 4.1 器件参数与能谱表征（前置设计）
- 忽略谐振器/控制线，四 transmon 系统用 Hamiltonian
  $\hat H = 4e^2 \hat n^{TC-1}\hat n - \sum_i E_{Ji}\cos\hat\phi_i - E_{J5}\cos(\hat\phi_4-\hat\phi_3-\phi_{ex})$（Eq. 1）数值对角化。
- **电路参数人工搜索流程**（附录 C，按序执行，3–5 可反复迭代）：

| 步 | 做什么 | 取值/经验 |
|---|---|---|
| 1 | 定数据比特频率与非谐性 | 4–5.5 GHz，非谐性 ≈ −200 MHz；失谐取 500 MHz（脱离 straddling） |
| 2 | 处理杂散电容 | C14、C23、C12 < 1 fF 可忽略；C34 视为定值（Sonnet/COMSOL 有限元给出） |
| 3 | 定耦合电容 C<sub>g</sub> = C13 = C24 | 分电容的 5–10%（对应 100–200 MHz 耦合），本文 5.7 fF；**越大 ζ 越大，但留给其他参数的容差越小** |
| 4 | 固定 α = E<sub>J5</sub>/E<sub>J3</sub> 扫 {C<sub>c</sub>, E<sub>Jc</sub>} | α 经验上应 < 0.25，本文 0.216；C<sub>c</sub>↑、E<sub>Jc</sub>↓ → 耦合器模式频率↓ → ζ<sub>min</sub> 与 ζ<sub>max</sub> 都↑，需权衡 |
| 5 | 固定 C<sub>c</sub> 扫 {E<sub>Jc</sub>, α} | α↑ → 两耦合器耦合更强 → 间接增大比特间 ZZ，同样在 ζ<sub>min</sub>/ζ<sub>max</sub> 间权衡 |

- 扫描结果与实验能谱对照微调参数（原文 Fig. 1(d)），得到的模型是后续所有控制仿真的基准。
- 实际工艺值与设计值略有偏差（见 §6 电路参数表）。

### 4.2 Step 1：idle bias point 定标（ZZ 最小点）

**方法：JAZZ（联合放大 ZZ 相互作用）+ 线性扫描最终 π/2 的相位**
- 序列须按原文 Fig. 2(a) 实现带双比特 π 翻转的 JAZZ 回波结构，不能用“Q2 始终为 |1⟩ 的普通 Ramsey”替代；后者还包含单比特动态相位。扫描参数 t 与 Z 平顶持续时间 t/2 的关系也应按原图实现。
- ZZ 导致叠加态累积相对相位 φ，P<sub>0</sub> = (1 − cos φ)/2。脉冲为"高斯沿 + 平顶（时长 t/2）"，故
  $$\varphi = \frac{\zeta t}{2} + \varphi_0$$
- **关键技巧**：idle 点附近 |ζ| 很小，振荡太慢、拟合困难。于是让最终 π/2 的相位随时间**线性变化** Φ<sub>π/2</sub>(t)，人为引入基线频率 ω<sub>b</sub>，把振荡频率抬高：
  $$\zeta = 2(\omega_m - \omega_b),\qquad \omega_b > |\zeta/2|$$
- 结果：φ<sub>ex</sub>/2π = 0.309（解析估计 0.319），ζ<sub>min</sub> = 2π×(−6.3) kHz。
- 该点同步做单比特门校准（见 4.3）。
- 大 ZZ 的测量不需要基线频率，直接把最终门固定为 Y<sub>π/2</sub>：
  $$\zeta = 2\omega_m \;\Rightarrow\; \zeta_{\max} = 2\pi\times(-82.5)\ \text{MHz at } \varphi_{ex}/2\pi = 0.47$$

### 4.3 Step 2：单比特门精标（DRAG + RB）
- 在 idle bias point 用 **DRAG** 校正 X/Y rotation，门长 48 ns。
- 用**单独 RB**（另一个比特置 |0⟩）与**同时 RB** 两种方式验证残余 ZZ 不损伤单比特性能：

| 测量方式 | Q1 | Q2 |
|---|---|---|
| 单独 RB | 99.985% | 99.981% |
| 同时 RB | 99.985% | 99.983% |

- 两者几乎相同 ⇒ 2π×(−6.3) kHz 的残余 ZZ 不是当前限制因素。
- 每个 Clifford 的物理门数按分解为 **1.875** 折算单门误差：r = (1−p)(1−1/d)，d = 2，r<sub>SQ</sub> = r/1.875。
- **单门非相干误差预算**（附录 G）：
  $$r_{\text{SQ}}^{\text{incoherent}} = \frac{t_{\text{gate}}}{3}\left(\frac{1}{T_1}+\frac{1}{T_\varphi}\right),\qquad \frac{1}{T_\varphi}=\frac{1}{T_2^E}-\frac{1}{2T_1}$$
  Q1：T₁ 与 Tφ 贡献 7.0×10⁻⁵ 与 6×10⁻⁶ ⇒ 非相干仅占总误差约 50%，**残余来自控制脉冲不稳的相干误差**；
  Q2：7.8×10⁻⁵ 与 8.3×10⁻⁵ ⇒ 非相干占 >85%（其中 Tφ 项可能因磁通噪声被高估）。
- 相干性（idle 点）：T₁ = 228.6 / 205.3 μs，T₂ᴿ = 132.4 / 62.4 μs，T₂ᴱ = 358.9 / 129.8 μs（Q1/Q2）。

### 4.4 Step 3：Z 脉冲失真标定与预畸变

这是本方案**最具借鉴价值**的一步：不需要耦合器读出谐振器，用 Q2 当传感器。

**为什么能用 Q2 当传感器**：在 0.375 < φ<sub>ex</sub>/2π < 0.45 区间，Q2 频率被耦合器能级明显排斥，因而对磁通灵敏（代价：该点 T₂ᴿ≈3 μs）。

**测量序列（Ramsey 型 + 单比特相位层析）**
1. **短期失真**（原文 Fig. 12a）：把磁通用线圈偏置到 φ<sub>ex</sub>/2π ≈ 0.42，X<sub>π/2</sub> 建立叠加态 → 加一个 48 ns 方波 Z → 等待 t → 做 phase tomography。为避免 Z 脉冲把系统推过 anticrossing，这里**取较小幅度**（累积相位虽小但够用）。
2. **长期失真**（原文 Fig. 12b）：先加 10 μs 方波（此时所有比特都在基态，可以取大幅值提高 SNR），等待 t 后做固定 ~500 ns 的 Ramsey + 层析。用于捕捉超出比特退相干时间的瞬态。

**模型与拟合**（多指数步响应，Eq. H1）：

$$\varphi(t) = \varphi_0 \sum_k a_k\left(e^{-t/\tau_k} - e^{-(t+\tau_{\text{pulse}})/\tau_k}\right)$$

拟合得到 {a<sub>k</sub>, τ<sub>k</sub>}，回代 AWG / HDAWG 的预补偿实现**预畸变（predistortion）**。

**本文拟合结果（Table V）**

| 短期标定 | 幅度 a<sub>k</sub> | 长期标定 | 幅度 a<sub>k</sub> |
|---|---|---|---|
| τ = 603.3 ns | −0.0104 | τ = 400.5 μs | +0.12 |
| τ = 79.45 ns | −0.0137 | τ = 71.02 μs | +0.038 |
| τ = 13.60 ns | +0.00525 | — | — |

> 注意：短期拟合幅度有正有负，长期两项为正；仅凭符号不能确定瞬态来源。分两种序列是为了覆盖不同时间尺度，最终统一验证补偿效果。

### 4.5 Step 4：CZ 相位（θ<sub>CZ</sub> = π）优化

#### 4.5.1 4a 物理预标定：JAZZ-N 扫幅度
- 用 **(π − Z)<sup>N</sup>** 重复模式放大对脉冲**幅度**的灵敏度；N = 4k+1（实测 N = 9, 41, 101，k = 2, 10, 25）。
- 序列保真度（Q2 的 P₁，Eq. I1）：
  $$P_{|1\rangle} = \frac{1-\cos\big((2k+1)\theta_{CZ}\big)}{2}$$
  **优点**：单比特旋转自动抵消，保真度与 VZ 门无关 → 直接对准 θ<sub>CZ</sub> = π。
- N 越大，幅度标定越细。

#### 4.5.2 初始脉冲形状：Slepian
- 由两态还原 Hamiltonian 的 θ(t) = arctan(H<sub>x</sub>/H<sub>z</sub>)（Eq. I2–I3），令 dθ/dt = S(t)（Slepian 函数，Eq. I4），θ(t) = ∫S(t)dt，再用 H<sub>z</sub>/H<sub>x</sub> ≈ C z²(t) 映射到 AWG 电压 z(t)。
- 实例：总长 48 ns，首尾各 2 ns 零 padding，归一化到峰值 1。
- Slepian 只是"初始猜测"，Gaussian / cosine 均可——论文在该实验中认为初始形状主要影响优化效率；在有限训练预算、噪声和带宽约束下，不能保证任意初值都达到相同性能。

#### 4.5.3 4b RL(PPO) 优化形状：JAZZ2-N 作 reward
- **JAZZ2-N**（N = 2k，Eq. I5）：两端各加 π/2，测 P<sub>|00⟩</sub>：
  $$P_{|00\rangle} = \frac{1-\cos\big((2k+1)\theta_{CZ}\big)}{2} \;\xrightarrow{\theta_{CZ}\to\pi}\; 1$$
  对形状更敏感、且对单比特门误差更鲁棒。
- **PPO agent**（代码改编自 Ref. [85]）：每 epoch 由策略生成 **20 个波形**，用 JAZZ2-N 的保真度作 reward；约 100 epoch 收敛；单机无 GPU 训练约 2 h（主要耗时在波形上传与策略更新）。
- 波形用 **20 个控制点 + 2 GSa/s + 三次样条插值**生成（参数空间小，收敛快）。
- 对"优化后 − Slepian"的差做 FFT，发现 agent 在注入特定频率分量——推测是为了补偿残余短瞬态失真，或抑制 anticrossing 附近高能级跃迁。
- 需要更精细时可逐步增大 N，并在上一次优化结果上继续训。

### 4.6 Step 5：VZ（virtual-Z）相位优化

为什么必须加 VZ：每个数据比特虽只有一个约瑟夫森结（形似定频），但在 CZ 期间被耦合器能级排斥，**频率动态变化**，会各自积累额外 Z 旋转 θ₁、θ₂，需用 VZ 补偿。

#### 5a 物理预标定（逐比特）
- Ramsey 型序列（测 Q1 时 Q2 置 |0⟩）：π/2 → Z 脉冲（重复 N 次，每次给 θ<sub>i</sub>/2）→ tomography，线性拟合累积相位 N·θ<sub>i</sub>/2。
- 本文结果：**139.33°/次（Q1，图中 θ₁/2 的斜率）**、**−102.05°/次（Q2，图中 θ₂/2 的斜率）**。
- 注意：比特频率标不准会引入附加相位污染 VZ 标定，需用 echo 型序列（π/2 − Z − π − I − Tomo）或先精标比特频率。

#### 5b RL(PPO) 微调
- reward = 两比特 Clifford RB 的序列保真度；**Z 脉冲固定不动**，只把 5a 得到的 VZ 相位作初值，每 epoch 生成 20 组 (VZ<sub>Q1</sub>, VZ<sub>Q2</sub>)，用 5 条随机 Clifford（m = 20）测均值。
- 约 100 epoch 收敛；精度最终受读出保真度限制。**结论：用 RB 优化 VZ 是压低相位误差、提升 CZ 保真度的关键一步。**

### 4.7 Step 6：QPT / PTM 复核
- 36 个初态（{X±, Y±, Z±}^⊗2，由 {Y<sub>π/2</sub>, −Y<sub>π/2</sub>, −X<sub>π/2</sub>, X<sub>π/2</sub>, I, X<sub>π</sub>}^⊗2 制备）→ 施加 CZ → 36 个投影测量（共 1296 个结果）→ 最大似然重建 16×16 Pauli 转移矩阵。
- 保真度（Eq. J1）$F = \frac{\mathrm{Tr}(R_{ideal}^{T}R)+d}{d(d+1)}$，d = 4 → **0.959（被 SPAM 低估，仅作定性确认）**。
- 用途：直观证明"这个门就是 CZ"、重建时施加了 trace-preserving 约束；这不是无泄漏的实验证据，泄漏需另行测量。
- PTM 结构要点：CZ 共轭把 XI↔XZ、YI↔YZ、IX↔ZX、IY↔ZY、XX↔YY（正号）、XY↔−YX；对角仅 II/IZ/ZI/ZZ 为 +1。

### 4.8 Step 7：SRB / IRB / LRB 正式评精度

因为有泄漏，标准 RB 需改造为 **LRB**（附录 K）：

- 同时测两个量：
  - 总残留在计算子空间的概率 $P_{X1} = P_{|0000\rangle}+P_{|0100\rangle}+P_{|1000\rangle}+P_{|1100\rangle}$；
  - 理想输出态存活率 $P_{id} = P_{|0000\rangle}$。
- 拟合式（避免双指数、把 SPAM 误差与模型参数分离，Eq. K14–K15）：
  $$P_{X1}(m) = A + B\lambda_L^{m}, \qquad P_{id}(m) - P_{X1}(m)/d = C\lambda_r^{m} + D,\quad d=4$$
  其中 λ<sub>L</sub> = 1 − L₁ − L₂（泄漏+渗出），λ<sub>r</sub> = (1−L₁)(1−p<sub>D</sub>)（去极化）。
- 泄漏率与 CZ 门误差：
  $$L_{1}^{SRB(IRB)} = (1-\lambda_{L}^{SRB(IRB)})(1-A^{SRB(IRB)})$$
  $$L_{1}^{CZ} = 1 - \frac{1-L_{1}^{IRB}}{1-L_{1}^{SRB}},\qquad r_{CZ} \approx \frac{d-1}{d}\left(1-\frac{\lambda_r^{IRB}}{\lambda_r^{SRB}}\right)$$
- 总保真度（含泄漏）：
  $$\bar F = 1 - \frac{L_1^{CZ}}{d} - r_{CZ} \;\simeq\; 1 - L_1^{CZ} - \frac{d-1}{d}p_D^{CZ}$$
  后一形式明确拆成"泄漏 + 去极化"两项；并给出去极化型误差 $r_D^{CZ} = r_{CZ} - \frac{d-1}{d}L_1^{CZ}$。

**本文结果**：r<sub>CZ</sub> = 0.090 ± 0.009%，L₁<sup>CZ</sup> = 0.027 ± 0.006% ⇒ **F̄ = 99.90 ± 0.01%**。
**稳定性**：12 h 内 6 次测量 r<sub>CZ</sub> 均 < 0.1%（平均 0.09 ± 0.01%），期间 CZ 参数只优化一次，单比特门每次 RB 前重新标定。

### 4.9 Step 8：误差预算
- **控制误差**：非绝热跃迁可能造成泄漏；计算子空间内还存在 CZ/VZ 相位误差（相位误差 < 1.5° ⇒ < 0.01%，当前阶段可忽略）。
- **泄漏误差**：LRB 直接给出，0.010–0.035%（40 ns 门长时约 0.1%，视为 RL/带宽失效的异常点；> 100 ns 略升，因为优化敏感度下降）。
  - 40 ns 点的诊断：用**理论最优脉冲**做数值仿真可达 >99.99% ⇒ 差距来自**器件带宽限制**和/或 **RL 优化未收敛**，而非方案本身。
- **非相干误差**：建模为（Eq. 15）
  $$r_{D}^{CZ} = r_{\text{incoherent}} + r_0 = \frac{2}{5}\frac{t_{\text{gate}}}{T_{\text{eff}}} + r_0$$
  线性拟合 r₀ = −0.0007 ± 0.008%（Pearson ≈ 0.995, p < 10⁻⁹），模型成立。
- $T_{\text{eff}}^{\text{exp}} = 23.9 \pm 1.5$ μs，明显低于由 idle 点 T₁/Tφ 估算的 $T_{\text{eff}}^{\text{est}} = 67.6 \pm 11.4$ μs ⇒ 存在"CZ 期间的额外退相干"（可能是多能级混合 + Z 脉冲激活的额外噪声通道，含 CZ dephasing）。
- **1/f 磁通噪声可忽略**：由 Hahn-echo 退相干率 vs 磁通灵敏度拟合出 √A<sub>Φ</sub> ≈ 4.89 / 4.79 μΦ₀（Q1/Q2，平均 4.84 μΦ₀），Monte Carlo（1000 次采样偏磁通）给出 40–320 ns 门长内误差 < 10⁻⁶。
- 非相干误差分解（附录 L，Kraus 算子 + 平均保真度公式）：
  $$1-F \simeq \frac{2}{5}\frac{t}{T_1^{(1)}}+\frac{2}{5}\frac{t}{T_1^{(2)}}+\frac{2}{5}\frac{t}{T_\varphi^{(1)}}+\frac{2}{5}\frac{t}{T_\varphi^{(2)}}+\frac{3}{10}\frac{t}{T_{CZ}} \equiv \frac{2}{5}\frac{t}{T_{\text{eff}}}$$
  （振幅阻尼与纯退相各贡献 (2/5)t/T，CZ 退相贡献 (3/10)t/T<sub>CZ</sub>）

---

## 5. 关键公式速查

> 按用途分组；括号内为原文公式编号。

### A. ZZ 与能谱
| 公式 | 说明 |
|---|---|
| $\zeta = E_{|1100\rangle}-E_{|1000\rangle}-E_{|0100\rangle}+E_{|0000\rangle}$ (3) | ZZ 相互作用定义，四 transmon 本征能级 |
| $g_{\text{eff}} = \frac{g_{1p}g_{2p}}{2}(\frac{1}{\Delta_{1p}}+\frac{1}{\Delta_{2p}}) - \frac{g_{1m}g_{2m}}{2}(\frac{1}{\Delta_{1m}}+\frac{1}{\Delta_{2m}})$ (4) | 玩具模型有效耦合；p 固定、m 可调且与 Q2 反号 |
| $\cos(2\varphi_m+\varphi_{ex})=0,\;\sin\varphi_m=\pm\alpha$ (D7) | idle 点解析条件；α=0.216 → φ/2π ≈ 0.319 |

### B. ZZ 测量（JAZZ）
| 公式 | 说明 |
|---|---|
| $P_{|0\rangle}=(1-\cos\varphi)/2,\;\varphi=\zeta t/2+\varphi_0$ | 平顶 t/2 + 固定高斯沿 |
| $\zeta = 2(\omega_m-\omega_b),\ \omega_b>|\zeta/2|$ | 引入基线频率提速小 ζ 拟合 |
| $\zeta = 2\omega_m$ | 大 ζ 情形（固定最终门为 Y<sub>π/2</sub>） |

### C. Z 脉冲失真
| 公式 | 说明 |
|---|---|
| $\varphi(t)=\varphi_0\sum_k a_k[e^{-t/\tau_k}-e^{-(t+\tau_{pulse})/\tau_k}]$ (H1) | 多指数步响应，拟合 {a<sub>k</sub>,τ<sub>k</sub>} 做预畸变 |
| $\theta=\arctan(H_x/H_z),\ d\theta/dt=S(t),\ H_z/H_x\approx Cz^2(t)$ (I2–I4) | Slepian 初值脉冲生成 |

### D. CZ 相位标定
| 公式 | 说明 |
|---|---|
| $U_{\rm CPHASE}=\mathrm{diag}(e^{i\theta_0},e^{i\theta_1},e^{i\theta_2},e^{i\theta_3})$，$\theta_{CZ}=\theta_3-\theta_1-\theta_2+\theta_0=\pi$ | CZ 条件；θ₁、θ₂ 由 VZ 补偿 |
| $P_{|1\rangle}=\frac{1-\cos[(2k+1)\theta_{CZ}]}{2}$ (I1) | JAZZ-N，N=4k+1，标幅度，与 VZ 无关 |
| $P_{|00\rangle}=\frac{1-\cos[(2k+1)\theta_{CZ}]}{2}\to 1$ (I5) | JAZZ2-N，N=2k，标形状，RL 的 reward |

### E. 精度评估（LRB / SRB / IRB）
| 公式 | 说明 |
|---|---|
| $P_{X1}=A+B\lambda_L^m$ (K14) | 泄漏 RB，λ<sub>L</sub>=1−L₁−L₂ |
| $P_{id}-P_{X1}/d=C\lambda_r^m+D$ (K15) | 去极化项，d=4，单指数 |
| $L_1=(1-\lambda_L)(1-A)$ (K20) | 单次序列泄漏率 |
| $r=(1-\lambda_r)(1-1/d)$ | 单次序列误差率 |
| $L_1^{CZ}=1-\frac{1-L_1^{IRB}}{1-L_1^{SRB}}$ (K22) | CZ 泄漏误差 |
| $r_{CZ}\approx\frac{d-1}{d}\big(1-\frac{\lambda_r^{IRB}}{\lambda_r^{SRB}}\big)$ (11) | CZ 门误差 |
| $\bar F=1-\frac{L_1^{CZ}}{d}-r_{CZ}$ (12)；$\bar F\simeq 1-L_1^{CZ}-\frac{d-1}{d}p_D^{CZ}$ (13) | 总平均保真度两种写法 |
| $r_D^{CZ}=r_{CZ}-\frac{d-1}{d}L_1^{CZ}$ (14) | 去极化型误差（用于门长拟合） |
| $F=\frac{\mathrm{Tr}(R_{ideal}^TR)+d}{d(d+1)}$ (J1) | QPT/PTM 保真度（受 SPAM 低估） |

### F. 误差预算
| 公式 | 说明 |
|---|---|
| $r_D^{CZ}=\frac{2}{5}\frac{t_{gate}}{T_{eff}}+r_0$ (15) | 门长线性模型 |
| $1-F\simeq\frac{2}{5}\big(\frac{t}{T_1^{(1)}}+\frac{t}{T_1^{(2)}}+\frac{t}{T_\varphi^{(1)}}+\frac{t}{T_\varphi^{(2)}}\big)+\frac{3}{10}\frac{t}{T_{CZ}}$ (L17) | 振幅阻尼+纯退相+CZ 退相 |
| $r_{SQ}^{inc}=\frac{t}{3}\big(\frac{1}{T_1}+\frac{1}{T_\varphi}\big)$ (G3) | 单比特非相干误差；$\frac{1}{T_\varphi}=\frac{1}{T_2^E}-\frac{1}{2T_1}$ |
| $\Gamma_\Phi^E=\frac{2\pi}{\Phi_0}\sqrt{A_\Phi\ln 2}\,\lvert\partial\omega/\partial\varphi_{ex}\rvert$ (L3) | 1/f 磁通噪声退相干率 vs 磁通灵敏度 |
| $r=(1-p)(1-1/d),\ r_{SQ}=r/1.875$ (G2) | RB → 单门误差折算 |

---

## 6. 实验结果与关键参数汇总

### 6.1 性能指标
| 量 | 数值 | 出处 |
|---|---|---|
| CZ 平均保真度 F̄ | **99.90 ± 0.01 %** | Eq. 12 |
| r<sub>CZ</sub>（Clifford IRB） | 0.090 ± 0.009 % | Eq. 11 |
| 泄漏 L₁<sup>CZ</sup> | 0.027 ± 0.006 % | Eq. K22 |
| 12 h 稳定性 | r<sub>CZ</sub> 6 次全 < 0.1%（均值 0.09 ± 0.01%），L₁ 0.030 ± 0.003% | Fig. 4(d) |
| QPT 保真度（参考） | 0.959（SPAM 主导低估） | Eq. J1 |
| 单比特门（单独/同时） | Q1 99.985 / 99.985 %；Q2 99.981 / 99.983 % | 附录 G |
| ZZ on/off 比 | 1.3 × 10⁴（−6.3 kHz ↔ −82.5 MHz） | Fig. 2(d), 3(a) |
| CZ 门长 | 48 ns（+2 个 VZ） | — |

### 6.2 器件与电路参数（Table I/II）
| 项目 | 数值 | 项目 | 数值 |
|---|---|---|---|
| Q1 / Q2 频率 | 4.314 / 4.778 GHz | α = E<sub>J5</sub>/E<sub>J3</sub> | 0.216 |
| 非谐性 Q1 / Q2 | −212 / −199 MHz | C<sub>13</sub> / C<sub>24</sub> | 5.73 / 5.77 fF |
| p / m 模式频率 | 5.495 / 5.373 GHz | C<sub>34</sub> / C<sub>12</sub> | 1.73 / 0.04 fF |
| 读出谐振器 | 8.184 / 8.261 / 8.101 GHz | C<sub>11</sub> / C<sub>22</sub> / C<sub>33</sub> / C<sub>44</sub> | 91.86 / 91.79 / 110.27 / 106.36 fF |
| idle bias point | φ<sub>ex</sub>/2π = 0.309（解析 0.319） | I<sub>c1..5</sub> | 26.13 / 31.93 / 47.73 / 47.68 / 10.32 nA |
| 最大 ZZ | 2π×(−82.5) MHz @ 0.47 | 强耦合区 | 0.325 < φ<sub>ex</sub>/2π < 0.5 |

### 6.3 相干与噪声
| 量 | Q1 | Q2 |
|---|---|---|
| T₁ | 228.6 ± 30.4 μs | 205.3 ± 26.0 μs |
| T₂ᴿ（Ramsey） | 132.4 ± 59.9 μs | 62.4 ± 6.1 μs |
| T₂ᴱ（Hahn echo） | 358.9 ± 47 μs | 129.8 ± 13.3 μs |

- 1/f 磁通噪声幅度 √A<sub>Φ</sub>：4.89 / 4.79 μΦ₀（平均 4.84 μΦ₀）→ 对 CZ 误差贡献 < 10⁻⁶。
- T<sub>eff</sub><sup>exp</sup> = 23.9 ± 1.5 μs vs T<sub>eff</sub><sup>est</sup> = 67.6 ± 11.4 μs（差距 = CZ 期间额外退相干，未完全解释）。

### 6.4 读出与状态判别（Table IV）
| 制备 \ 读出 | Q1：\|0⟩ | \|1⟩ | \|2⟩ | Q2：\|0⟩ | \|1⟩ | \|2⟩ |
|---|---|---|---|---|---|---|
| \|0⟩ | 0.9933 | 0.0006 | 0.0061 | 0.9973 | 0.0017 | 0.0010 |
| \|1⟩ | 0.0121 | 0.9787 | 0.0092 | 0.0231 | 0.9704 | 0.0065 |
| \|2⟩ | 0.0060 | 0.0257 | 0.9683 | 0.0196 | 0.0398 | 0.9406 |

耦合器（共用谐振器）：P(\|00⟩\|\|00⟩) = 0.9871，P(\|xx⟩\|\|xx⟩) = 0.9378 —— 只需区分"基态 vs 任一激发态"，够用于泄漏表征。
分类方法：单发 IQ 散点 + **k-means 无监督聚类**。

---

## 7. 对本项目（CZCali）的落地清单

### 7.1 建议校准顺序（数值目标需自定）（按依赖排序）

| # | 步骤 | 关键动作 | 产出 | 验收 |
|---|---|---|---|---|
| 0 | 链路准备 | AWG 预补偿、dc 低通、铷钟同步 | — | 基线噪声 |
| 1 | 读出与分类 | 谐振器频率/功率、单发 IQ、k-means | assignment matrix | 按实验误差预算设目标；需记录分类矩阵 |
| 2 | XY 门粗标 | DRAG（幅度/β/频率） | X/Y 脉冲 | idle 确定后重新精标并验证 RB |
| 3 | idle 点及 XY 精标 | dc 粗扫 + JAZZ 找 ZZ 零点 | φ<sub>ex</sub>/2π、ζ<sub>min</sub> | 残余 ZZ 量级 kHz |
| 4 | Z 失真 | 短期/长期 Ramsey+层析 → {a<sub>k</sub>,τ<sub>k</sub>} | 预畸变参数 | 校正前后相位残差 |
| 5 | CZ 相位 | JAZZ-N 扫幅度 → Slepian → JAZZ2-N+PPO | Z 脉冲形状 | θ<sub>CZ</sub> = π |
| 6 | VZ 相位 | Ramsey 型初值 → RB+PPO | 两个 VZ 相位 | 相位误差 < 1.5° |
| 7 | 验证 | QPT → LRB → 门长扫描 | F̄、r<sub>CZ</sub>、L₁ | 稳定性 ≥ 12 h |

### 7.2 易踩坑（论文明确提到）
1. 不先标 Z 失真就上 RL：agent 会用容量补偿未知瞬态，效率低甚至不收敛。
2. 在 DTC 中联合优化 Z 形状与 VZ 相位"极难"（能级排斥致 VZ 对形状极敏感）——必须分两步。
3. 门长短于约 40–48 ns：泄漏误差可能因 RL/带宽限制而反弹（本文 40 ns 约 0.1%）。
4. 门长长于 100 ns：泄漏占比变小后 RL 优化敏感度下降，泄漏又缓慢上升。
5. 单比特门要反复重标，否则温度漂移污染 CZ/VZ 相位评估。
6. 读出误差和测量噪声会限制此实验的 VZ/RL 优化灵敏度；它不是门保真度的普适上限。
7. QPT 保真度受 SPAM 严重低估（95.9%），最终以 RB 为准；RB 需改造为 LRB 才能计入泄漏。

### 7.3 硬件取舍
- **可省**：耦合器读出谐振器（失真标定用比特传感器即可），但保留它做泄漏表征更方便。
- **建议保留**：AWG 侧实时预补偿、dc 低通（<1.6 kHz + π）、μ 金属双层屏蔽、IMP 放大器（提高单发分类质量）。

---

## 8. 结论与展望（原文观点）

- 选择 **48 ns** 门长是在"泄漏误差"与"非相干误差"之间的平衡点。
- 提升方向：更快门长（需更大耦合电容，但会恶化相干性）+ 更有效的短脉冲 RL 优化；理解和抑制"CZ 期间额外退相干"（T<sub>eff</sub> 缺口）。
- 玩具模型指出：JJ3/JJ4 可换成线性电感仍能实现近零 ZZ，但 **JJ5 的非线性必须保留**（提供可调耦合强度与大 ZZ）。
- DTC 的系统级优势：失谐大、频率碰撞概率低、空间排布灵活、只有一个磁通调控自由度、spectator 误差小。

---

## 附录 A · 原文附录结构导航（A–L）

| 附录 | 内容 | 解决什么问题 | 对应本文 |
|---|---|---|---|
| **A** Fabrication | Ta 薄膜 135 nm、CF₄ 刻蚀、Dolan 桥 Al 结、TSV（Bosch） | 高相干工艺 | §6.3 |
| **B** Measurement Setup | SHFQC/HDAWG8/GS200/FS725、IMPA、HEMT、diplexer、滤波与屏蔽 | 链路与噪声控制 | §4.0 |
| **C** Device Parameters | Table I/II + **5 步电路参数搜索流程** | 器件设计 | §4.1 |
| **D** Toy Model of the DTC | p/m 模式推导、耦合系数 g<sub>ip</sub>/g<sub>im</sub>、idle 点解析 | 物理理解 | §2.2 |
| **E** Qubit Coherence | T₁/T₂ᴿ/T₂ᴱ 随磁通变化 + 17.5 h 稳定性 | 相干基线 | §6.3 |
| **F** State Discrimination | 单发 IQ + k-means + assignment matrix | 读出分类 | §6.4 |
| **G** Single-Qubit Gate Fidelities | RB（单独/同时）+ 非相干误差分解 | 单门校准 | §4.3 |
| **H** Z-Pulse Distortion | 短期/长期序列 + {a<sub>k</sub>,τ<sub>k</sub>} | 预畸变 | §4.4 |
| **I** CZ-Gate Optimization by RL | JAZZ-N / JAZZ2-N + Slepian + PPO + VZ 优化 | **核心校准** | §4.5, §4.6 |
| **J** Quantum Process Tomography | 36×36 + 最大似然 PTM | 定性确认 | §4.7 |
| **K** CZ-Gate Fidelity | LRB 模型推导（K1–K26） | 精度评估 | §4.8 |
| **L** Incoherent Error | 1/f 磁通噪声 + Kraus 通道 + T<sub>eff</sub> | 误差预算 | §4.9 |

---

## 附录 B · 术语中英对照

| 英文 | 中文 | 备注 |
|---|---|---|
| double-transmon coupler (DTC) | 双 transmon 耦合器 | 本文方案 |
| single-transmon coupler (STC) | 单 transmon 可调耦合器 | 对照方案 |
| straddling regime | 跨越区间（失谐 < 非谐性） | 外于此区可避免频率碰撞 |
| idle bias point | 空闲偏置点 | ZZ 最小点 |
| residual ZZ interaction | 残余 ZZ 相互作用 | 关断后的剩余耦合 |
| on-off ratio | 开关比 | ζ<sub>max</sub>/ζ<sub>min</sub> |
| flux transient | 磁通瞬态 | Z 脉冲断电后的残留 |
| predistortion | 预畸变 | 逆补偿线路失真 |
| Slepian pulse | 斯列皮安脉冲 | 平滑绝热初始脉冲 |
| reinforcement learning (RL) | 强化学习 | 本文用 PPO |
| model-free | 无模型 | 不依赖系统辨识 |
| virtual-Z (VZ) gate | 虚拟 Z 门 | 零时长，只改帧相位 |
| Clifford interleaved RB | Clifford 交错随机基准 | IRB |
| leakage / seepage error | 泄漏 / 渗出误差 | 计算子空间 ↔ 泄漏子空间 |
| depolarizing error | 去极化误差 | 计算子空间内 |
| incoherent / coherent error | 非相干 / 相干误差 | — |
| effective coherence time | 有效相干时间 | T<sub>eff</sub> |
| Pauli transfer matrix (PTM) | Pauli 转移矩阵 | 过程层析的实数表示 |
| spectator error | 旁观者误差 | 未参与比特受扰动 |
| through-silicon via (TSV) | 硅通孔 | 背面接地 |
| Purcell filter | 珀塞尔滤波器 | 抑制弛豫通道 |

---

*注：本文所有公式编号沿用原文，便于对照检索。若需复现，请以原文附录 **H（Z 脉冲失真）**、**I（CZ 优化）**、**K（保真度）**、**L（非相干误差）** 为核心；配套图解见 `docs/cz-calibration.html`。*
