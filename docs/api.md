# SLO Sentinel API Documentation

## Overview

SLO Sentinel provides a comprehensive RESTful API for managing Service Level Objectives (SLOs), Service Level Indicators (SLIs), and error budgets. This API follows OpenSLO specification standards and provides real-time monitoring capabilities with AI-powered predictions.

**Base URL**: `https://api.slo-sentinel.dev`  
**API Version**: `v1`  
**Authentication**: Bearer Token  
**Content Type**: `application/json`

## Authentication

All API requests require authentication using Bearer tokens in the Authorization header:

```bash
Authorization: Bearer <your-api-token>
````

### Obtaining API Tokens

```bash
curl -X POST "https://api.slo-sentinel.dev/v1/auth/tokens" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "your-username",
    "password": "your-password"
  }'
```

## SLO Management

### List All SLOs

**Endpoint**: `GET /v1/slos`

**Description**: Retrieve a paginated list of all SLOs with optional filtering.

**Query Parameters**:

| Parameter | Type    | Description                                 | Default | Required |
| --------- | ------- | ------------------------------------------- | ------- | -------- |
| `page`    | integer | Page number (1-based)                       | 1       | No       |
| `limit`   | integer | Items per page (max 100)                    | 20      | No       |
| `service` | string  | Filter by service name                      | -       | No       |
| `status`  | string  | Filter by status (healthy/warning/critical) | -       | No       |
| `labels`  | string  | Filter by labels (comma-separated)          | -       | No       |

**Example Request**:

```bash
curl -X GET "https://api.slo-sentinel.dev/v1/slos?service=user-api&status=critical&limit=50" \
  -H "Authorization: Bearer <token>"
```

**Example Response** (`200 OK`):

```json
{
  "data": [
    {
      "apiVersion": "openslo/v1",
      "kind": "SLO",
      "metadata": {
        "name": "user-api-availability",
        "displayName": "User API Availability",
        "service": "user-api",
        "labels": {
          "team": "platform",
          "environment": "production"
        },
        "createdAt": "2024-01-15T10:00:00Z",
        "updatedAt": "2024-01-20T14:30:00Z"
      },
      "spec": {
        "indicator": {
          "name": "user-api-success-rate",
          "type": "ratio",
          "ratioMetric": {
            "counter": true,
            "good": {
              "source": "prometheus",
              "queryType": "promql",
              "query": "sum(rate(http_requests_total{service=\"user-api\",status!~\"5..\"}[5m]))"
            },
            "total": {
              "source": "prometheus", 
              "queryType": "promql",
              "query": "sum(rate(http_requests_total{service=\"user-api\"}[5m]))"
            }
          }
        },
        "objectives": [
          {
            "displayName": "99.9% availability over 30 days",
            "target": 0.999,
            "timeWindow": {
              "duration": "30d",
              "isRolling": true
            }
          }
        ]
      },
      "status": {
        "currentValue": 0.9985,
        "target": 0.999,
        "errorBudget": {
          "remaining": 0.0005,
          "consumed": 0.0005,
          "consumedPercentage": 50.0,
          "estimatedTimeToExhaustion": "15d"
        },
        "burnRate": {
          "current": 1.2,
          "threshold": 2.0
        },
        "status": "warning",
        "lastUpdated": "2024-01-20T15:00:00Z"
      }
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 1,
    "totalPages": 1
  }
}
```

### Get SLO by Name

**Endpoint**: `GET /v1/slos/{name}`

**Description**: Retrieve detailed information about a specific SLO.

**Path Parameters**:

| Parameter | Type   | Description         | Required |
| --------- | ------ | ------------------- | -------- |
| `name`    | string | SLO name identifier | Yes      |

**Example Request**:

```bash
curl -X GET "https://api.slo-sentinel.dev/v1/slos/user-api-availability" \
  -H "Authorization: Bearer <token>"
```

**Example Response** (`200 OK`):
\[Same as single SLO object above]

**Error Responses**:

| Status Code | Description   | Example                                                |
| ----------- | ------------- | ------------------------------------------------------ |
| `404`       | SLO not found | `{"error": "SLO 'invalid-name' not found"}`            |
| `401`       | Unauthorized  | `{"error": "Invalid or missing authentication token"}` |

### Create SLO

**Endpoint**: `POST /v1/slos`

**Description**: Create a new SLO based on OpenSLO specification.

**Request Body**: OpenSLO-compliant SLO definition

**Example Request**:

```bash
curl -X POST "https://api.slo-sentinel.dev/v1/slos" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "apiVersion": "openslo/v1",
    "kind": "SLO",
    "metadata": {
      "name": "payment-api-latency",
      "displayName": "Payment API P95 Latency",
      "service": "payment-api",
      "labels": {
        "team": "payments",
        "environment": "production"
      }
    },
    "spec": {
      "indicator": {
        "name": "payment-api-p95-latency",
        "type": "threshold",
        "thresholdMetric": {
          "source": "prometheus",
          "queryType": "promql",
          "query": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{service=\"payment-api\"}[5m]))",
          "threshold": {
            "operator": "lte",
            "value": 0.2
          }
        }
      },
      "objectives": [
        {
          "displayName": "P95 latency under 200ms",
          "target": 0.95,
          "timeWindow": {
            "duration": "7d",
            "isRolling": true
          }
        }
      ]
    }
  }'
```

**Example Response** (`201 Created`):

```json
{
  "message": "SLO created successfully",
  "data": {
    "name": "payment-api-latency",
    "status": "created",
    "createdAt": "2024-01-20T15:30:00Z"
  }
}
```

**Error Responses**:

| Status Code | Description                                |
| ----------- | ------------------------------------------ |
| `400`       | Invalid SLO definition or validation error |
| `409`       | SLO with same name already exists          |
| `422`       | OpenSLO specification validation failed    |

### Update SLO

**Endpoint**: `PUT /v1/slos/{name}`

**Description**: Update an existing SLO configuration.

**Path Parameters**:

| Parameter | Type   | Description         | Required |
| --------- | ------ | ------------------- | -------- |
| `name`    | string | SLO name identifier | Yes      |

**Request Body**: Complete OpenSLO-compliant SLO definition

**Example Request**:

```bash
curl -X PUT "https://api.slo-sentinel.dev/v1/slos/payment-api-latency" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "apiVersion": "openslo/v1",
    "kind": "SLO",
    "metadata": {
      "name": "payment-api-latency",
      "displayName": "Payment API P95 Latency - Updated",
      "service": "payment-api"
    },
    "spec": {
      "objectives": [
        {
          "target": 0.98,
          "timeWindow": {
            "duration": "7d",
            "isRolling": true
          }
        }
      ]
    }
  }'
