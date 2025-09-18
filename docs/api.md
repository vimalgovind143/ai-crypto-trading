# API Documentation - AI Cryptocurrency Trading System

## Overview

The AI Cryptocurrency Trading System provides a comprehensive REST API and WebSocket interface for trading automation, market data access, and system management. The API follows RESTful principles with JSON request/response format.

**Base URL**: `https://localhost:8000/api/v1`  
**Authentication**: JWT Bearer tokens  
**Rate Limiting**: 1000 requests per minute per user  
**API Version**: v1.0  

---

## Authentication

### JWT Token Authentication
All API endpoints require authentication via JWT tokens in the Authorization header.

```http
Authorization: Bearer <jwt_token>
```

### Login Endpoint
```http
POST /auth/login
Content-Type: application/json

{
    "username": "string",
    "password": "string"
}
```

**Response**:
```json
{
    "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...",
    "token_type": "bearer",
    "expires_in": 3600,
    "refresh_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...",
    "user": {
        "id": 1,
        "username": "trader1",
        "email": "trader@example.com"
    }
}
```

---

## Market Data API

### Get Real-time Market Data
```http
GET /market-data/{symbol}
```

**Parameters**:
- `symbol` (path): Trading pair symbol (e.g., "BTCUSDT")
- `timeframe` (query): Optional timeframe filter ("1m", "5m", "1h", "1d")
- `limit` (query): Number of records to return (default: 100, max: 1000)

**Response**:
```json
{
    "symbol": "BTCUSDT",
    "data": [
        {
            "timestamp": "2025-09-18T09:00:00Z",
            "timeframe": "1m",
            "open": "43250.50",
            "high": "43280.75",
            "low": "43240.25",
            "close": "43275.00",
            "volume": "125.45",
            "quote_volume": "5425678.25"
        }
    ],
    "count": 1,
    "has_more": false
}
```

### Get Historical Market Data
```http
GET /market-data/{symbol}/history
```

**Parameters**:
- `symbol` (path): Trading pair symbol
- `timeframe` (query): Required timeframe ("1m", "5m", "15m", "1h", "4h", "1d", "1w")
- `start_time` (query): Start timestamp (ISO 8601)
- `end_time` (query): End timestamp (ISO 8601)
- `limit` (query): Number of records (max: 5000)

**Response**:
```json
{
    "symbol": "BTCUSDT",
    "timeframe": "1h",
    "data": [
        {
            "timestamp": "2025-09-18T08:00:00Z",
            "open": "43200.00",
            "high": "43350.00",
            "low": "43150.00",
            "close": "43275.00",
            "volume": "1250.75",
            "trades_count": 15420
        }
    ],
    "start_time": "2025-09-18T00:00:00Z",
    "end_time": "2025-09-18T09:00:00Z",
    "count": 9
}
```

### Get Order Book
```http
GET /market-data/{symbol}/orderbook
```

**Parameters**:
- `symbol` (path): Trading pair symbol
- `limit` (query): Number of price levels (default: 100, max: 1000)

**Response**:
```json
{
    "symbol": "BTCUSDT",
    "timestamp": "2025-09-18T09:00:00Z",
    "bids": [
        ["43270.00", "2.5"],
        ["43269.50", "1.2"]
    ],
    "asks": [
        ["43275.00", "1.8"],
        ["43275.50", "3.1"]
    ],
    "last_update_id": 1234567890
}
```

---

## Trading API

### Place Order
```http
POST /trading/orders
Content-Type: application/json

{
    "symbol": "BTCUSDT",
    "side": "BUY",
    "type": "LIMIT",
    "quantity": "0.1",
    "price": "43250.00",
    "time_in_force": "GTC",
    "client_order_id": "my_order_123"
}
```

**Response**:
```json
{
    "order_id": "12345678",
    "client_order_id": "my_order_123",
    "symbol": "BTCUSDT",
    "side": "BUY",
    "type": "LIMIT",
    "quantity": "0.1",
    "price": "43250.00",
    "status": "NEW",
    "time_in_force": "GTC",
    "created_at": "2025-09-18T09:00:00Z",
    "updated_at": "2025-09-18T09:00:00Z"
}
```

### Get Orders
```http
GET /trading/orders
```

**Parameters**:
- `symbol` (query): Filter by symbol
- `status` (query): Filter by status ("NEW", "FILLED", "CANCELED", "EXPIRED")
- `limit` (query): Number of records (default: 50, max: 500)
- `start_time` (query): Start timestamp filter
- `end_time` (query): End timestamp filter

