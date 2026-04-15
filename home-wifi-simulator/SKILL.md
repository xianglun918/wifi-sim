---
name: home-wifi-simulator
description: "自包含的家庭宽带 WiFi 环境仿真 Skill。封装三个可视化能力：1) 预制户型+AP数 → 可配置栅格信号热力图 PNG（默认 40×40）；2) 基于信号 → 可配置栅格点状 RTMP 卡顿率栅格图 PNG（默认 40×40）；3) 自动 AP 补点推荐并输出补点前后的双图对比 + 矩阵数据（.npy / .json）。matplotlib 已配置中文字体，无需引用外部项目文件。"
triggers:
  - "生成 WiFi 信号热力图"
  - "输出 RTMP 卡顿率栅格图"
  - "AP 补点优化对比"
  - "户型 WiFi 仿真"
  - "WiFi 覆盖分析"
---

# Home WiFi Simulator Skill

**自包含 Skill**。所有模型、引擎逻辑和户型预设均已内嵌到 `skill.py` 中，无需引用任何外部文件。

## 前置依赖

- **Python >= 3.11**
- **numpy**, **pyyaml**, **matplotlib**

安装方式（推荐在 skill 目录旁创建虚拟环境）：

```bash
# 示例：使用 uv
uv venv --python 3.11 .venv
source .venv/bin/activate
uv pip install numpy pyyaml matplotlib
```

（以下命令均假设虚拟环境已激活，直接使用 `python`）

## 使用方式

进入 skill 目录并执行：

```bash
cd skills/home-wifi-simulator

# 能力 1+2：生成信号热力图 + 卡顿率栅格图（默认大平层，栅格 40×40，显示门洞）
python skill.py --preset 大平层 --ap 1 --out-dir ./output --show-doors

# 使用 20×20 栅格（更快）
python skill.py --preset 大平层 --ap 1 --grid-size 20 --out-dir ./output --show-doors

# 能力 3：1AP → 3AP 补点优化对比（默认大平层，显示门洞）
python skill.py --preset 大平层 --ap 1 --target-ap 3 --compare --out-dir ./output --show-doors

# 大平层不显示门洞（恢复完整墙体）
python skill.py --preset 大平层 --ap 1 --out-dir ./output
```

Python API 调用示例：

```python
from skill import (
    generate_rssi_heatmap,
    generate_stall_grid,
    generate_ap_optimization_comparison,
)

# 能力 1（默认大平层，信号较差场景）
generate_rssi_heatmap("大平层", 1, "./output/rssi.png")

# 能力 2
generate_stall_grid("大平层", 1, "./output/stall.png")

# 能力 3：对比 1AP 与补点后 3AP 的效果
result = generate_ap_optimization_comparison(
    "大平层", current_ap_count=1, target_ap_count=3, output_dir="./output"
)
# result 包含对比图路径 + 8 个矩阵数据文件路径：
# {
#   "rssi_comparison": "./output/大平层_rssi_comparison.png",
#   "stall_comparison": "./output/大平层_stall_comparison.png",
#   "rssi_before_npy": "./output/大平层_rssi_before.npy",
#   "rssi_after_npy": "./output/大平层_rssi_after.npy",
#   "stall_before_npy": "./output/大平层_stall_before.npy",
#   "stall_after_npy": "./output/大平层_stall_after.npy",
#   "rssi_before_json": "./output/大平层_rssi_before.json",
#   "rssi_after_json": "./output/大平层_rssi_after.json",
#   "stall_before_json": "./output/大平层_stall_before.json",
#   "stall_after_json": "./output/大平层_stall_after.json",
# }
```

## 能力详解

### 1. 信号热力图 `generate_rssi_heatmap`

- **输入**：户型名（`一居室` / `两居室` / `三居室` / `大平层`）+ AP 数量
- **AP 布局策略**：
  - 1 个 AP → 户型几何中心
  - 2~4 个 AP → 按常见拓扑均匀分布（左右 / 三角 / 四角）
  - >4 个 AP → 网格均匀分布
- **信号调弱（Mock）**：
  - AP 发射功率从默认 20dBm 降至 **10dBm**
  - 增加 **10dB 环境衰减**，使全屋信号整体偏弱，更贴近边缘覆盖场景
- **信号模型**：
  - **FSPL 自由空间路径损耗** + **墙体穿透衰减**
  - 多 AP 时每个格点取信号最大值
- **输出密度**：栅格大小可配置，默认 **40×40**
- **输出示例**：`{preset_name}_rssi_ap{ap_count}.png`
  - 颜色：`-90dBm=红` → `-60dBm=黄` → `-30dBm=绿`
  - 图上叠加房间名称、墙体、AP 位置

### 2. RTMP 卡顿率栅格图 `generate_stall_grid`

- **输入**：同上（户型 + AP 数）
- **计算方式**：
  - 对可配置栅格（默认 **40×40**）的每个格点分别取最优 RSSI，仅修改 `wifi_rssi`，其余参数使用默认值
  - 运行 **10 秒短时 RTMP 仿真**（2000 步，5ms/步）
  - **0.5 dB RSSI 量化缓存**：相同 RSSI 只跑一次仿真，大幅加速
- **输出示例**：`{preset_name}_stall_ap{ap_count}.png`
  - 每格渲染为**独立的彩色圆点**，1 万个圆点组成栅格
  - 颜色动态归一化到全图最大卡顿率（红=高卡顿，绿=低卡顿）

