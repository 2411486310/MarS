# MarS 项目上手指南（中文）

本文给你一条“从 0 到能跑、再到能改”的最短路径，适合第一次接触本项目。

## 1. 先理解项目分层

你可以把仓库拆成两层：

- `mlib/`：底层事件驱动撮合与仿真引擎（订单簿、事件、延迟、状态更新）。
- `market_simulation/`：上层应用（模型服务、示例脚本、Demo、配置）。

建议先把 `mlib` 当“市场操作系统”，`market_simulation` 当“业务 App”。

---

## 2. 环境准备（推荐容器）

项目官方建议使用 Docker / Dev Container。

### 2.1 安装依赖

```bash
pip install -e .[dev]
```

### 2.2 配置数据目录

默认目录由 `market_simulation/conf.py` 中 `C.directory.input_root_dir` 和 `output_root_dir` 控制。

可通过环境变量覆盖（嵌套配置分隔符是 `__`）：

```bash
export DIRECTORY__INPUT_ROOT_DIR=/your/data/path
export DIRECTORY__OUTPUT_ROOT_DIR=/your/output/path
```

---

## 3. 第一个可运行闭环：NoiseAgent 仿真

先不要碰大模型，先确认“引擎链路是通的”。

```bash
python market_simulation/examples/run_simulation.py
```

该脚本会：

1. 创建交易所与交易时段。
2. 注册 `TradeInfoState`。
3. 使用 `NoiseAgent` 持续发单。
4. 通过 `Env` 的 `for observation in env.env()` 驱动事件。
5. 输出价格轨迹图到 `tmp/price_curves.png`。

如果这一步跑通，说明你的基础仿真能力已经可用。

---

## 4. 必读源码顺序（按收益排序）

建议按下面顺序阅读：

1. `market_simulation/examples/run_simulation.py`
   - 看最小可用示例如何组装。
2. `mlib/core/env.py`
   - 看“类 Gym 的 generator 接口”怎么把事件流暴露给 Agent。
3. `mlib/core/engine.py`
   - 看事件队列、订单接收、撮合回报、状态唤醒、延迟处理。
4. `market_simulation/models/order_model.py`
   - 看订单特征如何 token 化并用自回归模型采样下一订单。
5. `market_simulation/rollout/order_model_serving.py` 与 `model_client.py`
   - 看服务部署与调用链路。

---

## 5. 从“会跑”到“会改”的三步

### 第一步：改一个 Agent 参数

在 `run_simulation.py` 改 `NoiseAgent` 的 `interval_seconds`、`init_price`、仿真时长，观察价格曲线变化。

### 第二步：加一个 State

参考 `market_simulation/states/`，新增你的统计状态（比如盘口不平衡度、成交量桶），并在 `exchange.register_state(...)` 注册。

### 第三步：把结果接到分析脚本

在 `market_simulation/examples/` 下复制一个脚本，先输出 CSV/图，再慢慢替换为你自己的策略与指标。

---

## 6. 模型与服务链路（进阶）

当你需要大模型生成订单时，再开启这条链路：

1. 启动 Ray Serve（项目提供了 `ray_serving.yaml` 和启动脚本）。
2. 使用 `ModelClient` 向 `http://ip:port/model_name` 发请求。
3. 校验返回 token 是否符合你的转换器和状态定义。

实践建议：

- 本地调试把 `MODEL_SERVING__TEMPERATURE` 设为较低值，便于复现实验。
- 先单标的、短窗口跑通，再扩到多标的与并行。

---

## 7. 常见坑位与排查建议

1. **数据路径不对**：优先检查 `DIRECTORY__INPUT_ROOT_DIR`。
2. **模型不可用**：当前 README 提到模型公开状态受审核流程影响，先使用 NoiseAgent 跑业务逻辑。
3. **GPU/依赖问题**：优先在官方 Docker / Dev Container 里跑。
4. **时序错误**：重点检查 agent 的 `communication_delay` 与下次唤醒时间是否满足约束。

---

## 8. 推荐你的 7 天学习路线

- Day 1：跑通 `run_simulation.py`，看懂 `Env` 循环。
- Day 2：读 `Engine`，把事件类型画成流程图。
- Day 3：新增一个自定义 `State`。
- Day 4：实现一个简单规则 `Agent`（例如只在价差超过阈值下单）。
- Day 5：接入指标统计，导出结果。
- Day 6：尝试 `forecast.py` 或 `market_impact.py`。
- Day 7：写一个你自己的实验脚本并固定随机种子复现。

如果你愿意，我可以下一步按你的目标（研究/生产/策略回测）给你做一版“个性化上手路线图 + 文件级 TODO 清单”。
