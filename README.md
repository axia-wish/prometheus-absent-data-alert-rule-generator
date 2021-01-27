# prometheus-rule-generate

`prometheus-rule-generate` is a tool to generate Prometheus rules based on our
existing rules. At the moment it is just used to generate alerts for missing
data in time-series that we use in our other alerts and recordings.

## Usage

This is a normal Rust project and can be managed using Cargo.

```shell
$ cargo run -- --help
prometheus-absent-data-alert-rule-generator [OPTIONS] <PATH>

ARGS:
    PATH            Path to the directory containing the Prometheus rules files.

OPTIONS:
    -h, --help      Print this help information.
    --dry-run       Dry run. Don't output the generated rules files.
    --output-file   File to write the absent rules to. Defaults to absent.rules.yml in PATH.
```

All our rules are kep in `operations/docker/prometheus/alerts/` so we can call
`prometheus-absent-data-alert-rule-generator` like so:

```shell
cargo run -- $(git rev-parse --show-toplevel)/operations/docker/prometheus/alerts
```

This will create an `absent.rules.yml` file in
`operations/docker/prometheus/alerts`.

## Absent time-series alert generation

The high-level overview of how the alerts are generated is:

1. Parse all the `*.rules.yml` files in the given directory and pull out the
   expressions from the `expr` field
2. In each expression, pull out all the time-series selector used (e.g.
   `stack:public_http_errors_5xx_non_L3:rate1m_sum` or
   `aws_firehose_delivery_to_redshift_success_minimum[1h]`)
3. Create a unique list of all the selectors we use
4. For each selector generate a rule of the form:
```yaml
- expr: "absent(<selector>)"
  alert: absent_<selector name>
  annotations:
    description: "No data for '<selector>'. This alert rule was generated by   prometheus-absent-data-alert-rule-generator."
    summary: "No data for '<selector>' data"
  for: 5m
  labels:
    severity: business_hours_page
```
**NOTE:** For range-vector selectors (e.g.
`aws_firehose_delivery_to_redshift_success_minimum[1h]`) the `absent_over_time`
function is used because it's the range-vector equivalent of `absent`.

5. Dump all the rules to `absent.rules.yml` in the input directory or to the
   specified output file.