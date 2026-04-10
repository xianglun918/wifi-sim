---
name: wifi-simulation-workflow
description: |
  WiFi 覆盖与网络性能仿真完整工作流。从户型图输入开始，依次执行：① 户型图分割生成栅格地图、② 信号强度仿真、③ 网络性能分析，最终输出吞吐量/延迟/丢包率等指标。

  当用户提到以下场景时触发：
  - "WiFi 仿真"、"信号覆盖仿真"、"网络性能分析"
  - "户型图转信号强度"、"floorplan to network"
  - 需要串起 01、02、03 三个脚本的完整流程
  - "从户型图分析 WiFi 覆盖"、"AP 布置优化"
  - 用户提供了户型图并希望得到网络性能指标

  本 skill 会依次调用三个脚本，确保数据在它们之间正确传递。
---

# WiFi 仿真工作流 Skill

本 skill 将三个独立脚本串成完整工作流：**户型图 → 栅格地图 → 信号强度 → 网络性能**。

## 工作流概览

```
户型图图片
    │
    ▼
[ Step 1: 户型图处理 (深度学习) ]
    │ grid_map.npy + walls.json
    ▼
[ Step 2: 信号强度仿真 ]
    │ rssi_matrix.npy
    ▼
[ Step 3: 网络性能分析 ]
    │
    ▼
网络性能指标 (吞吐量、延迟、丢包率)
```

## 目录结构

```
wifi-simulation-workflow/
├── SKILL.md                        # 本文件
├── models/                        # 模型文件目录
│   ├── 65s.pt                     # YOLO 分割模型 (目标态)
│   └── 911checkpoint_best_ema.pth  # RFDETR 检测模型 (目标态)
├── scripts/
│   ├── 01_floorplan_process.py    # 户型图处理脚本 (集成深度学习)
│   ├── 02_signal_simulation.py    # 信号强度仿真脚本
│   └── 03_network_simulation.py   # 网络性能仿真脚本
├── data/
│   ├── mock_floorplan_100/        # Mock 栅格数据
│   │   ├── grid_map.npy
│   │   └── grid_info.json
│   └── test_data/                  # 测试数据
│       └── walls.json              # Mock 墙体坐标
└── evals/
    └── evals.json
```

## 输入要求

| 字段 | 类型 | 说明 |
|------|------|------|
| `floorplan_image` | Path | 户型图图片路径（支持 jpg/png） |
| `ap_count` | int | AP 数量（默认: 2） |
| `wifi_standard` | str | WiFi 标准（默认: wifi6） |
| `output_dir` | Path | 输出目录（默认: output/wifi_simulation） |

## 输出产物

| 文件 | 说明 |
|------|------|
| `grid_map.npy` | 400×400 栅格地图 |
| `walls.json` | 原始墙体坐标 (Step 1) |
| `rssi_matrix.npy` | 信号强度矩阵（dBm） |
| `rssi_heatmap.png` | 信号覆盖热力图 |
| `throughput_map.npy` | 吞吐量矩阵（Mbps） |
| `latency_map.npy` | 延迟矩阵（ms） |
| `packet_loss_map.npy` | 丢包率矩阵（%） |
| `network_dashboard.png` | 网络性能看板 |
| `network_metrics.json` | 网络指标统计 |

## 执行步骤

### Step 1: 安装依赖

```bash
# 在 skill 目录下安装依赖
cd /path/to/wifi-simulation-workflow

# 创建虚拟环境（推荐）
python -m venv .venv
source .venv/bin/activate

# 基础依赖
pip install numpy opencv-python matplotlib typer rich scikit-learn

# Step 1 深度学习依赖 (目标态需要)
pip install ultralytics torch
# rfdetr 和 shapely
pip install rfdetr shapely
```

### Step 2: 执行完整工作流

```bash
SKILL_DIR="$(cd "$(dirname "$0")" && pwd)"

# Step 1: 户型图处理 (使用深度学习)
# 模型文件存在时使用深度学习，否则使用轮廓检测备用
python "$SKILL_DIR/scripts/01_floorplan_process.py" \
    data/floorplan.jpg \
    --output-dir output/floorplan

# Step 2: 信号仿真
python "$SKILL_DIR/scripts/02_signal_simulation.py" \
    --grid output/floorplan/grid_map.npy \
    --ap-count 2 \
    --fast \
    --output-dir output/signal

# Step 3: 网络仿真
python "$SKILL_DIR/scripts/03_network_simulation.py" \
    --rssi output/signal/rssi_matrix.npy \
    --grid output/floorplan/grid_map.npy \
    --output-dir output/network
```

### Step 3: 查看结果