### 3. AP 补点优化对比 `generate_ap_optimization_comparison`

- **输入**：户型 + 当前 AP 数 + 目标 AP 数
- **自动流程**：
  1. 计算当前布局的 RSSI 热力图 + 卡顿率栅格图（栅格大小可配置，默认 40×40）
  2. 调用 `recommend_ap_positions()` **贪心推荐**补点位置
     - 评分公式：自适应权重（`stall_rate > 10%` 时 stall 权重 0.7，否则 0.5）
     - 排除现有 AP 周边区域（动态 `min_dist ≈ 户型短边 × 0.35`，提升覆盖互补性）
     - 排除户型边缘 margin 区域（避免 AP 被推到墙角）
     - 对候选点做 3×3 局部重心平滑，使推荐位置更稳定
  3. 自动添加推荐 AP 到户型中
  4. 重新计算补点后的 RSSI 和卡顿率
  5. 输出 **左右对比图** + **矩阵数据文件**
- **输出文件**：
  - **对比图**：`{preset_name}_rssi_comparison.png`、`{preset_name}_stall_comparison.png`
  - **矩阵 .npy**（NumPy 二进制）：`{preset_name}_rssi_before.npy` / `_after.npy`、`{preset_name}_stall_before.npy` / `_after.npy`
  - **矩阵 .json**（带元数据）：`{preset_name}_rssi_before.json` / `_after.json`、`{preset_name}_stall_before.json` / `_after.json`
    - JSON 包含字段：`preset_name`、`ap_count`、`shape`、`mean_rssi` / `worst_rssi` / `mean_stall_rate` / `max_stall_rate`、`data`（行优先二维数组）

## 户型预设

| 户型名 | 尺寸 (宽×高 m) | 房间构成 |
|--------|----------------|----------|
| 一居室 | 8 × 6 | 客厅 + 卧室 + 厨卫 |
| 两居室 | 10 × 8 | 客厅 + 主卧 + 次卧 + 厨房 + 卫生间 |
| 三居室 | 12 × 10 | 客厅 + 主卧 + 次卧×2 + 厨房 + 卫生间 |
| 大平层 | 15 × 12 | 开放客餐厅 + 主卧套 + 次卧×2 + 书房 + 厨房 + 双卫 |

每个户型已内置外墙（高衰减 ~10dB）和内墙（低衰减 4~6dB）。

**大平层门洞模拟**：
- 通过 `--show-doors`（CLI）或 `show_doors=True`（API）可启用**门洞模式**。
- 所有 CLI 使用示例均默认携带 `--show-doors`，但参数本身为可选项；不传则大平层与一/二/三居室一样显示完整墙体。
- 门洞布局遵循户型逻辑：
  - 主卫仅与主卧相通（套房逻辑）
  - 每个房间至少有一扇门
  - 每扇门仅连接两个房间
  - 共 **8 处门洞**
- 门洞处无线段，因此信号穿墙时**不计算该段墙体衰减**，全屋平均 RSSI 约提升 +1~2 dB。
- 图上仅显示墙体空缺，不绘制门扇符号

## 中文字体配置

`skill.py` 启动时会自动探测并配置 matplotlib 中文字体，探测顺序：

1. `PingFang SC`（macOS）
2. `Heiti SC`
3. `SimHei`
4. `WenQuanYi Micro Hei`（Linux）
5. `Noto Sans CJK SC`
6. `Source Han Sans SC`
7. `Microsoft YaHei`（Windows）
8. `Arial Unicode MS`

同时关闭 `axes.unicode_minus`，防止负号显示为方块。

## 接口速查

```python
generate_rssi_heatmap(
    preset_name: str,         # 一居室 | 两居室 | 三居室 | 大平层
    ap_count: int,            # AP 数量
    output_path: str,         # PNG 输出路径
    grid_size: int = 40,      # 栅格分辨率 NxN
    show_doors: bool = False, # 仅大平层有效：显示门洞并影响仿真
)

generate_stall_grid(
    preset_name: str,
    ap_count: int,
    output_path: str,
    grid_size: int = 40,
    show_doors: bool = False, # 仅大平层有效：显示门洞并影响仿真
)

generate_ap_optimization_comparison(
    preset_name: str,
    current_ap_count: int,
    target_ap_count: int,
    output_dir: str,
    grid_size: int = 40,
    show_doors: bool = False, # 仅大平层有效：显示门洞并影响仿真
) -> dict[str, str]
# 返回字典包含以下键：
#   rssi_comparison, stall_comparison,
#   rssi_before_npy, rssi_after_npy,
#   stall_before_npy, stall_after_npy,
#   rssi_before_json, rssi_after_json,
#   stall_before_json, stall_after_json
```

## 注意事项

- `generate_stall_grid` 会实际调用 `SimulationEngine.simulate()`，**默认 40×40 网格约需 5~15 秒**（取决于 CPU 性能）。栅格越大计算量越大，可通过 `--grid-size` 调小以换取速度；得益于 0.5dB RSSI 缓存，实际只需执行约 140 次仿真。
- 若运行环境缺少中文字体，PNG 中的中文可能显示为方框。请确保系统安装了上述候选字体之一。
- **本 Skill 完全自包含**：`skill.py` 不依赖项目根目录或任何外部 Python 文件，复制到任意目录即可直接运行。