```

**Example Response** (`200 OK`):

```json
{
  "message": "SLO updated successfully",
  "data": {
    "name": "payment-api-latency",
    "status": "updated",
    "updatedAt": "2024-01-20T16:00:00Z"
  }
}
```

### Delete SLO

**Endpoint**: `DELETE /v1/slos/{name}`

**Description**: Delete an existing SLO.

**Path Parameters**:

| Parameter | Type   | Description         | Required |
| --------- | ------ | ------------------- | -------- |
| `name`    | string | SLO name identifier | Yes      |

**Example Request**:

```bash
curl -X DELETE "https://api.slo-sentinel.dev/v1/slos/payment-api-latency" \
  -H "Authorization: Bearer <token>"
```

**Example Response** (`204 No Content`):
*Empty response body*

## SLO Status and Metrics

### Get SLO Current Status

**Endpoint**: `GET /v1/slos/{name}/status`

**Description**: Get real-time status and metrics for a specific SLO.

**Query Parameters**:

| Parameter             | Type    | Description            | Default | Required |
| --------------------- | ------- | ---------------------- | ------- | -------- |
| `include_predictions` | boolean | Include AI predictions | false   | No       |
| `include_history`     | boolean | Include recent history | false   | No       |

**Example Request**:

```bash
curl -X GET "https://api.slo-sentinel.dev/v1/slos/user-api-availability/status?include_predictions=true" \
  -H "Authorization: Bearer <token>"
```

**Example Response** (`200 OK`):

```json
{
  "name": "user-api-availability",
  "status": {
    "overall": "warning",
    "currentValue": 0.9985,
    "target": 0.999,
    "compliance": 99.85,
    "lastUpdated": "2024-01-20T15:00:00Z"
  },
  "errorBudget": {
    "policy": "monthly",
    "remaining": 0.0005,
    "consumed": 0.0005,
    "consumedPercentage": 50.0,
    "estimatedTimeToExhaustion": "15d",
    "burnRate": {
      "current": 1.2,
      "alert": 2.0,
      "critical": 5.0
    }
  },
  "predictions": {
    "trend": "stable",
    "riskLevel": "medium",
    "estimatedComplianceIn24h": 99.82,
    "estimatedComplianceIn7d": 99.91,
    "confidence": 0.85,
    "lastPredictionAt": "2024-01-20T14:45:00Z"
  }
}
```

### Get SLO Historical Data

**Endpoint**: `GET /v1/slos/{name}/history`

**Description**: Retrieve historical SLO performance data.

**Query Parameters**:

| Parameter    | Type   | Description                      | Default | Required |
| ------------ | ------ | -------------------------------- | ------- | -------- |
| `start`      | string | Start time (RFC3339)             | 24h ago | No       |
| `end`        | string | End time (RFC3339)               | now     | No       |
| `resolution` | string | Data resolution (1m, 5m, 1h, 1d) | 5m      | No       |

**Example Request**:

```bash
curl -X GET "https://api.slo-sentinel.dev/v1/slos/user-api-availability/history?start=2024-01-19T00:00:00Z&end=2024-01-20T00:00:00Z&resolution=1h" \
  -H "Authorization: Bearer <token>"
