# 量化研究平台重构计划 V2.0

## 文档修订说明

本文档是对 `trade_platform_refactor.md` 的修订版本。修订过程中遵循以下原则：

1. **严格分层**：API层与后端服务完全分离
2. **职责单一**：后端仅负责调用脚本、维护数据库、提供API
3. **文件交互**：服务间通过本地文件进行信息交换
4. **CLI标准化**：所有服务脚本遵循统一的CLI接口规范
5. **不局限现有实现**：所有代码包括服务脚本都可重构

---

## 核心技术实现原理

### 架构设计哲学

本重构采用**微服务CLI架构**，核心设计理念：

#### 1. 绝对隔离原则

```
API层                    服务脚本
  │                         │
  ├─ subprocess.run() ─────>│ 独立进程
  │                         │
  │<──── 读取result.json ───┤
  │                         │
  ├─ 写入config.json ──────>│ 文件通信
```

**关键点：**
- ✅ API层**绝不导入**服务脚本模块
- ✅ 服务脚本**绝不导入**backend任何模块
- ✅ 唯一通信方式：`config.json` → CLI → `result.json`
- ✅ 每个CLI调用都是独立进程，完成后立即销毁

**实现示例：**
```python
# ❌ 错误：API层导入服务模块
from backend.scripts.backtest_service import run_backtest
run_backtest(...)  # 违反隔离原则

# ✅ 正确：通过subprocess调用
subprocess.run([
    "python",
    "backend/scripts/backtest_service.py",
    "--config", "config.json",
    "--output", "result.json"
])
```

#### 2. 无状态服务原则

**CLI脚本只返回最终结果，不提供中间状态**

```python
# 服务脚本的执行流程
def run_backtest(task_id, workdir, parameters):
    start_time = datetime.now()

    try:
        # 步骤1：数据准备（不输出进度）
        prepare_data(...)

        # 步骤2：模型训练（不输出进度）
        train_model(...)

        # 步骤3：执行回测（不输出进度）
        execute_backtest(...)

        # 只在最后返回结果
        return {
            "status": "completed",
            "metrics": {...}
        }

    except Exception as e:
        return {
            "status": "failed",
            "error": str(e),
            "traceback": traceback.format_exc()
        }
```

**前端状态判断逻辑：**
```python
# API层判断状态
def get_task_status(task_id):
    # 1. 检查subprocess是否还在运行
    if subprocess_alive(task_id):
        return "running"

    # 2. 检查result.json是否存在
    if not result_file_exists():
        return "failed"  # 进程异常退出

    # 3. 读取result.json
    result = read_result_file()
    return result["status"]  # "completed" or "failed"
```

#### 3. 插件式扩展架构

**策略插件发现（硬编码）：**
```python
STRATEGY_MAP = {
    "pointwise": "strategies.pointwise_service:PointwiseStrategy",
    "pairwise": "strategies.pairwise_service:PairwiseStrategy",
    "listwise": "strategies.listwise_service:ListwiseStrategy",
}

def load_strategy(strategy_type):
    # 动态加载
    module_path, class_name = STRATEGY_MAP[strategy_type].split(":")
    module = __import__(module_path, fromlist=[class_name])
    return getattr(module, class_name)()
```

**新增策略步骤：**
1. 创建 `strategies/new_strategy_service.py`
2. 继承 `BaseStrategy`
3. 实现必需方法
4. 在 `STRATEGY_MAP` 中注册

#### 4. 文件存储隔离原则

**每个任务独立存储，不共享模型**

```
data/tasks/
├── backtest_20240101_abc123/
│   ├── config.json          # 配置文件（可复现）
│   ├── result.json          # 最终结果
│   ├── models/model.pkl     # 独立模型
│   ├── predictions/         # 预测结果
│   └── logs/task.log        # 日志
├── backtest_20240115_def456/
│   ├── config.json
│   ├── result.json
│   └── models/model.pkl     # 另一个独立模型
```

**关键点：**
- ✅ 每个任务有自己的模型文件
- ✅ 不共享模型文件
- ✅ 模拟盘关联时复制模型路径（而非共享）

#### 5. 数据层独立性

**Qlib数据维护在默认目录**

```python
# 服务脚本中初始化Qlib
import qlib
qlib.init(provider="arrow", region="cn")
# 数据存储在 ~/.qlib/qlib_data/

# ❌ 不使用项目内目录
qlib.init(provider="arrow", region="cn", uri="./data/qlib")
```

**好处：**
- 数据可被多个项目共享
- 服务脚本更轻量（不携带数据）
- 符合Qlib最佳实践

#### 6. Schema即配置模板

**Schema定义了CLI的所有可配置参数**

```python
# CLI --schema 输出
{
  "properties": {
    "data.instruments": {"type": "string", "enum": [...]},
    "model.type": {"type": "string", "enum": [...]},
    "strategy.topk": {"type": "integer"},
    # ... 所有参数
  }
}
```

**前端使用：**
- 读取Schema渲染动态表单
- 用户选择性填写部分参数
- 未填写参数使用默认值

#### 7. 关联关系通过ID维护

**模拟盘 → 回测任务 → 模型**

```python
# 创建模拟盘时
POST /api/portfolios/strategies
{
  "backtest_task_id": "backtest_20240115_abc123"  # 指定回测任务
}

# API层处理
backtest_task = db.get_task(backtest_task_id)
model_path = backtest_task.model_path  # 从回测结果读取

strategy = PortfolioStrategy(
    backtest_task_id=backtest_task_id,
    model_path=model_path  # 保存模型路径
)

# 推理时使用
config = {
    "model_path": strategy.model_path  # 使用保存的路径
}
```

---

## 1. 系统架构

### 1.1 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                         前端 (Vue3 / React)                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ 策略配置页面 │  │ 回测结果页面 │  │ 模拟交易监控页面     │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                         API层 (FastAPI)                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ 策略管理API  │  │ 回测任务API  │  │ 模拟交易API         │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              服务调用管理 & 任务调度                      │  │
│  │         - 调用CLI脚本                                    │  │
│  │         - 读写结果文件                                   │  │
│  │         - 维护任务状态                                   │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│  数据服务层    │    │  回测服务层    │    │  推理服务层    │
│  (Python CLI)  │    │  (Python CLI)  │    │  (Python CLI)  │
│  data_service  │    │  backtest_svc  │    │  inference_svc │
│               │    │               │    │               │
│  - 独立程序    │    │  - 独立程序    │    │  - 独立程序    │
│  - CLI接口     │    │  - CLI接口     │    │  - CLI接口     │
│  - 文件输入    │    │  - 文件输入    │    │  - 文件输入    │
│  - 文件输出    │    │  - 文件输出    │    │  - 文件输出    │
└───────────────┘    └───────────────┘    └───────────────┘
        │                    │                    │
        ▼                    ▼                    ▼
┌─────────────────────────────────────────────────────────────┐
│                        数据存储层                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │  SQLite     │  │  Qlib数据   │  │  结果文件存储        │  │
│  │ (元数据)     │  │ (~/.qlib/)  │  │ (模型/日志/回测结果)  │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 架构原则

| 原则 | 说明 |
|------|------|
| **严格分层** | API层不直接使用Qlib，仅通过CLI调用服务脚本 |
| **独立进程** | 每个服务脚本作为独立程序运行，不共享内存 |
| **文件通信** | 服务间通过本地文件交换数据（config.json → result.json） |
| **职责单一** | 后端API层：调用脚本、读写数据库、提供接口 |
| **绝对隔离** | 服务脚本不提供中间状态反馈，只返回最终结果 |
| **独立存储** | 每个任务的模型文件独立存储，不共享 |

### 1.3 组件职责

| 组件 | 技术选型 | 职责 | 不负责 |
|------|----------|------|--------|
| 前端 | Vue3 + Element Plus / React + Ant Design | 用户交互、表单渲染、数据可视化 | ❌ 直接调用Qlib、❌ 执行回测 |
| API层 | FastAPI + SQLAlchemy | 调用CLI脚本、维护数据库、提供REST接口 | ❌ 模型训练、❌ 预测计算、❌ 进程管理 |
| 数据服务 | Python CLI | 更新市场数据、查询数据状态 | ❌ HTTP服务、❌ 数据库操作 |
| 回测服务 | Python CLI | 执行回测、训练模型、生成报告 | ❌ HTTP服务、❌ 任务调度 |
| 推理服务 | Python CLI | 加载模型、执行预测、生成信号 | ❌ HTTP服务、❌ 持仓管理 |
| 持久化层 | SQLite + SQLAlchemy ORM | 存储元数据（任务、策略、配置） | ❌ 存储行情数据 |

---

## 2. 服务层CLI接口规范

### 2.1 统一CLI接口

所有服务脚本必须支持以下CLI接口：

```bash
# 1. 查看帮助
python <service>.py --help

# 2. 获取参数Schema（用于前端动态表单）
python <service>.py --schema

# 3. 执行服务（标准模式）
python <service>.py --config <input.json> [--output <output.json>]

# 4. 执行服务（指定工作目录）
python <service>.py --config <input.json> --workdir <dir> --output <output.json>
```

### 2.2 输入输出格式

#### 输入格式 (config.json)

```json
{
  "task_id": "backtest_20240115_abc123",
  "workdir": "/data/tasks/backtest_20240115_abc123",
  "parameters": {
    // 服务特定参数
  }
}
```

#### 输出格式 (result.json)

```json
{
  "task_id": "backtest_20240115_abc123",
  "status": "completed",
  "start_time": "2024-01-15T10:00:00",
  "end_time": "2024-01-15T12:00:00",
  "error": null,
  "metrics": {
    // 服务特定指标
  },
  "outputs": {
    "model": "models/model.pkl",
    "predictions": "predictions/predictions.parquet",
    "report": "reports/report.html",
    "log": "logs/task.log"
  }
}
```

### 2.3 Schema格式 (JSON Schema)

```json
{
  "type": "object",
  "properties": {
    "model_type": {
      "type": "string",
      "title": "模型类型",
      "enum": ["lgbm", "mlp", "twin_tower"],
      "default": "lgbm"
    },
    "learning_rate": {
      "type": "number",
      "title": "学习率",
      "minimum": 0.0001,
      "maximum": 1.0,
      "default": 0.1
    }
  },
  "required": ["model_type"]
}
```

---

## 3. 数据结构设计

### 3.1 SQLite表结构

#### 策略表 (strategies)

```sql
CREATE TABLE strategies (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- 因子配置 (JSON)
    factors JSON NOT NULL,

    -- 预处理配置 (JSON)
    preprocessors JSON,

    -- 模型配置
    model_type VARCHAR(50) DEFAULT 'LGBM',
    model_params JSON,

    -- 关联信息
    latest_backtest_id INTEGER,
    latest_simulation_id INTEGER,

    FOREIGN KEY (latest_backtest_id) REFERENCES backtests(id),
    FOREIGN KEY (latest_simulation_id) REFERENCES simulations(id)
);
```

