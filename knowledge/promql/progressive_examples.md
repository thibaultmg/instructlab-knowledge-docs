# PromQL Learning Path: From Basic to Advanced

  ## Level 1: Basic Metrics and Selectors

  1. Simple metric query
    http_requests_total
  Description: Returns all time series with the metric name http_requests_total.
  2. Label matching
    http_requests_total{job="prometheus"}
  Description: Returns http_requests_total metrics only for the job labeled "prometheus".
  3. Multiple label matchers
    http_requests_total{job="prometheus", instance="localhost:9090"}
  Description: Returns metrics matching both label conditions.
  4. Negative matching
    http_requests_total{job!="prometheus"}
  Description: Returns metrics where the job label is not "prometheus".
  5. Regex matching
    http_requests_total{job=~"prom.*"}
  Description: Returns metrics where the job label matches the regex pattern "prom.*".

## Level 2: Range Vectors and Simple Functions

  6. Range vector selection
    http_requests_total[5m]
  Description: Returns the last 5 minutes of samples for all time series in http_requests_total.
  7. Basic rate function
    rate(http_requests_total[5m])
  Description: Calculates the per-second average rate of increase over the last 5 minutes.
  8. Increase function
    increase(http_requests_total[1h])
  Description: Shows the increase in the metric over the last hour.
  9. Sum by label
    sum(http_requests_total) by (job)
  Description: Sums all http_requests_total values, grouped by the job label.
  10. Count by label
    count(http_requests_total) by (instance)
  Description: Counts the number of time series for each instance.

## Level 3: Arithmetic and Comparison Operators

  11. Simple arithmetic
    http_requests_total{status="200"} / http_requests_total
  Description: Calculates the ratio of successful requests to total requests.
  12. Multiplication with scalar
    rate(http_requests_total[5m]) * 60
  Description: Converts per-second rate to per-minute rate.
  13. Comparison operator
    http_requests_total > 100
  Description: Returns 1 for time series with values greater than 100, otherwise returns no data.
  14. Boolean filter
    http_requests_total{status!="200"} and http_requests_total > 0
  Description: Returns non-200 status metrics with values greater than 0.
  15. Unless operator
    http_requests_total unless on(instance) node_boot_time
  Description: Returns http_requests_total time series that don't have a matching node_boot_time
  series with the same instance label.

## Level 4: Advanced Functions

  16. Quantiles
    histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
  Description: Calculate the 95th percentile from a histogram over the last 5 minutes.
  17. Prediction functions
    predict_linear(node_filesystem_free_bytes[1h], 4 * 3600)
  Description: Predicts the value of the time series 4 hours from now based on the last 1 hour of
  data.
  18. Moving averages
    avg_over_time(rate(http_requests_total[5m])[1h:])
  Description: Calculates the average of the 5-minute rates over a 1-hour window.
  19. Deriv function
    deriv(node_temperature_celsius{instance="localhost:9100"}[1h])
  Description: Calculates the per-second derivative of the time series over the last hour.
  20. Label joins
    sum by (app) (
      rate(http_requests_total[5m]) * on(instance) group_left(app) node_meta{env="prod"}
    )
  Description: Calculates request rates, then joins with node metadata to include app label in the
  result.

## Level 5: Complex Queries and Patterns

  21. Error ratio calculation
    sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
      /
    sum(rate(http_requests_total[5m])) by (job)
  Description: Calculates the ratio of HTTP 5xx errors to total requests by job.
  22. Top N time series
    topk(5, sum(rate(http_requests_total[5m])) by (pod))
  Description: Returns the 5 pods with the highest request rates.
  23. Subquery and max_over_time
    max_over_time(rate(http_requests_total[5m])[1h:])
  Description: Finds the maximum 5-minute rate observed over the last hour.
  24. Alert trigger condition
    sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
      /
    sum(rate(http_requests_total[5m])) by (job) > 0.01
  Description: Returns 1 when the error rate exceeds 1% for any job.
  25. Complex aggregation
    clamp_max(
      sum by(instance) (
        rate(node_cpu_seconds_total{mode!="idle"}[5m])
      ) /
      count by(instance) (
        sum by(instance, cpu) (rate(node_cpu_seconds_total[5m]))
      ),
      1
    )
  Description: Calculates CPU utilization by instance, clamping the value to a maximum of 1 (100%).

