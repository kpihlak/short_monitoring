# TICK Stack with Chuck Norris Monitor - Part 2

## Setting Up Kapacitor (Optional Alerting)

Kapacitor is the alerting component of the TICK stack. While the rest of the stack focuses on collecting and visualizing data, Kapacitor watches for specific conditions and can notify you when they occur.

### Configuring Kapacitor

First, let's finish setting up the Kapacitor configuration file:

```bash
cat > ~/tick-stack/kapacitor/kapacitor.conf << 'EOL'
# The hostname of this node.
hostname = "localhost"
# Directory for storing task state
data_dir = "/var/lib/kapacitor"

[http]
  # HTTP API Server
  bind-address = ":9092"
  auth-enabled = false
  log-enabled = true
  write-tracing = false
  pprof-enabled = false
  https-enabled = false
  https-certificate = "/etc/ssl/kapacitor.pem"

[replay]
  # Location for replay files
  dir = "/var/lib/kapacitor/replay"

[storage]
  # Where to store Kapacitor database
  boltdb = "/var/lib/kapacitor/kapacitor.db"

[task]
  # Task refresh rate
  dir = "/var/lib/kapacitor/tasks"
  snapshot-interval = "60s"

[[influxdb]]
  # InfluxDB connection parameters
  enabled = true
  name = "default"
  default = true
  urls = ["http://influxdb:8086"]
  token = "${INFLUX_TOKEN}"
  organization = "${INFLUX_ORG}"
  insecure-skip-verify = false

[logging]
  # Configure logging
  file = "STDERR"
  level = "INFO"
EOL
```

Now let's create a basic alerts configuration file:

```bash
cat > ~/tick-stack/kapacitor/tasks/alerts.tick << 'EOL'
// CPU Alert
var data = stream
    |from()
        .measurement('cpu')
        .groupBy('host')
    |window()
        .period(1m)
        .every(1m)
    |mean('usage_idle')
        .as('usage_idle')
    |eval(lambda: 100.0 - "usage_idle")
        .as('usage')
    |alert()
        .crit(lambda: "usage" > 80)
        .message('Host {{ index .Tags "host" }} CPU usage is {{ index .Fields "usage" }}%')
        .details('''
Host: {{ index .Tags "host" }}
Current Usage: {{ index .Fields "usage" }}%
Time: {{ .Time }}
Please check system processes if usage remains high.
''')
        .id('{{ index .Tags "host" }}_cpu')

// HTTP Response Alert (Chuck Norris App)
var http_data = stream
    |from()
        .measurement('http_response')
        .groupBy('host', 'server')
    |window()
        .period(1m)
        .every(1m)
    |alert()
        .crit(lambda: "http_response_code" >= 500 OR "response_time" > 5.0)
        .message('App is experiencing issues. Response code: {{ index .Fields "http_response_code" }}')
        .details('''
Host: {{ index .Tags "host" }}
Server: {{ index .Tags "server" }}
Response Code: {{ index .Fields "http_response_code" }}
Response Time: {{ index .Fields "response_time" }}s
Time: {{ .Time }}
''')
        .id('{{ index .Tags "host" }}_http')

// Log Error Alert
var log_data = stream
    |from()
        .measurement('chuck_norris_logs')
        .where(lambda: "level" == 'ERROR' OR "level" == 'CRITICAL')
    |window()
        .period(5m)
        .every(1m)
    |count('level')
    |alert()
        .crit(lambda: "count" > 5)
        .message('High error rate detected in Chuck Norris application logs')
        .details('''
Host: {{ index .Tags "host" }}
Error Count: {{ index .Fields "count" }}
Time: {{ .Time }}
''')
        .id('{{ index .Tags "host" }}_logs')
EOL
```

**Why Kapacitor is configured this way:**
- Streams data directly from InfluxDB
- Uses windows to aggregate data over time periods
- Creates alerts based on thresholds
- Templates alert messages with relevant data
- Includes unique IDs to avoid duplicate alerts

### Understanding Time-Series Data and Monitoring

The TICK stack is built around time-series data, which has some important characteristics:

1. **Time is the primary axis:** Each data point has a timestamp
2. **High write volume:** Monitoring generates a lot of data points
3. **Downsampling:** Older data is often aggregated to save space
4. **Regular patterns:** Many metrics follow daily/weekly patterns

#### Exploring Your Monitoring Data

Let's look at some ways to explore your monitoring data through Chronograf:

1. **Creating a basic CPU usage query:**
   - Select the `cpu` measurement
   - Filter by `cpu = 'cpu-total'`
   - Apply the function: `mean(usage_user)`
   - Group by `time(1m)`

2. **Monitoring HTTP response times:**
   - Select the `http_response` measurement
   - Filter by `server = 'flask-app:5000'`
   - Apply the function: `mean(response_time)`
   - Group by `time(1m)`

3. **Tracking Chuck Norris API response times:**
   - Select the `chuck_norris_api` measurement
   - Apply the function: `mean(response_time)`
   - Group by `time(1m)`

4. **Counting error logs:**
   - Select the `chuck_norris_logs` measurement
   - Filter by `level = 'ERROR'`
   - Apply the function: `count()`
   - Group by `time(15m)`

### Building Useful Dashboards

A good monitoring dashboard should:
1. Answer specific questions
2. Show related metrics together
3. Include both high-level and detailed views

Here's a suggested dashboard setup:

#### System Health Dashboard

