# Prometheus and Grafana Monitoring

### Student Information

Name: Ebubechukwu Ogbonna

## 1. Objective

Build a local monitoring stack using Prometheus, Node Exporter, and Grafana. Node Exporter exposes system-level metrics for the host machine, Prometheus scrapes and stores those metrics, and Grafana visualizes them through a custom-built dashboard. The project also demonstrates PromQL query writing, Grafana Explore, and hands-on troubleshooting of a broken metrics pipeline.

## 2-4. Prometheus and Node Exporter Setup

Node Exporter and Prometheus were installed as standalone binaries (downloaded, extracted, and run directly - no package manager needed for these two). Node Exporter runs on port 9100 and exposes hardware/OS metrics at /metrics. Prometheus runs on port 9090 and was configured with the following prometheus.yml to scrape both itself and Node Exporter every 15 seconds:

    global:
      scrape_interval: 15s

    scrape_configs:
      - job_name: "prometheus"
        static_configs:
          - targets:
              - "localhost:9090"

      - job_name: "node"
        static_configs:
          - targets:
              - "localhost:9100"

Both targets were confirmed UP on the Prometheus Targets page (http://localhost:9090/targets).

## 5. Prometheus Metric Investigation

### CPU

Query: rate(node_cpu_seconds_total{mode="idle"}[5m])

Result: one series per CPU core, each showing the fraction of time (a value near 1.0) that core spent idle over the last 5 minutes. On this machine, cores showed values between about 0.94 and 0.98, meaning each core was idle roughly 94-98% of the time.

CPU utilization percentage query:

    100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

Result: 4.25 (percent)

This averages the idle rate across all cores for the instance, converts it to a percentage, then subtracts from 100 to get the inverse - the percentage of time the CPU was actually busy across all cores combined.

### Memory

Query: node_memory_MemTotal_bytes
Result: 4056530944 bytes (about 3.78 GB) - the total physical memory on the machine.

Query: node_memory_MemAvailable_bytes
Result: 3478470656 bytes (about 3.24 GB) - memory currently available for new processes, accounting for reclaimable cache/buffers.

Memory utilization percentage query:

    100 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100)

Result: 14.36 (percent)

This divides available memory by total memory to get the fraction still free, converts to a percentage, then subtracts from 100 to get the percentage of memory actually in use.

### Disk

Query: node_filesystem_size_bytes
Result: multiple series, one per mounted filesystem. The root filesystem (mountpoint="/") showed 1081101176832 bytes (about 1 TB) total size.

Query: node_filesystem_avail_bytes
Result: same mountpoints. Root filesystem showed 1014185295872 bytes (about 945 GB) available.

Filesystem utilization percentage query (filtered to root):

    100 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100)

Result: 6.19 (percent)

### Network

Query: rate(node_network_receive_bytes_total[5m])
Result: one series per interface. eth0 showed about 19-49 bytes/sec incoming; loopback (lo) showed a much higher rate (around 12,000 bytes/sec); docker/bridge interfaces showed 0.

Query: rate(node_network_transmit_bytes_total[5m])
Result: eth0 showed about 5.4 bytes/sec outgoing; lo showed about 1749 bytes/sec.

### System Load

node_load1 -> 0.14
node_load5 -> 0.07
node_load15 -> 0.15

These represent average system load over 1, 5, and 15 minutes respectively. load1 reacts fastest to a sudden spike; load15 smooths out short spikes and shows the longer-term trend. Comparing all three distinguishes a brief spike (high load1, low load15) from a sustained problem (all three elevated).

## 6. PromQL Queries

**Query 1**
Query: up{job="node"}
What it does: Checks whether the Node Exporter target is currently being scraped successfully.
Expected result: 1 (up) or 0 (down). Result observed: 1.

**Query 2**
Query: rate(node_cpu_seconds_total{mode="idle"}[5m])
What it does: Calculates the per-second rate of CPU idle time over the last 5 minutes, per core.
Result observed: values between 0.94 and 0.98 across 4 cores.

**Query 3**
Query: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
What it does: Uses rate(), avg by(), and arithmetic together to compute overall CPU utilization percentage.
Result observed: 4.25.

**Query 4**
Query: node_memory_MemTotal_bytes
What it does: Reports total physical memory installed.
Result observed: 4056530944.

