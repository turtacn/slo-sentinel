# SLO Sentinel

A high-performance, AI-ready SLO (Service Level Objective) monitoring and enforcement engine that serves as the foundation for TBOaaS (Trusted Business Outcomes as a Service).

[中文版本](README-zh.md) | [Architecture Documentation](docs/architecture.md) | [API Documentation](docs/api.md)

## Overview

SLO Sentinel is more than just another monitoring tool—it's a forward-thinking SLO computation engine designed to bridge the gap between traditional IT operations and AI-driven business outcome assurance. As a production-ready reference implementation of the [OpenSLO specification](https://github.com/OpenSLO/OpenSLO), it provides standardized, quantifiable methods for defining and measuring business outcomes like "99.99% availability with zero data breaches."

### Core Pain Points Addressed

| Traditional Challenge | SLO Sentinel Solution |
|----------------------|----------------------|
| **Vendor Lock-in** | Plugin-based architecture supports multiple data sources (Prometheus, Datadog, custom APIs) |
| **Limited Scalability** | Independent compute engine with VictoriaMetrics for high-performance time series storage |
| **Reactive Monitoring** | AI-ready interfaces for predictive analysis and proactive intervention |
| **Complex SLO Definition** | Simple YAML-based configuration following OpenSLO standards |
| **Siloed Data** | Unified data ingestion through OpenTelemetry Collector |

### Core Value Propositions

🎯 **Future-Proof Architecture**: Built for the AI era with native support for LLM integration and AI Agent workflows  
⚡ **High Performance**: Optimized for real-time computation with minimal resource consumption  
🔧 **Extensible Design**: Plugin-based data source adapters and modular architecture  
📊 **Standards Compliant**: Full OpenSLO specification compliance ensures ecosystem compatibility  
🚀 **Production Ready**: Enterprise-grade reliability with comprehensive observability

## Key Features

### 🎪 YAML-Based SLO Definition
Define complex SLOs using intuitive YAML syntax:

```yaml
apiVersion: openslo/v1
kind: SLO
metadata:
  name: api-availability
  displayName: "API Availability SLO"
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

### 🔌 Multi-Source Data Integration

Seamlessly integrate with multiple monitoring systems:

```go
// Example: Plugin-based data source configuration
type DataSourceConfig struct {
    Type       string            `yaml:"type"`
    Name       string            `yaml:"name"`
    Endpoint   string            `yaml:"endpoint"`
    Auth       AuthConfig        `yaml:"auth"`
    Parameters map[string]string `yaml:"parameters"`
}

// Supported data sources
var supportedSources = []string{
    "prometheus",
    "datadog",
    "victoriametrics",
    "custom-api",
    "opentelemetry",
}
```

### ⚡ Real-time Computation Engine

High-performance SLO calculation with comprehensive metrics:

```go
// SLO Status Response
type SLOStatus struct {
    Name            string    `json:"name"`
    CurrentValue    float64   `json:"currentValue"`
    Target          float64   `json:"target"`
    ErrorBudget     float64   `json:"errorBudget"`
    BurnRate        float64   `json:"burnRate"`
    Status          string    `json:"status"` // "healthy", "warning", "critical"
    LastUpdated     time.Time `json:"lastUpdated"`
    TimeWindow      string    `json:"timeWindow"`
    Predictions     *AIPredict `json:"predictions,omitempty"`
}

type AIPredict struct {
    NextHourBurnRate   float64 `json:"nextHourBurnRate"`
    BudgetDepletionETA *time.Time `json:"budgetDepletionETA"`
    Confidence         float64 `json:"confidence"`
}
```

### 🤖 AI-Ready Architecture

Built-in support for AI Agent integration:

```go
// AI Agent Integration Interface
type AIAgentInterface interface {
    // Provide structured SLO data for AI analysis
    GetSLOContext(sloName string) (*SLOContext, error)
    
    // Execute AI-driven recommendations
    ExecuteRecommendation(rec *AIRecommendation) error
    
    // Register AI-driven SLO target adjustment
    RegisterDynamicTarget(sloName string, predictor TargetPredictor) error
}