**数据示例：**
```json
{
  "id": 1,
  "name": "沪深300 Pairwise 策略",
  "description": "使用自适应锦标赛排序的配对学习策略",
  "factors": [
    {"name": "CLOSE0", "enabled": true},
    {"name": "MA5", "enabled": true},
    {"name": "VOLUME0", "enabled": true}
  ],
  "preprocessors": [
    {"class": "DropnaLabel", "kwargs": {}},
    {"class": "CSZScoreNorm", "kwargs": {"fields_group": "label"}}
  ],
  "model_type": "PAIRWISE_LGBM",
  "model_params": {
    "tournament": "adaptive",
    "k_pairs_per_stock": 3,
    "num_leaves": 210,
    "learning_rate": 0.2
  }
}
```

#### 回测任务表 (backtests)

```sql
CREATE TABLE backtests (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    strategy_id INTEGER NOT NULL,
    name VARCHAR(100) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,

    -- 回测配置 (JSON)
    config JSON NOT NULL,

    -- 回测结果 (JSON)
    results JSON,

    -- 文件路径
    workdir VARCHAR(500),
    model_path VARCHAR(500),
    log_path VARCHAR(500),
    report_path VARCHAR(500),

    FOREIGN KEY (strategy_id) REFERENCES strategies(id)
);
```

**数据示例：**
```json
{
  "id": 101,
  "strategy_id": 1,
  "name": "2024Q1回测",
  "status": "completed",
  "config": {
    "data": {
      "instruments": "csi300",
      "train_period": ["2017-01-01", "2020-12-31"],
      "test_period": ["2021-01-01", "2023-12-31"]
    },
    "model": {
      "type": "pairwise",
      "architecture": "lgbm"
    },
    "strategy": {
      "topk": 50,
      "n_drop": 5
    },
    "trading": {
      "initial_capital": 100000000,
      "benchmark": "SH000300"
    }
  },
  "results": {
    "ic_metrics": {
      "mean_ic": 0.1099,
      "ic_std": 0.1148,
      "ic_ir": 0.9566
    },
    "performance_metrics": {
      "annual_return": 0.156,
      "sharpe_ratio": 1.23,
      "max_drawdown": -0.156
    }
  },
  "workdir": "/data/tasks/backtest_20240115_abc123"
}
```

#### 模拟交易表 (simulations)

```sql
CREATE TABLE simulations (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    strategy_id INTEGER NOT NULL,
    backtest_id INTEGER,
    name VARCHAR(100) NOT NULL,
    status VARCHAR(20) DEFAULT 'running',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_run_at TIMESTAMP,

    -- 交易配置 (JSON)
    config JSON NOT NULL,

    -- 当前状态
    positions JSON,
    cash FLOAT,
    total_value FLOAT,

    FOREIGN KEY (strategy_id) REFERENCES strategies(id),
    FOREIGN KEY (backtest_id) REFERENCES backtests(id)
);
```

#### 交易记录表 (trades)

```sql
CREATE TABLE trades (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    simulation_id INTEGER NOT NULL,
    trade_date DATE NOT NULL,
    stock_code VARCHAR(10) NOT NULL,
    action VARCHAR(4) NOT NULL,
    price FLOAT NOT NULL,
    volume INTEGER NOT NULL,
    amount FLOAT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (simulation_id) REFERENCES simulations(id)
);
```

#### 因子元数据表 (factors_meta)

```sql
CREATE TABLE factors_meta (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name VARCHAR(50) NOT NULL UNIQUE,
    display_name VARCHAR(100),
    category VARCHAR(50),
    expression TEXT,
    description TEXT,
    data_type VARCHAR(20) DEFAULT 'float'
);
```

### 3.2 TypeScript类型定义

```typescript
// types/backtest.ts

export interface BacktestConfig {
  name: string;
  description?: string;
  config: {
    data: {
      instruments: string;
      train_period: [string, string];
      valid_period: [string, string];
      test_period: [string, string];
    };
    model: {
      type: 'pointwise' | 'pairwise';
      architecture: 'lgbm' | 'mlp' | 'twin_tower';
      lgbm_params?: LGBMParams;
      mlp_params?: MLPParams;
      pairwise_params?: PairwiseParams;
    };
    strategy: {
      topk: number;
      n_drop: number;
    };
    trading: {
      initial_capital: number;
      benchmark: string;
      open_cost: number;
      close_cost: number;
    };
  };
}

export interface BacktestTask {
  task_id: string;
  strategy_id: number;
  name: string;
  status: 'pending' | 'running' | 'completed' | 'failed';
  created_at: string;
  config: BacktestConfig['config'];
  progress?: number;
  error_message?: string;
}

export interface ICMetrics {
  mean_ic: number;
  ic_std: number;
  ic_min: number;
  ic_max: number;
  ic_ir: number;
}

export interface PerformanceMetrics {
  annual_return: number;
  volatility: number;
  sharpe_ratio: number;
  max_drawdown: number;
  cumulative_return: number;
  calmar_ratio: number;
}

export interface BacktestResult {
  task_id: string;
  name: string;
  status: string;
  config: BacktestConfig['config'];
  ic_metrics?: ICMetrics;
  performance_metrics?: PerformanceMetrics;
  plot_paths?: Record<string, string>;
  completed_at?: string;
}
```

---

## 4. 服务层实现规范

### 4.1 数据服务 (data_service.py)

**位置：** `backend/scripts/data_service.py`

```python
#!/usr/bin/env python3
"""
数据服务 CLI

负责：
- 更新本地Qlib市场数据
- 查询数据状态
- 提供数据信息

不负责：
- HTTP服务
- 数据库操作
"""

import argparse
import json
from pathlib import Path
from datetime import datetime

def update_data(source: str = "akshare", target_market: str = "csi300"):
    """
    更新市场数据到最近交易日

    CLI内部自动：
    1. 检测当前已有数据的最新日期
    2. 获取市场最新交易日
    3. 增量更新缺失的数据
    """
    import qlib
    from qlib.data import D

    # 使用默认目录 ~/.qlib/qlib_data/
    qlib.init(provider="arrow", region="cn")

    # 检测最新交易日并增量更新
    # ... 数据更新逻辑 ...

    return {
        "source": source,
        "target_market": target_market,
        "latest_date": "2024-01-15",  # 实际更新到的日期
        "update_count": 500,           # 更新的股票数
        "status": "success"
    }

def get_data_status():
    """获取数据状态"""
    import qlib
    qlib.init(provider="arrow")

    from qlib.data import D
    instruments = D.instruments()

    return {
        "last_update": datetime.now().isoformat(),
        "stock_count": len(instruments),
        "status": "ok"
    }

def main():
    parser = argparse.ArgumentParser(description="数据服务CLI")
    parser.add_argument('--schema', action='store_true', help='输出配置Schema')
    parser.add_argument('--config', type=str, help='配置文件路径')
    parser.add_argument('--output', type=str, help='输出文件路径')

    args = parser.parse_args()

    if args.schema:
        schema = {
            "type": "object",
            "properties": {
                "action": {
                    "type": "string",
                    "enum": ["update", "status"],
                    "title": "操作类型"
                },
                "source": {
                    "type": "string",
                    "enum": ["akshare", "tushare"],
                    "title": "数据源",
                    "default": "akshare"
                },
                "target_market": {
                    "type": "string",
                    "enum": ["csi300", "csi500", "csi800"],
                    "title": "目标市场",
                    "default": "csi300"
                }
            }
        }
        print(json.dumps(schema, indent=2))
        return

    if args.config:
        with open(args.config) as f:
            config = json.load(f)

        action = config.get("action")

        if action == "update":
            result = update_data(
                source=config.get("source", "akshare"),
                start_date=config.get("start_date"),
                end_date=config.get("end_date")
            )
        elif action == "status":
            result = get_data_status()
        else:
            result = {"error": f"Unknown action: {action}"}

        if args.output:
            with open(args.output, 'w') as f:
                json.dump(result, f, indent=2)

        print(json.dumps(result, indent=2))

if __name__ == "__main__":
    main()
```

**使用方式：**
```bash
# 获取Schema
python backend/scripts/data_service.py --schema

# 更新数据（更新到最近交易日）
echo '{"action": "update", "source": "akshare", "target_market": "csi300"}' > /tmp/config.json
python backend/scripts/data_service.py --config /tmp/config.json --output /tmp/result.json

# 查询状态
echo '{"action": "status"}' > /tmp/config.json
python backend/scripts/data_service.py --config /tmp/config.json --output /tmp/result.json
```

### 4.2 市场数据服务 (market_service.py)

**位置：** `backend/scripts/market_service.py`

```python
#!/usr/bin/env python3
"""
市场数据服务 CLI

负责：
- 提供K线数据
- 提供实时行情
- 提供股票池信息
- 提供交易日历

不负责：
- HTTP服务
- 数据库操作
"""

import argparse
import json
from pathlib import Path

def get_kline(instrument: str, days: int = 30):
    """获取K线数据"""
    import qlib
    qlib.init(provider="arrow", region="cn")

    from qlib.data import D

    # 获取K线数据
    df = D.features([instrument], ["open", "high", "low", "close", "volume"],
                    end_time="now", freq="day")

    return {
        "instrument": instrument,
        "data": df.to_dict(),
        "count": len(df)
    }

def get_realtime(instruments: list):
    """获取实时行情（使用akshare）"""
    import akshare as ak

    quotes = []
    for instrument in instruments:
        # 转换股票代码格式
        code = instrument.replace("SH", "sh").replace("SZ", "sz")

        try:
            df = ak.stock_zh_a_spot_em()
            row = df[df['代码'] == code]

            if not row.empty:
                quotes.append({
                    "instrument": instrument,
                    "name": row['名称'].values[0],
                    "price": row['最新价'].values[0],
                    "change": row['涨跌幅'].values[0],
                    "volume": row['成交量'].values[0]
                })
        except Exception as e:
            pass  # 跳过获取失败的股票

    return {
        "quotes": quotes,
        "count": len(quotes)
    }

def get_instruments(market: str = "csi300"):
    """获取股票池"""
    import qlib
    qlib.init(provider="arrow", region="cn")

    from qlib.data import D

    instruments = D.instruments(market=market)

    return {
        "market": market,
        "instruments": list(instruments),
        "count": len(instruments)
    }

def main():
    parser = argparse.ArgumentParser(description="市场数据服务CLI")
    parser.add_argument('--schema', action='store_true', help='输出配置Schema')
    parser.add_argument('--config', type=str, help='配置文件路径')
    parser.add_argument('--output', type=str, help='输出文件路径')

    args = parser.parse_args()

    if args.schema:
        schema = {
            "type": "object",
            "properties": {
                "action": {
                    "type": "string",
                    "enum": ["kline", "realtime", "instruments"],
                    "title": "操作类型"
                },
                "instrument": {
                    "type": "string",
                    "title": "股票代码"
                },
                "instruments": {
                    "type": "array",
                    "items": {"type": "string"},
                    "title": "股票代码列表"
                },
                "market": {
                    "type": "string",
                    "enum": ["csi300", "csi500", "csi800"],
                    "title": "市场"
                },
                "days": {
                    "type": "integer",
                    "title": "天数",
                    "default": 30
                }
            }
        }
        print(json.dumps(schema, indent=2))
        return

    if args.config:
        with open(args.config) as f:
            config = json.load(f)

        action = config.get("action")

        if action == "kline":
            result = get_kline(
                instrument=config["instrument"],
                days=config.get("days", 30)
            )
        elif action == "realtime":
            result = get_realtime(config["instruments"])
        elif action == "instruments":
            result = get_instruments(config.get("market", "csi300"))
        else:
            result = {"error": f"Unknown action: {action}"}

        if args.output:
            with open(args.output, 'w') as f:
                json.dump(result, f, indent=2)

        print(json.dumps(result, indent=2))

if __name__ == "__main__":
    main()
```