```

**Example Response** (`200 OK`):

```json
{
  "name": "user-api-availability",
  "timeRange": {
    "start": "2024-01-19T00:00:00Z",
    "end": "2024-01-20T00:00:00Z",
    "resolution": "1h"
  },
  "data": [
    {
      "timestamp": "2024-01-19T00:00:00Z",
      "value": 0.9992,
      "errorBudgetConsumed": 0.0008,
      "errorBudgetRemaining": 0.0002
    },
    {
      "timestamp": "2024-01-19T01:00:00Z", 
      "value": 0.9989,
      "errorBudgetConsumed": 0.0011,
      "errorBudgetRemaining": -0.0001
    }
  ],
  "summary": {
    "averageValue": 0.9985,
    "minValue": 0.9980,
    "maxValue": 0.9995,
    "totalErrorBudgetConsumed": 0.0015
  }
}
```

## Error Budget Management

### Get Error Budget Status

**Endpoint**: `GET /v1/slos/{name}/error-budget`

**Description**: Get detailed error budget information for a specific SLO.

**Example Request**:

```bash
curl -X GET "https://api.slo-sentinel.dev/v1/slos/user-api-availability/error-budget" \
  -H "Authorization: Bearer <token>"
```

**Example Response** (`200 OK`):

```json
{
  "sloName": "user-api-availability",
  "errorBudget": {
    "policy": {
      "name": "monthly-policy",
      "resetCycle": "monthly",
      "alertThresholds": [
        {
          "level": "warning",
          "consumedPercentage": 50
        },
        {
          "level": "critical", 
          "consumedPercentage": 90
        }
      ]
    },
    "current": {
      "remaining": 0.0005,
      "consumed": 0.0005,
      "consumedPercentage": 50.0,
      "totalBudget": 0.001
    },
    "burnRate": {
      "current": 1.2,
      "1h": 0.8,
      "6h": 1.1,
      "24h": 1.3,
      "7d": 1.0
    },
    "projections": {
      "estimatedExhaustionTime": "2024-02-05T10:30:00Z",
      "timeToExhaustion": "15d",
      "projectedMonthlyConsumption": 75.5
    }
  }
}
```

### Reset Error Budget

**Endpoint**: `POST /v1/slos/{name}/error-budget/reset`

**Description**: Manually reset the error budget for a specific SLO.

**Request Body**:

```json
{
  "reason": "Planned maintenance completed",
  "resetTo": "full"
}
```

**Example Request**:

```bash
curl -X POST "https://api.slo-sentinel.dev/v1/slos/user-api-availability/error-budget/reset" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "reason": "Incident resolved - planned maintenance",
    "resetTo": "full"
  }'
```

**Example Response** (`200 OK`):

```json
{
  "message": "Error budget reset successfully",
  "data": {
    "sloName": "user-api-availability",
    "resetAt": "2024-01-20T16:00:00Z",
    "reason": "Incident resolved - planned maintenance",
    "newBudget": {
      "remaining": 0.001,
      "consumed": 0.0,
      "consumedPercentage": 0.0
    }
  }
}
```

## Data Source Management

### List Data Sources

**Endpoint**: `GET /v1/datasources`

**Description**: Get all configured data sources and their status.

**Example Request**:

```bash
curl -X GET "https://api.slo-sentinel.dev/v1/datasources" \
  -H "Authorization: Bearer <token>"
```

**Example Response** (`200 OK`):

```json
{
  "data": [
    {
      "name": "prometheus-production",
      "type": "prometheus",
      "url": "https://prometheus.prod.company.com",
      "status": "healthy",
      "lastHealthCheck": "2024-01-20T15:00:00Z",
      "capabilities": [
        "promql",
        "range_queries",
        "instant_queries"
      ],
      "metrics": {
        "queryLatency": "45ms",
        "availability": "99.95%",
        "queriesPerSecond": 150
      }
    },
    {
      "name": "datadog-monitoring",
      "type": "datadog", 
      "status": "healthy",
      "lastHealthCheck": "2024-01-20T15:00:00Z",
      "capabilities": [
        "datadog_query",
        "metrics_query",
        "logs_query"
      ]
    }
  ]
}
```

### Register Data Source

**Endpoint**: `POST /v1/datasources`

**Description**: Register a new data source.

**Request Body**:

```json
{
  "name": "prometheus-staging",
  "type": "prometheus",
  "config": {
    "url": "https://prometheus.staging.company.com",
    "timeout": "30s",
    "auth": {
      "type": "bearer",
      "token": "prometheus-bearer-token"
    }
  }
}
```

**Example Response** (`201 Created`):

```json
{
  "message": "Data source registered successfully",
  "data": {
    "name": "prometheus-staging",
    "status": "registered",
    "registeredAt": "2024-01-20T16:00:00Z"
  }
}
```

### Test Data Source Connection

**Endpoint**: `POST /v1/datasources/{name}/test`

**Description**: Test connectivity and functionality of a data source.

**Example Request**:

```bash
curl -X POST "https://api.slo-sentinel.dev/v1/datasources/prometheus-production/test" \
  -H "Authorization: Bearer <token>"