```bash
# 查看网络指标
cat output/network/network_metrics.json | python -m json.tool

# 查看信号统计
cat output/signal/signal_stats.json | python -m json.tool
```

## Step 1 详解：深度学习户型图处理

### 模型文件 (目标态)

将模型文件放入 `models/` 目录：

| 文件 | 说明 |
|------|------|
| `models/65s.pt` | YOLO 分割模型 |
| `models/911checkpoint_best_ema.pth` | RFDETR 检测模型 |

### 工作流程

```
输入图片
    │
    ▼
YOLO 分割 + RFDETR 检测
    │
    ▼
墙体线段提取 (merge/extend/split/delete)
    │
    ├── walls.json (墙体坐标)
    │
    ▼
walls_json_to_grid() 转换
    │
    ▼
grid_map.npy (400×400)
```

### 墙体 JSON 格式

```json
{
  "walls": [
    {"x": {"start": 100, "end": 400}, "y": {"start": 100, "end": 100}},
    {"x": {"start": 400, "end": 400}, "y": {"start": 100, "end": 300}}
  ]
}
```

### 备用方案

当模型文件不存在时，`01_floorplan_process.py` 会自动使用轮廓检测作为备用：
```bash
python "$SKILL_DIR/scripts/01_floorplan_process.py" \
    <image> \
    --no-dl \
    --output-dir output/floorplan
```

## 快速测试命令

```bash
SKILL_DIR="$(cd "$(dirname "$0")" && pwd)"

# 使用现有 mock 数据快速测试 (跳过 Step 1)
python "$SKILL_DIR/scripts/02_signal_simulation.py" \
    --grid "$SKILL_DIR/data/mock_floorplan_100/grid_map.npy" \
    --ap-count 2 \
    --fast \
    --output-dir output/quick_test/signal

python "$SKILL_DIR/scripts/03_network_simulation.py" \
    --rssi output/quick_test/signal/rssi_matrix.npy \
    --grid "$SKILL_DIR/data/mock_floorplan_100/grid_map.npy" \
    --output-dir output/quick_test/network
```

## 关键参数说明

### Step 1 (户型图处理)

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `INPUT_IMAGE` | 输入户型图路径 (positional) | 必需 |
| `--output-dir` | 输出目录 | output/floorplan |
| `--grid-size` | 栅格大小 | 400 |
| `--scale` | 比例尺 (m/px) | 自动计算 |
| `--no-dl` | 禁用深度学习，使用备用 | false |

### Step 2 (信号仿真)

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `--ap-count N` | 自动优化 N 个 AP 位置 | 2 |
| `--ap x,y` | 手动指定 AP 位置（米） | - |
| `--fast` | 启用快速模式（400×400 推荐） | false |
| `--tx-power` | AP 发射功率（dBm） | 20 |
| `--frequency` | WiFi 频率（GHz） | 5.0 |

### Step 3 (网络仿真)

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `--standard` | WiFi 标准（wifi4/5/6/6e/7） | wifi6 |
| `--bandwidth` | 信道带宽（MHz） | 80 |
| `--spatial-streams` | 空间流数量 | 2 |
| `--sta-count` | STA 设备数量 | 1 |

## 数据流向

```
01_floorplan_process.py (深度学习模型)
Input:  floorplan.jpg (户型图)
Process: YOLO + RFDETR 检测墙体
Output: walls.json (墙体坐标列表)
        grid_map.npy (400×400 栅格)
        grid_info.json (元数据)

02_signal_simulation.py
Input:  grid_map.npy
Output: rssi_matrix.npy (信号强度 dBm)
        rssi_heatmap.png
        optimized_ap_positions.json

03_network_simulation.py
Input:  rssi_matrix.npy
        grid_map.npy
Output: throughput_map.npy (Mbps)
        latency_map.npy (ms)
        packet_loss_map.npy (%)
        network_dashboard.png
        network_metrics.json
```

## 故障排查

| 问题 | 解决方案 |
|------|----------|
| `ModuleNotFoundError` | 激活虚拟环境或使用 `uv run python` |
| 模型文件不存在 | 使用 `--no-dl` 备用方案 |
| 信号仿真太慢 | 添加 `--fast` 参数启用向量化 |
| 网络仿真报错 | 确认 rssi_matrix.npy 存在且格式正确 |

## 依赖

### 基础依赖
- Python >= 3.11
- numpy, opencv-python, matplotlib
- scikit-learn (Step 2 的 K-means)
- typer, rich (CLI 框架)

### Step 1 深度学习依赖 (目标态)
- ultralytics (YOLO)
- torch
- rfdetr
- shapely

### 快速测试 (仅 Step 2-3)
- 无需深度学习依赖