### 4.3 回测服务（插件式架构）

**插件目录结构：**
```
backend/scripts/strategies/
├── __init__.py
├── base_strategy.py        # 策略基类
├── pointwise_service.py     # Pointwise策略（对应MyAlpha）
├── pairwise_service.py      # Pairwise策略（对应pairwise_rank）
└── listwise_service.py      # Listwise策略（未来扩展）
```

**策略发现机制（硬编码）：**

```python
# backend/scripts/backtest_service.py

def load_strategy(strategy_type: str):
    """
    动态加载策略插件（硬编码方式）

    新增策略时需要在此处添加映射
    """
    STRATEGY_MAP = {
        "pointwise": "strategies.pointwise_service:PointwiseStrategy",
        "pairwise": "strategies.pairwise_service:PairwiseStrategy",
        "listwise": "strategies.listwise_service:ListwiseStrategy",
    }

    if strategy_type not in STRATEGY_MAP:
        raise ValueError(f"Unknown strategy type: {strategy_type}")

    module_path, class_name = STRATEGY_MAP[strategy_type].split(":")
    module = __import__(module_path, fromlist=[class_name])
    return getattr(module, class_name)()
```

**策略基类定义：**

```python
# backend/scripts/strategies/base_strategy.py

from abc import ABC, abstractmethod
from pathlib import Path
from typing import Dict, Any

class BaseStrategy(ABC):
    """策略基类，所有策略插件必须继承此类"""

    @abstractmethod
    def get_schema(self) -> Dict[str, Any]:
        """返回策略的参数Schema"""
        pass

    @abstractmethod
    def train(self, workdir: Path, parameters: Dict[str, Any]) -> Dict[str, Any]:
        """
        训练模型

        返回：
        {
            "model_path": "path/to/model.pkl",
            "training_time": 120.5,
            "metrics": {...}
        }
        """
        pass

    @abstractmethod
    def predict(self, workdir: Path, parameters: Dict[str, Any]) -> Dict[str, Any]:
        """
        执行预测

        返回：
        {
            "predictions_path": "path/to/predictions.parquet",
            "instruments": ["SH600000", ...],
            "scores": [0.85, ...]
        }
        """
        pass

    @abstractmethod
    def backtest(self, workdir: Path, parameters: Dict[str, Any]) -> Dict[str, Any]:
        """
        执行回测

        返回：
        {
            "ic_metrics": {...},
            "performance_metrics": {...},
            "plot_paths": {...}
        }
        """
        pass
```

**回测服务入口：**

```python
# backend/scripts/backtest_service.py

#!/usr/bin/env python3
"""
回测服务 CLI（插件式架构）

负责：
- 根据策略类型动态加载策略插件
- 执行回测任务
- 生成回测报告

不负责：
- HTTP服务
- 任务调度（由API层负责）
"""

import argparse
import json
import traceback
from pathlib import Path
from datetime import datetime

def load_strategy(strategy_type: str):
    """动态加载策略插件"""
    if strategy_type == "pairwise":
        from strategies.pairwise_service import PairwiseStrategy
        return PairwiseStrategy()
    elif strategy_type == "pointwise":
        from strategies.pointwise_service import PointwiseStrategy
        return PointwiseStrategy()
    else:
        raise ValueError(f"Unknown strategy type: {strategy_type}")

def run_backtest(task_id: str, workdir: Path, parameters: dict):
    """执行回测"""

    start_time = datetime.now()

    # 创建工作目录
    workdir = Path(workdir)
    workdir.mkdir(parents=True, exist_ok=True)
    (workdir / "models").mkdir(exist_ok=True)
    (workdir / "predictions").mkdir(exist_ok=True)
    (workdir / "reports").mkdir(exist_ok=True)
    (workdir / "logs").mkdir(exist_ok=True)

    # 保存配置用于复现
    with open(workdir / "config.json", 'w') as f:
        json.dump(parameters, f, indent=2)

    try:
        # === 步骤1: 初始化Qlib ===
        # 使用默认目录 ~/.qlib/qlib_data/
        import qlib
        qlib.init(provider="arrow", region="cn")

        # === 步骤2: 加载策略插件 ===
        model_config = parameters["model"]
        strategy_type = model_config.get("type", "pairwise")
        strategy = load_strategy(strategy_type)

        # === 步骤3: 训练模型 ===
        train_result = strategy.train(workdir, parameters)

        # === 步骤4: 执行回测 ===
        backtest_result = strategy.backtest(workdir, parameters)

        # === 步骤5: 生成结果 ===
        end_time = datetime.now()

        result = {
            "task_id": task_id,
            "status": "completed",
            "start_time": start_time.isoformat(),
            "end_time": end_time.isoformat(),
            "error": None,
            "config": parameters,  # 用于复现
            "metrics": {
                "ic_metrics": backtest_result.get("ic_metrics", {}),
                "performance_metrics": backtest_result.get("performance_metrics", {})
            },
            "outputs": {
                "model": str(train_result.get("model_path")),
                "predictions": str(backtest_result.get("predictions_path")),
                "report": str(workdir / "reports" / "report.html"),
                "log": str(workdir / "logs" / "task.log"),
                "config": str(workdir / "config.json")  # 用于复现
            }
        }

        return result

    except Exception as e:
        return {
            "task_id": task_id,
            "status": "failed",
            "start_time": start_time.isoformat(),
            "end_time": datetime.now().isoformat(),
            "error": str(e),
            "traceback": traceback.format_exc(),
            "config": parameters  # 失败时也保留配置用于复现
        }

def main():
    parser = argparse.ArgumentParser(description="回测服务CLI")
    parser.add_argument('--schema', action='store_true', help='输出配置Schema')
    parser.add_argument('--config', type=str, help='配置文件路径')
    parser.add_argument('--output', type=str, help='输出文件路径')

    args = parser.parse_args()

    if args.schema:
        # 输出Schema供前端动态表单使用
        # Schema是配置模板的定义，声明所有支持的配置项
        # 前端可以选择性填写其中一部分
        schema = {
            "type": "object",
            "title": "回测配置Schema",
            "description": "CLI调用方式及用户可填写的参数定义",
            "properties": {
                "task_id": {
                    "type": "string",
                    "title": "任务ID",
                    "description": "唯一标识符"
                },
                "workdir": {
                    "type": "string",
                    "title": "工作目录",
                    "description": "任务数据存储路径"
                },
                "parameters": {
                    "type": "object",
                    "title": "回测参数",
                    "properties": {
                        # 数据配置
                        "data.instruments": {
                            "type": "string",
                            "title": "股票池",
                            "enum": ["csi300", "csi500", "csi800"],
                            "default": "csi300"
                        },
                        "data.train_period": {
                            "type": "array",
                            "title": "训练期",
                            "items": {"type": "string", "format": "date"},
                            "description": "训练开始和结束日期"
                        },
                        "data.valid_period": {
                            "type": "array",
                            "title": "验证期",
                            "items": {"type": "string", "format": "date"}
                        },
                        "data.test_period": {
                            "type": "array",
                            "title": "测试期",
                            "items": {"type": "string", "format": "date"}
                        },

                        # 模型配置
                        "model.type": {
                            "type": "string",
                            "title": "模型类型",
                            "enum": ["pointwise", "pairwise", "listwise"]
                        },
                        "model.architecture": {
                            "type": "string",
                            "title": "模型架构",
                            "enum": ["lgbm", "mlp", "twin_tower"]
                        },

                        # Pairwise特定参数
                        "model.pairwise_params.tournament": {
                            "type": "string",
                            "title": "锦标赛排序方法",
                            "enum": ["single_elim", "double_elim", "hybrid", "adaptive"],
                            "default": "adaptive"
                        },
                        "model.pairwise_params.k_pairs_per_stock": {
                            "type": "integer",
                            "title": "每股票配对数",
                            "minimum": 1,
                            "maximum": 10,
                            "default": 3
                        },

                        # 策略配置
                        "strategy.topk": {
                            "type": "integer",
                            "title": "TopK数量",
                            "minimum": 1,
                            "default": 50
                        },
                        "strategy.n_drop": {
                            "type": "integer",
                            "title": "调出数量",
                            "minimum": 0,
                            "default": 5
                        },

                        # 交易配置
                        "trading.initial_capital": {
                            "type": "number",
                            "title": "初始资金",
                            "default": 100000000
                        },
                        "trading.benchmark": {
                            "type": "string",
                            "title": "基准指数",
                            "enum": ["SH000300", "SZ000905", "SH000905"],
                            "default": "SH000300"
                        },
                        "trading.open_cost": {
                            "type": "number",
                            "title": "买入费率",
                            "minimum": 0,
                            "maximum": 0.01,
                            "default": 0.0005
                        },
                        "trading.close_cost": {
                            "type": "number",
                            "title": "卖出费率",
                            "minimum": 0,
                            "maximum": 0.01,
                            "default": 0.0015
                        }
                    }
                }
            }
        }
        print(json.dumps(schema, indent=2))
        return

    if args.config:
        with open(args.config) as f:
            config = json.load(f)

        result = run_backtest(
            task_id=config["task_id"],
            workdir=Path(config["workdir"]),
            parameters=config["parameters"]
        )

        if args.output:
            with open(args.output, 'w') as f:
                json.dump(result, f, indent=2)

        print(json.dumps(result, indent=2))

if __name__ == "__main__":
    main()
```

### 4.4 推理服务 (inference_service.py)

**位置：** `backend/scripts/inference_service.py`

