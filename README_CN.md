# AI Auto Trading: 基于多大语言模型认知决策的自主量化交易系统

[English](README.md) | [简体中文](README_CN.md)

> **基线开源声明（12个月阶段性成果）**  
> 本代码库为历时 12 个月持续研发的基础架构基线开源版本。商用版本与深度生产环境迭代保留于独立私有仓库维护。本项目依据 MIT 协议开源。

**AI Auto Trading (LLM Trader)** 是一个完全解耦、支持多大模型（OpenAI GPT-4o、Google Gemini、DeepSeek、本地 Ollama）认知决策的自主量化实盘/模拟交易系统。通过三层记忆架构、全时区交易感知、高频紧凑 Prompt 压缩与盈透证券（Interactive Brokers）直连接口，赋予 LLM 像专业操盘手一样理解盘面、做出交易决策并自主复盘进化的能力。

---

## ✨ 核心特性

1. **多大模型支持与竞技场 (Multi-LLM Race Arena)**
   - 原生支持 **OpenAI (GPT-4o/o1)**、**Google Gemini (2.0 Flash)**、**DeepSeek (V3/R1)**、**DeepSeek-NVIDIA** 以及本地私有化部署的 **Ollama (Qwen, Llama3)**。
   - 内置 **LLM Race 竞技模式**：相同初始本金、相同标的组合，多模型并行实盘/模拟对决，直观对比不同 LLM 的胜率、夏普比率与交易决策风格。

2. **三层记忆与认知进化系统 (3-Layer Memory Architecture)**
   - **情景记忆 (Episodic Memory)**：完整记录历史成交、建仓点位、持仓时长与盈亏状态。
   - **策略偏好记忆 (Policy Memory)**：根据市场风格与胜负经验，动态固化风险偏好与仓位策略。
   - **反思评估记忆 (Reflection Memory)**：在交易周期或每日收盘后自主触发批判性复盘，沉淀教训并写入交易手记。

3. **紧凑 Prompt 压缩技术 (83% Token 成本降低)**
   - 独创分级 Prompt 机制：高频执行阶段采用结构化微观数据流，降低 83% 的 Prompt Token 开销。
   - 低频反思阶段采用完整宏观上下文推理，兼顾高频执行响应与深度逻辑思考。

4. **全生命周期时间感知与风控 (Time Awareness & Session Guard)**
   - 严格遵循美东交易时钟（盘前 4:00-9:30、常规盘 RTH 9:30-16:00、盘后 16:00-20:00、夜盘 20:00-4:00）。
   - 内置 **EOD 强制平仓熔断 (End-of-Day Flattening)**：收盘前 5 分钟自动禁止开仓并强制清空日内仓位，规避隔夜跳空风险。
   - 数据降级保护 (Data Degradation)：自动检测 1m/5s 数据完整性，数据缺失时触发只读保护。

5. **实盘直连、消息队列接入与账本持久化**
   - 直连盈透证券 IBKR Gateway / TWS，支持限价单、止盈止损挂单、防滑点重试与成交回报对齐。
   - 原生支持接入上游 **Apache Kafka / MQ 消息总线** (`events.bars`) 与 **Redis 状态总线**，同时支持 ClickHouse 时序账本双写，全链路审计每笔决策的 Prompt、Raw LLM 输出与成交回报。

---

## 🏗 系统架构图

```mermaid
flowchart TD
    subgraph MarketData[ 行情与特征流 (Market & MQ) ]
      IB[IBKR Gateway 行情]
      KF[Apache Kafka / MQ 事件流 events.bars]
      YF[Yahoo Finance / 新闻流]
      CH[(ClickHouse 历史时序)]
      RC[(Redis 实时缓存)]
      AG[Realtime Aggregator 1s/5s]
    end

    subgraph LLM_Engine[ LLM 交易大脑 ]
      TA[Time Awareness 时间感知]
      MS[Memory System 三层记忆]
      PB[Compact / Universal Prompt Builder]
      
      subgraph Providers[ 多模型统一路由 ]
        P_OAI[OpenAI gpt-4o]
        P_GEM[Google Gemini 2.0]
        P_DS[DeepSeek V3/R1]
        P_OLL[Local Ollama]
      end
    end

    subgraph Execution[ 执行与风险层 ]
      PM[Portfolio Manager 资产管理]
      PO[Pending Orders Manager 挂单管理]
      RG[Risk Gate 风控熔断]
      EXEC[IBKR Executor 订单执行]
    end

    subgraph Storage[ 审计与账本 ]
      LOG[LLM Logger 结构化日志]
      LEDGER[(ClickHouse Trade Ledger)]
    end

    IB --> AG
    KF --> AG
    YF --> AG
    CH --> AG
    AG --> TA
    RC --> PM
    TA --> PB
    MS --> PB
    PM --> PB
    PB --> Providers
    Providers --> LOG
    Providers --> RG
    RG --> PO
    PO --> EXEC
    EXEC --> IB
    EXEC --> MS
    EXEC --> LEDGER
```

