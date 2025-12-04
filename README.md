# 🚀 Project: Zero-Shot Embodied Object Navigation via VLFM
### 基于视觉语言大模型(VLM)的零样本具身导航与 Sim2Real 迁移

![Python](https://img.shields.io/badge/Python-3.8%2B-blue)
![PyTorch](https://img.shields.io/badge/PyTorch-2.0-ee4c2c)
![Habitat](https://img.shields.io/badge/Simulator-Habitat-green)
![Status](https://img.shields.io/badge/Status-Completed-success)

> **摘要 (Abstract)：** > 本项目复现并改进了 **VLFM (Vision-Language Frontier Maps)** 算法，构建了一个能够理解自然语言指令（如 *"Find a bed"*) 的移动机器人导航系统。不仅在仿真环境（Habitat）中实现了基于 **GroundingDINO** 和 **BLIP-2** 的开放词汇目标搜索，更针对 **Sim2Real（仿真到真机）** 的迁移鸿沟，设计了鲁棒的底层控制策略，解决了传感器盲区与动力学死区问题。

---

## 📺 1. 系统可视化演示 (System Visualization)

<div align="center">
  <img src="success_bed.gif" width="800px" alt="Demo Navigation">
  <p><em>图 1: 寻找卧室中的床 (Find a Bed) - 完整导航过程演示</em></p>
</div>

### 👁️ 画面布局解析 (Layout)
如上图所示，系统的可视化界面包含两个核心视角，直观展示了机器人的“所见”与“所想”：

* **左侧 (Perception - 感知层):** * **RGB 视角:** 机器人第一人称画面，叠加了 GroundingDINO 的实时目标检测框（Bounding Box）。
    * **Depth 视角:** 实时深度图，用于避障和 3D 投影。
* **右侧 (Reasoning - 决策层):** * **实时构建的 2D 地图:** 白色为已探索区域，灰色为未知区域。
    * **Value Map (热力图):** 叠加在地图上的彩色阴影。颜色越暖（红/黄），代表 VLM 推理认为该区域存在目标的概率越高。
    * **蓝色/红圈:** 算法生成的 **Frontier Waypoint** (前沿探索点)。

---

## 🏆 2. 成功案例展示 (Success Cases)

本项目在多种家庭场景下进行了测试，展示了算法在复杂环境中的语义推理能力。

| 场景 A：卫生间导航 (Find a Toilet) | 场景 B：客厅沙发导航 (Find a Couch) |
| :---: | :---: |
| <img src="success_toilet.gif" width="100%"> | <img src="success_couch.gif" width="100%"> |
| **任务描述：** 机器人从走廊出发寻找卫生间。<br>**亮点：** 即使起始位置没有任何线索，Value Map 成功指引机器人优先探索狭窄的门口区域，而非开阔的客厅。 | **任务描述：** 在充满障碍物的客厅中定位沙发。<br>**亮点：** 展示了开放词汇检测能力。机器人成功规划路径，绕过了茶几和椅子的阻挡，停在沙发前方。 |

---

## 🧐 3. 挑战与失败案例分析 (Corner Case Analysis)

> 💡 **工程师笔记：** 在具身智能研究中，分析 "Corner Cases" (长尾/极端情况) 往往比展示成功更有价值。以下展示了我们在开发过程中遇到的典型挑战。

### 案例分析：感知幻觉与过早停止 (Perception Hallucination)

<div align="center">
  <img src="fail_bed.gif" width="700px" alt="Failure Case">
</div>

* **视频说明：** 任务目标是寻找“床 (Bed)”。
* **现象描述：** 机器人虽然行驶方向正确（SoftSPL 指标提升），但在距离目标尚有距离时突然停止。
* **深度归因 (Root Cause Analysis)：**
    这是一个典型的 **"Local Distractor" (局部干扰项)** 问题。
    1.  **感知幻觉：** 机器人在远处看到了一个纹理与床极其相似的物体（例如米色的单人沙发或堆叠的织物），GroundingDINO 给出了高置信度的误检。
    2.  **过早决策：** 状态机逻辑较为激进，一旦检测置信度超过阈值，立即从“探索态”切换至“导航态”并触发停止。
    3.  **调试数据佐证：** 如下图 Log 所示，`stop_called=1.00` 且 `distance_to_goal=6.46m`，证明机器人“自以为”到了，实际上还很远。


* **改进方案 (Optimization)：**
    * 引入 **时序一致性校验 (Temporal Consistency)**：要求目标必须在连续 5 帧中被检测到，且空间位置方差小于阈值，才确认为真目标。
    * 引入 **VQA 二次确认**：在检测到目标后，调用 BLIP-2 进行 Visual Question Answering ("Is this really a bed?") 进行双重验证。

---

## 🛠 4. 核心技术模块 (Core Methodology)

本项目摒弃了不可解释的端到端黑盒方案，采用了 **模块化 (Modular)** 架构，确保了系统的可解释性与鲁棒性。

### A. 语义价值地图 (Semantic Value Mapping)
让机器人拥有“常识”。
* 利用 **BLIP-2** 计算当前 RGB 视野与目标文本（Prompt）的余弦相似度。
* 通过坐标变换，将 2D 相似度投影到 3D 空间并更新 2D 栅格地图。
* **效果：** 机器人不再盲目乱撞，而是像人一样“顺藤摸瓜”（例如：找灶台会优先探索看起来像厨房的区域）。

### B. 基于前沿的自主探索 (Frontier-based Exploration)
解决“下一步去哪”的问题。
* 实时提取 **已探索区域 (Free)** 与 **未知区域 (Unknown)** 的交界线（Frontiers）。
* 设计综合打分函数： $Score = \alpha \cdot \text{Value} - \beta \cdot \text{Distance}$，在“探索收益”与“移动代价”之间取得平衡。

### C. Sim2Real 工程落地 (Engineering for Real World)
为了让算法从 Habitat 仿真迁移到真实机器人（Kobuki 底盘 + Realsense），解决了以下关键 Gap：
1.  **传感器盲区补偿：** 针对深度相机 20cm 内的物理盲区（导致 Costmap 错误清除障碍物），设计了基于近场红外数据的 Costmap 保持机制。
2.  **动力学死区处理：** 针对真实电机在低速下的静摩擦力问题，设计了 **脉冲蠕行控制 (Creep Control)**，解决了起步时的“窜动”和微调时的不精准问题。

---

## 📊 5. 实验结论

在 HM3D 数据集的验证集上，通过引入上述优化策略，导航成功率 (Success Rate) 相比基线提升了 **约 7%**。实验证明，结合 VLM 的语义推理能力与传统机器人控制的鲁棒性，是解决复杂室内非结构化环境导航的有效路径。
