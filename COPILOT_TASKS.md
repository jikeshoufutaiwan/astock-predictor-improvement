# Copilot 执行任务清单

> 本文档包含所有可执行的改进任务，每个任务都有具体的代码和验收标准

---

## 🔴 第一批 - 紧急修复

### Task 1: 清理冗余文件

**目标**: 删除临时/测试文件

**执行命令**:
```bash
# 1. 列出所有临时文件
find ml-service -name "temp_*.py" -o -name "quick_*.py" -o -name "test_*.py" -o -name "check_*.py" -o -name "debug_*.py" -o -name "diagnose_*.py"

# 2. 删除 (注意保留 tests/ 目录下的正式测试)
# rm -f ml-service/temp_*.py
# rm -f ml-service/quick_*.py
# ... 等等
```

**验收标准**:
- [ ] 临时文件删除完毕
- [ ] 核心功能正常运行
- [ ] 提交代码

---

### Task 2: 修复 OOS Sharpe 异常

**目标**: 修复 `backtest_blackbox_v2.py` 仓位计算逻辑

**修改文件**: `ml-service/backtest_blackbox_v2.py`

**修改内容**:
```python
class VaRManager:
    def get_alloc_by_var(self, cvar: float, base_alloc: float, capital: float) -> float:
        """根据 CVaR 调整仓位"""
        if cvar <= 0:
            return base_alloc
        
        # 添加边界检查
        cvar = max(cvar, 0.001)  # 防止除零
        
        max_alloc = capital * self.cap_limit / cvar
        adjusted = min(base_alloc, max_alloc)
        
        # 添加详细日志
        import logging
        logger = logging.getLogger(__name__)
        logger.info(f"VaR调整: cvar={cvar:.4f}, base_alloc={base_alloc:.4f}, "
                   f"capital={capital:.0f}, max_alloc={max_alloc:.4f}, "
                   f"adjusted={adjusted:.4f}")
        
        return adjusted
```

**验收标准**:
- [ ] 所有 OOS Sharpe < 10
- [ ] 日志清晰显示仓位调整
- [ ] 提交代码

---

### Task 3: 统一数据层

**新建文件**: `ml-service/data_layer/__init__.py`

```python
from .unified_store import UnifiedDataStore, DuckDBStore, SQLiteStore

__all__ = ['UnifiedDataStore', 'DuckDBStore', 'SQLiteStore']
```

**新建文件**: `ml-service/data_layer/unified_store.py`

```python
"""
统一数据访问层
================
主: DuckDB (嵌入式，无需维护)
备: SQLite (兼容旧代码)
"""

import logging
from typing import List, Optional
import pandas as pd

logger = logging.getLogger(__name__)


class DuckDBStore:
    """DuckDB 存储 (主存储)"""
    
    def __init__(self, db_path: str = "data/quant.duckdb"):
        import duckdb
        self.conn = duckdb.connect(db_path, read_only=False)
        logger.info(f"DuckDB 初始化: {db_path}")
    
    def get_prices(self, codes: List[str], start: str, end: str) -> pd.DataFrame:
        codes_str = ','.join([f"'{c}'" for c in codes])
        query = f"""
        SELECT * FROM stock_prices
        WHERE code IN ({codes_str})
          AND date >= '{start}'
          AND date <= '{end}'
        ORDER BY code, date
        """
        return self.conn.execute(query).fetchdf()
    
    def close(self):
        self.conn.close()


class SQLiteStore:
    """SQLite 存储 (备用)"""
    
    def __init__(self, db_path: str = "data/stock_data.db"):
        import sqlite3
        self.conn = sqlite3.connect(db_path)
        logger.info(f"SQLite 初始化: {db_path}")
    
    def get_prices(self, codes: List[str], start: str, end: str) -> pd.DataFrame:
        codes_str = ','.join([f"'{c}'" for c in codes])
        query = f"""
        SELECT * FROM stock_prices
        WHERE code IN ({codes_str})
          AND date >= '{start}'
          AND date <= '{end}'
        ORDER BY code, date
        """
        return pd.read_sql(query, self.conn)
    
    def close(self):
        self.conn.close()


class UnifiedDataStore:
    """
    统一数据访问层
    
    Usage:
        store = UnifiedDataStore()
        df = store.get_prices(['000001'], '2024-01-01', '2024-12-31')
    """
    
    def __init__(self, primary: str = "duckdb", fallback: bool = True):
        self.primary_name = primary
        self.fallback_enabled = fallback
        
        if primary == "duckdb":
            self.primary = DuckDBStore()
        else:
            self.primary = SQLiteStore()
        
        self.legacy = SQLiteStore() if primary != "sqlite" else None
    
    def get_prices(self, codes: List[str], start: str, end: str) -> pd.DataFrame:
        try:
            return self.primary.get_prices(codes, start, end)
        except Exception as e:
            logger.warning(f"主存储失败 ({self.primary_name}): {e}")
            if self.fallback_enabled and self.legacy:
                logger.info("降级到 SQLite...")
                return self.legacy.get_prices(codes, start, end)
            raise
    
    def close(self):
        self.primary.close()
        if self.legacy:
            self.legacy.close()
```