```python
#!/usr/bin/env python3
"""
推理服务 CLI

负责：
- 加载训练好的模型
- 生成预测分数
- 生成交易信号

不负责：
- HTTP服务
- 持仓管理（由API层负责）
"""

import argparse
import json
from pathlib import Path
from datetime import datetime

def run_inference(task_id: str, workdir: Path, parameters: dict):
    """执行推理"""

    try:
        import qlib
        # 使用默认目录 ~/.qlib/qlib_data/
        qlib.init(provider="arrow", region="cn")

        model_path = Path(parameters["model_path"])
        date = parameters["date"]

        # 加载模型并生成预测
        # ... 推理逻辑 ...

        predictions = {
            "SH600000": 0.85,
            "SZ000001": -0.23,
            # ...
        }

        # 转换为交易信号
        signals = []
        for instrument, score in predictions.items():
            if score > 0.5:
                action = "buy"
            elif score < -0.2:
                action = "sell"
            else:
                action = "hold"

            signals.append({
                "instrument": instrument,
                "score": score,
                "action": action
            })

        return {
            "task_id": task_id,
            "status": "completed",
            "date": date,
            "predictions": predictions,
            "signals": signals,
            "outputs": {
                "predictions": str(workdir / "predictions.json"),
                "signals": str(workdir / "signals.json")
            }
        }

    except Exception as e:
        return {
            "task_id": task_id,
            "status": "failed",
            "error": str(e)
        }

def main():
    parser = argparse.ArgumentParser(description="推理服务CLI")
    parser.add_argument('--schema', action='store_true', help='输出配置Schema')
    parser.add_argument('--config', type=str, help='配置文件路径')
    parser.add_argument('--output', type=str, help='输出文件路径')

    args = parser.parse_args()

    if args.schema:
        schema = {
            "type": "object",
            "properties": {
                "task_id": {"type": "string"},
                "workdir": {"type": "string"},
                "parameters": {
                    "type": "object",
                    "properties": {
                        "model_path": {"type": "string"},
                        "date": {"type": "string", "format": "date"},
                        "topk": {"type": "integer"},
                        "n_drop": {"type": "integer"}
                    }
                }
            }
        }
        print(json.dumps(schema, indent=2))
        return

    if args.config:
        with open(args.config) as f:
            config = json.load(f)

        result = run_inference(
            task_id=config["task_id"],
            workdir=Path(config["workdir"]),
            parameters=config["parameters"]
        )

        if args.output:
            with open(args.output, 'w') as f:
                json.dump(result, f, indent=2)

        print(json.dumps(result, indent=2))

if __name__ == "__main__":
    main()
```

---

## 5. API层实现规范

### 5.1 API层职责

API层是**唯一**的Web服务层，负责：

1. **调用CLI脚本**：通过 subprocess 调用服务脚本
2. **维护数据库**：管理任务状态、策略配置
3. **提供REST接口**：响应前端请求
4. **文件管理**：创建工作目录、读写结果文件

API层**不负责**：

- ❌ 模型训练
- ❌ 预测计算
- ❌ 进程管理（只用 subprocess）
- ❌ 直接使用 Qlib

### 5.2 并发控制规范

**实现方式：** API层使用线程池控制并发CLI调用

```python
# backend/core/concurrent_executor.py

from concurrent.futures import ThreadPoolExecutor, as_completed
from typing import Callable, Any
import threading

class ConcurrentExecutor:
    """并发执行器 - 控制同时运行的CLI脚本数量"""

    def __init__(self, max_workers: int = 3):
        """
        参数：
            max_workers: 最大并发任务数（建议3-5）
        """
        self.executor = ThreadPoolExecutor(max_workers=max_workers)
        self.lock = threading.Lock()
        self.running_tasks = {}

    def submit_cli(self, func: Callable, *args, **kwargs) -> str:
        """
        提交CLI任务到线程池

        返回：任务ID
        """
        future = self.executor.submit(func, *args, **kwargs)

        task_id = f"task_{id(future)}"
        with self.lock:
            self.running_tasks[task_id] = {
                "future": future,
                "status": "running"
            }

        # 任务完成后自动清理
        future.add_done_callback(lambda f: self._on_task_done(task_id))

        return task_id

    def _on_task_done(self, task_id: str):
        with self.lock:
            self.running_tasks.pop(task_id, None)

    def get_stats(self):
        """获取执行器统计"""
        return {
            "max_workers": self.executor._max_workers,
            "running_count": len(self.running_tasks)
        }
```

**在API层使用：**

```python
# backend/api/backtests.py

from backend.core.concurrent_executor import ConcurrentExecutor

executor = ConcurrentExecutor(max_workers=3)

@router.post("/create")
async def create_backtest(config: dict, db: Session):
    # ... 准备配置文件 ...

    # 提交到并发执行器（不阻塞）
    task_id = executor.submit_cli(
        subprocess.run,
        [
            "python",
            "backend/scripts/backtest_service.py",
            "--config", str(config_file),
            "--output", str(output_file)
        ],
        check=True,
        capture_output=True,
        timeout=14400  # 4小时超时
    )

    return {"task_id": task_id, "status": "pending"}
```

### 5.3 API缓存规范

**实现方式：** API层对K线等市场数据提供内存缓存

```python
# backend/core/cache_manager.py

from functools import lru_cache
from datetime import datetime, timedelta
import threading

class CacheManager:
    """API层缓存管理器"""

    def __init__(self):
        self._cache = {}
        self._timestamps = {}
        self._lock = threading.Lock()

    def get(self, key: str, max_age_seconds: int = 300):
        """
        获取缓存数据

        参数：
            key: 缓存键
            max_age_seconds: 最大缓存时间（秒），默认5分钟
        """
        with self._lock:
            if key not in self._cache:
                return None

            timestamp = self._timestamps.get(key)
            if datetime.now() - timestamp > timedelta(seconds=max_age_seconds):
                # 缓存过期
                del self._cache[key]
                del self._timestamps[key]
                return None

            return self._cache[key]

    def set(self, key: str, value: Any):
        with self._lock:
            self._cache[key] = value
            self._timestamps[key] = datetime.now()

    def invalidate(self, key: str):
        with self._lock:
            self._cache.pop(key, None)
            self._timestamps.pop(key, None)

cache_manager = CacheManager()
```

**在市场数据API中使用：**

```python
# backend/api/market.py

from backend.core.cache_manager import cache_manager

@router.get("/kline/{instrument}")
async def get_kline(instrument: str, days: int = 30):
    # 检查缓存
    cache_key = f"kline:{instrument}:{days}"
    cached_data = cache_manager.get(cache_key, max_age_seconds=300)

    if cached_data:
        return cached_data

    # 缓存未命中，调用CLI
    config = {
        "action": "kline",
        "instrument": instrument,
        "days": days
    }

    result = call_market_cli(config)

    # 写入缓存
    cache_manager.set(cache_key, result)

    return result
```

### 5.4 任务状态判断规范

**重要：** 服务脚本不提供中间状态反馈，只返回最终结果

**前端状态判断逻辑：**

```python
# backend/api/backtests.py

@router.get("/{task_id}/status")
async def get_task_status(task_id: str, db: Session):
    """
    获取任务状态

    状态判断逻辑：
    1. 运行中：subprocess进程还在运行
    2. 已完成：result.json文件存在且status="completed"
    3. 错误：result.json文件存在且status="failed"或subprocess异常退出
    """
    task = task_manager.get_task(db, task_id)
    if not task:
        raise HTTPException(status_code=404, detail="任务不存在")

    # 检查subprocess是否还在运行
    if executor.is_task_running(task_id):
        return {
            "task_id": task_id,
            "status": "running",
            "message": "任务执行中"
        }

    # 检查结果文件
    result_file = Path(task.workdir) / "result.json"

    if not result_file.exists():
        # 进程已结束但无结果文件 = 异常失败
        return {
            "task_id": task_id,
            "status": "failed",
            "message": "任务异常终止，结果文件不存在"
        }

    # 读取结果
    with open(result_file) as f:
        result = json.load(f)

    return {
        "task_id": task_id,
        "status": result.get("status"),
        "error": result.get("error")
    }
```

**并发执行器扩展：**

```python
# backend/core/concurrent_executor.py

class ConcurrentExecutor:
    # ... 现有代码 ...

    def is_task_running(self, task_id: str) -> bool:
        """检查任务是否还在运行"""
        with self.lock:
            task_info = self.running_tasks.get(task_id)
            if not task_info:
                return False

            future = task_info["future"]
            return not future.done()
```

### 5.5 任务删除规范

**删除逻辑：** 同步删除工作目录

```python
# backend/api/backtests.py

@router.delete("/{task_id}")
async def delete_backtest(task_id: str, db: Session):
    """
    删除回测任务

    删除内容：
    1. 数据库记录
    2. 工作目录（包含模型、结果、日志等所有文件）
    """
    task = task_manager.get_task(db, task_id)
    if not task:
        raise HTTPException(status_code=404, detail="任务不存在")

    # 等待任务完成（如果还在运行）
    if executor.is_task_running(task_id):
        raise HTTPException(status_code=400, detail="任务正在运行，无法删除")

    # 删除工作目录
    workdir = Path(task.workdir)
    if workdir.exists():
        import shutil
        shutil.rmtree(workdir)

    # 删除数据库记录
    db.delete(task)
    db.commit()

    return {"message": "任务已删除"}
```

### 5.6 错误处理规范

**CLI脚本错误返回格式：**

```json
{
  "task_id": "backtest_20240115_abc123",
  "status": "failed",
  "start_time": "2024-01-15T10:00:00",
  "end_time": "2024-01-15T10:05:00",
  "error": {
    "type": "ValueError",
    "message": "learning_rate must be positive",
    "traceback": "Traceback (most recent call last):\n  File ...",
    "config": {
      // 完整的输入参数，用于复现
      "model": {"learning_rate": -0.1, ...}
    }
  }
}
```

**关键要求：**
- ✅ 完整堆栈信息（traceback）
- ✅ 完整配置参数（config）- 用于复现
- ❌ 不需要详细的用户友好错误解释

### 5.4 回测任务API实现

**位置：** `backend/api/backtests.py`