**Response**:
```json
{
    "orders": [
        {
            "order_id": "12345678",
            "client_order_id": "my_order_123",
            "symbol": "BTCUSDT",
            "side": "BUY",
            "type": "LIMIT",
            "quantity": "0.1",
            "price": "43250.00",
            "executed_quantity": "0.05",
            "status": "PARTIALLY_FILLED",
            "created_at": "2025-09-18T09:00:00Z",
            "updated_at": "2025-09-18T09:01:00Z"
        }
    ],
    "count": 1,
    "has_more": false
}
```

### Cancel Order
```http
DELETE /trading/orders/{order_id}
```

**Response**:
```json
{
    "order_id": "12345678",
    "status": "CANCELED",
    "canceled_at": "2025-09-18T09:02:00Z"
}
```

### Get Positions
```http
GET /trading/positions
```

**Parameters**:
- `symbol` (query): Filter by symbol
- `side` (query): Filter by side ("LONG", "SHORT")

**Response**:
```json
{
    "positions": [
        {
            "symbol": "BTCUSDT",
            "side": "LONG",
            "size": "0.5",
            "entry_price": "43200.00",
            "mark_price": "43275.00",
            "unrealized_pnl": "37.50",
            "percentage": "0.87",
            "margin": "2160.00",
            "opened_at": "2025-09-18T08:30:00Z",
            "updated_at": "2025-09-18T09:00:00Z"
        }
    ],
    "count": 1
}
```

---

## Portfolio API

### Get Account Balance
```http
GET /portfolio/balance
```

**Response**:
```json
{
    "account_type": "futures",
    "total_wallet_balance": "10000.00",
    "available_balance": "8500.00",
    "locked_balance": "1500.00",
    "unrealized_pnl": "125.50",
    "margin_balance": "8625.50",
    "assets": [
        {
            "asset": "USDT",
            "wallet_balance": "10000.00",
            "unrealized_pnl": "125.50",
            "margin_balance": "8625.50",
            "available_balance": "8500.00",
            "locked_balance": "1500.00"
        }
    ],
    "updated_at": "2025-09-18T09:00:00Z"
}
```

### Get Portfolio Performance
```http
GET /portfolio/performance
```

**Parameters**:
- `period` (query): Time period ("1d", "7d", "30d", "90d", "1y", "all")
- `start_date` (query): Custom start date (ISO 8601)
- `end_date` (query): Custom end date (ISO 8601)

**Response**:
```json
{
    "period": "30d",
    "start_date": "2025-08-18T00:00:00Z",
    "end_date": "2025-09-18T09:00:00Z",
    "initial_balance": "10000.00",
    "final_balance": "10875.50",
    "total_return": "8.76",
    "total_return_percentage": "8.76",
    "daily_returns": [
        {
            "date": "2025-09-18",
            "return": "1.25",
            "return_percentage": "0.12",
            "balance": "10875.50"
        }
    ],
    "metrics": {
        "sharpe_ratio": "1.85",
        "sortino_ratio": "2.12",
        "max_drawdown": "-3.25",
        "max_drawdown_percentage": "-3.25",
        "volatility": "15.8",
        "win_rate": "65.5",
        "profit_factor": "1.45"
    }
}
```

---

## Strategy API

### Get Strategies
```http
GET /strategies
```

**Parameters**:
- `is_active` (query): Filter by active status (true/false)
- `strategy_type` (query): Filter by strategy type

**Response**:
```json
{
    "strategies": [
        {
            "id": 1,
            "name": "BTC Momentum Strategy",
            "description": "RSI-based momentum trading for BTC",
            "strategy_type": "momentum",
            "is_active": true,
            "is_paper_trading": false,
            "symbols": ["BTCUSDT"],
            "timeframes": ["1h", "4h"],
            "parameters": {
                "rsi_period": 14,
                "rsi_oversold": 30,
                "rsi_overbought": 70,
                "position_size": 0.1
            },
            "performance": {
                "total_return": "12.5",
                "sharpe_ratio": "1.8",
                "max_drawdown": "-5.2",
                "win_rate": "68.5"
            },
            "created_at": "2025-09-01T00:00:00Z",
            "updated_at": "2025-09-18T09:00:00Z"
        }
    ],
    "count": 1
}
```

