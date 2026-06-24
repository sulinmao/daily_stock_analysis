---
name: "stock_analyzer"
description: "分析股票和市场。当用户想要分析单个或多个股票，或进行市场复盘时调用。"
---

# 股票分析器

本技能基于 `src/services/analyzer_service.py` 的逻辑，提供分析股票和整体市场的功能。

## 输出结构 (`AnalysisResult`)

单只股票分析函数返回一个 `AnalysisResult` 对象；批量分析返回该对象列表。`perform_market_review` 是一个单独的市场复盘函数，返回 `Optional[str]` 而非 `AnalysisResult`，以避免返回类型不一致。

`AnalysisResult` 对象结构如下：

- `code` (str): 股票代码。
- `name` (str): 股票名称。
- `sentiment_score` (float): 情绪得分，范围 -1.0 到 1.0。
- `operation_advice` (str): 主要操作建议。
- `dashboard` (object): 核心分析内容。
  - `core_conclusion` (object): 核心结论。
    - `summary` (str): 单句总结。
    - `signal_type` (str): 允许值为 `buy`、`sell`、`hold`、`watch`。
    - `position_recommendation` (str): 允许值示例：`full`、`half`、`light`、`avoid`。
  - `data_perspective` (object): 技术面数据。
    - `trend` (str): 允许值为 `up`、`down`、`sideways`。
    - `price_position` (str): 价格位置描述。
    - `volume_analysis` (str): 量能分析描述。
    - `chip_structure` (str): 筹码结构描述。
  - `intelligence` (list[object]): 额外定性情报。
    - 每项对象包含 `type` (str) 和 `text` (str)。
  - `battle_plan` (object): 行动计划。
    - `sniper_points` (list[str]): 关键狙击点。
    - `position_strategy` (str): 仓位策略描述。
    - `risk_controls` (list[str]): 风险控制清单。

调用者应确保单只股票分析与批量分析的结果使用上述结构，避免自由文本或不一致的字段名称。

## 配置 (`Config`)

所有分析函数都可以接受一个可选的 `config` 对象。该对象包含 API 密钥、数据源、时间窗、缓存和通知设置。

配置优先级：
1. 如果调用时提供了 `config`，则使用该对象中明确指定的字段。
2. 否则使用从 `.env` 文件加载的全局单例配置。
3. 仅对已知字段进行处理；未知字段应忽略，不影响默认行为。

支持的 `Config` 字段示例：
- `api_key` (str)
- `data_source` (str): 例如 `akshare`、`baostock`、`tushare`、`yfinance`
- `timeframe` (str): 例如 `1d`、`1w`、`1m`
- `cache_ttl` (int): 缓存过期时间，单位秒。
- `notification_settings` (object)
- `retry_policy` (object): 例如 `{"max_attempts": 2, "backoff_seconds": 1}`。

如果 `config` 中包含本技能未知的字段，应忽略它们并继续使用有效配置。

所有函数在遇到网络/API 错误、超时或提供者限流时，应重试不超过 2 次，采用指数退避。如果仍然失败：
- 单只股票分析返回 `None`。
- 批量分析跳过失败符号，保留成功结果，并在可用的上下文中记录失败符号和原因。

如果未提供 `config`，函数仍应使用全局配置并保持可用。

**参考:** [`Config`](src/config.py)

## 函数

### 1. 分析单只股票

**描述:** 分析单只股票并返回分析结果。

**何时使用:** 当用户要求分析特定股票时。

**输入:**
- `stock_code` (str): 要分析的股票代码。
- `config` (Config, 可选): 配置对象。默认为 `None`。
- `full_report` (bool, 可选): 是否生成完整报告。默认为 `False`。
- `notifier` (NotificationService, 可选): 通知服务对象。默认为 `None`。