```python
"""
回测任务 API 路由

职责：
1. 接收前端请求
2. 调用 backtest_service.py CLI
3. 维护任务状态
4. 返回结果给前端
"""

from fastapi import APIRouter, HTTPException
from sqlalchemy.orm import Session
from backend.dependencies import get_db
from backend.models.backtest import BacktestTaskResponse
from backend.core.task_manager import TaskManager
import subprocess
import json
from pathlib import Path

router = APIRouter()
task_manager = TaskManager()

@router.post("/create", response_model=dict)
async def create_backtest(
    config: dict,
    db: Session = Depends(get_db)
):
    """
    创建回测任务

    流程：
    1. 创建任务记录到数据库
    2. 创建工作目录
    3. 生成配置文件
    4. 调用 backtest_service.py CLI
    5. 更新任务状态
    """

    # 1. 生成任务ID
    task_id = f"backtest_{datetime.now().strftime('%Y%m%d_%H%M%S')}"

    # 2. 创建工作目录
    workdir = Path(f"./data/tasks/{task_id}")
    workdir.mkdir(parents=True, exist_ok=True)

    # 3. 保存配置到文件
    config_file = workdir / "config.json"
    config_with_task = {
        "task_id": task_id,
        "workdir": str(workdir),
        "parameters": config
    }
    with open(config_file, 'w') as f:
        json.dump(config_with_task, f, indent=2)

    # 4. 保存到数据库
    task = task_manager.create_task(
        db=db,
        task_id=task_id,
        name=config.get("name", "Unnamed"),
        config=config,
        workdir=str(workdir)
    )

    # 5. 调用 CLI 脚本（异步）
    output_file = workdir / "result.json"

    # 使用后台任务执行
    def run_cli():
        subprocess.run([
            "python",
            "backend/scripts/backtest_service.py",
            "--config", str(config_file),
            "--output", str(output_file)
        ], check=True)

        # 读取结果并更新数据库
        with open(output_file) as f:
            result = json.load(f)

        task_manager.update_task(db, task_id, result)

    # 在后台线程中运行
    import threading
    thread = threading.Thread(target=run_cli)
    thread.start()

    return {
        "task_id": task_id,
        "status": "pending",
        "message": "回测任务已创建"
    }

@router.get("/{task_id}/result")
async def get_backtest_result(
    task_id: str,
    db: Session = Depends(get_db)
):
    """获取回测结果"""
    task = task_manager.get_task(db, task_id)

    if not task:
        raise HTTPException(status_code=404, detail="任务不存在")

    # 从结果文件读取
    result_file = Path(task.workdir) / "result.json"

    if not result_file.exists():
        return {
            "task_id": task_id,
            "status": task.status,
            "message": "结果文件不存在"
        }

    with open(result_file) as f:
        result = json.load(f)

    return result

@router.get("/config-schema")
async def get_config_schema():
    """
    获取配置Schema

    调用 CLI 的 --schema 接口
    """
    result = subprocess.run([
        "python",
        "backend/scripts/backtest_service.py",
        "--schema"
    ], capture_output=True, text=True)

    return json.loads(result.stdout)
```

### 5.3 任务管理器

**位置：** `backend/core/task_manager.py`

```python
"""
任务管理器

负责：
- 任务CRUD操作
- 状态管理
- 数据库交互
"""

from pathlib import Path
from datetime import datetime
from typing import Optional, Dict, Any
from sqlalchemy.orm import Session

class TaskManager:
    def create_task(
        self,
        db: Session,
        task_id: str,
        name: str,
        config: Dict[str, Any],
        workdir: str
    ) -> Any:
        """创建任务记录"""
        from backend.models.task import TaskDB

        task = TaskDB(
            task_id=task_id,
            name=name,
            status="pending",
            config=json.dumps(config),
            workdir=workdir,
            created_at=datetime.now()
        )

        db.add(task)
        db.commit()

        return task

    def update_task(
        self,
        db: Session,
        task_id: str,
        result: Dict[str, Any]
    ):
        """更新任务状态"""
        task = db.query(TaskDB).filter(
            TaskDB.task_id == task_id
        ).first()

        if task:
            task.status = result.get("status", "unknown")
            task.results = json.dumps(result.get("metrics", {}))
            task.completed_at = datetime.now()
            db.commit()

    def get_task(self, db: Session, task_id: str) -> Optional[Any]:
        """获取任务"""
        return db.query(TaskDB).filter(
            TaskDB.task_id == task_id
        ).first()
```

### 5.4 模拟盘关联回测任务

**关联逻辑：** 创建模拟盘时必须指定已完成的回测任务

```python
# backend/api/portfolios.py

@router.post("/strategies")
async def add_portfolio_strategy(
    backtest_task_id: str,
    allocation: float = 1.0,
    db: Session = Depends(get_db)
):
    """
    添加模拟盘策略

    必须指定一个已完成的回测任务，回测任务中包含：
    - 模型路径
    - 策略配置
    - 所有必要信息
    """
    # 1. 验证回测任务存在且已完成
    from backend.models.task import TaskDB

    backtest_task = db.query(TaskDB).filter(
        TaskDB.task_id == backtest_task_id
    ).first()

    if not backtest_task:
        raise HTTPException(status_code=404, detail="回测任务不存在")

    if backtest_task.status != "completed":
        raise HTTPException(status_code=400, detail="回测任务未完成，无法创建模拟盘")

    # 2. 读取回测结果获取模型路径
    result_file = Path(backtest_task.workdir) / "result.json"
    with open(result_file) as f:
        backtest_result = json.load(f)

    model_path = backtest_result["outputs"]["model"]

    # 3. 创建模拟盘策略
    from backend.models.portfolio import PortfolioStrategyDB
    import uuid

    strategy = PortfolioStrategyDB(
        id=str(uuid.uuid4()),
        backtest_task_id=backtest_task_id,
        name=f"Portfolio_{backtest_task_id[:8]}",
        allocation=allocation,
        status="active",
        model_path=model_path,  # 保存模型路径
        config=backtest_task.config,  # 保存策略配置
        initial_capital=1000000,
        added_at=datetime.now()
    )

    db.add(strategy)
    db.commit()

    return {"strategy_id": strategy.id, "message": "模拟盘策略已创建"}
```

**推理服务调用：**

```python
# backend/api/portfolios.py

@router.post("/trade")
async def trigger_daily_trade(db: Session = Depends(get_db)):
    """触发所有活跃模拟盘的每日交易"""
    from backend.models.portfolio import PortfolioStrategyDB

    strategies = db.query(PortfolioStrategyDB).filter(
        PortfolioStrategyDB.status == "active"
    ).all()

    results = []
    for strategy in strategies:
        # 从数据库读取模型路径（不需要用户指定）
        config = {
            "task_id": f"inference_{datetime.now().strftime('%Y%m%d_%H%M%S')}",
            "workdir": f"./data/tasks/{strategy.id}",
            "parameters": {
                "model_path": strategy.model_path,  # 使用保存的模型路径
                "date": datetime.now().strftime("%Y-%m-%d"),
                "topk": 50,
                "n_drop": 5
            }
        }

        # 调用推理CLI
        result = subprocess.run([
            "python",
            "backend/scripts/inference_service.py",
            "--config", json.dumps(config)
        ], capture_output=True, text=True)

        results.append(json.loads(result.stdout))

    return {
        "executed_at": datetime.now().isoformat(),
        "count": len(strategies),
        "results": results
    }
```

---

## 6. API端点设计

### 6.1 策略管理API

```python
# 基础路径: /api/v1/strategies

GET    /                        # 获取策略列表
GET    /{id}                     # 获取策略详情
POST   /                         # 创建新策略
PUT    /{id}                     # 更新策略
DELETE /{id}                     # 删除策略
GET    /{id}/backtests           # 获取策略的回测列表
GET    /{id}/simulations         # 获取策略的模拟交易列表
GET    /factors                  # 获取可用因子列表
GET    /model-schema             # 获取模型参数Schema
```

### 6.2 回测任务API

```python
# 基础路径: /api/v1/backtests

GET    /                         # 获取回测任务列表
GET    /{id}                     # 获取回测任务详情
POST   /                         # 创建新回测任务
DELETE /{id}                     # 删除回测任务
GET    /{id}/status              # 获取任务状态
GET    /{id}/result              # 获取回测结果
GET    /{id}/chart-data          # 获取累积收益曲线数据
GET    /{id}/download/{file_type} # 下载结果文件
GET    /config-schema            # 获取回测配置Schema
```

### 6.3 模拟交易API

```python
# 基础路径: /api/v1/simulations

GET    /                         # 获取模拟交易列表
GET    /{id}                     # 获取模拟交易详情
POST   /                         # 创建新模拟交易
PUT    /{id}/status              # 更新状态（启动/暂停/停止）
DELETE /{id}                     # 删除模拟交易
GET    /{id}/trades               # 获取交易记录
GET    /{id}/positions            # 获取当前持仓
GET    /{id}/nav                  # 获取净值曲线
GET    /config-schema            # 获取模拟交易配置Schema
```

### 6.4 系统配置API

```python
# 基础路径: /api/v1/config

GET    /schemas/backtest         # 获取回测Schema
GET    /schemas/portfolio        # 获取模拟盘Schema
GET    /instruments              # 获取可用股票池
GET    /benchmarks               # 获取可用基准指数
```

---

## 7. 定时任务规范

### 7.1 模拟盘定时执行

**触发方式：** 系统cron → 调用API → API调用CLI

**crontab配置：**
```bash
# 每个交易日收盘后执行（15:30）
30 15 * * 1-5 curl -X POST http://localhost:8000/api/v1/portfolios/trade
```

**API端点实现：**
```python
# backend/api/portfolios.py

@router.post("/trade")
async def trigger_daily_trade(db: Session = Depends(get_db)):
    """
    触发所有活跃模拟盘的每日交易

    由系统cron调用，执行流程：
    1. 获取所有status='active'的模拟盘策略
    2. 为每个策略调用inference_service.py CLI
    3. 保存交易信号和持仓更新
    """
    from backend.models.portfolio import PortfolioStrategyDB

    # 获取活跃策略
    strategies = db.query(PortfolioStrategyDB).filter(
        PortfolioStrategyDB.status == "active"
    ).all()

    results = []
    for strategy in strategies:
        # 准备配置
        config = {
            "task_id": f"inference_{datetime.now().strftime('%Y%m%d_%H%M%S')}",
            "workdir": f"./data/tasks/{strategy.id}",
            "parameters": {
                "model_path": strategy.model_path,
                "date": datetime.now().strftime("%Y-%m-%d"),
                "topk": 50,
                "n_drop": 5
            }
        }

        # 调用推理CLI
        result = subprocess.run([
            "python",
            "backend/scripts/inference_service.py",
            "--config", json.dumps(config)
        ], capture_output=True, text=True)

        results.append(json.loads(result.stdout))

    return {
        "executed_at": datetime.now().isoformat(),
        "count": len(strategies),
        "results": results
    }
```

### 7.2 数据更新定时任务

**crontab配置：**
```bash
# 每天收盘后更新数据（16:00）
0 16 * * 1-5 cd /path/to/quant_platform && python backend/scripts/data_service.py --config '{"action":"update","source":"akshare","target_market":"csi300"}' --output /tmp/data_update_result.json
```

**关键点：**
- ✅ 直接调用CLI脚本（不经过API层）
- ✅ 结果输出到日志文件
- ✅ CLI内部自动检测增量并更新

---

## 8. 前端实现规范

### 7.1 前端技术选型

| 模块 | 推荐方案 | 备选方案 |
|------|----------|----------|
| 框架 | React 18 | Vue 3.4 |
| 语言 | TypeScript | TypeScript |
| UI组件库 | Ant Design 5.x | Element Plus |
| 状态管理 | Zustand / Jotai | Pinia |
| 数据请求 | TanStack Query | VueUse + axios |
| 图表 | ECharts | ECharts |
| 构建 | Vite | Vite |

