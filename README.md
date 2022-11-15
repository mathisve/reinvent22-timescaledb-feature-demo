# reinvent22-timescaledb-feature-demo

## Launch Docker container
```
docker run -d --name timescaledb -p 5432:5432 -e POSTGRES_PASSWORD=password timescale/timescaledb-ha:pg14-latest
```

```
psql -h localhost -U postgres
```


## Create hypertables
```
CREATE TABLE stocks_real_time (
  time TIMESTAMPTZ NOT NULL,
  symbol TEXT NOT NULL,
  price DOUBLE PRECISION NULL,
  day_volume INT NULL
);
```

```
SELECT create_hypertable('stocks_real_time','time');
```

```
CREATE INDEX ix_symbol_time ON stocks_real_time (symbol, time DESC);
```

```
CREATE TABLE company (
  symbol TEXT NOT NULL,
  name TEXT NOT NULL
);
```

## Insert data
```
\COPY stocks_real_time from './small_tutorial_sample_tick.csv' DELIMITER ',' CSV HEADER;

\COPY company from './tutorial_sample_company.csv' DELIMITER ',' CSV HEADER;
```

## Query data
```
SELECT * FROM company;
SELECT * FROM stocks_real_time limit 50;

SELECT 
    symbol, 
    first(price, time), 
    last(price, time) 
FROM stocks_real_time 
GROUP BY symbol 
ORDER BY symbol;

SELECT 
    time_bucket('30 minutes', time) AS bucket,
    symbol, 
    avg(price)
FROM stocks_real_time
WHERE symbol = 'AMD'
GROUP BY bucket, symbol
ORDER BY bucket, symbol; 
```

## CAGGs

```
SELECT
    time_bucket('30 minutes', "time") AS bucket,
    symbol,
    max(price) AS high,
    min(price) AS low,
    first(price, time) AS open,
    last(price, time) AS close
FROM stocks_real_time srt
WHERE symbol = 'AMD'
GROUP BY bucket, symbol
ORDER BY bucket DESC, symbol;


CREATE MATERIALIZED VIEW stock_candlestick_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', "time") AS day,
    symbol,
    max(price) AS high,
    min(price) AS low,
    first(price, time) AS open,
    last(price, time) AS close
FROM stocks_real_time srt
GROUP BY day, symbol;

SELECT add_continuous_aggregate_policy('stock_candlestick_hourly',
  start_offset => INTERVAL '3 hours',
  end_offset => INTERVAL '15 minutes',
  schedule_interval => INTERVAL '1 hour');
```

talk about compression, data retention, hyperfunctions