### Create Strategy
```http
POST /strategies
Content-Type: application/json

{
    "name": "ETH Mean Reversion",
    "description": "Bollinger Bands mean reversion for ETH",
    "strategy_type": "mean_reversion",
    "symbols": ["ETHUSDT"],
    "timeframes": ["15m", "1h"],
    "parameters": {
        "bb_period": 20,
        "bb_std": 2,
        "position_size": 0.2,
        "stop_loss": 0.02,
        "take_profit": 0.04
    },
    "risk_parameters": {
        "max_position_size": 1000,
        "max_drawdown": 0.1
    },
    "is_paper_trading": true
}
```

**Response**:
```json
{
    "id": 2,
    "name": "ETH Mean Reversion",
    "description": "Bollinger Bands mean reversion for ETH",
    "strategy_type": "mean_reversion",
    "is_active": false,
    "is_paper_trading": true,
    "symbols": ["ETHUSDT"],
    "timeframes": ["15m", "1h"],
    "parameters": {
        "bb_period": 20,
        "bb_std": 2,
        "position_size": 0.2,
        "stop_loss": 0.02,
        "take_profit": 0.04
    },
    "created_at": "2025-09-18T09:00:00Z"
}
```

### Start/Stop Strategy
```http
POST /strategies/{strategy_id}/start
POST /strategies/{strategy_id}/stop
```

**Response**:
```json
{
    "strategy_id": 1,
    "status": "started",
    "message": "Strategy started successfully",
    "started_at": "2025-09-18T09:00:00Z"
}
```

---

## Machine Learning API

### Get ML Models
```http
GET /ml/models
```

**Parameters**:
- `symbol` (query): Filter by symbol
- `model_type` (query): Filter by model type
- `is_active` (query): Filter by active status

**Response**:
```json
{
    "models": [
        {
            "id": 1,
            "name": "BTC Price Predictor v2.1",
            "model_type": "lstm",
            "symbol": "BTCUSDT",
            "timeframe": "1h",
            "version": "2.1",
            "is_active": true,
            "validation_score": 0.8542,
            "test_score": 0.8234,
            "features": [
                "close_price",
                "volume",
                "rsi_14",
                "macd_signal",
                "bb_upper",
                "bb_lower"
            ],
            "created_at": "2025-09-15T00:00:00Z",
            "updated_at": "2025-09-18T09:00:00Z"
        }
    ],
    "count": 1
}
```

### Get Predictions
```http
GET /ml/predictions
```

**Parameters**:
- `symbol` (query): Filter by symbol
- `model_id` (query): Filter by model ID
- `prediction_type` (query): Filter by prediction type
- `start_time` (query): Start timestamp filter
- `end_time` (query): End timestamp filter

**Response**:
```json
{
    "predictions": [
        {
            "id": 12345,
            "model_id": 1,
            "symbol": "BTCUSDT",
            "prediction_type": "price_direction",
            "predicted_value": "43350.00",
            "confidence": 0.85,
            "actual_value": "43275.00",
            "error": "-75.00",
            "prediction_timestamp": "2025-09-18T10:00:00Z",
            "created_at": "2025-09-18T09:00:00Z"
        }
    ],
    "count": 1,
    "accuracy_metrics": {
        "directional_accuracy": 0.68,
        "mean_absolute_error": 125.50,
        "root_mean_square_error": 185.25
    }
}
```

### Train Model
```http
POST /ml/models/{model_id}/train
Content-Type: application/json

{
    "start_date": "2025-01-01T00:00:00Z",
    "end_date": "2025-08-31T23:59:59Z",
    "validation_split": 0.2,
    "hyperparameters": {
        "learning_rate": 0.001,
        "batch_size": 32,
        "epochs": 100,
        "lstm_units": 50
    }
}
```

**Response**:
```json
{
    "training_id": "train_123456",
    "model_id": 1,
    "status": "started",
    "estimated_duration": "2 hours",
    "started_at": "2025-09-18T09:00:00Z"
}
```

---

## Backtesting API

### Create Backtest
```http
POST /backtesting/backtests
Content-Type: application/json

{
    "name": "BTC Momentum Backtest Q3 2025",
    "strategy_id": 1,
    "start_date": "2025-07-01T00:00:00Z",
    "end_date": "2025-09-30T23:59:59Z",
    "initial_capital": 10000,
    "symbols": ["BTCUSDT"],
    "timeframes": ["1h"],
    "parameters": {
        "commission": 0.001,
        "slippage": 0.0005,
        "market_impact": true
    }
}
```