### 8.2 目录结构

```
frontend/
├── src/
│   ├── components/          # 通用组件
│   │   ├── DynamicForm.tsx # 动态表单（基于Schema）
│   │   ├── TaskMonitor.tsx # 任务监控（轮询状态）
│   │   └── ChartPanel.tsx  # 图表面板
│   ├── pages/              # 页面
│   │   ├── BacktestList.tsx
│   │   ├── BacktestCreate.tsx
│   │   ├── BacktestResult.tsx
│   │   ├── PortfolioList.tsx
│   │   └── Dashboard.tsx
│   ├── services/           # API服务
│   │   ├── api.ts         # API客户端
│   │   ├── backtest.ts    # 回测API
│   │   └── portfolio.ts   # 模拟盘API
│   ├── stores/            # 状态管理
│   ├── types/             # 类型定义
│   │   ├── backtest.ts
│   │   └── portfolio.ts
│   └── App.tsx
├── package.json
└── vite.config.ts
```

### 8.3 动态表单组件

```typescript
// src/components/DynamicForm.tsx

import React from 'react';
import { Form, Input, Select, DatePicker, InputNumber } from 'antd';
import type { FormInstance } from 'antd';

interface JsonSchema {
  type: string;
  properties: Record<string, SchemaField>;
  required?: string[];
}

interface SchemaField {
  type: string;
  title: string;
  description?: string;
  enum?: string[];
  minimum?: number;
  maximum?: number;
  default?: any;
}

interface DynamicFormProps {
  schema: JsonSchema;
  onSubmit: (values: any) => void;
  form?: FormInstance;
}

export const DynamicForm: React.FC<DynamicFormProps> = ({
  schema,
  onSubmit,
  form
}) => {
  const renderField = (name: string, field: SchemaField) => {
    const isRequired = schema.required?.includes(name);

    const commonProps = {
      name,
      label: field.title,
      help: field.description,
      rules: isRequired ? [{ required: true }] : undefined,
    };

    switch (field.type) {
      case 'string':
        if (field.enum) {
          return (
            <Form.Select {...commonProps} options={field.enum.map(v => ({ label: v, value: v }))} />
          );
        }
        return <Form.Input {...commonProps} />;

      case 'number':
      case 'integer':
        return (
          <Form.InputNumber
            {...commonProps}
            min={field.minimum}
            max={field.maximum}
          />
        );

      default:
        return null;
    }
  };

  return (
    <Form form={form} onFinish={onSubmit} layout="vertical">
      {Object.entries(schema.properties).map(([name, field]) =>
        renderField(name, field)
      )}
    </Form>
  );
};
```

### 8.4 任务监控组件（无轮询）

**重要变更：** 前端不再自动轮询，用户手动刷新

```typescript
// src/components/TaskMonitor.tsx

import { useState } from 'react';
import { Button, Alert, Progress, Spin } from 'antd';

interface TaskMonitorProps {
  taskId: string;
}

export const TaskMonitor: React.FC<TaskMonitorProps> = ({ taskId }) => {
  const [task, setTask] = useState<any>(null);
  const [loading, setLoading] = useState(false);

  const checkStatus = async () => {
    setLoading(true);
    const response = await fetch(`/api/backtests/${taskId}/status`);
    const data = await response.json();
    setTask(data);
    setLoading(false);
  };

  if (!task) {
    // 首次加载
    checkStatus();
    return <Spin />;
  }

  if (task.status === 'running') {
    return (
      <div>
        <Alert type="info" message="任务执行中（预计最多4小时）" />
        <Button onClick={checkStatus} loading={loading}>
          刷新状态
        </Button>
      </div>
    );
  }

  if (task.status === 'failed') {
    return (
      <div>
        <Alert type="error" message={task.error || '任务失败'} />
        <Button onClick={checkStatus}>重试</Button>
      </div>
    );
  }

  // status === 'completed'
  return (
    <div>
      <Alert type="success" message="任务完成" />
      <Button onClick={() => window.location.href = `/backtests/${taskId}`}>
        查看详情
      </Button>
    </div>
  );
};
```

**关键点：**
- ❌ 不使用 `useQuery` 的 `refetchInterval`
- ✅ 用户手动点击"刷新状态"按钮
- ✅ 首次加载自动调用一次
- ✅ 任务完成后提供"查看详情"链接

```typescript
// src/components/TaskMonitor.tsx

import { useEffect } from 'react';
import { useQuery } from '@tanstack/react-query';
import { Progress, Alert } from 'antd';

interface TaskMonitorProps {
  taskId: string;
  onComplete?: () => void;
}

export const TaskMonitor: React.FC<TaskMonitorProps> = ({
  taskId,
  onComplete
}) => {
  const { data, isLoading } = useQuery({
    queryKey: ['task', taskId],
    queryFn: () => fetch(`/api/backtests/${taskId}/status`).then(r => r.json()),
    refetchInterval: (data) => {
      // 完成后停止轮询
      return data?.status === 'completed' ? false : 2000;
    }
  });

  useEffect(() => {
    if (data?.status === 'completed' && onComplete) {
      onComplete();
    }
  }, [data?.status, onComplete]);

  if (isLoading) return <div>加载中...</div>;

  if (data?.status === 'failed') {
    return <Alert type="error" message={data.error_message} />;
  }

  if (data?.status === 'running') {
    return (
      <div>
        <div>执行中...</div>
        <Progress percent={data.progress || 0} />
      </div>
    );
  }

  return <Alert type="success" message="执行完成" />;
};
```

---

## 9. 部署结构

### 9.1 目录结构

```
quant_platform/
├── frontend/              # Vue3/React前端项目
│   ├── src/
│   ├── package.json
│   └── vite.config.ts
├── backend/
│   ├── api/               # FastAPI路由
│   │   ├── main.py        # 应用入口
│   │   ├── backtests.py   # 回测API
│   │   ├── portfolios.py  # 模拟盘API
│   │   ├── market.py      # 市场数据API
│   │   └── strategies.py  # 策略API
│   ├── models/            # SQLAlchemy模型
│   │   ├── task.py
│   │   ├── strategy.py
│   │   └── portfolio.py
│   ├── core/              # 核心逻辑
│   │   ├── task_manager.py # 任务管理
│   │   ├── concurrent_executor.py # 并发控制（线程池）
│   │   └── config.py      # 配置
│   ├── scripts/           # CLI服务脚本（独立程序，插件式扩展）
│   │   ├── data_service.py        # 数据更新CLI
│   │   ├── market_service.py      # 市场数据查询CLI
│   │   ├── backtest_service.py    # 回测CLI
│   │   ├── inference_service.py   # 推理CLI
│   │   └── strategies/            # 策略插件目录
│   │       ├── pointwise_service.py   # Pointwise策略
│   │       ├── pairwise_service.py    # Pairwise策略
│   │       └── listwise_service.py    # Listwise策略（未来）
│   ├── db/                # SQLite数据库
│   ├── data/              # 数据目录
│   │   └── tasks/         # 任务工作目录（用户手动删除）
│   ├── dependencies.py
│   └── main.py
└── README.md
```

### 9.2 启动命令

```bash
# 初始化数据库
python backend/init_db.py

# 启动API服务
cd backend
uvicorn main:app --reload --port 8000

# 启动前端开发服务器
cd frontend
npm run dev

# 数据更新（定时任务）
crontab -e
0 18 * * * cd /path/to/quant_platform && python backend/scripts/data_service.py --config config/update.json
```

---

## 10. 实现路线图

### 第一阶段：基础架构（Week 1-2）

**后端任务：**
- [ ] 搭建 FastAPI 项目结构
- [ ] 实现 SQLite 数据库模型
- [ ] 实现 TaskManager 任务管理器
- [ ] 创建回测服务 CLI 脚本框架
- [ ] 实现调用 CLI 的基础逻辑

**前端任务：**
- [ ] 初始化 React + TypeScript 项目
- [ ] 配置 Ant Design
- [ ] 实现 API 客户端
- [ ] 实现基础布局和路由

### 第二阶段：回测功能（Week 3-5）

**后端任务：**
- [ ] 完善 backtest_service.py CLI
- [ ] 实现回测 API 端点
- [ ] 实现配置 Schema 接口
- [ ] 实现任务状态轮询

**前端任务：**
- [ ] 实现回测列表页面
- [ ] 实现动态表单组件
- [ ] 实现回测配置页面
- [ ] 实现结果展示页面
- [ ] 集成 ECharts 图表

### 第三阶段：模拟盘功能（Week 6-7）

**后端任务：**
- [ ] 实现 inference_service.py CLI
- [ ] 实现模拟盘 API 端点
- [ ] 实现交易信号生成

**前端任务：**
- [ ] 实现模拟盘列表页面
- [ ] 实现持仓展示页面
- [ ] 实现交易历史页面

### 第四阶段：优化与部署（Week 8）

**任务：**
- [ ] 错误处理完善
- [ ] 日志记录规范
- [ ] 性能优化
- [ ] Docker 容器化
- [ ] 文档完善

---

## 11. 实施说明

### 11.1 现有Backend处理

**重要：** 重构开始时，现有 `backend/` 文件夹将被完全删除。

**保留的参考文件：**
- `pairwise_rank.py` → 作为新策略插件实现的参考
- `pair_wise_ranking/` → 作为重构 `pairwise_service.py` 的逻辑参考
- 其他所有 `backend/` 内容将被删除

**重构策略：**
1. 删除 `backend/` 文件夹
2. 按照本文档重新创建目录结构
3. 从现有脚本中提取核心逻辑，重构为标准CLI接口
4. 重新实现API层（仅调用CLI、维护数据库、提供REST接口）

### 11.2 数据迁移

**需要迁移的数据：**
- SQLite数据库：重新创建表结构，从旧数据库导出元数据
- Qlib数据：保留在 `~/.qlib/qlib_data/`，无需迁移
- 历史回测结果：保留在旧 `backend/data/`，新系统无法访问（或手动迁移）

### 11.3 配置兼容性

**不兼容：**
- 旧API端点与新端点不同
- 旧前端无法直接使用新后端

**迁移建议：**
- 旧系统继续运行，新系统并行开发
- 逐步将策略和配置迁移到新系统
- 完成验证后切换

---

## 12. 与原计划的变更说明

