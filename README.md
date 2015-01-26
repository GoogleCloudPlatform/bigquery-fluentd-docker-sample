# Fluentd + Google BigQuery Getting Started Sample

This sample launches a [Docker](https://www.docker.com/) container that's preconfigured with [nginx](http://nginx.org/en/) and [Fluentd](http://www.fluentd.org/) and uses a [Fluentd-to-Bigquery plugin](https://github.com/kaizenplatform/fluent-plugin-bigquery) to load web server access logs into [Google BigQuery](https://cloud.google.com/bigquery/) in near-real-time.

For a step-by-step tutorial, see: https://cloud.google.com/solutions/real-time/fluentd-bigquery, which walks you through the following steps:
 * Run an nginx web server in a Google Compute Engine instance.
 * Log browser traffic to that server using Fluentd.
 * Query the logged data using the BigQuery Browser Tool and using Google Apps Script from a Google Spreadsheet.
 * Visualize the query results in a chart within a Google Spreadsheet that automatically refreshes.

## License

* See [LICENSE](LICENSE)
