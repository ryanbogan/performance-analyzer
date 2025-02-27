[![Java CI](https://github.com/opensearch-project/performance-analyzer/workflows/Java%20CI/badge.svg)](https://github.com/opensearch-project/performance-analyzer/actions?query=workflow%3A%22Java+CI%22)
[![CD](https://github.com/opensearch-project/performance-analyzer/workflows/CD/badge.svg)](https://github.com/opensearch-project/performance-analyzer/actions?query=workflow%3ACD)
[![codecov](https://codecov.io/gh/opensearch-project/performance-analyzer/branch/main/graph/badge.svg)](https://codecov.io/gh/opensearch-project/performance-analyzer)
[![Documentation](https://img.shields.io/badge/api-reference-blue.svg)](https://opensearch.org/docs/monitoring-plugins/pa/api/)
[![Chat](https://img.shields.io/badge/chat-on%20forums-blue)](https://forum.opensearch.org/c/plugins/performance-analyzer/9)
![PRs welcome!](https://img.shields.io/badge/PRs-welcome!-success)

<img src="https://opensearch.org/assets/img/opensearch-logo-themed.svg" height="64px">

<!-- TOC -->

- [OpenSearch Performance Analyzer](#opensearch-performace-analyzer)
- [Documentation](#documentation)
- [Contributing](#contributing)
- [Code of Conduct](#code-of-conduct)
- [Security](#security)
- [License](#license)
- [Copyright](#copyright)

<!-- /TOC -->

# OpenSearch Performance Analyzer
Performance Analyzer exposes a REST API that allows you to query numerous performance metrics for your cluster, including aggregations of those metrics, independent of the Java Virtual Machine (JVM). PerfTop is the default command line interface (CLI) for displaying those metrics.

## Setup



## Performance Analyzer API
Performance Analyzer uses a single HTTP method and URI for all requests:

GET `<endpoint>/_plugins/_performanceanalyzer/metrics`

Then you provide parameters for metrics, aggregations, dimensions, and nodes (optional):

```
?metrics=<metrics>&agg=<aggregations>&dim=<dimensions>&nodes=all"
```

* metrics - comma separated list of metrics you are interested in. For a full list of metrics, see Metrics Reference.
* agg - comma separated list of agg to be used on each metric. Possible values are sum, avg, min and max. Length of the list should be equal to the number of metrics specified.
* dim - comma separated list of dimensions. For the list of dimensions supported by each metric, see Metrics Reference.
* nodes - If the string all is passed, metrics from all nodes in the cluster are returned. For any other value, metrics from only the local node is returned.

### SAMPLE REQUEST
GET `_plugins/_performanceanalyzer/metrics?metrics=Latency,CPU_Utilization&agg=avg,max&dim=ShardID&nodes=all`


## Batch Metrics API
While the metrics api associated with performance analyzer provides the last 5 seconds worth of metrics, the batch metrics api provides more detailed metrics and from longer periods of time. See the [design doc](https://github.com/opensearch-project/performance-analyzer-rca/blob/main/docs/batch-metrics-api.md) for more information.

In order to access the batch metrics api, first enable it using one of the following HTTP request:

```
POST localhost:9200/_plugins/_performanceanalyzer/batch/config -H ‘Content-Type: application/json’ -d ‘{"enabled": true}’
POST localhost:9200/_plugins/_performanceanalyzer/batch/cluster/config -H ‘Content-Type: application/json’ -d ‘{"enabled": true}’
```

The former enables batch metrics on a single node, while the latter enables it on nodes across the entire cluster. Batch metrics can be disabled using analogous queries with `{"enabled": false}`.

You can then query either the config or cluster config apis to see how many minutes worth of batch metrics data will be retained by nodes in the cluster (`batchMetricsRetentionPeriodMinutes`):

```
GET localhost:9200/_plugins/_performanceanalyzer/config

{"performanceAnalyzerEnabled":true,"rcaEnabled":false,"loggingEnabled":false,"shardsPerCollection":0,"batchMetricsEnabled":true,"batchMetricsRetentionPeriodMinutes":7}

GET localhost:9200/_plugins/_performanceanalyzer/cluster/config

{"currentPerformanceAnalyzerClusterState":9,"shardsPerCollection":0,"batchMetricsRetentionPeriodMinutes":7}
```

The default retention period is 7 minutes. However, the cluster owner can adjust this by setting `batch-metrics-retention-period-minutes` in performance-analyzer.properties (note, setting this value will require a restart so that the cluster can read the new value upon startup). The value must be between 1 and 60 minutes (inclusive) — the range is capped like so in order to prevent excessive data retention on the cluster, which would eat up a lot of storage.

You can then access the batch metrics available at each node via queries of the following format:

```
GET localhost:9600/_plugins/_performanceanalyzer/batch?metrics=<metrics>&starttime=<starttime>&endtime=<endtime>&samplingperiod=<samplingperiod>
```

* metrics - Comma separated list of metrics you are interested in. For a full list of metrics, see Metrics Reference.
* starttime - Unix timestamp (difference between the current time and midnight, January 1, 1970 UTC) in milliseconds determining the oldest data point to return. starttime is inclusive — data points from at or after the starttime will be returned. Note, the starttime and endtime supplied by the user will both be rounded down to the nearest samplingperiod. starttime must be no less than `now - retention_period` and it must be less than the endtime (after the rounding).
* endtime - Unix timestamp in milliseconds determining the freshest data point to return. endtime is exclusive — only datapoints from before the endtime will be returned. endtime must be no greater than the system time at the node, and it must be greater than the startime (after being rounded down to the nearest samplingperiod).
* samplingperiod - Optional parameter indicating the sampling period in seconds (default is 5s). The requested time range will be partitioned according to the sampling period, and data from the first available 5s interval in each partition will be returned to the user. Must be at least 5s, must be less than the retention period, and must be a multiple of 5.

Note, the maximum number of datapoints that a single query can request for via API is capped at 100,800 datapoints (in order to prevent excessive memory consumption by the datapoints). If a query exceeds this limit, an error is returned. The query parameters can be adjusted on such queries to request for fewer datapoints at a time.

Note, unlike with the metrics api, there is no `nodes=all` parameter for the batch metrics api. You must query a specific node in order to obtain metrics from that node.

Note, the default retention period is 7 minutes because a typical use-case would be to query for 5 minutes worth of data from the node. In order to do this, a client would actually select a starttime of now-6min and an endtime of now-1min (this one minute offset will give sufficient time for the metrics in the time range to be available at the node). Atop this 6 minutes of retention, we need an extra 1 minute of retention to account for the time that would have passed by the time the query arrives at the node, and for the fact that starttime and endtime will be rounded down to the nearest samplingperiod.

### SAMPLE REQUEST
GET `_plugins/_performanceanalyzer/batch?metrics=CPU_Utilization,IO_TotThroughput&starttime=1594412250000&endtime=1594412260000&samplingperiod=5`

See the [design doc](https://github.com/opensearch-project/performance-analyzer-rca/blob/main/docs/batch-metrics-api.md) for the expected response.

## Documentation

Please refer to the [technical documentation](https://opensearch.org/docs/monitoring-plugins/pa/index/) for detailed information on installing and configuring Performance Analyzer.

## Contributing

See [developer guide](DEVELOPER_GUIDE.md) and [how to contribute to this project](CONTRIBUTING.md).

## Code of Conduct

This project has adopted the [Amazon Open Source Code of Conduct](CODE_OF_CONDUCT.md). For more information see the [Code of Conduct FAQ](https://aws.github.io/code-of-conduct-faq), or contact [opensource-codeofconduct@amazon.com](mailto:opensource-codeofconduct@amazon.com) with any additional questions or comments.

## Security

If you discover a potential security issue in this project we ask that you notify AWS/Amazon Security via our [vulnerability reporting page](http://aws.amazon.com/security/vulnerability-reporting/). Please do **not** create a public GitHub issue.

## License

This project is licensed under the [Apache v2.0 License](LICENSE.txt).

## Copyright

Copyright OpenSearch Contributors. See [NOTICE](NOTICE) for details.