**参数语义:**
- `config` 优先于全局配置；如果提供，则覆盖全局配置中对应字段。
- `full_report=True` 返回：`dashboard`、最近 365 天原始时间序列、指标表、完整新闻列表和详细行动计划。
- `full_report=False` 返回：`dashboard.summary`、`data_perspective.trend`、`battle_plan.sniper_points` 和 `intelligence` 中最多 3 条最重要内容。
- `notifier` 如果提供，调用 `notifier.notify(event, payload)`，事件值为 `analysis_start`、`analysis_complete`、`analysis_error`，并传入 `payload` 包含 `{code, status, summary}`。
- 如果 `notifier.notify` 抛出异常，应捕获并记录 `notifier_failed` 警告，不中断分析。

**输出:** `Optional[AnalysisResult]`
一个包含分析结果的 `AnalysisResult` 对象，如果分析失败则为 `None`。

**错误处理:**
- 数据提供者错误或网络/API 超时：返回 `None`，记录系统日志；如果提供 `notifier`，调用 `notifier.notify('analysis_error', {"code": stock_code, "reason": "DATA_ERROR"})`。
- 无效代码或未找到数据：返回 `None`，记录错误；应包含错误信息对象 `{code: 'INVALID_SYMBOL'|'NOT_FOUND', message: str}` 或按实现抛出 `StockNotFoundError`。

**示例:**

```python
from src.services.analyzer_service import analyze_stock

# 分析单只股票
result = analyze_stock("600989")
if result:
    print(f"股票: {result.name} ({result.code})")
    print(f"情绪得分: {result.sentiment_score}")
    print(f"操作建议: {result.operation_advice}")
```

**参考:** [`analyze_stock`](src/services/analyzer_service.py)

### 2. 分析多只股票

**描述:** 分析一个股票列表并返回分析结果列表。

**何时使用:** 当用户想要一次分析多只股票时。

**输入:**
- `stock_codes` (List[str]): 要分析的股票代码列表。
- `config` (Config, 可选): 配置对象。默认为 `None`。
- `full_report` (bool, 可选): 是否为每只股票生成完整报告。默认为 `False`。
- `notifier` (NotificationService, 可选): 通知服务对象。默认为 `None`。

**参数语义:**
- 空列表返回空列表。
- 重复符号按首次出现顺序去重，返回结果顺序与去重后的顺序一致。
- 如果 `len(stock_codes) > 50`，应按 50 个符号一批处理，并可在返回结果中包含 `batch_info`，例如 `{requested: 120, processed: 120, batches: 3}`。
- `full_report` 的含义同单只股票分析。
- `notifier` 行为同单只股票分析；`event` 中的 `code` 可以为每个符号或 `batch`。

**输出:** `List[AnalysisResult]`
成功分析的 `AnalysisResult` 对象列表。

**错误处理:**
- 若部分符号失败，不要使整个批次失败；跳过失败符号，保留成功结果。
- 在可用上下文中记录失败符号和失败原因，并可返回额外的 `errors` 列表，格式为 `[{code: symbol, reason: str}]`。

**示例:**

```python
from src.services.analyzer_service import analyze_stocks

# 分析多只股票
results = analyze_stocks(["600989", "000001"])
for result in results:
    print(f"股票: {result.name}, 操作建议: {result.operation_advice}")
```

**参考:** [`analyze_stocks`](src/services/analyzer_service.py)


### 3. 执行大盘复盘

**描述:** 对整体市场进行复盘并返回一份报告。

**何时使用:** 当用户要求市场概览、摘要或复盘时。

**输入:**
- `config` (Config, 可选): 配置对象。默认为 `None`。
- `notifier` (NotificationService, 可选): 通知服务对象。默认为 `None`。

**输出:** `Optional[str]`
一个包含市场复盘报告的字符串，如果失败则为 `None`。

**错误处理:**
- 网络/API 错误、超时或限流导致复盘失败时，重试不超过 2 次；如果仍失败，返回 `None` 并记录日志。
- 如果提供 `notifier`，遇到错误时调用 `notifier.notify('analysis_error', {"code": "market_review", "reason": "DATA_ERROR"})`。

**示例:**

```python
from src.services.analyzer_service import perform_market_review

# 执行大盘复盘
report = perform_market_review()
if report:
    print(report)
```

**参考:** [`perform_market_review`](src/services/analyzer_service.py)