```

**Example Response** (`200 OK`):

```json
{
  "name": "prometheus-production",
  "testResults": {
    "connectivity": "passed",
    "authentication": "passed", 
    "queryCapability": "passed",
    "latency": "42ms"
  },
  "testedAt": "2024-01-20T16:05:00Z"
}
```

## AI and Predictions

### Get SLO Predictions

**Endpoint**: `GET /v1/slos/{name}/predictions`

**Description**: Get AI-powered predictions for SLO performance and risks.

**Query Parameters**:

| Parameter              | Type   | Description                               | Default | Required |
| ---------------------- | ------ | ----------------------------------------- | ------- | -------- |
| `horizon`              | string | Prediction time horizon (1h, 6h, 24h, 7d) | 24h     | No       |
| `confidence_threshold` | number | Minimum confidence level (0-1)            | 0.7     | No       |

**Example Request**:

```bash
curl -X GET "https://api.slo-sentinel.dev/v1/slos/user-api-availability/predictions?horizon=7d&confidence_threshold=0.8" \
  -H "Authorization: Bearer <token>"
```

**Example Response** (`200 OK`):

```json
{
  "sloName": "user-api-availability",
  "predictions": {
    "horizon": "7d",
    "generatedAt": "2024-01-20T15:00:00Z",
    "model": {
      "version": "v2.1",
      "confidence": 0.87
    },
    "forecasts": [
      {
        "timestamp": "2024-01-21T15:00:00Z",
        "predictedValue": 0.9987,
        "confidence": 0.89,
        "upperBound": 0.9995,
        "lowerBound": 0.9979
      },
      {
        "timestamp": "2024-01-27T15:00:00Z",
        "predictedValue": 0.9991,
        "confidence": 0.83,
        "upperBound": 0.9998,
        "lowerBound": 0.9984
      }
    ],
    "riskAssessment": {
      "overallRisk": "low",
      "riskFactors": [
        {
          "factor": "deployment_pattern",
          "impact": "medium",
          "description": "Increased deployment frequency detected"
        }
      ],
      "recommendations": [
        {
          "priority": "medium",
          "action": "monitor_deployment_impact",
          "description": "Monitor SLO impact during peak deployment hours"
        }
      ]
    }
  }
}
```

### Get Anomaly Detection

**Endpoint**: `GET /v1/slos/{name}/anomalies`

**Description**: Get detected anomalies and their analysis.

**Query Parameters**:

| Parameter  | Type   | Description                            | Default | Required |
| ---------- | ------ | -------------------------------------- | ------- | -------- |
| `start`    | string | Start time (RFC3339)                   | 7d ago  | No       |
| `end`      | string | End time (RFC3339)                     | now     | No       |
| `severity` | string | Filter by severity (low, medium, high) | -       | No       |

**Example Request**:

```bash
curl -X GET "https://api.slo-sentinel.dev/v1/slos/user-api-availability/anomalies?severity=high&start=2024-01-13T00:00:00Z" \
  -H "Authorization: Bearer <token>"
```

**Example Response** (`200 OK`):

```json
{
  "sloName": "user-api-availability", 
  "anomalies": [
    {
      "id": "anomaly-001",
      "detectedAt": "2024-01-18T14:30:00Z",
      "startTime": "2024-01-18T14:25:00Z",
      "endTime": "2024-01-18T14:45:00Z",
      "severity": "high",
      "type": "performance_degradation",
      "description": "Significant drop in success rate detected",
      "impact": {
        "sloValueDrop": 0.005,
        "errorBudgetConsumed": 0.002
      },
      "rootCause": {
        "confidence": 0.78,
        "primaryCause": "external_dependency_failure",
        "contributingFactors": [
          "database_connection_timeout",
          "high_traffic_volume"
        ]
      },
      "recommendations": [
        {
          "action": "investigate_database_performance",
          "priority": "urgent"
        }
      ]
    }
  ]
}
```

## System Health and Monitoring

### System Health Check

**Endpoint**: `GET /v1/health`

**Description**: Get overall system health status.

**Example Request**:

```bash
curl -X GET "https://api.slo-sentinel.dev/v1/health" \
  -H "Authorization: Bearer <token>"
```

**Example Response** (`200 OK`):

```json
{
  "status": "healthy",
  "timestamp": "2024-01-20T15:00:00Z",
  "version": "1.0.0",
  "components": {
    "api": {
      "status": "healthy",
      "responseTime": "15ms"
    },
    "database": {
      "status": "healthy",
      "connectionPool": "8/20"
    },
    "computeEngine": {
      "status": "healthy",
      "activeWorkers": 5,
      "queuedJobs": 12
    },
    "dataSource": {
      "status": "degraded",
      "healthyCount": 2,
      "totalCount": 3,
      "details": {
        "prometheus-production": "healthy",
        "prometheus-staging": "unhealthy",
        "datadog-monitoring": "healthy"
      }
    }
  },
  "metrics": {
    "totalSLOs": 156,
    "activeSLOs": 149,
    "computationsPerMinute": 445,
    "avgComputationTime": "125ms"
  }
}
```

### System Readiness Check

**Endpoint**: `GET /v1/ready`

**Description**: Check if the system is ready to accept traffic.

**Example Response** (`200 OK`):

```json
{
  "status": "ready",
  "timestamp": "2024-01-20T15:00:00Z",
  "readinessChecks": {
    "database": "ready",
    "computeEngine": "ready", 
    "primaryDataSource": "ready"
  }
}
```

### System Metrics

**Endpoint**: `GET /v1/metrics`

**Description**: Get system performance metrics in Prometheus format.

**Example Response** (`200 OK`):

```
# HELP slo_sentinel_slos_total Total number of configured SLOs
# TYPE slo_sentinel_slos_total gauge
slo_sentinel_slos_total 156

