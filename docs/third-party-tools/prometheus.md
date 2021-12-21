---
title: Prometheus monitoring and alerting
sidebar_label: Prometheus
description:
  This document describes how to monitor QuestDB metrics such as memory
  consumption using the Prometheus metrics endpoint, and how to log alerts to
  Prometheus Alertmanager.
---

Prometheus is an open-source systems monitoring and alerting toolkit. Prometheus
collects and stores metrics as time series data, i.e. metrics information is
stored with the timestamp at which it was recorded, alongside optional key-value
pairs called labels.

Users can measure the internal status of a QuestDB instance via an HTTP endpoint
exposed by QuestDB at port `9003`. This document describes how to enable metrics
via this endpoint, how to configure Prometheus to scrape metrics from a QuestDB
instance, and how to enable alerting from QuestDB to Prometheus Alertmanager.

## Prerequisites

- **QuestDB** must be running and accessible. You can do so from
  [Docker](/docs/get-started/docker/), the
  [binaries](/docs/get-started/binaries/), or
  [Homebrew](/docs/get-started/homebrew/) for macOS users.

- **Prometheus** can be installed using
  [homebrew](https://formulae.brew.sh/formula/prometheus),
  [Docker](https://hub.docker.com/u/prom), or directly as a binary. For more
  details, refer to the official Prometheus
  [installation instructions](https://prometheus.io/download/).

- **Alertmanager** can be run using
  [Docker](https://hub.docker.com/r/prom/alertmanager/) or
  [Quay](https://quay.io/repository/prometheus/alertmanager), or can be built
  from source by following the
  [build instructions on GitHub](https://github.com/prometheus/alertmanager#compiling-the-binary).

## Scraping Prometheus metrics from QuestDB

QuestDB has a `/metrics` HTTP endpoint on port `9003` to expose Prometheus
metrics. Before being able to query metrics, they must be enabled via the
`metrics.enabled` key in server configuration:

```bash title="/path/to/server.conf"
metrics.enabled=true
```

When running QuestDB via Docker, port `9003` must be exposed and the metrics
configuration can be enabled via the `QDB_METRICS_ENABLED` environment variable:

```bash
docker run \
  -e QDB_METRICS_ENABLED=TRUE \
  -p 8812:8812 -p 9000:9000 -p 9003:9003 -p 9009:9009 \
  questdb/questdb:6.1.3
```

To verify that metrics are being exposed correctly by QuestDB, navigate to
`http://<questdb_ip>:9003/metrics` in a browser, where `<questdb_ip>` is the IP
address of an instance, or execute a basic curl like the following example:

```bash title="Given QuestDB running at 127.0.0.1"
curl http://127.0.0.1:9003/metrics
# TYPE questdb_json_queries_total counter
questdb_json_queries_total 0

# TYPE questdb_memory_tag_MMAP_DEFAULT gauge
questdb_memory_tag_MMAP_DEFAULT 77872

# TYPE questdb_memory_malloc_count gauge
questdb_memory_malloc_count 659

# ...
```

To configure Prometheus to scrape these metrics, provide the QuestDB instance IP
and port `9003` as a target. The following example configuration file
`questdb.yml` assumes there is a running QuestDB instance on localhost
(127.0.0.1) with port `9003` available:

```shell title="questdb.yml"
global:
  scrape_interval: 5s
  external_labels:
    monitor: 'questdb'

scrape_configs:
  - job_name: 'questdb'
    scrape_interval: 5s
    static_configs:
      - targets: ['127.0.0.1:9003']
```

Start Prometheus and pass this configuration on launch:

```bash
prometheus --config.file=questdb.yml
```

Prometheus should be available on `0.0.0.0:9090` and navigating to
`http://0.0.0.0:9090/targets` should show that QuestDB is being scraped
successfully:

import Screenshot from "@theme/Screenshot"

<Screenshot
  alt="Prometheus targets tab showing a QuestDB instance status"
  height={320}
  src="/img/guides/prometheus/healthy-targets.png"
  width={750}
/>

In the graphing tab of Prometheus (`http://0.0.0.0:9090/graph`), autocomplete
can be used to graph QuestDB-specific metrics which are all prefixed with
`questdb_`:

<Screenshot
  alt="Prometheus graphing tab showing QuestDB instance metrics on a chart"
  height={320}
  src="/img/guides/prometheus/graphing-metrics.png"
  width={750}
/>

## Configuring Prometheus Alertmanager

QuestDB includes a log writer that sends any message logged at critical level
(by default) to Prometheus
[Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) over a
TCP/IP socket connection. To configure this writer, add it to the `writers`
config alongside other log writers. Details on logging configuration can be
found on the
[server configuration documentation](/docs/reference/configuration/#configuration-file).

Alertmanager may be started via Docker with the following command:

```
docker run -p 127.0.0.1:9093:9093 --name alertmanager quay.io/prometheus/alertmanager
```

To discover the IP address of this container, run the following command which
specifies `alertmanager` as the container name:

```bash
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' alertmanager
```

To run QuestDB and point it towards Alertmanager for alerting, first create a
file `./conf/log.conf` with the following contents. `172.17.0.2` in this case is
the IP address of the docker container for alertmanager that was discovered by
running the `docker inspect ` command above.

```bash title="./conf/log.conf"
# Which writers to enable
writers=stdout,alert

# stdout
w.stdout.class=io.questdb.log.LogConsoleWriter
w.stdout.level=INFO

# Prometheus Alerting
w.alert.class=io.questdb.log.LogAlertSocketWriter
w.alert.level=CRITICAL
w.alert.alertTargets=172.17.0.2:9093
```

Start up QuestDB in Docker using the following command:

```bash "Mounting a volume with the log.conf file"
docker run \
  -p 9000:9000 -p 8812:8812 -p 9009:9009 -p 9003:9003 \
  -v "$(pwd):/root/.questdb/" \
  questdb/questdb:6.1.3
```

When alerts are successfully triggered, QuestDB logs will indicate the sent and
received status:

```txt
2021-12-14T18:42:54.222967Z I i.q.l.LogAlertSocketWriter Sending: 2021-12-14T18:42:54.122874Z I i.q.l.LogAlertSocketWriter Sending: 2021-12-14T18:42:54.073978Z I i.q.l.LogAlertSocketWriter Received [0] 172.17.0.2:9093: {"status":"success"}
2021-12-14T18:42:54.223377Z I i.q.l.LogAlertSocketWriter Received [0] 172.17.0.2:9093: {"status":"success"}
```

:::info

The template used by QuestDB for alerts is user-configurable and is described in
more detail in the
[server configuration](/docs/reference/configuration/#prometheus-alertmanager)
documentation.