**Response**:
```json
{
    "backtest_id": 123,
    "name": "BTC Momentum Backtest Q3 2025",
    "status": "queued",
    "estimated_duration": "15 minutes",
    "created_at": "2025-09-18T09:00:00Z"
}
```

### Get Backtest Results
```http
GET /backtesting/backtests/{backtest_id}/results
```

**Response**:
```json
{
    "backtest_id": 123,
    "name": "BTC Momentum Backtest Q3 2025",
    "status": "completed",
    "start_date": "2025-07-01T00:00:00Z",
    "end_date": "2025-09-30T23:59:59Z",
    "initial_capital": 10000.00,
    "final_capital": 11250.00,
    "results": {
        "total_return": 1250.00,
        "total_return_percentage": 12.50,
        "annual_return": 50.00,
        "volatility": 18.5,
        "sharpe_ratio": 1.85,
        "sortino_ratio": 2.12,
        "max_drawdown": -8.5,
        "max_drawdown_percentage": -8.5,
        "calmar_ratio": 5.88,
        "win_rate": 65.5,
        "profit_factor": 1.45,
        "total_trades": 156,
        "winning_trades": 102,
        "losing_trades": 54,
        "avg_trade_return": 8.01,
        "avg_win": 18.50,
        "avg_loss": -12.25,
        "largest_win": 125.00,
        "largest_loss": -85.50
    },
    "equity_curve": [
        {
            "timestamp": "2025-07-01T00:00:00Z",
            "equity": 10000.00,
            "drawdown": 0.00
        }
    ],
    "trades": [
        {
            "entry_time": "2025-07-01T10:00:00Z",
            "exit_time": "2025-07-01T14:00:00Z",
            "symbol": "BTCUSDT",
            "side": "BUY",
            "quantity": 0.1,
            "entry_price": 30000.00,
            "exit_price": 30150.00,
            "pnl": 15.00,
            "pnl_percentage": 0.50,
            "commission": 3.00
        }
    ],
    "completed_at": "2025-09-18T09:15:00Z"
}
```

---

## Risk Management API

### Get Risk Limits
```http
GET /risk/limits
```

**Response**:
```json
{
    "limits": [
        {
            "id": 1,
            "limit_type": "max_position_size",
            "symbol": "BTCUSDT",
            "limit_value": 1000.00,
            "current_value": 500.00,
            "utilization": 50.0,
            "is_active": true
        },
        {
            "id": 2,
            "limit_type": "daily_loss_limit",
            "limit_value": 500.00,
            "current_value": 125.50,
            "utilization": 25.1,
            "is_active": true
        }
    ],
    "count": 2
}
```

### Update Risk Limits
```http
PUT /risk/limits/{limit_id}
Content-Type: application/json

{
    "limit_value": 1500.00,
    "is_active": true
}
```

### Get Risk Events
```http
GET /risk/events
```

**Parameters**:
- `severity` (query): Filter by severity ("LOW", "MEDIUM", "HIGH", "CRITICAL")
- `start_time` (query): Start timestamp filter
- `end_time` (query): End timestamp filter

**Response**:
```json
{
    "events": [
        {
            "id": 1001,
            "event_type": "position_limit_exceeded",
            "severity": "HIGH",
            "symbol": "BTCUSDT",
            "description": "Position size exceeded 80% of limit",
            "risk_value": 850.00,
            "limit_value": 1000.00,
            "action_taken": "position_reduced",
            "timestamp": "2025-09-18T08:45:00Z"
        }
    ],
    "count": 1
}
```

---

## WebSocket API

### Connection
```
wss://localhost:8000/ws
```

### Authentication
Send authentication message after connection:
```json
{
    "type": "auth",
    "token": "your_jwt_token"
}
```

### Subscribe to Market Data
```json
{
    "type": "subscribe",
    "channel": "market_data",
    "symbol": "BTCUSDT",
    "timeframe": "1m"
}
```

**Market Data Stream**:
```json
{
    "type": "market_data",
    "channel": "market_data",
    "symbol": "BTCUSDT",
    "timeframe": "1m",
    "data": {
        "timestamp": "2025-09-18T09:00:00Z",
        "open": "43250.50",
        "high": "43280.75",
        "low": "43240.25",
        "close": "43275.00",
        "volume": "125.45"
    }
}
```