**Panel 1: System Resources**
- CPU Usage: `SELECT mean("usage_user") FROM "cpu" WHERE ("cpu" = 'cpu-total') GROUP BY time(1m)`
- Memory Usage: `SELECT mean("used_percent") FROM "mem" GROUP BY time(1m)`
- Disk Space: `SELECT mean("used_percent") FROM "disk" GROUP BY time(1m), "device"`

**Panel 2: Docker Containers**
- Container CPU: `SELECT mean("usage_percent") FROM "docker_container_cpu" GROUP BY time(1m), "container_name"`
- Container Memory: `SELECT mean("usage_percent") FROM "docker_container_mem" GROUP BY time(1m), "container_name"`

#### Application Performance Dashboard

**Panel 1: HTTP Response Times**
- Flask App: `SELECT mean("response_time") FROM "http_response" WHERE ("server" = 'flask-app:5000') GROUP BY time(1m)`
- Chuck Norris API: `SELECT mean("response_time") FROM "chuck_norris_api" GROUP BY time(1m)`

**Panel 2: Request Success Rates**
- Success Rate: `SELECT count("http_response_code") FROM "http_response" WHERE ("http_response_code" < 400) GROUP BY time(1m)`
- Error Rate: `SELECT count("http_response_code") FROM "http_response" WHERE ("http_response_code" >= 400) GROUP BY time(1m)`

**Panel 3: Log Analysis**
- Error Count: `SELECT count("level") FROM "chuck_norris_logs" WHERE ("level" = 'ERROR' OR "level" = 'CRITICAL') GROUP BY time(15m)`
- Warning Count: `SELECT count("level") FROM "chuck_norris_logs" WHERE ("level" = 'WARNING') GROUP BY time(15m)`

## Understanding the Metrics Flow

Let's understand how metrics flow through the TICK stack:

1. **Sources Generate Data:**
   - System metrics (CPU, memory, disk)
   - Docker container metrics
   - HTTP endpoints (Flask app, Chuck Norris API)
   - Application logs

2. **Telegraf Collects Data:**
   - Runs collection agents at regular intervals
   - Applies transformations if needed
   - Batches data for efficiency

3. **InfluxDB Stores Data:**
   - Organizes data into buckets
   - Indexes by time
   - Compresses and optimizes storage
   - Provides query capability

4. **Chronograf Visualizes Data:**
   - Queries InfluxDB
   - Renders data in dashboards
   - Provides UI for exploring data

5. **Kapacitor Processes Data (Optional):**
   - Monitors data streams in real-time
   - Applies rules and transformations
   - Triggers alerts when conditions are met

## Common Issues and Troubleshooting

### InfluxDB Connection Issues

**Problem:** Telegraf can't connect to InfluxDB
**Checks:**
```bash
# Check InfluxDB is running
docker-compose ps influxdb

# Check InfluxDB logs
docker-compose logs influxdb

# Verify token in .env matches what's in InfluxDB
docker-compose exec influxdb influx auth list
```

### Log Collection Issues

**Problem:** Not seeing log data in Chronograf
**Checks:**
```bash
# Verify log file exists
docker-compose exec flask-app ls -la /app/logs/app.log

# Check Telegraf logs for parsing errors
docker-compose logs telegraf

# Make sure the log format matches the Grok pattern
docker-compose exec flask-app tail /app/logs/app.log
```

### HTTP Monitoring Issues

**Problem:** HTTP response metrics not appearing
**Checks:**
```bash
# Test HTTP endpoint directly
docker-compose exec telegraf wget -q -O- http://flask-app:5000/health

# Check for network connectivity
docker-compose exec telegraf ping -c 3 flask-app

# Verify the URL in telegraf.conf
docker-compose exec telegraf cat /etc/telegraf/telegraf.conf | grep urls
```

## Advanced Options

### Adding More Metrics

You can extend Telegraf to collect additional metrics:

1. **Monitor specific processes:**
   Add to telegraf.conf:
   ```
   [[inputs.procstat]]
     pattern = "python"
   ```

2. **Track network connections:**
   Add to telegraf.conf:
   ```
   [[inputs.netstat]]
   ```

3. **Monitor API webhook calls:**
   Add to telegraf.conf:
   ```
   [[inputs.webhooks]]
     [inputs.webhooks.github]
       path = "/github"
       service_key = ""
   ```

### Telegraf Plugins Overview

Telegraf has a vast library of plugins:

1. **Input plugins:** Collect data from various sources
   - System metrics (cpu, disk, mem)
   - Network (ping, dns, netstat)
   - Applications (nginx, redis, postgres)
   - Message queues (kafka, rabbitmq)
   - IoT (mqtt, modbus)

2. **Processor plugins:** Transform data in-flight
   - regex: Extract fields using regex
   - converter: Convert data types
   - topk: Keep only top values
   - enum: Map numeric values to strings

3. **Aggregator plugins:** Aggregate data over time
   - basicstats: Calculate min, max, mean
   - histogram: Create histograms
   - valuecounter: Count occurrences

4. **Output plugins:** Send data to various destinations
   - influxdb: Send to InfluxDB
   - file: Write to files
   - kafka: Send to Kafka
   - cloudwatch: Send to AWS CloudWatch

## Conclusion

You now have a fully functioning TICK stack monitoring a Flask application that interacts with the Chuck Norris API. This setup demonstrates:

1. **Data collection** with Telegraf from multiple sources
2. **Time-series storage** with InfluxDB
3. **Visualization** with Chronograf
4. **Optional alerting** with Kapacitor

This monitoring stack gives you visibility into:
- System performance
- Application health
- API interactions
- Error patterns