# HELP slo_sentinel_computations_total Total number of SLO computations
# TYPE slo_sentinel_computations_total counter
slo_sentinel_computations_total 45123

# HELP slo_sentinel_computation_duration_seconds SLO computation duration
# TYPE slo_sentinel_computation_duration_seconds histogram
slo_sentinel_computation_duration_seconds_bucket{le="0.1"} 1034
slo_sentinel_computation_duration_seconds_bucket{le="0.5"} 3456
slo_sentinel_computation_duration_seconds_bucket{le="1.0"} 4123
slo_sentinel_computation_duration_seconds_bucket{le="+Inf"} 4200
```

## Error Handling

### Standard Error Response Format

All API errors follow a consistent format:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The request body contains invalid data",
    "details": [
      {
        "field": "spec.objectives[0].target",
        "issue": "must be between 0 and 1"
      }
    ],
    "requestId": "req-123e4567-e89b-12d3-a456-426614174000"
  }
}
```

### HTTP Status Codes

| Status Code | Description           | Usage                                       |
| ----------- | --------------------- | ------------------------------------------- |
| `200`       | OK                    | Successful GET, PUT requests                |
| `201`       | Created               | Successful POST requests                    |
| `204`       | No Content            | Successful DELETE requests                  |
| `400`       | Bad Request           | Invalid request format or parameters        |
| `401`       | Unauthorized          | Authentication required or failed           |
| `403`       | Forbidden             | Insufficient permissions                    |
| `404`       | Not Found             | Resource not found                          |
| `409`       | Conflict              | Resource already exists                     |
| `422`       | Unprocessable Entity  | Valid format but failed business validation |
| `429`       | Too Many Requests     | Rate limit exceeded                         |
| `500`       | Internal Server Error | Unexpected server error                     |
| `503`       | Service Unavailable   | System temporarily unavailable              |

### Common Error Codes

| Error Code                 | Description                             |
| -------------------------- | --------------------------------------- |
| `VALIDATION_ERROR`         | Request validation failed               |
| `SLO_NOT_FOUND`            | SLO with specified name not found       |
| `SLO_ALREADY_EXISTS`       | SLO with same name already exists       |
| `DATASOURCE_UNAVAILABLE`   | Data source is not available            |
| `COMPUTATION_FAILED`       | SLO computation failed                  |
| `INVALID_OPENSLO_SPEC`     | OpenSLO specification validation failed |
| `RATE_LIMIT_EXCEEDED`      | API rate limit exceeded                 |
| `INSUFFICIENT_PERMISSIONS` | User lacks required permissions         |

## Rate Limiting

API requests are subject to rate limiting:

* **Default**: 1000 requests per hour per API key
* **Burst**: 100 requests per minute per API key
* **Headers**: Rate limit information in response headers

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1642694400
```

## WebSocket API

### Real-time SLO Status Updates

**Endpoint**: `WSS /v1/ws/slos/{name}/status`

**Description**: Subscribe to real-time SLO status updates via WebSocket.

**Connection Example**:

```javascript
const ws = new WebSocket('wss://api.slo-sentinel.dev/v1/ws/slos/user-api-availability/status?token=<your-token>');