**Query 5**
Query: 100 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100)
What it does: Uses arithmetic to compute memory utilization percentage from two raw byte metrics.
Result observed: 14.36.

**Query 6**
Query: 100 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100)
What it does: Uses label filtering with arithmetic to compute disk utilization for a single filesystem.
Result observed: 6.19.

**Query 7**
Query: rate(node_network_receive_bytes_total[5m])
What it does: Calculates incoming network traffic rate per interface.
Result observed: eth0 around 19-49 bytes/sec, lo around 12,000 bytes/sec.

**Query 8**
Query: rate(node_network_transmit_bytes_total[5m])
What it does: Calculates outgoing network traffic rate per interface.
Result observed: eth0 around 5.4 bytes/sec, lo around 1749 bytes/sec.

**Query 9**
Query: sum(rate(node_network_receive_bytes_total[5m]))
What it does: Uses sum() to aggregate incoming network traffic across every interface into one combined total.

**Query 10**
Query: node_load1, node_load5, node_load15
What it does: Retrieves the 1-, 5-, and 15-minute system load averages.
Result observed: 0.14, 0.07, 0.15.

**Query 11**
Query: node_filesystem_avail_bytes{mountpoint="/"}
What it does: Uses label filtering to isolate a single series (root filesystem) from the full metric, which otherwise returns one series per mountpoint.
Result observed: 1014185295872.

## 7. Grafana Installation

Grafana was installed via its official APT repository (not a raw binary), which is the standard method:

    sudo mkdir -p /etc/apt/keyrings/
    wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
    echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
    sudo apt-get update
    sudo apt-get install -y grafana
    sudo service grafana-server start

Grafana was confirmed running by successfully loading the login page at http://localhost:3000.

## 8. Grafana Data Source

