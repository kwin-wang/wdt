# CLAUDE.md

此文件为Claude Code (claude.ai/code)提供在此代码库中工作的指导。

## WonderTrader概述

WonderTrader是一个高性能量化交易框架，主要使用C++编写，并提供Python绑定(wtpy)。该框架为量化交易生命周期的所有方面提供了全面的功能：

- 数据收集、清洗和存储
- 回测和策略分析
- 实盘交易执行
- 运维调度和监控

主要特点包括：
- 多种交易引擎(CTA, SEL, HFT, UFT)适用于不同的交易场景
- 基于M+1+N架构的策略和执行分离
- 面向专业机构的投资组合管理
- 全面的风险管理
- 支持多种市场和资产类别
- 跨语言兼容性(主要是C++和Python)

## 构建说明

### Linux/Unix构建说明

在release模式下构建WonderTrader:
```bash
cd src
./build_release.sh
```

在debug模式下构建:
```bash
cd src
./build_debug.sh
```

### macOS开发环境设置

在Mac上，推荐使用Docker进行开发：

```bash
# 安装必要工具
softwareupdate --install-rosetta
brew install qemu
brew install colima
brew install docker

# 启动一个x86_64架构的虚拟机(可根据需要调整内存/CPU)
colima start --arch x86_64 --memory 16 --cpu 8

# 验证状态
colima list
```

## 项目架构

WonderTrader组织为几个关键功能区域：

### 核心组件
- **WtCore**: 中央实时交易引擎
- **WtPorter**: 用于跨语言集成的C接口导出库
- **WtRunner**: 框架执行的主入口点

### 交易引擎
- **CTA Engine**: 同步策略引擎(1-2μs延迟)
- **SEL Engine**: 异步策略引擎，用于处理多个交易品种
- **HFT Engine**: 高频交易引擎(1-2μs延迟)
- **UFT Engine**: 超高速交易引擎(<200ns延迟)

### 数据组件
- **WtDataStorage**: 数据存储模块
- **WtDtCore**: 核心数据采集和存储库

### 回测系统
- **WtBtCore**: 核心回测框架
- **WtBtPorter**: 回测框架的C接口

### 市场数据解析器
为不同的渠道和交易所提供了各种解析器：
- 期货(CTP, CTPMini, Femas等)
- 期权(CTPOpt等)
- 股票(XTP, HuaX, ATP, OES)

### 其他重要组件
- **WtExeFact**: 算法交易的执行工厂
- **WtRiskMonFact**: 风控单元
- **WtMsgQue**: 通信消息队列模块

### 目录结构

```bash
# 只描述主要目录
docker/ # docker配置相关信息
src/  # WondertTrader的主要C++代码
python/ # 绑定接口(wtpy)的python代码相关实现
script/
...

```
**重要**： `python/`目录是`wtpy`完整的实现

## 开发工作流

在使用WonderTrader时，请记住：

1. 核心功能使用C++实现以确保性能
2. Python绑定(wtpy)提供了更易于使用的策略开发接口
3. 系统将信号生成与订单执行分离
4. 数据缓存在内存中以实现高性能访问
5. 回测和实盘交易环境使用相同的核心代码

## 代码指南

- 遵循每个模块中的现有代码风格
- 对于性能关键的组件，使用C++并遵循其他地方使用的优化模式
- 策略实现可以使用C++(获得最大性能)或Python(便于开发)
- 单元测试位于TestUnits目录中

## Python使用

### 运行CTA策略(实时)

```python
from wtpy import WtEngine, EngineType

# 创建交易环境并添加策略
env = WtEngine(EngineType.ET_CTA)
env.init('path/to/config/dir/', "config.yaml")

# 创建并添加策略实例
strategy = YourStrategy(name='strategy_name', code='EXCHANGE.SYMBOL.HOT', ...)
env.add_cta_strategy(strategy)

# 运行引擎
env.run(True)
```

### 运行回测

```python
from wtpy import WtBtEngine, EngineType
from wtpy.apps import WtBtAnalyst

# 创建回测引擎
engine = WtBtEngine(EngineType.ET_CTA)
engine.init('path/to/config/dir/', "configbt.yaml")
engine.configBacktest(startTime, endTime)  # 格式: YYYYMMDDHHMM
engine.configBTStorage(mode="csv", path="path/to/storage/")
engine.commitBTConfig()

# 添加策略
strategy = YourStrategy(name='strategy_name', code='EXCHANGE.SYMBOL.HOT', ...)
engine.set_cta_strategy(strategy, slippage=0)

# 运行回测
engine.run_backtest(bAsync=False)

# 分析结果
analyst = WtBtAnalyst()
analyst.add_strategy("strategy_name", folder="./outputs_bt/", init_capital=500000)
analyst.run_new()
```

### 使用监控服务

监控服务提供HTTP和WebSocket接口用于监控交易策略：

```python
from wtpy.monitor import WtMonSvr

# 启动监控服务器
monitor = WtMonSvr()
monitor.run(port=8099)
```