**验收标准**:
- [ ] 新建 `data_layer/` 目录
- [ ] 文件可正常导入
- [ ] 提交代码

---

## 🟡 第二批 - 重要改进

### Task 4: 增强熊市识别

**修改文件**: `ml-service/regime_adaptive_engine.py`

**添加代码** (在 `RegimeDetector.detect` 方法中):

```python
def detect(self, date: str) -> RegimeState:
    """识别指定日期的市场环境"""
    # ... 现有代码 ...
    
    # ========== 新增预警指标 ==========
    
    # 1. MA5 下穿 MA20 提前预警
    if len(self.df) > 5:
        row = self.df[self.df['trade_date'] <= date].iloc[-1]
        prev_row = self.df[self.df['trade_date'] <= date].iloc[-2]
        
        ma5 = row.get('ma5', ma20)
        ma5_prev = prev_row.get('ma5', ma20)
        ma20_prev = prev_row.get('ma20', ma20)
        
        ma5_cross_down = (ma5 < ma20) and (ma5_prev >= ma20_prev)
        
        # 2. 波动率突然放大
        vol_20d_avg = row.get('vol60', 0.25) * 0.8
        vol_spike = vol > vol_20d_avg * 2
        
        # 触发预警
        if ma5_cross_down or vol_spike:
            logger.warning(f"⚠️ [{date}] 熊市预警: "
                         f"MA5下穿={ma5_cross_down}, 波动突增={vol_spike}")
            return self._build_state(MarketRegime.BEAR, confidence=0.8)
    
    # ... 继续原有逻辑 ...
```

**验收标准**:
- [ ] 添加预警日志
- [ ] 重新跑 Walk-Forward
- [ ] 提交代码

---

### Task 5: 升级 LSTM 模型

**修改文件**: `ml-service/models.py`

**添加代码**:

```python
class BiLSTMAttentionPredictor:
    """双向 LSTM + Attention 股票预测模型"""
    
    def __init__(
        self,
        seq_length: int = 60,
        n_features: int = 5,
        d_model: int = 64,
        num_heads: int = 4,
        dropout: float = 0.2
    ):
        self.seq_length = seq_length
        self.n_features = n_features
        self.d_model = d_model
        self.num_heads = num_heads
        self.dropout = dropout
        self.model = self._build_model()
    
    def _build_model(self) -> keras.Model:
        inputs = keras.Input(shape=(self.seq_length, self.n_features))
        
        # 投影
        x = layers.Dense(self.d_model)(inputs)
        
        # 双向 LSTM
        x = layers.Bidirectional(
            layers.LSTM(128, return_sequences=True, dropout=self.dropout)
        )(x)
        x = layers.Bidirectional(
            layers.LSTM(64, return_sequences=True, dropout=self.dropout)
        )(x)
        
        # Self-Attention
        attention = layers.MultiHeadAttention(
            num_heads=self.num_heads, 
            key_dim=self.d_model // self.num_heads,
            dropout=self.dropout
        )(x, x)
        x = layers.LayerNormalization(epsilon=1e-6)(x + attention)
        
        # Output
        x = layers.GlobalAveragePooling1D()(x)
        x = layers.Dropout(self.dropout)(x)
        x = layers.Dense(32, activation="relu")(x)
        outputs = layers.Dense(1)(x)
        
        model = keras.Model(inputs=inputs, outputs=outputs, name="bilstm_attention")
        model.compile(
            optimizer=keras.optimizers.Adam(learning_rate=0.001),
            loss="mse",
            metrics=["mae"]
        )
        
        logger.info(f"BiLSTM+Attention 模型完成: {model.count_params()} 参数")
        return model
    
    def predict(self, X: np.ndarray) -> np.ndarray:
        return self.model.predict(X, verbose=0)
```

