# Database Schema Design - AI Cryptocurrency Trading System

## Overview

The database schema is designed to efficiently store and retrieve high-frequency trading data, market information, and machine learning features. The system uses PostgreSQL with TimescaleDB extension for time-series data optimization.

## Schema Architecture

### Time-Series Data Strategy
- **TimescaleDB**: Optimized for time-series data with automatic partitioning
- **Hypertables**: Used for market data and trading records
- **Compression**: Automatic compression for historical data
- **Retention Policies**: Automated data lifecycle management

### Indexing Strategy
- **Primary Indexes**: Time-based partitioning keys
- **Secondary Indexes**: Symbol, strategy, and user-based queries
- **Composite Indexes**: Multi-column indexes for complex queries
- **Partial Indexes**: Conditional indexes for specific data subsets

---

## Core Tables

### 1. Market Data Tables

#### market_data (Hypertable)
```sql
CREATE TABLE market_data (
    id BIGSERIAL,
    symbol VARCHAR(20) NOT NULL,
    timestamp TIMESTAMPTZ NOT NULL,
    timeframe VARCHAR(10) NOT NULL,
    open DECIMAL(20,8) NOT NULL,
    high DECIMAL(20,8) NOT NULL,
    low DECIMAL(20,8) NOT NULL,
    close DECIMAL(20,8) NOT NULL,
    volume DECIMAL(20,8) NOT NULL,
    quote_volume DECIMAL(20,8),
    trades_count INTEGER,
    taker_buy_volume DECIMAL(20,8),
    taker_buy_quote_volume DECIMAL(20,8),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (timestamp, symbol, timeframe)
);

-- Convert to hypertable
SELECT create_hypertable('market_data', 'timestamp', chunk_time_interval => INTERVAL '1 day');

-- Indexes
CREATE INDEX idx_market_data_symbol_time ON market_data (symbol, timestamp DESC);
CREATE INDEX idx_market_data_timeframe ON market_data (timeframe, timestamp DESC);
```

#### order_book_snapshots (Hypertable)
```sql
CREATE TABLE order_book_snapshots (
    id BIGSERIAL,
    symbol VARCHAR(20) NOT NULL,
    timestamp TIMESTAMPTZ NOT NULL,
    bids JSONB NOT NULL,
    asks JSONB NOT NULL,
    last_update_id BIGINT,
    PRIMARY KEY (timestamp, symbol)
);

SELECT create_hypertable('order_book_snapshots', 'timestamp', chunk_time_interval => INTERVAL '1 hour');
CREATE INDEX idx_orderbook_symbol_time ON order_book_snapshots (symbol, timestamp DESC);
```

#### trades (Hypertable)
```sql
CREATE TABLE trades (
    id BIGSERIAL,
    symbol VARCHAR(20) NOT NULL,
    timestamp TIMESTAMPTZ NOT NULL,
    trade_id BIGINT NOT NULL,
    price DECIMAL(20,8) NOT NULL,
    quantity DECIMAL(20,8) NOT NULL,
    quote_quantity DECIMAL(20,8) NOT NULL,
    is_buyer_maker BOOLEAN NOT NULL,
    PRIMARY KEY (timestamp, symbol, trade_id)
);

SELECT create_hypertable('trades', 'timestamp', chunk_time_interval => INTERVAL '1 hour');
CREATE INDEX idx_trades_symbol_time ON trades (symbol, timestamp DESC);
```

### 2. Trading Tables

#### users
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    api_key_hash VARCHAR(255),
    api_secret_hash VARCHAR(255),
    is_active BOOLEAN DEFAULT true,
    risk_profile VARCHAR(20) DEFAULT 'moderate',
    max_position_size DECIMAL(10,2) DEFAULT 10000.00,
    max_daily_loss DECIMAL(10,2) DEFAULT 1000.00,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_users_username ON users (username);
