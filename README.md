# CloudWatch Logs Insights Queries

This repository contains a number of useful queries you can copy, paste and run using
[CloudWatch Logs Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html).

For an overview of CloudWatch Logs Insights, see
[Operating Lambda: Using CloudWatch Logs Insights](https://aws.amazon.com/blogs/compute/operating-lambda-using-cloudwatch-logs-insights/)
on the AWS Compute Blog. While this blog post focuses on querying logs from AWS Lambda, CloudWatch Logs
Insights may be used to analyze logs from any logs stored in CloudWatch.

## Save queries using CloudFormation

Queries described below can be persisted in your CloudWatch Logs Insights page using the CloudFormation template in cloudformation.yaml, or by clicking [![Launch CloudFormation template](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home#/stacks/new?stackName=cwlogs-insights-sample-queries&templateURL=https://raw.githubusercontent.com/aws-samples/cloudwatch-logs-insights-queries/main/cloudformation.yaml)

---

## General queries

### 25 most recent logs

```sql
fields @timestamp, @message
| sort @timestamp desc
| limit 25
```

### Number of exceptions per hour

```sql
filter @message like /Exception/
| stats count(*) as exceptionCount by bin(1h)
| sort exceptionCount desc
```

## Lambda Queries

### Average and max duration

```sql
filter @type ="REPORT"
| parse @message /Duration: (?<ms>\S+)/
| stats avg(ms),
  max(ms)
by bin(30s)
```

### Top 100 highest billed invocations

```sql
filter @type = "REPORT"
| fields @requestId, @billedDuration
| sort by @billedDuration desc
| limit 100
```

### Find count, average duration, max duration and average memory for cold starts

```sql
filter @type ="REPORT"
| parse @message /Init Duration: (?<init>\S+)/
| stats count() as total,
  count(init) as coldStarts,
  avg(init) as avgInitDuration,
  max(init) as maxInitDuration,
  avg(@maxMemoryUsed)/1000/1000 as memoryused
by bin (1min)
```

### Percentage of cold starts in total invocations

```sql
filter @type = "REPORT"
| stats
  sum(strcontains(@message, "Init Duration"))/count(*) * 100 as coldStartPct,
  avg(@duration)
by bin(1m)
```

### Percentile report of Lambda duration

```sql
filter @type = "REPORT"
| stats
  avg(@billedDuration) as Average,
  pct(@billedDuration, 99) as NinetyNinth,
  pct(@billedDuration, 95) as NinetyFifth,
  pct(@billedDuration, 90) as Ninetieth
by bin(1m)
```

### Percentile report of Lambda memory usage

```sql
filter @type = "REPORT"
| stats avg(@maxMemoryUsed / 1024 / 1024) as mean_MemoryUsed,
  min(@maxMemoryUsed / 1024 / 1024) as min_MemoryUsed,
  max(@maxMemoryUsed / 1024 / 1024) as max_MemoryUsed,
  pct(@maxMemoryUsed / 1024 / 1024, 95) as Percentile
```

### Invocations using 75% or more of assigned memory

```sql
filter @type = "REPORT" and @maxMemoryUsed >= (@memorySize * 0.75)
| stats
  count_distinct(@requestId)
by bin(30m)
```

### Latency report

```sql
filter @type = "REPORT"
| stats avg(@duration),
  max(@duration),
  min(@duration)
by bin(1m)
```

### Free memory

```sql
filter @type = "REPORT"
| stats max(@memorySize / 1024 / 1024) as provisonedMemMB,
  min(@maxMemoryUsed / 1024 / 1024) as smallestMemReqMB,
  avg(@maxMemoryUsed / 1024 / 1024) as avgMemUsedMB,
  max(@maxMemoryUsed / 1024 / 1024) as maxMemUsedMB,
  provisonedMemMB - maxMemUsedMB as overProvisionedMB
```

### Billed GB-S

```sql
filter @type = "REPORT"
| stats sum(@billedDuration)/1000 * avg(@memorySize)/1024000000 as billedGBs by bin(1m)
```

### Compare cold and warm start durations

```sql
filter @type = "REPORT"
| fields greatest(@initDuration, 0) + @duration as duration,
  ispresent(@initDuration) as coldStart
| stats count(*) as count,
  pct(duration, 50) as p50,
  pct(duration, 90) as p90,
  pct(duration, 99) as p99,
  max(duration) as max
by coldStart
```

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
