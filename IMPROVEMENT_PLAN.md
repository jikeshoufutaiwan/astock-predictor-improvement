# 改进计划

> 版本: 1.0 | 日期: 2026-03-05

---

## 改进阶段

### 🔴 第一阶段: 紧急修复 (1-2周)

#### 1.1 清理冗余文件

**目标**: 删除临时/测试文件，保持代码整洁

**文件列表** (预计删除 50+ 个):
```
ml-service/temp_*.py
ml-service/quick_*.py
ml-service/test_*.py (保留 tests/ 目录)
ml-service/check_*.py
ml-service/debug_*.py
ml-service/diagnose_*.py
```

**验收标准**:
- 临时文件数量减少到 0
- 核心功能不受影响

---

#### 1.2 修复 OOS Sharpe 异常

**目标**: 确保回测结果合理

**修改文件**: `ml-service/backtest_blackbox_v2.py`

**修改内容**:
1. 检查 VaR 仓位计算逻辑
2. 添加详细日志输出
3. 验证交易成本扣除

**预期结果**:
- 所有 OOS Sharpe < 5
- 重新生成测试报告

---

#### 1.3 统一数据层

**目标**: 解决三套数据库并存问题

**新建文件**: `ml-service/data_layer/unified_store.py`

**设计**:
```
UnifiedDataStore
├── primary: DuckDB (嵌入式，新主存储)
└── legacy: SQLite (兼容旧代码)
```

**验收标准**:
- 新建数据访问层
- 更新核心模块使用新接口

---

### 🟡 第二阶段: 重要改进 (2-4周)

#### 2.1 增强熊市识别

**目标**: 提前识别熊市，避免大幅亏损

**修改文件**: `ml-service/regime_adaptive_engine.py`

**新增功能**:
1. MA5 下穿 MA20 提前预警
2. 波动率突增检测 (>2倍均值)
3. 下跌股票占比监控

**验收标准**:
- 2022年初能提前识别熊市
- 重新跑 Walk-Forward 验证效果

---

#### 2.2 升级 LSTM 模型

**目标**: 提升模型预测能力

**修改文件**: `ml-service/models.py`

**新增类**: `BiLSTMAttentionPredictor`

**架构**:
```
Input → BiLSTM(128) → BiLSTM(64) → Self-Attention → GlobalAvgPool → Dense(1)
```

**验收标准**:
- 新增模型可用
- MSE 有改善

---

#### 2.3 完善 RL 优化器

**目标**: 确保 RL 不可用时有降级方案

**修改文件**: `ml-service/rl_alpha_optimizer.py`

**新增类**: `ICBasedDynamicWeights`

**功能**: 基于 IC 的规则权重，当 stable-baselines3 未安装时自动降级

---

### 🟢 第三阶段: 长期优化 (1-3月)

#### 3.1 建立代码规范

**新建文件**:
- `.editorconfig`
- `pyproject.toml`

**工具**:
- black (代码格式化)
- isort (import 排序)
- mypy (类型检查)

---

#### 3.2 添加 CI/CD

**新建文件**: `.github/workflows/test.yml`

**流程**:
```yaml
push → format check → type check → test → coverage
```

---

#### 3.3 创建因子研究模板

**新建文件**: `ml-service/factor_research_template.py`

**功能**:
- 标准化因子研究流程
- 自动计算 IC/ICIR/衰减
- 生成推荐结果

---

#### 3.4 创建模型监控模块

**新建文件**: `ml-service/model_monitor.py`

**功能**:
- 每日检查模型预测 vs 实际
- IC 低于阈值自动告警
- 生成监控报告

---

## 验证任务

### V1: 下载历史数据

```bash
python overfit_check_and_longterm.py --download
```

**目标**: 获取 2010-2024 完整数据

---

### V2: 完整 Walk-Forward 验证

```bash
python overfit_check_and_longterm.py --walkfwd
```

**目标**: 12 窗口验证

---

### V3: 发布验证报告

**新建文件**: `ml-service/VALIDATION_REPORT_20260305.md`

**内容**:
1. 数据概况
2. 各窗口结果
3. 过拟合分析
4. 改进建议

---

## 时间线

```
Week 1-2:  第一阶段 (紧急修复)
Week 3-4:  第二阶段 (重要改进)
Month 2-3: 第三阶段 (长期优化)
```

---

## 风险评估

| 任务 | 风险 | 缓解措施 |
|------|------|----------|
| 删除文件 | 误删有用代码 | 先备份，增量提交 |
| 修改回测逻辑 | 影响现有结果 | 保留旧版本对比 |
| 升级模型 | 训练失败 | 保留原模型 |
| 统一数据层 | 兼容性问题 | 渐进式迁移 |

---

*本计划将根据实际进展动态调整。*