CREATE INDEX idx_users_email ON users (email);
```

#### accounts
```sql
CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    exchange VARCHAR(20) NOT NULL,
    account_type VARCHAR(20) DEFAULT 'spot',
    balance DECIMAL(20,8) DEFAULT 0,
    available_balance DECIMAL(20,8) DEFAULT 0,
    locked_balance DECIMAL(20,8) DEFAULT 0,
    total_wallet_balance DECIMAL(20,8) DEFAULT 0,
    unrealized_pnl DECIMAL(20,8) DEFAULT 0,
    margin_balance DECIMAL(20,8) DEFAULT 0,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_accounts_user_id ON accounts (user_id);
CREATE INDEX idx_accounts_exchange ON accounts (exchange);
```

#### orders (Hypertable)
```sql
CREATE TABLE orders (
    id BIGSERIAL,
    user_id INTEGER NOT NULL,
    account_id INTEGER NOT NULL,
    order_id VARCHAR(50) NOT NULL,
    client_order_id VARCHAR(50),
    symbol VARCHAR(20) NOT NULL,
    side VARCHAR(10) NOT NULL CHECK (side IN ('BUY', 'SELL')),
    type VARCHAR(20) NOT NULL CHECK (type IN ('MARKET', 'LIMIT', 'STOP', 'STOP_MARKET', 'TAKE_PROFIT', 'TAKE_PROFIT_MARKET')),
    time_in_force VARCHAR(10) DEFAULT 'GTC',
    quantity DECIMAL(20,8) NOT NULL,
    price DECIMAL(20,8),
    stop_price DECIMAL(20,8),
    executed_quantity DECIMAL(20,8) DEFAULT 0,
    cumulative_quote_quantity DECIMAL(20,8) DEFAULT 0,
    status VARCHAR(20) NOT NULL DEFAULT 'NEW',
    strategy_id INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (created_at, user_id, order_id)
);

SELECT create_hypertable('orders', 'created_at', chunk_time_interval => INTERVAL '1 day');
CREATE INDEX idx_orders_user_id ON orders (user_id, created_at DESC);
CREATE INDEX idx_orders_symbol ON orders (symbol, created_at DESC);
CREATE INDEX idx_orders_status ON orders (status, created_at DESC);
CREATE INDEX idx_orders_strategy ON orders (strategy_id, created_at DESC);
```

#### positions (Hypertable)
```sql
CREATE TABLE positions (
    id BIGSERIAL,
    user_id INTEGER NOT NULL,
    account_id INTEGER NOT NULL,
    symbol VARCHAR(20) NOT NULL,
    side VARCHAR(10) NOT NULL CHECK (side IN ('LONG', 'SHORT')),
    size DECIMAL(20,8) NOT NULL,
    entry_price DECIMAL(20,8) NOT NULL,
    mark_price DECIMAL(20,8),
    unrealized_pnl DECIMAL(20,8) DEFAULT 0,
    percentage DECIMAL(10,4) DEFAULT 0,
    margin DECIMAL(20,8) DEFAULT 0,
    isolated_margin DECIMAL(20,8) DEFAULT 0,
    is_auto_add_margin BOOLEAN DEFAULT false,
    position_side VARCHAR(10) DEFAULT 'BOTH',
    max_notional_value DECIMAL(20,8),
    strategy_id INTEGER,
    opened_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    closed_at TIMESTAMPTZ,
    PRIMARY KEY (opened_at, user_id, symbol, side)
);

