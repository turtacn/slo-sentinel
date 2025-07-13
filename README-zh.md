# SLO Sentinel

一个高性能、AI就绪的SLO（服务等级目标）监控和执行引擎，作为TBOaaS（可信业务成果即服务）的技术基石。

[English Version](README.md) | [架构文档](docs/architecture.md) | [API文档](docs/api.md)

## 项目概述

SLO Sentinel不仅仅是另一个监控工具——它是一个前瞻性的SLO计算引擎，旨在弥合传统IT运维与AI驱动的业务成果保障之间的差距。作为[OpenSLO规范](https://github.com/OpenSLO/OpenSLO)的生产就绪参考实现，它提供了标准化、可量化的方法来定义和度量业务成果，如"99.99%可用性与零数据泄露"。

### 核心痛点解决方案

| 传统挑战 | SLO Sentinel解决方案 |
|---------|-------------------|
| **厂商锁定** | 基于插件的架构支持多种数据源（Prometheus、Datadog、自定义API） |
| **扩展性限制** | 独立计算引擎配合VictoriaMetrics实现高性能时序存储 |
| **被动监控** | AI就绪接口支持预测分析和主动干预 |
| **复杂SLO定义** | 基于YAML的简单配置，遵循OpenSLO标准 |
| **数据孤岛** | 通过OpenTelemetry Collector统一数据摄取 |

### 核心价值主张

🎯 **面向未来的架构**：为AI时代构建，原生支持LLM集成和AI Agent工作流  
⚡ **高性能**：优化的实时计算，资源消耗最小化  
🔧 **可扩展设计**：基于插件的数据源适配器和模块化架构  
📊 **标准兼容**：完全符合OpenSLO规范，确保生态系统兼容性  
🚀 **生产就绪**：企业级可靠性，全面的可观测性

## 主要功能特性

### 🎪 基于YAML的SLO定义
使用直观的YAML语法定义复杂的SLO：

```yaml
apiVersion: openslo/v1
kind: SLO
metadata:
  name: api-availability
  displayName: "API可用性SLO"
spec:
  service: payment-service
  indicator:
    spec:
      ratioMetric:
        counter: true
        good:
          source: prometheus
          queryType: promql
          query: 'sum(rate(http_requests_total{job="payment-service",code!~"5.."}[5m]))'
        total:
          source: prometheus
          queryType: promql
          query: 'sum(rate(http_requests_total{job="payment-service"}[5m]))'
  objectives:
    - target: 0.999
      timeWindow:
        duration: 30d
        isRolling: true
````

### 🔌 多源数据集成

无缝集成多个监控系统：

```go
// 示例：基于插件的数据源配置
type DataSourceConfig struct {
    Type       string            `yaml:"type"`
    Name       string            `yaml:"name"`
    Endpoint   string            `yaml:"endpoint"`
    Auth       AuthConfig        `yaml:"auth"`
    Parameters map[string]string `yaml:"parameters"`
}

// 支持的数据源
var supportedSources = []string{
    "prometheus",
    "datadog",
    "victoriametrics",
    "custom-api",
    "opentelemetry",
}
```

### ⚡ 实时计算引擎

高性能SLO计算，提供全面的指标：

```go
// SLO状态响应
type SLOStatus struct {
    Name            string    `json:"name"`
    CurrentValue    float64   `json:"currentValue"`    // 当前值
    Target          float64   `json:"target"`          // 目标值
    ErrorBudget     float64   `json:"errorBudget"`     // 错误预算
    BurnRate        float64   `json:"burnRate"`        // 燃烧率
    Status          string    `json:"status"`          // "healthy", "warning", "critical"
    LastUpdated     time.Time `json:"lastUpdated"`     // 最后更新时间
    TimeWindow      string    `json:"timeWindow"`      // 时间窗口
    Predictions     *AIPredict `json:"predictions,omitempty"` // AI预测
}

type AIPredict struct {
    NextHourBurnRate   float64 `json:"nextHourBurnRate"`   // 下一小时燃烧率
    BudgetDepletionETA *time.Time `json:"budgetDepletionETA"` // 预算耗尽预计时间
    Confidence         float64 `json:"confidence"`         // 置信度
}
```

### 🤖 AI就绪架构

内置AI Agent集成支持：

```go
// AI Agent集成接口
type AIAgentInterface interface {
    // 为AI分析提供结构化SLO数据
    GetSLOContext(sloName string) (*SLOContext, error)
    
    // 执行AI驱动的建议
    ExecuteRecommendation(rec *AIRecommendation) error
    
    // 注册AI驱动的SLO目标调整
    RegisterDynamicTarget(sloName string, predictor TargetPredictor) error
}

type SLOContext struct {
    Current      SLOStatus     `json:"current"`      // 当前状态
    Historical   []SLOPoint    `json:"historical"`   // 历史数据
    Correlations []Correlation `json:"correlations"` // 相关性
    Causality    []CausalLink  `json:"causality"`    // 因果关系
}
```

## 架构概览

SLO Sentinel采用针对性能和可扩展性优化的分层架构：

```
┌─────────────────────────────────────────────────────────────┐
│                     API层                                   │
├─────────────────────────────────────────────────────────────┤
│                   应用层                                     │
├─────────────────────────────────────────────────────────────┤
│                   领域层                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │SLO聚合根    │  │SLI计算器    │  │错误预算服务         │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│                 基础设施层                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │数据源       │  │VictoriaMetrics│  │OpenTelemetry      │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

详细架构文档请参见[docs/architecture.md](docs/architecture.md)。

## 快速开始

### 前置条件

* Go 1.20.2 或更高版本
* VictoriaMetrics实例
* Prometheus或其他支持的监控系统

### 安装

```bash
# 克隆仓库
git clone https://github.com/turtacn/slo-sentinel.git
cd slo-sentinel

# 构建二进制文件
make build

# 安装依赖
go mod download
```

### 基本使用

1. **创建SLO配置**：

```bash
# 创建第一个SLO定义
cat > config/slo/api-availability.yaml << EOF
apiVersion: openslo/v1
kind: SLO
metadata:
  name: api-availability
spec:
  service: payment-service
  indicator:
    spec:
      ratioMetric:
        good:
          source: prometheus
          query: 'sum(rate(http_requests_total{code!~"5.."}[5m]))'
        total:
          source: prometheus
          query: 'sum(rate(http_requests_total[5m]))'
  objectives:
    - target: 0.999
      timeWindow:
        duration: 30d
        isRolling: true
EOF
```

2. **配置数据源**：

```bash
# 配置Prometheus数据源
cat > config/datasources/prometheus.yaml << EOF
type: prometheus
name: main-prometheus
endpoint: http://localhost:9090
auth:
  type: none
parameters:
  timeout: 30s
  maxQueryRange: 7d
EOF
```

3. **启动引擎**：

```bash
# 启动SLO Sentinel
./bin/slo-sentinel \
  --config=config/sentinel.yaml \
  --slo-dir=config/slo \
  --datasource-dir=config/datasources
```

4. **查询SLO状态**：

```bash
# 通过API检查SLO状态
curl -X GET http://localhost:8080/api/v1/slo/api-availability/status

# 获取错误预算信息
curl -X GET http://localhost:8080/api/v1/slo/api-availability/error-budget

# 查询历史数据
curl -X GET http://localhost:8080/api/v1/slo/api-availability/history?duration=24h
```

## 配置说明

### 主配置文件

```yaml
# config/sentinel.yaml
server:
  port: 8080
  host: "0.0.0.0"

storage:
  type: victoriametrics
  endpoint: http://localhost:8428
  retention: 90d

computation:
  interval: 1m
  workers: 4
  batchSize: 1000

ai:
  enabled: true
  endpoints:
    prediction: http://localhost:8081/predict
    recommendation: http://localhost:8081/recommend
```

### 高级功能

#### AI Agent集成

```go
// 注册AI预测器用于动态SLO目标
sentinel.RegisterAIPredictor("api-availability", &CustomPredictor{
    ModelEndpoint: "http://ai-service:8080/predict",
    UpdateInterval: 1 * time.Hour,
})

// 查询AI增强的SLO状态
status, err := sentinel.GetAIEnhancedStatus("api-availability")
if err != nil {
    log.Fatal(err)
}

fmt.Printf("当前值: %.4f, 预测值: %.4f, 置信度: %.2f\n", 
    status.CurrentValue, 
    status.Predictions.NextHourBurnRate, 
    status.Predictions.Confidence)
```

## API文档

### 核心端点

| 端点                                | 方法  | 描述         |
| --------------------------------- | --- | ---------- |
| `/api/v1/slo/{name}/status`       | GET | 获取当前SLO状态  |
| `/api/v1/slo/{name}/error-budget` | GET | 获取错误预算信息   |
| `/api/v1/slo/{name}/history`      | GET | 获取历史SLO数据  |
| `/api/v1/slo/{name}/predict`      | GET | 获取基于AI的预测  |
| `/api/v1/slo`                     | GET | 列出所有配置的SLO |
| `/api/v1/health`                  | GET | 健康检查端点     |

完整API文档请参见[docs/api.md](docs/api.md)。

## 贡献指南

我们欢迎贡献！请查看我们的[贡献指南](CONTRIBUTING.md)了解详情。

### 开发环境设置

```bash
# Fork并克隆仓库
git clone https://github.com/yourusername/slo-sentinel.git
cd slo-sentinel

# 安装开发依赖
make dev-setup

# 运行测试
make test

# 运行代码检查
make lint

# 开发构建
make dev-build
```

## 许可证

本项目基于Apache License 2.0许可证 - 详情请参见[LICENSE](LICENSE)文件。

## 社区

* **GitHub Issues**: [报告问题或请求功能](https://github.com/turtacn/slo-sentinel/issues)
* **讨论**: [加入我们的社区讨论](https://github.com/turtacn/slo-sentinel/discussions)
* **文档**: [完整文档](https://slo-sentinel.readthedocs.io/)

## 致谢

* [OpenSLO社区](https://github.com/OpenSLO/OpenSLO) 提供的规范
* [VictoriaMetrics](https://github.com/VictoriaMetrics/VictoriaMetrics) 提供的高性能存储
* [OpenTelemetry](https://opentelemetry.io/) 提供的可观测性标准

---

**SLO Sentinel** - 驱动AI驱动的业务成果保障未来。