ws.onmessage = function(event) {
  const data = JSON.parse(event.data);
  console.log('SLO Status Update:', data);
};
```

**Message Format**:

```json
{
  "type": "status_update",
  "timestamp": "2024-01-20T15:00:00Z",
  "sloName": "user-api-availability",
  "data": {
    "currentValue": 0.9985,
    "status": "warning",
    "errorBudget": {
      "remaining": 0.0005,
      "consumedPercentage": 50.0
    }
  }
}
```

## SDK and Client Libraries

SLO Sentinel provides official client libraries for popular programming languages:

* **Go**: `go get github.com/slo-sentinel/go-client`
* **Python**: `pip install slo-sentinel-client`
* **JavaScript/Node.js**: `npm install @slo-sentinel/client`
* **Java**: Available via Maven Central

[View client library documentation](https://docs.slo-sentinel.dev/clients) and [API client examples](https://github.com/slo-sentinel/api-examples).

## API Versioning

SLO Sentinel API uses semantic versioning with backward compatibility guarantees:

* **Current Version**: `v1`
* **Deprecation Policy**: 12 months notice before removing features
* **Version Header**: `API-Version: v1` (optional)
* **URL Versioning**: `/v1/` in path (recommended)

### Backward Compatibility

When we introduce breaking changes, we will:

1. Release a new API version
2. Maintain the previous version for at least 12 months
3. Provide migration guides and tools
4. Send deprecation notices via email and API headers

[Learn more about API versioning best practices](https://restfulapi.net/versioning/).

## Batch Operations

### Batch SLO Operations

**Endpoint**: `POST /v1/slos/batch`

**Description**: Perform multiple SLO operations in a single request for improved efficiency.

**Request Body**:

```json
{
  "operations": [
    {
      "operation": "create",
      "data": {
        "apiVersion": "openslo/v1",
        "kind": "SLO",
        "metadata": {
          "name": "batch-slo-1",
          "service": "test-service"
        },
        "spec": {
          "objectives": [
            {
              "target": 0.99,
              "timeWindow": {
                "duration": "30d",
                "isRolling": true
              }
            }
          ]
        }
      }
    },
    {
      "operation": "update",
      "name": "existing-slo",
      "data": {
        "spec": {
          "objectives": [
            {
              "target": 0.995,
              "timeWindow": {
                "duration": "30d", 
                "isRolling": true
              }
            }
          ]
        }
      }
    },
    {
      "operation": "delete",
      "name": "obsolete-slo"
    }
  ]
}
```

**Example Request**:

```bash
curl -X POST "https://api.slo-sentinel.dev/v1/slos/batch" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d @batch-operations.json
```

**Example Response** (`207 Multi-Status`):

```json
{
  "results": [
    {
      "operation": "create",
      "name": "batch-slo-1",
      "status": "success",
      "statusCode": 201,
      "message": "SLO created successfully"
    },
    {
      "operation": "update", 
      "name": "existing-slo",
      "status": "success",
      "statusCode": 200,
      "message": "SLO updated successfully"
    },
    {
      "operation": "delete",
      "name": "obsolete-slo", 
      "status": "error",
      "statusCode": 404,
      "message": "SLO not found",
      "error": {
        "code": "SLO_NOT_FOUND",
        "details": "SLO 'obsolete-slo' does not exist"
      }
    }
  ],
  "summary": {
    "total": 3,
    "successful": 2,
    "failed": 1
  }
}
```

### Batch Status Query

**Endpoint**: `POST /v1/slos/batch/status`

**Description**: Query status for multiple SLOs efficiently.

**Request Body**:

```json
{
  "sloNames": [
    "user-api-availability",
    "payment-api-latency", 
    "order-service-error-rate"
  ],
  "includeHistory": false,
  "includePredictions": true
}
```

**Example Response** (`200 OK`):

```json
{
  "data": {
    "user-api-availability": {
      "status": "healthy",
      "currentValue": 0.9995,
      "target": 0.999,
      "errorBudget": {
        "remaining": 0.0005,
        "consumedPercentage": 50.0
      }
    },
    "payment-api-latency": {
      "status": "warning", 
      "currentValue": 0.945,
      "target": 0.95,
      "errorBudget": {
        "remaining": 0.005,
        "consumedPercentage": 75.0
      }
    },
    "order-service-error-rate": {
      "status": "critical",
      "currentValue": 0.985,
      "target": 0.99,
      "errorBudget": {
        "remaining": 0.0,
        "consumedPercentage": 100.0
      }
    }
  },
  "retrievedAt": "2024-01-20T15:00:00Z"
}
```

## Alerting and Notifications

### List Alert Rules

**Endpoint**: `GET /v1/alerts/rules`

**Description**: Get all configured alerting rules for SLOs.

**Example Response** (`200 OK`):

```json
{
  "data": [
    {
      "id": "alert-rule-001",
      "name": "High Error Budget Consumption",
      "sloName": "user-api-availability",
      "condition": {
        "type": "error_budget_consumption",
        "threshold": 80,
        "window": "1h"
      },
      "notifications": [
        {
          "type": "slack",
          "channel": "#sre-alerts",
          "severity": "warning"
        },
        {
          "type": "email",
          "recipients": ["sre-team@company.com"],
          "severity": "critical"
        }
      ],
      "enabled": true,
      "createdAt": "2024-01-15T10:00:00Z"
    }
  ]
}
```

### Create Alert Rule

**Endpoint**: `POST /v1/alerts/rules`

**Description**: Create a new alerting rule for SLO monitoring.

**Request Body**:

```json
{
  "name": "SLO Burn Rate Alert",
  "sloName": "payment-api-latency",
  "condition": {
    "type": "burn_rate",
    "threshold": 5.0,
    "window": "5m"
  },
  "notifications": [
    {
      "type": "pagerduty",
      "serviceKey": "your-pagerduty-service-key",
      "severity": "critical"
    }
  ],
  "enabled": true
}
```

**Example Response** (`201 Created`):

```json
{
  "message": "Alert rule created successfully",
  "data": {
    "id": "alert-rule-002",
    "name": "SLO Burn Rate Alert", 
    "status": "active",
    "createdAt": "2024-01-20T16:00:00Z"
  }
}
```

## Configuration Management

### Get System Configuration

**Endpoint**: `GET /v1/config`

**Description**: Retrieve current system configuration (admin only).

**Example Response** (`200 OK`):

```json
{
  "computation": {
    "defaultResolution": "5m",
    "maxHistoryDays": 90,
    "computeIntervalSeconds": 300,
    "concurrentWorkers": 10
  },
  "storage": {
    "retentionDays": 365,
    "compressionEnabled": true,
    "backupEnabled": true
  },
  "ai": {
    "predictionEnabled": true,
    "modelVersion": "v2.1",
    "confidenceThreshold": 0.7,
    "updateIntervalHours": 24
  },
  "api": {
    "rateLimit": {
      "requestsPerHour": 1000,
      "burstSize": 100
    },
    "authentication": {
      "tokenExpiryHours": 24,
      "refreshEnabled": true
    }
  }
}
```

### Update Configuration

**Endpoint**: `PATCH /v1/config`

**Description**: Update system configuration (admin only).

**Request Body**:

```json
{
  "computation": {
    "computeIntervalSeconds": 60,
    "concurrentWorkers": 20
  },
  "ai": {
    "confidenceThreshold": 0.8
  }
}
```

**Example Response** (`200 OK`):

```json
{
  "message": "Configuration updated successfully",
  "data": {
    "updatedFields": [
      "computation.computeIntervalSeconds",
      "computation.concurrentWorkers", 
      "ai.confidenceThreshold"
    ],
    "updatedAt": "2024-01-20T16:30:00Z"
  }
}
```

## Import/Export

### Export SLOs

**Endpoint**: `GET /v1/export/slos`

**Description**: Export SLO configurations in OpenSLO format.

**Query Parameters**:

| Parameter       | Type    | Description                | Default | Required |
| --------------- | ------- | -------------------------- | ------- | -------- |
| `format`        | string  | Export format (yaml, json) | yaml    | No       |
| `service`       | string  | Filter by service name     | -       | No       |
| `includeStatus` | boolean | Include current status     | false   | No       |

**Example Request**:

```bash
curl -X GET "https://api.slo-sentinel.dev/v1/export/slos?format=yaml&service=user-api" \
  -H "Authorization: Bearer <token>" \
  -H "Accept: application/x-yaml"
