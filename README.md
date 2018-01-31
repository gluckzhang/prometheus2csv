# metrics2csv
This is a small tool written in Python 3. It helps us query multiple metrics from prometheus, and save them into a csv file.

## Why do we need this?
Needless to say, we have lots of requirements towards docker monitoring. And what's more, we want to dig into metrics exporting methods. In order to do some data analysis, we want to extract metrics in a usable/simple format (e.g. CSV). It's easy to query some metrics, but the tools paid less attention to export multiple metrics joined by timestamp(this can be some scenarios about data analysis, or dev daily report), then metrics2csv was born. More info about 2 basic docker monitoring solutions, you can visit my blog [here](http://blog.gluckzhang.com/archives/145/).

## How to use it?
`metrics2csv` is a command line tool for Python 3. Basic useage is as follows:

```bash
python metrics2csv.py -h http://prometheus:9090 -c blc_server -o test.csv -s 10s --period=120
```

`http://prometheus:9090` is your Prometheus server's address, `blc_server` is the name of the container which you want to query multiple metrics, `test.csv` is the target csv file's name(default is result.csv), `10s` indicates query resolution step width in Prometheus query API, `120` means that you will get 120 **minutes** data which is in the most recent period.

### All arguments of metrics2csv

Required arguments:

- -h / --host: Prometheus server address
- -c / --container: The name of the container which you want to query multiple metrics

Optional arguments:

- -o / --outfile: Query result's CSV filename, default is `result.csv`
- -s / --step: Query resolution step width in Prometheus query API, default is `10s`
- --period: Indicate this to get most recent period's data, it's an integer, in minute. For example, when you use `--period=120`, you will get the data from 120 mins ago till now
- --start: Start time in the query, use timestamp or rfc3339
- --end: End time in the query, use timestamp or rfc3339
- --help: Get the basic help info

*Attention: you can use start&end OR period to query data, if these 3 arguments all exist, `period` will be the priority.*

## Ideal data format

```
-------------------------------------------------
| timestame + metric1 + metric2 + metric3 + ... |
-------------------------------------------------
| xxxxxxxxx + value1  + value2  + value3  + ... |
-------------------------------------------------
| ...       + ...     + ...     + ...     + ... |
-------------------------------------------------
```

## Implementation details

Mainly use [Prometheus HTTP API](https://prometheus.io/docs/prometheus/latest/querying/api/#range-queries) to get the result (query and query_range).

### Step1: Get all the metric names based on the container's name

It's a little tricky here, because Prometheus doesn't provide any query functions to get this info, but we can query like this:

```
http://prometheus:9090/api/v1/query?query=sum by(__name__)({name="blc_server"})
```

Then you will get all the metrics' names(get them by `__name__`) relevant to container 'blc_server'. Hence `timestamp + these_metrics_names` will be our csv file's header.

### Step2: Query every metric's values and join them together by timestamp

Construct series of query urls based on the names of metrics, then we can get timestamp-value info for every metric. Use a dictionary to store these data, timestamp can be keys and each key corresponds to a list.

*As far as I know, Prometheus doesn't provide such method to get multiple metrics' values joined by timestamp, so we have to make series of queries as well. Sometimes you can get similar csv file in Grafana, this is an end-user method talked in [my blog](http://blog.gluckzhang.com/archives/145/)*

## TODO

- Generate a config file for the first run: then you can update the query easily, and you can also choose some of the metrics you are interested in, instead of query all metrics.
- Multiple query jobs: maybe we need to query series of containers' monitoring data, so it's better to run metrics2csv one time and get all the info you want (maybe with multiple csv files, classified by containers' names).
- For some metrics in Prometheus (acturally from cAdvisor), they have the same `__name__`, but some other labels' names differ. For example, you might get series of results for `container_fs_io_time_seconds_total`, because the container might have many `device` values. metrics2csv should handle this circumstances.