A data source in Grafana is a connection to wherever data actually lives - Grafana itself does not store or collect metrics, it is purely a visualization layer. Prometheus was configured as a data source (URL: http://localhost:9090), which lets Grafana send PromQL queries to Prometheus's API and render the results. This separation means Grafana can visualize many different backends through one consistent interface without needing to know how each stores data internally. Save & Test confirmed a successful connection.

## 9. Grafana Explore

Grafana Explore is a free-form, ad-hoc query interface for investigating metrics without needing to build or save a dashboard panel first. It is meant for quick, exploratory troubleshooting - testing a PromQL query or digging into an active incident - where the exact final visualization isn't yet known. A dashboard, in contrast, is for pre-built, repeatable, at-a-glance monitoring a team checks regularly. A DevOps engineer reaches for Explore during an active investigation to quickly test queries and iterate, then only builds a permanent dashboard panel once they know what is worth tracking continuously.

Five queries were run in Explore covering CPU, memory, disk, network, and load, each rendering correctly against the Prometheus data source.

## 10. Grafana Dashboard - "Linux Server Monitoring"

A 10-panel dashboard was built manually in the Grafana UI:

1. CPU Usage - Time series
2. Memory Usage - Gauge
3. Available Memory - Stat
4. Disk Usage - Gauge
5. Available Disk - Stat
6. Network Receive - Time series
7. Network Transmit - Time series
8. System Load - Time series (three queries: node_load1, node_load5, node_load15 together)
9. Running Processes - Stat
10. System Uptime - Stat

## 11. Visualization Selection Reasoning

- CPU Usage -> Time series: CPU usage changes continuously, so a line graph best shows trends and spikes over time.
- Memory Usage -> Gauge: gives an immediate, at-a-glance sense of how full memory is relative to its maximum, like a fuel gauge - ideal for a single current percentage.
- Available Memory -> Stat: a single current numeric value, so a big clear Stat panel is the most direct way to show it.
- Disk Usage -> Gauge: same reasoning as memory - a percentage relative to a full range is intuitive as a gauge.
- Network Receive / Transmit -> Time series: throughput fluctuates constantly, so a time series is essential to see traffic patterns over time.
- System Load -> Time series: showing load1, load5, and load15 together over time lets you compare short-term spikes against longer-term trends.
- Running Processes / System Uptime -> Stat: simple single current values that do not need historical context in the main view.

## 12. Dashboard Organization

Panels are arranged in three rows:

Row 1: CPU Usage, Memory Usage, Disk Usage, Network Transmit
Row 2: Available Memory, Available Disk, Network Receive, System Load
Row 3: Running Processes, System Uptime

This groups the core resource-usage percentages together at the top (the panels most likely to be checked first), availability/network/load metrics in the middle, and lower-priority process/uptime information at the bottom.

## 13. Dashboard Time Range

The dashboard was tested across four time ranges - Last 5 minutes, Last 15 minutes, Last 1 hour, and Last 6 hours - and all panels correctly updated their x-axis and data to reflect each selected window.

## 14. Dashboard Refresh

An auto-refresh interval was set on the dashboard. A monitoring dashboard should not refresh every second because it creates unnecessary load on both Prometheus (constant repeated queries) and the browser (constant re-rendering), without adding real value, since most system metrics do not meaningfully change second-to-second. A 10-30 second refresh balances catching real problems quickly against wasting resources.

## 15. Dashboard Export

The dashboard was exported as JSON via the dashboard settings -> JSON Model, and saved to grafana/dashboards/linux-monitoring.json in this repository. This makes the dashboard portable - it can be re-imported into any Grafana instance with a matching Prometheus data source, recreating all 10 panels instantly without manually rebuilding them.

## 16. Troubleshooting Challenges

### Challenge 1 - Broken Node Exporter target

prometheus.yml was edited to point the node job at the wrong port (localhost:9999 instead of localhost:9100), and Prometheus was restarted. The Targets page then showed the node job as DOWN, while the prometheus job remained UP - confirming Prometheus itself stayed healthy even though one specific target could not be reached. The config was reverted to localhost:9100, Prometheus was restarted again, and the target returned to UP.

### Challenge 2 - Broken Grafana data source

The Prometheus data source's URL was changed from http://localhost:9090 to http://localhost:9999 and Save & Test was clicked. Grafana returned the error:

    Post "http://localhost:9999/api/v1/query": dial tcp 127.0.0.1:9999: connect: connection refused

This confirmed Grafana correctly detects and reports when it cannot reach its configured data source. The URL was reverted to http://localhost:9090 and Save & Test confirmed success again.

### Challenge 3 - Bad panel query

A temporary panel was created using a deliberately nonexistent metric name: node_totally_fake_metric_bytes. Rather than an error message, the panel simply displayed "No data." This is a useful distinction from Challenges 1 and 2: a connectivity failure (wrong host/port) produces a loud, explicit error, while a metric name that simply does not exist produces a silent "No data" result, since Prometheus just returns an empty result set rather than treating it as invalid. The temporary panel was removed afterward.

## 17. Repository Structure

    prometheus-grafana-monitoring/
    |-- README.md
    |-- prometheus/
    |   `-- prometheus.yml
    |-- grafana/
    |   `-- dashboards/
    |       `-- linux-monitoring.json
    `-- screenshots/
        |-- 01-prometheus-running.png
        |-- 02-node-exporter-metrics.png
        |-- 03-prometheus-targets.png
        |-- 04-prometheus-query.png
        |-- 05-grafana-running.png
        |-- 06-grafana-datasource.png
        |-- 07-grafana-explore.png
        |-- 08-grafana-dashboard.png
        |-- 09-troubleshooting-prometheus.png
        `-- 10-troubleshooting-grafana.png

## 18. Key Takeaways

1. Prometheus, Node Exporter, and Grafana each play a distinct role: Node Exporter collects raw system metrics, Prometheus scrapes and stores them as time series, and Grafana visualizes them - none of the three duplicates the others' job.
2. PromQL's real power comes from combining functions (rate, avg by, sum), label filters, and arithmetic together to turn raw counters and gauges into meaningful percentages and rates.
3. A Service/target being reachable and a metric actually existing are two separate failure modes - Prometheus reports the former loudly (target DOWN) and the latter silently (No data) - which matters a lot when troubleshooting a real monitoring pipeline.
4. Dashboards are for repeatable, at-a-glance monitoring; Explore is for one-off, ad-hoc investigation - using the right one for the right situation keeps a monitoring workflow efficient.
5. Exporting a dashboard as JSON makes it portable and reproducible - an entire dashboard's panels, queries, and layout can be recreated instantly on any compatible Grafana instance.