```

**Example Response** (`200 OK`):

```yaml
apiVersion: openslo/v1
kind: SLO
metadata:
  name: user-api-availability
  displayName: User API Availability
  service: user-api
  labels:
    team: platform
    environment: production
spec:
  indicator:
    name: user-api-success-rate
    type: ratio
    ratioMetric:
      counter: true
      good:
        source: prometheus
        queryType: promql
        query: sum(rate(http_requests_total{service="user-api",status!~"5.."}[5m]))
      total:
        source: prometheus
        queryType: promql
        query: sum(rate(http_requests_total{service="user-api"}[5m]))
  objectives:
    - displayName: 99.9% availability over 30 days
      target: 0.999
      timeWindow:
        duration: 30d
        isRolling: true
---
apiVersion: openslo/v1
kind: SLO
metadata:
  name: user-api-latency
  displayName: User API P95 Latency
  service: user-api
spec:
  indicator:
    name: user-api-p95-latency
    type: threshold
    thresholdMetric:
      source: prometheus
      queryType: promql
      query: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{service="user-api"}[5m]))
      threshold:
        operator: lte
        value: 0.1
  objectives:
    - displayName: P95 latency under 100ms
      target: 0.95
      timeWindow:
        duration: 7d
        isRolling: true
```

### Import SLOs

**Endpoint**: `POST /v1/import/slos`

**Description**: Import SLO configurations from OpenSLO format.

**Request Body**: OpenSLO YAML or JSON content

**Example Request**:

```bash
curl -X POST "https://api.slo-sentinel.dev/v1/import/slos" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/x-yaml" \
  --data-binary @slos-export.yaml