SELECT create_hypertable('positions', 'opened_at', chunk_time_interval => INTERVAL '1 day');
CREATE INDEX idx_positions_user_id ON positions (user_id, opened_at DESC);
CREATE INDEX idx_positions_symbol ON positions (symbol, opened_at DESC);
CREATE INDEX idx_positions_active ON positions (user_id, symbol) WHERE closed_at IS NULL;
```

#### trades_executed (Hypertable)
```sql
CREATE TABLE trades_executed (
    id BIGSERIAL,
    user_id INTEGER NOT NULL,
    account_id INTEGER NOT NULL,
    order_id VARCHAR(50) NOT NULL,
    trade_id VARCHAR(50) NOT NULL,
    symbol VARCHAR(20) NOT NULL,
    side VARCHAR(10) NOT NULL,
    quantity DECIMAL(20,8) NOT NULL,
    price DECIMAL(20,8) NOT NULL,
    quote_quantity DECIMAL(20,8) NOT NULL,
    commission DECIMAL(20,8) DEFAULT 0,
    commission_asset VARCHAR(10),
    realized_pnl DECIMAL(20,8) DEFAULT 0,
    strategy_id INTEGER,
    executed_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (executed_at, user_id, trade_id)
);

SELECT create_hypertable('trades_executed', 'executed_at', chunk_time_interval => INTERVAL '1 day');
CREATE INDEX idx_trades_executed_user_id ON trades_executed (user_id, executed_at DESC);
CREATE INDEX idx_trades_executed_symbol ON trades_executed (symbol, executed_at DESC);
CREATE INDEX idx_trades_executed_order ON trades_executed (order_id, executed_at DESC);
```

### 3. Strategy and ML Tables

#### strategies
```sql
CREATE TABLE strategies (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    strategy_type VARCHAR(50) NOT NULL,
    parameters JSONB NOT NULL,
    risk_parameters JSONB,
    is_active BOOLEAN DEFAULT false,
    is_paper_trading BOOLEAN DEFAULT true,
    max_position_size DECIMAL(20,8),
    max_drawdown DECIMAL(5,2),
    symbols TEXT[] DEFAULT '{}',
    timeframes TEXT[] DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_strategies_user_id ON strategies (user_id);
CREATE INDEX idx_strategies_type ON strategies (strategy_type);
CREATE INDEX idx_strategies_active ON strategies (is_active) WHERE is_active = true;
```

#### strategy_performance (Hypertable)
```sql
CREATE TABLE strategy_performance (
    id BIGSERIAL,
    strategy_id INTEGER NOT NULL,
    timestamp TIMESTAMPTZ NOT NULL,
    total_return DECIMAL(10,4),
    daily_return DECIMAL(10,4),
    cumulative_return DECIMAL(10,4),
    sharpe_ratio DECIMAL(8,4),
    max_drawdown DECIMAL(8,4),
    win_rate DECIMAL(5,2),
    profit_factor DECIMAL(8,4),
    total_trades INTEGER DEFAULT 0,
    winning_trades INTEGER DEFAULT 0,
    losing_trades INTEGER DEFAULT 0,
    avg_win DECIMAL(10,4),
    avg_loss DECIMAL(10,4),
    largest_win DECIMAL(10,4),
    largest_loss DECIMAL(10,4),
    PRIMARY KEY (timestamp, strategy_id)
);

SELECT create_hypertable('strategy_performance', 'timestamp', chunk_time_interval => INTERVAL '1 day');
CREATE INDEX idx_strategy_performance_id ON strategy_performance (strategy_id, timestamp DESC);
```

#### ml_models
```sql
CREATE TABLE ml_models (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    model_type VARCHAR(50) NOT NULL,
    version VARCHAR(20) NOT NULL,
    symbol VARCHAR(20) NOT NULL,
    timeframe VARCHAR(10) NOT NULL,
    features JSONB NOT NULL,
    hyperparameters JSONB,
    training_data_start TIMESTAMPTZ,
    training_data_end TIMESTAMPTZ,
    validation_score DECIMAL(8,6),
    test_score DECIMAL(8,6),
    model_path VARCHAR(255),
    is_active BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_ml_models_symbol ON ml_models (symbol, timeframe);
CREATE INDEX idx_ml_models_active ON ml_models (is_active) WHERE is_active = true;
CREATE UNIQUE INDEX idx_ml_models_unique ON ml_models (name, version, symbol, timeframe);
```

#### predictions (Hypertable)
```sql
CREATE TABLE predictions (
    id BIGSERIAL,
    model_id INTEGER NOT NULL,
    symbol VARCHAR(20) NOT NULL,
    timestamp TIMESTAMPTZ NOT NULL,
    prediction_timestamp TIMESTAMPTZ NOT NULL,
    prediction_type VARCHAR(20) NOT NULL,
    predicted_value DECIMAL(20,8) NOT NULL,
    confidence DECIMAL(5,4),
    actual_value DECIMAL(20,8),
    error DECIMAL(20,8),
    features JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (timestamp, model_id, prediction_timestamp)
);

SELECT create_hypertable('predictions', 'timestamp', chunk_time_interval => INTERVAL '1 day');
CREATE INDEX idx_predictions_model ON predictions (model_id, timestamp DESC);
CREATE INDEX idx_predictions_symbol ON predictions (symbol, prediction_timestamp DESC);
```

#### technical_indicators (Hypertable)
```sql
CREATE TABLE technical_indicators (
    id BIGSERIAL,
    symbol VARCHAR(20) NOT NULL,
    timeframe VARCHAR(10) NOT NULL,
    timestamp TIMESTAMPTZ NOT NULL,
    indicator_name VARCHAR(50) NOT NULL,
    value DECIMAL(20,8),
    additional_values JSONB,
    PRIMARY KEY (timestamp, symbol, timeframe, indicator_name)
);

SELECT create_hypertable('technical_indicators', 'timestamp', chunk_time_interval => INTERVAL '1 day');
CREATE INDEX idx_technical_indicators_symbol ON technical_indicators (symbol, timeframe, indicator_name, timestamp DESC);
```

### 4. Risk Management Tables

#### risk_limits
```sql
CREATE TABLE risk_limits (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    limit_type VARCHAR(50) NOT NULL,
    symbol VARCHAR(20),
    strategy_id INTEGER,
    limit_value DECIMAL(20,8) NOT NULL,
    current_value DECIMAL(20,8) DEFAULT 0,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_risk_limits_user_id ON risk_limits (user_id);
CREATE INDEX idx_risk_limits_type ON risk_limits (limit_type);
```

#### risk_events (Hypertable)
```sql
CREATE TABLE risk_events (
    id BIGSERIAL,
    user_id INTEGER NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    severity VARCHAR(20) NOT NULL,
    symbol VARCHAR(20),
    strategy_id INTEGER,
    description TEXT,
    risk_value DECIMAL(20,8),
    limit_value DECIMAL(20,8),
    action_taken VARCHAR(100),
    timestamp TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (timestamp, user_id, event_type)
);

SELECT create_hypertable('risk_events', 'timestamp', chunk_time_interval => INTERVAL '1 day');
CREATE INDEX idx_risk_events_user ON risk_events (user_id, timestamp DESC);
CREATE INDEX idx_risk_events_severity ON risk_events (severity, timestamp DESC);
```

### 5. Backtesting Tables

#### backtests
```sql
CREATE TABLE backtests (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    strategy_id INTEGER REFERENCES strategies(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    start_date TIMESTAMPTZ NOT NULL,
    end_date TIMESTAMPTZ NOT NULL,
    initial_capital DECIMAL(20,8) NOT NULL,
    symbols TEXT[] NOT NULL,
    timeframes TEXT[] NOT NULL,
    parameters JSONB,
    status VARCHAR(20) DEFAULT 'PENDING',
    progress INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ
);

CREATE INDEX idx_backtests_user_id ON backtests (user_id);
CREATE INDEX idx_backtests_strategy ON backtests (strategy_id);
CREATE INDEX idx_backtests_status ON backtests (status);
```

#### backtest_results
```sql
CREATE TABLE backtest_results (
    id SERIAL PRIMARY KEY,
    backtest_id INTEGER REFERENCES backtests(id) ON DELETE CASCADE,
    total_return DECIMAL(10,4),
    annual_return DECIMAL(10,4),
    volatility DECIMAL(10,4),
    sharpe_ratio DECIMAL(8,4),
    sortino_ratio DECIMAL(8,4),
    max_drawdown DECIMAL(8,4),
    max_drawdown_duration INTEGER,
    calmar_ratio DECIMAL(8,4),
    win_rate DECIMAL(5,2),
    profit_factor DECIMAL(8,4),
    total_trades INTEGER,
    winning_trades INTEGER,
    losing_trades INTEGER,
    avg_trade_return DECIMAL(10,4),
    avg_win DECIMAL(10,4),
    avg_loss DECIMAL(10,4),
    largest_win DECIMAL(10,4),
    largest_loss DECIMAL(10,4),
    avg_trade_duration INTERVAL,
    final_capital DECIMAL(20,8),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_backtest_results_backtest ON backtest_results (backtest_id);
```

#### backtest_trades (Hypertable)
```sql
CREATE TABLE backtest_trades (
    id BIGSERIAL,
    backtest_id INTEGER NOT NULL,
    symbol VARCHAR(20) NOT NULL,
    side VARCHAR(10) NOT NULL,
    quantity DECIMAL(20,8) NOT NULL,
    entry_price DECIMAL(20,8) NOT NULL,
    exit_price DECIMAL(20,8),
    entry_time TIMESTAMPTZ NOT NULL,
    exit_time TIMESTAMPTZ,
    pnl DECIMAL(20,8),
    pnl_percentage DECIMAL(10,4),
    commission DECIMAL(20,8) DEFAULT 0,
    strategy_signal VARCHAR(50),
    PRIMARY KEY (entry_time, backtest_id, symbol)
);

SELECT create_hypertable('backtest_trades', 'entry_time', chunk_time_interval => INTERVAL '1 month');
CREATE INDEX idx_backtest_trades_backtest ON backtest_trades (backtest_id, entry_time DESC);
```

### 6. System Tables

#### system_logs (Hypertable)
```sql
CREATE TABLE system_logs (
    id BIGSERIAL,
    timestamp TIMESTAMPTZ DEFAULT NOW(),
    level VARCHAR(10) NOT NULL,
    component VARCHAR(50) NOT NULL,
    message TEXT NOT NULL,
    user_id INTEGER,
    session_id VARCHAR(100),
    ip_address INET,
    additional_data JSONB,
    PRIMARY KEY (timestamp, component, level)
);

SELECT create_hypertable('system_logs', 'timestamp', chunk_time_interval => INTERVAL '1 day');
CREATE INDEX idx_system_logs_level ON system_logs (level, timestamp DESC);
CREATE INDEX idx_system_logs_component ON system_logs (component, timestamp DESC);
CREATE INDEX idx_system_logs_user ON system_logs (user_id, timestamp DESC) WHERE user_id IS NOT NULL;
```

#### api_usage (Hypertable)
```sql
CREATE TABLE api_usage (
    id BIGSERIAL,
    timestamp TIMESTAMPTZ DEFAULT NOW(),
    user_id INTEGER,
    endpoint VARCHAR(255) NOT NULL,
    method VARCHAR(10) NOT NULL,
    status_code INTEGER NOT NULL,
    response_time INTEGER,
    ip_address INET,
    user_agent TEXT,
    request_size INTEGER,
    response_size INTEGER,
    PRIMARY KEY (timestamp, user_id, endpoint)
);

SELECT create_hypertable('api_usage', 'timestamp', chunk_time_interval => INTERVAL '1 day');
CREATE INDEX idx_api_usage_user ON api_usage (user_id, timestamp DESC);
CREATE INDEX idx_api_usage_endpoint ON api_usage (endpoint, timestamp DESC);
```

---

## Data Retention Policies

### Automatic Data Retention
```sql
-- Market data retention (2 years for minute data, 5 years for daily)
SELECT add_retention_policy('market_data', INTERVAL '2 years');

-- Order book snapshots (30 days)
SELECT add_retention_policy('order_book_snapshots', INTERVAL '30 days');

-- Individual trades (90 days)
SELECT add_retention_policy('trades', INTERVAL '90 days');

-- System logs (1 year)
SELECT add_retention_policy('system_logs', INTERVAL '1 year');

-- API usage logs (6 months)
SELECT add_retention_policy('api_usage', INTERVAL '6 months');
```

### Data Compression
```sql
-- Enable compression for older data
ALTER TABLE market_data SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'symbol,timeframe',
    timescaledb.compress_orderby = 'timestamp DESC'
);

SELECT add_compression_policy('market_data', INTERVAL '7 days');
```

---

## Performance Optimization

### Materialized Views
```sql
-- Daily OHLCV aggregation
CREATE MATERIALIZED VIEW daily_ohlcv AS
SELECT 
    symbol,
    date_trunc('day', timestamp) as date,
    first(open, timestamp) as open,
    max(high) as high,
    min(low) as low,
    last(close, timestamp) as close,
    sum(volume) as volume
FROM market_data 
WHERE timeframe = '1m'
GROUP BY symbol, date_trunc('day', timestamp);

CREATE UNIQUE INDEX ON daily_ohlcv (symbol, date);

-- Refresh policy
SELECT add_continuous_aggregate_policy('daily_ohlcv',
    start_offset => INTERVAL '1 month',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');
```

### Partitioning Strategy
```sql
-- Partition large tables by user_id for better query performance
CREATE TABLE orders_partitioned (LIKE orders INCLUDING ALL)
PARTITION BY HASH (user_id);

-- Create partitions
CREATE TABLE orders_p0 PARTITION OF orders_partitioned FOR VALUES WITH (modulus 4, remainder 0);
CREATE TABLE orders_p1 PARTITION OF orders_partitioned FOR VALUES WITH (modulus 4, remainder 1);
CREATE TABLE orders_p2 PARTITION OF orders_partitioned FOR VALUES WITH (modulus 4, remainder 2);
CREATE TABLE orders_p3 PARTITION OF orders_partitioned FOR VALUES WITH (modulus 4, remainder 3);
```

---

## Security Considerations

### Row Level Security
```sql
-- Enable RLS for user data
ALTER TABLE accounts ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE positions ENABLE ROW LEVEL SECURITY;
ALTER TABLE trades_executed ENABLE ROW LEVEL SECURITY;

-- Create policies
CREATE POLICY user_accounts_policy ON accounts
    FOR ALL TO authenticated_users
    USING (user_id = current_user_id());

CREATE POLICY user_orders_policy ON orders
    FOR ALL TO authenticated_users
    USING (user_id = current_user_id());
```

### Data Encryption
```sql
-- Encrypt sensitive columns
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Function to encrypt API keys
CREATE OR REPLACE FUNCTION encrypt_api_key(key TEXT)
RETURNS TEXT AS $$
BEGIN
    RETURN encode(encrypt(key::bytea, 'encryption_key', 'aes'), 'base64');
END;
$$ LANGUAGE plpgsql;
```

---

## Monitoring and Maintenance

### Database Health Checks
```sql
-- Monitor table sizes
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size
FROM pg_tables 
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Monitor query performance
SELECT 
    query,
    calls,
    total_time,
    mean_time,
    rows
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 10;
```

### Backup Strategy
```sql
-- Continuous archiving setup
archive_mode = on
archive_command = 'cp %p /backup/archive/%f'
wal_level = replica

-- Point-in-time recovery setup
SELECT pg_start_backup('daily_backup');
-- File system backup
SELECT pg_stop_backup();
```

This database schema provides a robust foundation for the AI cryptocurrency trading system, with optimizations for time-series data, proper indexing strategies, and comprehensive data retention policies.