**验收标准**:
- [ ] 新类可正常实例化
- [ ] 测试预测功能
- [ ] 提交代码

---

### Task 6: 完善 RL 降级方案

**修改文件**: `ml-service/rl_alpha_optimizer.py`

**添加代码**:

```python
class ICBasedDynamicWeights:
    """
    基于 IC 的规则权重 (RL 降级方案)
    
    权重 = max(IC, 0) / sum(max(IC, 0))
    """
    
    def __init__(self, model_names: List[str] = None):
        self.model_names = model_names or AlphaWeightEnv.MODEL_NAMES
    
    def get_weights(self, recent_ic: Dict[str, float]) -> Dict[str, float]:
        positive_ic = {k: max(v, 0) for k, v in recent_ic.items()}
        total = sum(positive_ic.values())
        
        if total == 0:
            return {k: 1/len(self.model_names) for k in self.model_names}
        
        return {k: v/total for k, v in positive_ic.items()}


# 修改 RLAlphaOptimizer
class RLAlphaOptimizer:
    def __init__(self, use_rl: bool = True):
        self.use_rl = use_rl and SB3_AVAILABLE
        
        if self.use_rl:
            self.optimizer = self._init_ppo()
            logger.info("✅ RL 优化器 (PPO)")
        else:
            self.optimizer = ICBasedDynamicWeights()
            logger.info("✅ IC 规则权重 (降级模式)")
```

**验收标准**:
- [ ] 无 stable-baselines3 时可降级
- [ ] 权重计算正确
- [ ] 提交代码

---

## 🟢 第三批 - 长期优化

### Task 7: 建立代码规范

**新建文件**: `.editorconfig`

```ini
root = true

[*.py]
charset = utf-8
indent_style = space
indent_size = 4
max_line_length = 100
```

**新建文件**: `pyproject.toml`

```toml
[tool.black]
line-length = 100
target-version = ['py39', 'py310']

[tool.isort]
profile = "black"
line_length = 100

[tool.mypy]
python_version = "3.9"
ignore_missing_imports = true
```

**验收标准**:
- [ ] 运行 `black .` 成功
- [ ] 运行 `mypy ml-service/` 无错误

---

### Task 8: 添加 CI/CD

**新建文件**: `.github/workflows/test.yml`

```yaml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      - run: pip install -r requirements.txt
      - run: pip install black isort mypy pytest
      - run: black --check ml-service/
      - run: isort --check-only ml-service/
      - run: mypy ml-service/ --ignore-missing-imports
      - run: pytest tests/ -v
```

**验收标准**:
- [ ] Push 后 CI 运行
- [ ] 所有检查通过

---

## ✅ 验证任务

### V1: 下载历史数据

```bash
cd ml-service
python overfit_check_and_longterm.py --download
```

**验收标准**:
- [ ] 数据覆盖 2010-2024
- [ ] 数据质量检查通过

---

### V2: 完整 Walk-Forward

```bash
cd ml-service
python overfit_check_and_longterm.py --walkfwd
```

**验收标准**:
- [ ] 12 窗口全部完成
- [ ] OOS Sharpe 合理

---

### V3: 发布验证报告

**新建文件**: `ml-service/VALIDATION_REPORT_20260305.md`

**内容模板**:
```markdown
# Walk-Forward 验证报告

## 数据概况
- 时间范围: 2010-2024
- 股票数量: XXXX
- 数据来源: Tushare

## 验证结果
| OOS年份 | IC | Sharpe | MaxDD |
|--------|-----|--------|-------|
| 2013   |     |        |       |
| ...    |     |        |       |

## 结论
...
```

---

## 📋 执行检查清单

```markdown
## 第一批
- [ ] Task 1: 清理冗余文件
- [ ] Task 2: 修复 OOS Sharpe 异常
- [ ] Task 3: 统一数据层

## 第二批
- [ ] Task 4: 增强熊市识别
- [ ] Task 5: 升级 LSTM 模型
- [ ] Task 6: 完善 RL 降级方案

## 第三批
- [ ] Task 7: 建立代码规范
- [ ] Task 8: 添加 CI/CD

## 验证
- [ ] V1: 下载历史数据
- [ ] V2: 完整 Walk-Forward
- [ ] V3: 发布验证报告
```

---

*每完成一个任务，请勾选对应的复选框。*