### Subscribe to Trading Updates
```json
{
    "type": "subscribe",
    "channel": "trading_updates"
}
```

**Trading Update Stream**:
```json
{
    "type": "trading_update",
    "channel": "trading_updates",
    "data": {
        "order_id": "12345678",
        "status": "FILLED",
        "executed_quantity": "0.1",
        "avg_price": "43260.00",
        "timestamp": "2025-09-18T09:00:00Z"
    }
}
```

### Subscribe to Portfolio Updates
```json
{
    "type": "subscribe",
    "channel": "portfolio_updates"
}
```

**Portfolio Update Stream**:
```json
{
    "type": "portfolio_update",
    "channel": "portfolio_updates",
    "data": {
        "total_balance": "10875.50",
        "available_balance": "8500.00",
        "unrealized_pnl": "125.50",
        "positions": [
            {
                "symbol": "BTCUSDT",
                "side": "LONG",
                "size": "0.5",
                "unrealized_pnl": "37.50"
            }
        ],
        "timestamp": "2025-09-18T09:00:00Z"
    }
}
```

---

## Error Handling

### HTTP Status Codes
- `200 OK`: Successful request
- `201 Created`: Resource created successfully
- `400 Bad Request`: Invalid request parameters
- `401 Unauthorized`: Authentication required
- `403 Forbidden`: Insufficient permissions
- `404 Not Found`: Resource not found
- `429 Too Many Requests`: Rate limit exceeded
- `500 Internal Server Error`: Server error

### Error Response Format
```json
{
    "error": {
        "code": "INVALID_SYMBOL",
        "message": "The specified symbol is not supported",
        "details": {
            "symbol": "INVALID",
            "supported_symbols": ["BTCUSDT", "ETHUSDT"]
        },
        "timestamp": "2025-09-18T09:00:00Z",
        "request_id": "req_123456789"
    }
}
```

### Common Error Codes
- `INVALID_SYMBOL`: Unsupported trading symbol
- `INSUFFICIENT_BALANCE`: Not enough balance for operation
- `INVALID_ORDER_TYPE`: Unsupported order type
- `POSITION_NOT_FOUND`: Position does not exist
- `STRATEGY_NOT_FOUND`: Strategy does not exist
- `MODEL_NOT_FOUND`: ML model does not exist
- `BACKTEST_NOT_FOUND`: Backtest does not exist
- `RATE_LIMIT_EXCEEDED`: Too many requests
- `INVALID_TIMEFRAME`: Unsupported timeframe
- `INVALID_DATE_RANGE`: Invalid date range specified

---

## Rate Limiting

### Rate Limits by Endpoint Category
- **Market Data**: 100 requests per minute
- **Trading**: 50 requests per minute
- **Portfolio**: 60 requests per minute
- **Strategy Management**: 30 requests per minute
- **ML Operations**: 20 requests per minute
- **Backtesting**: 10 requests per minute

### Rate Limit Headers
```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1695024000
X-RateLimit-Retry-After: 60
```

---

## SDK Examples

### Python SDK Example
```python
from ai_trading_client import TradingClient

client = TradingClient(
    base_url="https://localhost:8000/api/v1",
    api_key="your_api_key"
)

# Get market data
market_data = client.market_data.get_current("BTCUSDT")

# Place order
order = client.trading.place_order(
    symbol="BTCUSDT",
    side="BUY",
    type="LIMIT",
    quantity=0.1,
    price=43250.00
)

# Get portfolio
portfolio = client.portfolio.get_balance()
```

### JavaScript SDK Example
```javascript
import { TradingClient } from 'ai-trading-js-client';

const client = new TradingClient({
    baseUrl: 'https://localhost:8000/api/v1',
    apiKey: 'your_api_key'
});

// Get market data
const marketData = await client.marketData.getCurrent('BTCUSDT');

// Place order
const order = await client.trading.placeOrder({
    symbol: 'BTCUSDT',
    side: 'BUY',
    type: 'LIMIT',
    quantity: 0.1,
    price: 43250.00
});

// WebSocket connection
const ws = client.websocket.connect();
ws.subscribe('market_data', { symbol: 'BTCUSDT' });
```

This comprehensive API documentation provides all the necessary endpoints and examples for integrating with the AI Cryptocurrency Trading System.