```

**Example Response** (`200 OK`):

```json
{
  "message": "SLOs imported successfully",
  "data": {
    "imported": [
      {
        "name": "user-api-availability",
        "status": "created"
      },
      {
        "name": "user-api-latency", 
        "status": "updated"
      }
    ],
    "errors": [
      {
        "name": "invalid-slo",
        "error": "Validation failed: target must be between 0 and 1"
      }
    ],
    "summary": {
      "total": 3,
      "successful": 2,
      "failed": 1
    }
  }
}
```

## Reporting

### Generate SLO Report

**Endpoint**: `POST /v1/reports/slo`

**Description**: Generate comprehensive SLO performance reports.

**Request Body**:

```json
{
  "reportType": "monthly",
  "timeRange": {
    "start": "2024-01-01T00:00:00Z",
    "end": "2024-01-31T23:59:59Z"
  },
  "sloNames": [
    "user-api-availability",
    "payment-api-latency"
  ],
  "format": "pdf",
  "includeCharts": true,
  "includeRecommendations": true
}
```

**Example Response** (`202 Accepted`):

```json
{
  "message": "Report generation started",
  "data": {
    "reportId": "report-20240120-001",
    "status": "processing",
    "estimatedCompletionTime": "2024-01-20T16:45:00Z",
    "downloadUrl": null
  }
}
```

### Get Report Status

**Endpoint**: `GET /v1/reports/{reportId}`

**Description**: Check the status of a report generation request.

**Example Response** (`200 OK`):

```json
{
  "reportId": "report-20240120-001",
  "status": "completed",
  "createdAt": "2024-01-20T16:30:00Z",
  "completedAt": "2024-01-20T16:42:00Z",
  "downloadUrl": "https://api.slo-sentinel.dev/v1/reports/report-20240120-001/download",
  "expiresAt": "2024-01-27T16:42:00Z"
}
```

## Advanced Features

### Custom Metrics Integration

**Endpoint**: `POST /v1/custom-metrics`

**Description**: Register custom metrics for SLO calculations.

**Request Body**:

```json
{
  "name": "custom-business-metric",
  "description": "Custom business success rate",
  "dataSource": "custom-api",
  "queryConfig": {
    "endpoint": "https://api.company.com/metrics/business-success",
    "method": "GET",
    "headers": {
      "Authorization": "Bearer ${API_TOKEN}"
    },
    "responseMapping": {
      "valueField": "data.successRate",
      "timestampField": "data.timestamp"
    }
  },
  "refreshInterval": "5m"
}
```

### SLO Templates

**Endpoint**: `GET /v1/templates/slo`

**Description**: Get predefined SLO templates for common use cases.

**Example Response** (`200 OK`):

```json
{
  "templates": [
    {
      "id": "web-service-availability",
      "name": "Web Service Availability",
      "description": "Standard availability SLO for web services",
      "category": "availability",
      "template": {
        "apiVersion": "openslo/v1",
        "kind": "SLO",
        "metadata": {
          "name": "${SERVICE_NAME}-availability",
          "service": "${SERVICE_NAME}"
        },
        "spec": {
          "indicator": {
            "type": "ratio",
            "ratioMetric": {
              "good": {
                "source": "prometheus",
                "query": "sum(rate(http_requests_total{service=\"${SERVICE_NAME}\",status!~\"5..\"}[5m]))"
              },
              "total": {
                "source": "prometheus", 
                "query": "sum(rate(http_requests_total{service=\"${SERVICE_NAME}\"}[5m]))"
              }
            }
          },
          "objectives": [
            {
              "target": 0.999,
              "timeWindow": {
                "duration": "30d",
                "isRolling": true
              }
            }
          ]
        }
      },
      "variables": [
        {
          "name": "SERVICE_NAME",
          "description": "Name of the service to monitor",
          "required": true
        }
      ]
    }
  ]
}
```

## Troubleshooting

### Common Issues and Solutions

#### 1. SLO Computation Delays

**Issue**: SLO values are not updating in real-time.

**Possible Causes**:

* Data source connectivity issues
* High computation load
* Configuration errors

**Debugging Steps**:

1. Check data source health: `GET /v1/datasources`
2. Verify SLO configuration: `GET /v1/slos/{name}`
3. Check computation status: `GET /v1/compute/status`

#### 2. Authentication Errors

**Issue**: `401 Unauthorized` responses.

**Solutions**:

* Verify token validity: `GET /v1/auth/verify`
* Check token expiration
* Ensure proper Authorization header format

#### 3. Data Source Connection Failures

**Issue**: "Data source unavailable" errors.

**Debugging**:

* Test data source: `POST /v1/datasources/{name}/test`
* Check network connectivity
* Verify credentials and permissions

### Debug Endpoints

**Enable Debug Mode**: `POST /v1/debug/enable`

**Get Debug Information**: `GET /v1/debug/slo/{name}`

**Response**:

```json
{
  "sloName": "user-api-availability",
  "debugInfo": {
    "lastComputation": {
      "timestamp": "2024-01-20T15:00:00Z",
      "duration": "125ms",
      "status": "success",
      "rawData": {
        "goodRequests": 9995,
        "totalRequests": 10000
      }
    },
    "dataSourceQueries": [
      {
        "source": "prometheus-production",
        "query": "sum(rate(http_requests_total{service=\"user-api\",status!~\"5..\"}[5m]))",
        "executionTime": "45ms",
        "result": 33.25
      }
    ],
    "computationSteps": [
      {
        "step": "fetch_good_events",
        "duration": "45ms",
        "result": 9995
      },
      {
        "step": "fetch_total_events", 
        "duration": "42ms",
        "result": 10000
      },
      {
        "step": "calculate_ratio",
        "duration": "1ms",
        "result": 0.9995
      }
    ]
  }
}
```

## Contact and Support

* **API Documentation**: [https://docs.slo-sentinel.dev](https://docs.slo-sentinel.dev)
* **GitHub Repository**: [https://github.com/slo-sentinel/slo-sentinel](https://github.com/slo-sentinel/slo-sentinel)
* **Issue Tracker**: [https://github.com/slo-sentinel/slo-sentinel/issues](https://github.com/slo-sentinel/slo-sentinel/issues)
* **Community Forum**: [https://community.slo-sentinel.dev](https://community.slo-sentinel.dev)
* **Support Email**: [support@slo-sentinel.dev](mailto:support@slo-sentinel.dev)

For additional help, please refer to our [comprehensive documentation](https://docs.slo-sentinel.dev) or [community resources](https://community.slo-sentinel.dev/getting-started).

---

*This API documentation follows [OpenAPI 3.0 specification](https://swagger.io/specification/) and [RESTful API design principles](https://restfulapi.net/). Last updated: January 20, 2024*