| 方面 | 原计划 | 修订后 | 理由 |
|------|--------|--------|------|
| 后端职责 | 调用脚本、维护数据库、提供API | **保持不变** | 核心理念不变 |
| 服务脚本 | 独立CLI程序 | **保持不变** | 核心理念不变 |
| 数据交互 | 本地文件 | **保持不变** | 核心理念不变 |
| Schema接口 | 新增 | **添加明确规范** | 支持动态表单 |
| 任务管理 | 未明确 | **细化实现** | 添加TaskManager |
| 文件路径约定 | 未明确 | **添加规范** | 确保一致性 |
| 状态反馈 | 前端轮询 | **手动刷新** | 保证绝对隔离 |
| Qlib数据目录 | 项目内 | **~/.qlib/** | 符合Qlib规范 |
| 任务删除 | 未明确 | **同步删除目录** | 用户操作一致性 |
| 策略发现 | 未明确 | **硬编码映射** | 简单可控 |
| API缓存 | 未明确 | **添加内存缓存** | 性能优化 |
| 配置模板 | 保存的实例 | **Schema定义** | 统一概念 |

---

*文档版本: v2.0*
*基于: trade_platform_refactor.md*
*修订日期: 2024-01-15*

---

## 附录：完整实施 TODO List

### 第一阶段：基础架构搭建（Week 1）

#### 目标
建立完整的目录结构和基础框架，确保所有组件能正常启动和通信。

#### 任务清单

**后端任务：**
- [ ] 删除现有 `backend/` 文件夹
- [ ] 创建新目录结构：
  ```
  backend/
  ├── api/
  ├── models/
  ├── core/
  ├── scripts/
  │   └── strategies/
  ├── db/
  └── data/
  ```
- [ ] 实现 SQLAlchemy 数据库模型：
  - [ ] `models/task.py` (TaskDB)
  - [ ] `models/backtest.py` (BacktestDB)
  - [ ] `models/portfolio.py` (PortfolioStrategyDB, PortfolioPositionDB, TradeRecordDB)
- [ ] 实现核心组件：
  - [ ] `core/task_manager.py`
  - [ ] `core/concurrent_executor.py`
  - [ ] `core/cache_manager.py`
  - [ ] `dependencies.py`
- [ ] 创建 FastAPI 应用入口：
  - [ ] `main.py`
  - [ ] 配置 CORS
  - [ ] 配置静态文件（API文档）
- [ ] 初始化数据库脚本：
  - [ ] `init_db.py`
  - [ ] 创建所有表
  - [ ] 插入测试数据

**验证任务：**
- [ ] 运行 `uvicorn main:app` 能正常启动
- [ ] 访问 `http://localhost:8000/docs` 能看到 API 文档
- [ ] 数据库文件能正常创建

#### 验收标准

| 检查项 | 标准 | 测试方法 |
|--------|------|----------|
| API服务启动 | 无报错，8000端口监听 | `curl http://localhost:8000/api/v1/config/schemas` |
| 数据库初始化 | SQLite文件存在，表结构正确 | `sqlite3 db/quant_platform.db ".schema"` |
| 依赖注入 | get_db能正常工作 | 创建测试API端点验证 |

---

### 第二阶段：数据服务实现（Week 1-2）

#### 目标
实现数据更新CLI脚本，能够自动更新Qlib市场数据到最新交易日。

#### 任务清单

**CLI开发：**
- [ ] `scripts/data_service.py`
  - [ ] 实现 `--schema` 接口
  - [ ] 实现 `update_data()` 函数
    - [ ] 检测当前数据最新日期
    - [ ] 获取市场最新交易日
    - [ ] 增量更新数据
    - [ ] 支持csi300/csi500/csi800
  - [ ] 实现 `get_data_status()` 函数
  - [ ] 错误处理和日志记录
- [ ] 单元测试：
  - [ ] 测试schema输出
  - [ ] 测试数据更新流程（使用小数据集）

**API集成：**
- [ ] `api/market.py` 路由（预留，稍后完善）
- [ ] 配置数据更新定时任务（文档说明）

#### 验收标准

| 检查项 | 标准 | 测试方法 |
|--------|------|----------|
| Schema输出 | JSON格式正确，包含所有必需字段 | `python scripts/data_service.py --schema` |
| 数据更新 | 能成功更新到最近交易日 | 手动执行，检查Qlib数据目录 |
| 增量检测 | 只更新缺失数据 | 查看日志中的update_count |
| 状态查询 | 返回正确日期和股票数 | `python scripts/data_service.py --config '{"action":"status"}'` |

---

### 第三阶段：策略插件框架（Week 2）

#### 目标
实现策略插件基类和加载机制，完成第一个Pairwise策略插件。

#### 任务清单

**框架开发：**
- [ ] `scripts/strategies/base_strategy.py`
  - [ ] 定义 `BaseStrategy` 抽象类
  - [ ] 定义抽象方法：`get_schema()`, `train()`, `predict()`, `backtest()`
- [ ] `scripts/backtest_service.py`
  - [ ] 实现 `load_strategy()` 函数（硬编码映射）
  - [ ] 实现 `--schema` 接口（合并所有策略的Schema）
  - [ ] 实现 `run_backtest()` 函数
    - [ ] 初始化Qlib
    - [ ] 保存配置文件（可复现）
    - [ ] 调用策略插件的train()和backtest()
    - [ ] 错误处理和traceback记录

**Pairwise策略插件：**
- [ ] `scripts/strategies/pairwise_service.py`
  - [ ] 继承 `BaseStrategy`
  - [ ] 实现 `get_schema()`
  - [ ] 实现 `train()` - 复用现有 `pair_wise_ranking` 逻辑
  - [ ] 实现 `predict()`
  - [ ] 实现 `backtest()` - 使用Qlib executor
- [ ] 确保模型保存到 `{workdir}/models/model.pkl`

**API开发：**
- [ ] `api/backtests.py`
  - [ ] `POST /create` - 创建回测任务
  - [ ] `GET /{task_id}/status` - 获取状态
  - [ ] `GET /{task_id}/result` - 获取结果
  - [ ] `DELETE /{task_id}` - 删除任务（含目录）
  - [ ] `GET /config-schema` - 获取Schema

#### 验收标准

| 检查项 | 标准 | 测试方法 |
|--------|------|----------|
| 策略加载 | 能正确加载pairwise策略 | 单元测试验证load_strategy() |
| CLI执行 | 能完成完整回测流程 | 手动执行最小配置回测 |
| 结果文件 | result.json包含所有必需字段 | 检查输出文件 |
| 模型文件 | model.pkl存在于正确位置 | `ls data/tasks/{task_id}/models/` |
| 配置可复现 | config.json包含完整输入参数 | 检查配置文件 |
| 错误处理 | 异常时返回traceback和config | 故意传入错误参数验证 |
| API调用 | 通过API能成功创建和查询任务 | 使用curl测试所有端点 |
| 删除功能 | 删除后数据库和文件都不存在 | 创建任务后删除，验证清理完整 |

---

### 第四阶段：市场数据API实现（Week 2-3）

#### 目标
实现市场数据查询CLI和API，提供K线、实时行情、股票池查询功能。

#### 任务清单

**CLI开发：**
- [ ] `scripts/market_service.py`
  - [ ] `get_kline()` - K线数据（使用Qlib）
  - [ ] `get_realtime()` - 实时行情（使用akshare）
  - [ ] `get_instruments()` - 股票池列表（使用Qlib）
  - [ ] 错误处理

**API开发：**
- [ ] `api/market.py`
  - [ ] `GET /kline/{instrument}` - K线接口
  - [ ] `GET /realtime/{instruments}` - 实时行情接口
  - [ ] `GET /instruments` - 股票池接口
  - [ ] 集成CacheManager（5分钟TTL）

**测试：**
- [ ] 验证K线数据格式
- [ ] 验证缓存是否生效
- [ ] 验证实时行情数据源

#### 验收标准

| 检查项 | 标准 | 测试方法 |
|--------|------|----------|
| K线数据 | 返回正确OHLCV数据 | 调用API验证数据格式 |
| 实时行情 | 返回当前价格和涨跌幅 | 在交易时间测试 |
| 股票池 | 返回正确股票列表 | 验证csi300股票数量 |
| 缓存生效 | 第二次调用速度快10倍以上 | 计时测试 |
| 缓存过期 | 5分钟后重新获取数据 | 等待5分钟后验证 |

---

### 第五阶段：模拟盘功能实现（Week 3-4）

#### 目标
实现模拟盘创建、交易信号生成、持仓管理功能。

#### 任务清单

**CLI开发：**
- [ ] `scripts/inference_service.py`
  - [ ] 实现 `run_inference()` 函数
  - [ ] 加载指定模型文件
  - [ ] 生成预测分数
  - [ ] 转换为交易信号（buy/sell/hold）
  - [ ] 输出signals.json

**API开发：**
- [ ] `api/portfolios.py`
  - [ ] `POST /strategies` - 创建模拟盘（指定回测任务）
  - [ ] `GET /strategies` - 获取策略列表
  - [ ] `DELETE /strategies/{id}` - 删除策略
  - [ ] `POST /trade` - 触发每日交易
  - [ ] `GET /positions` - 获取当前持仓
  - [ ] `GET /history` - 获取交易历史
- [ ] 实现回测任务关联逻辑
  - [ ] 验证回测任务已完成
  - [ ] 提取model_path
  - [ ] 保存到模拟盘策略

**数据库模型：**
- [ ] `models/portfolio.py`
  - [ ] PortfolioStrategyDB（增加model_path字段）
  - [ ] PortfolioPositionDB
  - [ ] TradeRecordDB

**定时任务：**
- [ ] 编写crontab配置示例
- [ ] 文档说明定时触发方式

#### 验收标准

| 检查项 | 标准 | 测试方法 |
|--------|------|----------|
| 创建模拟盘 | 必须指定已完成回测任务 | 尝试指定running任务应失败 |
| 模型关联 | 推理时使用正确模型 | 手动触发trade，检查日志 |
| 信号生成 | 返回buy/sell/hold信号 | 验证输出格式 |
| 持仓记录 | 正确记录买入卖出 | 查询positions API |
| 关联失败 | 回测任务不存在时返回404 | API测试 |
| 定时执行 | cron能成功调用API | 手动触发curl测试 |

---

### 第六阶段：前端基础开发（Week 4-5）

#### 目标
搭建React前端框架，实现基础布局和API通信。

#### 任务清单

**项目初始化：**
- [ ] 使用 Vite 创建 React + TypeScript 项目
- [ ] 安装依赖：
  - [ ] antd
  - [ ] @tanstack/react-query
  - [ ] zustand
  - [ ] echarts
  - [ ] axios
- [ ] 配置目录结构
- [ ] 配置环境变量（.env.development, .env.production）

**基础组件：**
- [ ] `src/components/DynamicForm.tsx` - 动态表单
- [ ] `src/components/TaskMonitor.tsx` - 任务监控（手动刷新）
- [ ] `src/components/ChartPanel.tsx` - 图表面板

**API服务层：**
- [ ] `src/services/api.ts` - Axios封装
- [ ] `src/services/backtest.ts` - 回测API
- [ ] `src/services/portfolio.ts` - 模拟盘API
- [ ] `src/services/market.ts` - 市场数据API

**路由和布局：**
- [ ] 使用 react-router-dom
- [ ] 主布局（Header, Sidebar, Content）
- [ ] 页面路由配置

**类型定义：**
- [ ] 从后端导出类型定义
- [ ] `src/types/backtest.ts`
- [ ] `src/types/portfolio.ts`
- [ ] `src/types/market.ts`

#### 验收标准

| 检查项 | 标准 | 测试方法 |
|--------|------|----------|
| 项目启动 | npm run dev 无报错 | 本地开发服务器运行 |
| API调用 | 能成功请求后端接口 | 浏览器Network验证 |
| 动态表单 | 能根据Schema渲染表单 | 提供Schema测试数据 |
| 路由导航 | 页面跳转正常 | 点击菜单验证 |

---

### 第七阶段：回测管理页面（Week 5-6）

#### 目标
实现回测任务列表、创建、结果展示页面。

#### 任务清单

**页面开发：**
- [ ] `src/pages/backtest/BacktestList.tsx`
  - [ ] 任务列表展示
  - [ ] 状态筛选（pending/running/completed/failed）
  - [ ] 删除按钮
  - [ ] 创建按钮
- [ ] `src/pages/backtest/BacktestCreate.tsx`
  - [ ] 使用DynamicForm渲染配置表单
  - [ ] 策略类型选择（pointwise/pairwise）
  - [ ] 表单验证
  - [ ] 提交创建
- [ ] `src/pages/backtest/BacktestResult.tsx`
  - [ ] 结果指标展示（IC指标、绩效指标）
  - [ ] 图表展示（ECharts）
  - [ ] 下载功能

**交互实现：**
- [ ] 手动刷新状态（不使用轮询）
- [ ] 任务完成后显示"查看详情"按钮
- [ ] 错误提示

**图表集成：**
- [ ] 累积收益曲线图
- [ ] IC分布图
- [ ] 回撤图

#### 验收标准

| 检查项 | 标准 | 测试方法 |
|--------|------|----------|
| 创建任务 | 表单提交后任务创建成功 | 创建后检查任务列表 |
| 状态显示 | 正确显示running/completed/failed | 观察状态变化 |
| 手动刷新 | 点击刷新后状态更新 | 计时验证 |
| 结果展示 | 指标和图表正确显示 | 创建测试回测验证 |
| 删除功能 | 删除后任务消失且目录被删除 | 检查文件系统 |

---

### 第八阶段：模拟盘页面（Week 6-7）

#### 目标
实现模拟盘策略管理、持仓展示、交易历史页面。

#### 任务清单

**页面开发：**
- [ ] `src/pages/portfolio/PortfolioList.tsx`
  - [ ] 策略列表
  - [ ] 状态显示
  - [ ] 创建按钮（选择回测任务）
- [ ] `src/pages/portfolio/PortfolioDetail.tsx`
  - [ ] 策略详情
  - [ ] 当前持仓表格
  - [ ] 净值曲线图
  - [ ] 触发交易按钮
- [ ] `src/pages/portfolio/TradeHistory.tsx`
  - [ ] 交易记录列表
  - [ ] 筛选功能

**交互实现：**
- [ ] 选择回测任务创建模拟盘
- [ ] 手动触发交易
- [ ] 查看交易信号

#### 验收标准

| 检查项 | 标准 | 测试方法 |
|--------|------|----------|
| 创建模拟盘 | 选择已完成回测后创建成功 | 测试关联逻辑 |
| 策略列表 | 显示所有策略及状态 | 检查数据库 |
| 持仓展示 | 正确显示当前持仓和盈亏 | 触发交易后验证 |
| 交易触发 | 点击后执行推理并更新持仓 | 检查API日志 |
| 交易历史 | 显示所有交易记录 | 验证数据完整性 |

---

### 第九阶段：市场数据页面（Week 7）

#### 目标
实现K线图展示、股票池管理页面。

#### 任务清单

**页面开发：**
- [ ] `src/pages/market/KlineView.tsx`
  - [ ] 股票代码输入
  - [ ] K线图（ECharts candlestick）
  - [ ] 时间范围选择
- [ ] `src/pages/market/InstrumentsList.tsx`
  - [ ] 股票池列表（csi300/csi500）
  - [ ] 搜索功能
- [ ] `src/pages/market/RealtimeQuote.tsx`
  - [ ] 实时行情展示
  - [ ] 自动刷新（可选）

#### 验收标准

| 检查项 | 标准 | 测试方法 |
|--------|------|----------|
| K线图 | 正确显示OHLCV数据 | 输入股票代码验证 |
| 股票池 | 显示正确股票列表 | 验证股票数量 |
| 实时行情 | 显示当前价格 | 交易时间测试 |

---

### 第十阶段：仪表盘页面（Week 7-8）

#### 目标
实现仪表盘页面，汇总显示关键信息。

#### 任务清单

**页面开发：**
- [ ] `src/pages/dashboard/Dashboard.tsx`
  - [ ] 运行中任务统计
  - [ ] 最近回测结果
  - [ ] 模拟盘汇总
  - [ ] 快捷操作按钮

#### 验收标准

| 检查项 | 标准 | 测试方法 |
|--------|------|----------|
| 统计数据 | 数值正确 | 手动计算对比 |
| 快捷操作 | 能快速创建任务/查看详情 | 点击测试 |

---

### 第十一阶段：集成测试（Week 8）

#### 目标
端到端测试所有功能，修复缺陷。

#### 任务清单

**功能测试：**
- [ ] 完整回测流程测试
  - [ ] 创建回测 → 等待完成 → 查看结果 → 删除任务
- [ ] 完整模拟盘流程测试
  - [ ] 完成回测 → 创建模拟盘 → 触发交易 → 查看持仓
- [ ] 数据更新测试
  - [ ] 执行data_service.py → 验证数据更新
- [ ] 并发测试
  - [ ] 同时创建3个回测任务 → 验证并发控制
  - [ ] 验证结果正确性

**性能测试：**
- [ ] 大量任务列表查询性能
- [ ] 图表渲染性能
- [ ] API缓存效果验证

**兼容性测试：**
- [ ] Chrome浏览器测试
- [ ] Firefox浏览器测试
- [ ] Safari浏览器测试（Mac）

**缺陷修复：**
- [ ] 记录所有发现的bug
- [ ] 优先级排序
- [ ] 逐个修复

#### 验收标准

| 检查项 | 标准 | 测试方法 |
|--------|------|----------|
| 回测流程 | 无阻塞，能完成全流程 | 手动执行完整流程 |
| 模拟盘流程 | 无阻塞，能完成全流程 | 手动执行完整流程 |
| 并发控制 | 最多3个任务同时运行 | 创建5个任务验证 |
| 错误处理 | 所有异常都有友好提示 | 故意触发错误验证 |
| 性能 | 页面加载时间<2秒 | 计时测试 |
| 浏览器兼容 | 三大浏览器功能正常 | 逐一测试 |

---

### 第十二阶段：部署与文档（Week 8-9）

#### 目标
完成生产部署配置和用户文档。

#### 任务清单

**部署配置：**
- [ ] 编写 `Dockerfile`（后端）
- [ ] 编写 `Dockerfile`（前端）
- [ ] 编写 `docker-compose.yml`
- [ ] 配置Nginx反向代理
- [ ] 配置SSL证书（Let's Encrypt）

**文档编写：**
- [ ] 用户手册
  - [ ] 安装指南
  - [ ] 快速开始
  - [ ] 功能说明
- [ ] 开发文档
  - [ ] 添加新策略指南
  - [ ] API文档
- [ ] 运维文档
  - [ ] 部署指南
  - [ ] 故障排查
  - [ ] 数据备份

**部署验证：**
- [ ] 本地Docker环境测试
- [ ] 生产环境部署
- [ ] 域名配置
- [ ] HTTPS配置验证

#### 验收标准

| 检查项 | 标准 | 测试方法 |
|--------|------|----------|
| Docker构建 | 镜像构建成功 | `docker build` |
| docker-compose | 服务启动成功 | `docker-compose up` |
| HTTPS访问 | 能通过https访问 | 浏览器测试 |
| 文档完整性 | 覆盖所有功能点 | 逐项检查 |

---

### 第十三阶段：清理与收尾（Week 9）

#### 目标
清理临时文件，归档代码，正式发布。

#### 任务清单

**清理工作：**
- [ ] 删除所有TODO注释
- [ ] 删除调试代码
- [ ] 优化日志输出
- [ ] 代码格式化（black, prettier）

**归档工作：**
- [ ] 创建Git标签 v1.0.0
- [ ] 编写Release Notes
- [ ] 归档旧代码（可选）

**发布：**
- [ ] 正式发布通知
- [ ] 更新README

---

## 测试标准总览

### 单元测试标准

| 组件 | 覆盖率要求 | 必测场景 |
|------|------------|----------|
| CLI脚本 | 70% | 正常流程、异常处理、参数验证 |
| API端点 | 80% | 正常请求、错误请求、权限验证 |
| 数据模型 | 90% | CRUD操作、约束验证 |
| 工具函数 | 90% | 边界条件、异常情况 |

### 集成测试标准

| 场景 | 验证点 |
|------|--------|
| 创建回测任务 | API → CLI → 文件 → 数据库 |
| 查询任务状态 | subprocess检测 → 文件读取 → API响应 |
| 删除任务 | API删除 → 数据库删除 → 文件删除 |
| 创建模拟盘 | 回测验证 → 模型提取 → 策略创建 |
| 触发交易 | API调用 → CLI推理 → 持仓更新 |

### 性能测试标准

| 指标 | 目标 | 测试方法 |
|------|------|----------|
| API响应时间 | < 500ms (p95) | JMeter压力测试 |
| 前端首屏加载 | < 2s | Lighthouse测试 |
| CLI执行时间 | < 4小时 | 实际回测执行 |
| 并发处理 | 支持3个并发任务 | 同时创建5个任务 |

### 用户验收测试（UAT）标准

| 功能 | 验收标准 | 测试方法 |
|------|----------|----------|
| 回测创建 | 能成功创建并完成回测 | 用户实际操作 |
| 结果查看 | 能查看完整结果和图表 | 用户实际操作 |
| 模拟盘 | 能创建策略并触发交易 | 用户实际操作 |
| 数据更新 | 能更新到最新交易日 | 运行CLI验证 |
| 错误处理 | 所有错误有清晰提示 | 触发各种错误 |

---

## 风险与缓解措施

| 风险 | 影响 | 概率 | 缓解措施 |
|------|------|------|----------|
| Qlib重复初始化性能差 | CLI执行慢 | 高 | 已接受，未来优化 |
| 并发控制失效 | 资源耗尽 | 中 | 线程池大小限制，监控 |
| 磁盘空间不足 | 任务失败 | 中 | 定期清理机制，用户手动删除 |
| 策略插件兼容性 | 新策略无法加载 | 低 | 严格基类定义，单元测试 |
| 数据更新失败 | 数据过期 | 低 | akshare备选方案，监控告警 |