---

## 📂 目录结构

```text
ai-auto-trading/
├── run_single_bot.py          # 单模型自主交易主程序 CLI
├── run_llm_race.py            # 多模型并行交易对决主程序 CLI
├── requirements.txt           # 项目依赖
├── .env.example               # 环境变量配置模板
├── pyproject.toml             # Python 项目构建元数据
├── LICENSE                    # MIT 开源协议
├── README.md                  # 英文项目文档
├── README_CN.md               # 中文项目文档
├── run_all_providers.sh       # 一键启动全模型实盘脚本
├── run_fast_providers.sh      # 启动低延迟模型脚本
├── run_premarket.sh           # 盘前策略执行脚本
└── llm_trader/                # 核心引擎代码包
    ├── llm_trader_bot.py      # LLMTraderBot 核心交易循环
    ├── providers/             # 大模型接口适配层 (OpenAI/Gemini/DeepSeek/Ollama)
    ├── data/                  # 市场数据摄取与预加载
    ├── portfolio/             # 仓位状态追踪与资金核算
    ├── sentiment/             # 财经新闻与情绪分析流水线
    ├── trading/               # 盈透证券订单执行封装
    ├── utils/                 # 时间感知、三层记忆、Prompt 构建器
    └── scripts/               # 辅助分析、ClickHouse 查询工具
```

---

## 🚀 快速上手

### 1. 安装环境
推荐使用 Python 3.10+：
```bash
git clone https://github.com/chanaliuxing/ai-auto-trading.git
cd ai-auto-trading
pip install -r requirements.txt
pip install -e .
```

### 2. 配置环境变量
复制配置文件并填入你拥有的大模型 API Key：
```bash
cp .env.example .env
```
主要配置项：
```env
OPENAI_API_KEY=sk-...
GEMINI_API_KEY=AIzaSy...
DEEPSEEK_API_KEY=sk-...
OLLAMA_BASE_URL=http://localhost:11434

# 交易标的与初始资金
TRADING_SYMBOLS=AAPL,TSLA,NVDA,QQQ
INITIAL_CASH=50000.0
```

### 3. 运行单模型模拟交易 (Dry-Run)
以 Google Gemini 为例，在模拟模式下运行交易机器人：
```bash
python run_single_bot.py --provider gemini --symbols AAPL TSLA NVDA --dry-run
```
启用三层记忆与紧凑 Prompt（大幅降低 Token 消耗）：
```bash
python run_single_bot.py \
  --provider deepseek \
  --symbols NVDA TSLA \
  --enable-v2 \
  --compact-prompt \
  --dry-run
```

### 4. 运行多模型并行竞技 (LLM Race)
让 OpenAI、Gemini 和 DeepSeek 同时在相同标的下运行，看谁的策略收益与风控更优：
```bash
python run_llm_race.py \
  --providers openai gemini deepseek \
  --symbols AAPL NVDA \
  --cash 50000 \
  --interval 60
```

### 5. 启动实盘交易 (Live Trading)
> ⚠️ 实盘交易需要连接运行中的 Interactive Brokers Gateway (默认端口 4002/4001)。建议在充分回测与模拟验证后再连接真实资金。
```bash
python run_single_bot.py \
  --provider deepseek \
  --symbols QQQ SPY \
  --close-at 15:55
```

---

## 🔐 安全与风控边界

1. **模拟模式优先 (Safe by Default)**：未显式指定 `--live` 时，所有订单操作均在内存模拟撮合，杜绝意外连击真实账户。
2. **尾盘平仓保护**：默认在收盘前 5 分钟（15:55 ET）强制全清仓位，避免隔夜非系统性风险。
3. **最大仓位限制**：单只标的限制最大资金占比（默认 30%），单笔下单金额受硬性上限保护。

---

## 📄 开源许可证

本项目依据 [MIT 许可证](LICENSE) 分发。  
Copyright (c) 2026 Wei Zhang