type SLOContext struct {
    Current      SLOStatus     `json:"current"`
    Historical   []SLOPoint    `json:"historical"`
    Correlations []Correlation `json:"correlations"`
    Causality    []CausalLink  `json:"causality"`
}
```

## Architecture Overview

SLO Sentinel follows a layered architecture optimized for performance and extensibility:

```
┌─────────────────────────────────────────────────────────────┐
│                     API Layer                               │
├─────────────────────────────────────────────────────────────┤
│                 Application Layer                           │
├─────────────────────────────────────────────────────────────┤
│                   Domain Layer                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │SLO Aggregate│  │SLI Calculator│  │Error Budget Service │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│                Infrastructure Layer                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │Data Sources │  │VictoriaMetrics│  │OpenTelemetry      │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

For detailed architecture documentation, see [docs/architecture.md](docs/architecture.md).

## Quick Start

### Prerequisites

* Go 1.20.2 or higher
* VictoriaMetrics instance
* Prometheus or other supported monitoring system

### Installation

```bash
# Clone the repository
git clone https://github.com/turtacn/slo-sentinel.git
cd slo-sentinel

# Build the binary
make build

# Install dependencies
go mod download
```

### Basic Usage

1. **Create SLO Configuration**:

```bash
# Create your first SLO definition
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

2. **Configure Data Sources**:

```bash
# Configure Prometheus data source
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

3. **Start the Engine**:

```bash
# Start SLO Sentinel
./bin/slo-sentinel \
  --config=config/sentinel.yaml \
  --slo-dir=config/slo \
  --datasource-dir=config/datasources
```

4. **Query SLO Status**:

```bash
# Check SLO status via API
curl -X GET http://localhost:8080/api/v1/slo/api-availability/status

# Get error budget information
curl -X GET http://localhost:8080/api/v1/slo/api-availability/error-budget

# Query historical data
curl -X GET http://localhost:8080/api/v1/slo/api-availability/history?duration=24h
```

## Configuration

### Main Configuration File

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

### Advanced Features

#### AI Agent Integration

```go
// Register AI predictor for dynamic SLO targets
sentinel.RegisterAIPredictor("api-availability", &CustomPredictor{
    ModelEndpoint: "http://ai-service:8080/predict",
    UpdateInterval: 1 * time.Hour,
})

// Query AI-enhanced SLO status
status, err := sentinel.GetAIEnhancedStatus("api-availability")
if err != nil {
    log.Fatal(err)
}

fmt.Printf("Current: %.4f, Predicted: %.4f, Confidence: %.2f\n", 
    status.CurrentValue, 
    status.Predictions.NextHourBurnRate, 
    status.Predictions.Confidence)
```

## API Documentation

### Core Endpoints

| Endpoint                          | Method | Description                  |
| --------------------------------- | ------ | ---------------------------- |
| `/api/v1/slo/{name}/status`       | GET    | Get current SLO status       |
| `/api/v1/slo/{name}/error-budget` | GET    | Get error budget information |
| `/api/v1/slo/{name}/history`      | GET    | Get historical SLO data      |
| `/api/v1/slo/{name}/predict`      | GET    | Get AI-based predictions     |
| `/api/v1/slo`                     | GET    | List all configured SLOs     |
| `/api/v1/health`                  | GET    | Health check endpoint        |

For complete API documentation, see [docs/api.md](docs/api.md).

## Contributing

We welcome contributions! Please see our [Contributing Guidelines](CONTRIBUTING.md) for details.

### Development Setup

```bash
# Fork and clone the repository
git clone https://github.com/yourusername/slo-sentinel.git
cd slo-sentinel

# Install development dependencies
make dev-setup

# Run tests
make test

# Run linting
make lint

# Build for development
make dev-build
```

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.

## Community

* **GitHub Issues**: [Report bugs or request features](https://github.com/turtacn/slo-sentinel/issues)
* **Discussions**: [Join our community discussions](https://github.com/turtacn/slo-sentinel/discussions)
* **Documentation**: [Full documentation](https://slo-sentinel.readthedocs.io/)

## Acknowledgments

* [OpenSLO Community](https://github.com/OpenSLO/OpenSLO) for the specification
* [VictoriaMetrics](https://github.com/VictoriaMetrics/VictoriaMetrics) for high-performance storage
* [OpenTelemetry](https://opentelemetry.io/) for observability standards

---

**SLO Sentinel** - Powering the future of AI-driven business outcome assurance.