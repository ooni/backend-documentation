# Backend documentation
This document describes the architecture of the main components of the
OONI infrastructure.

How to develop, test, deploy and monitor them.

The documentation is meant for core contributors.

Copyright 2023 ooni.org See the LICENSE.CC0 file for licensing.

If you are reading this document as a MarkDown file on github the
following button on the top-right menu provides a table of content:

![toc_button](images/gh_button.png)


# Conventions
  A few shorthands used in the document:

  * Backend: the whole software stack including the Fastpath, API, tools
  that transfer data around

  * FSN: The backend-fsn.ooni.org host, running most of the
  **production** backend infrastructure.

  * ams-pg-test: The ams-pg-test.ooni.org host, running a **test**
  backend infrastructure.

  Usually backend components have names that are kept consistent across:

  * Journald unit name

  * Systemd service name

  * Systemd timer name

  * StatsD metrics prefix

  When linking to the backend codebase a specific commit is used in order
  to avoid breaking links when the codebase changes:
  <https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/>

  Internal links across the document are indicated with small icons to
  illustrate the type of element they are linking to, as done in technical
  wikis.

  The icons in use are listed below:

  * API: 🐝

  * Bug: 🐞

  * Backend component: ⚙

  * Grafana dashboard: 📊

  * Backend host: 🖥

  * Jupyter notebook: 📔

  * Debian package: 📦

  * Runbook: 📒

  * Database table: ⛁

  * Network test: Ⓣ

  * Systemd timer: ⏲

  * Python script: 🐍

  * Tool: 🔧

  * Web UI: 🖱

  * General topic: 💡


# Architecture
  The backend infrastructure performs multiple functions:

  * Provide APIs for data consumers

  * Instruct probes on what measurements to perform

  * Receive measurements from probes, process them and store them in the database

* Upload new measurements to a bucket on [S3 data bucket](#s3-data-bucket)&thinsp;💡

  * Fetch data from external sources e.g. fingerprints from a GitHub repository


## Main data flows
  This diagram represent the main flow of measurement data.

  The rectangles represent processes. The ellipses represent data at rest:
  as files on disk, files on S3 or records in database tables.

  ![Diagram](https://kroki.io/blockdiag/svg/eNrFlD1PwzAQhvf-CstdSTOwFYEEKkgMSJUQU4Uqf5wbU8c2_kCVEP-d2E3SlrZjQzbfG_t9fL47qgxbc0lW6HuEOAgSVVj6ilhAt8iZqDlwajY3IzR3hoJHC2aUcY2Ix0JA8-ErVDkQKVKFYP20LFcyVJFOmKlLY7QsKWFr0LzghsUadCBBGl1SZWhZE6k7fXmgT2o-ttkTvzf2jxvb-ILbB0j2QmQZv16jD2-0wmjR4YNS0nq4IF92LIRULWSisMYHRvSgHK1nCzF72KYCBfof6QiEKuhJBPHBklANDtMZ7_Nw6dfoM0KEQVGSbbG1zRhvTSkTDg6fKOYLYtRAfHSQYkXsEDLQU5urgYG6J0oQ_YAp7hC-nz9Pt2vkwX1J1vRXFnagaXVUXv3el91V253d_MDpvmfP3y-Q_rA-Vzm0GzSn3Q4fuN3RDYVj8aBZj7v3vMdJ1D9__fwCHdgDSg==)

<!--
  blockdiag {
    default_shape = roundedbox;
    Probes [color = "#ffeeee", href = "@@probes"];
    Explorer [color = "#eeeeff"];
    "S3 jsonl" [shape = ellipse, href = "@@jsonl-files"];
    "S3 postcan" [shape = ellipse, href = "@@postcans"];
    "DB jsonl tbl" [shape = ellipse, href = "@@jsonl-table"];
    "DB fastpath tbl" [shape = ellipse, href = "@@fastpath-table"];
    "disk queue" [shape = ellipse, href = "@@disk-queue"];
    "Uploader" [color = "#eeeeff", href = "@@measurement-uploader"];
    "Fastpath" [color = "#eeeeff", href = "@@fastpath"];

    Probes -> "API: Probe services" -> "Fastpath" -> "DB fastpath tbl" -> "API: Measurements" -> "Explorer";
    "API: Probe services" -> "disk queue" -> "API: uploader" -> "S3 jsonl" -> "API: Measurements";
    "Uploader" -> "S3 postcan";
    "Uploader" -> "DB jsonl tbl";
    "DB jsonl tbl" -> "API: Measurements";
    "disk queue" -> "API: Measurements";
  }
-->

Ellipses represent data; rectangles represent processes. Click on the
image and then click on each shape to see related documentation.

Probes submit measurements to the API with a POST at the following path:
<https://api.ooni.io/apidocs/#/default/post_report__report_id_> The
measurement is optionally decompressed if zstd compression is detected.
It is then parsed and added with a unique ID and saved to disk. Very
little validation is done at this time in order to ensure that all
incoming measurements are accepted.

Measurements are enqueued on disk using one file per measurement. On
hourly intervals they are batched together, compressed and uploaded to
S3 by the [Measurement uploader](#measurement-uploader)&thinsp;⚙. The batching is
performed to allow efficient compression. See the
[dedicated subchapter](#measurement-uploader)&thinsp;⚙ for details.

The measurement is also sent to the [Fastpath](#fastpath)&thinsp;⚙. The
Fastpath runs as a dedicated daemon with a pool of workers. It
calculates scoring for the measurement and writes a record in the
fastpath table. Each measurement is processed individually in real time.
See the [dedicated subchapter](#fastpath)&thinsp;⚙ below.

The disk queue is also used by the API to access recent measurements
that have not been uploaded to S3 yet. See the
[measurement API](#getting-measurement-bodies)&thinsp;🐝 for details.


## Reproducibility
The measurement processing pipeline is meant to generate outputs that
can be equally generated by 3rd parties like external researchers and
other organizations.

This is meant to keep OONI accountable and as a proof that we do not
arbitrarily delete or alter measurements and that we score them as
accessible/anomaly/confirmed/failure in a predictable and transparent
way.

> **important**
> The only exceptions were due to privacy breaches that required removal
> of the affected measurements from the [S3 data bucket](#s3-data-bucket)&thinsp;💡
> bucket.

  As such, the backend infrastructure is
  [FOSS](https://en.wikipedia.org/wiki/Free_and_open-source_software) and
  can be deployed by 3rd parties. We encourage researchers to replicate
  our findings.

  Incoming measurements are minimally altered by the
  [Measurement uploader](#measurement-uploader)&thinsp;⚙ and uploaded to S3.


## Supply chain management

The backend implements software supply chain management by installing components and libraries from the Debian Stable archive.

This provides the following benefits:

 * Receiving targeted security updates from the OS without introducing breaking changes.
 * Protection against supply chain attacks.
 * Reproducible deployment for internal use (e.g. testbeds and new backend hosts) and for external researchers.

> **warning**
> [ClickHouse](#clickhouse)&thinsp;⚙ is installed from a 3rd party archive and receives limited security support. As a mitigation an LTS version is used yet the support time is 1 year.

The backend received a security assessment from Cure53 <https://cure53.de/> in 2022.

Low-criticality hosts e.g. the monitoring stack also have components installed from 3rd party archives.


# Application metrics
  All components of the backend are designed to output application
  metrics.

  Metrics are prefixed with the name of each application. The metrics are
  used in [Grafana](#grafana)&thinsp;🔧 for charts, monitoring and alarming.

  They use the [StatsD](#statsd)&thinsp;💡 protocol.

  Application metrics data flow:

  ![Diagram](https://kroki.io/blockdiag/svg/eNq9kc1qAyEUhffzFDLZNnGf0EBX7SoEkl0p4arXUaJe8QcKpe9eZ9Imkz5AXHo-OcdzhCN5VhYG9tUxhRqqK6dsICJ7ZolqUKgEfW469hKjsxKKpcDeJTlKjegXWmM7_UcjdlgUFJiro6Z1_8RMQj3emFJiXnM-2GKqWEnynChYLkCeMailIlk9hjL5cOFIcA82_OmnO33l1SJcTKcA-0Qei8GaH5shXn2nGK8JNIQH9zBcTKcA86mW29suDgS60T23d1ndjda4eX1X9O143B_-t9vg309uuu8fUvvJ0Q==)

<!--
blockdiag {
 default_shape = roundedbox;
 Application [color = "#ffeeee"];
 Netdata [color = "#eeeeff", href = "@@netdata"];
 Prometheus [color = "#eeeeff", href = "@@prometheus"];
 Grafana [color = "#eeeeff", href = "@@grafana"];
 Application -> Netdata [label = "statsd"];
 Netdata -> Prometheus [label = "HTTPS"];
 Prometheus -> Grafana;
}
-->

  Ellipses represent data; rectangles represent processes. Purple
  components belong to the backend. Click on the image and then click on
  each shape to see related documentation.

  [Prometheus](#tool:prometheus) and [Grafana](#grafana)&thinsp;🔧 provide
  historical charts for more than 90 days and are useful to investigate
  long-term trends.

  [Netdata](#netdata)&thinsp;🔧 provides a web UI with real-time metrics. See
  the dedicated subchapter for details.


## StatsD
  All backend components send StatsD metrics over UDP using localhost as destination.

  This guarantees that applications never block on metric generation in
  case the receiver slows down. The StatsD messages are received by
  [Netdata](#netdata)&thinsp;🔧. It automatically tracks any new metric,
  generates averages and summaries as needed and exposes it to
  [Prometheus](#prometheus)&thinsp;🔧 for scraping.
  In the codebase the statsd library is often used as:

  ```python
  from <package_name>.metrics import setup_metrics
  setup_metrics(name="<component_name>")
  metrics.gauge("<metric_name>", <value>)
  ```

  Because of this, a quick way to identify where metrics are being generated
  in the backend codebase is to search e.g.:

  * <https://github.com/search?q=repo%3Aooni%2Fbackend+metrics.gauge&type=code>
  * <https://github.com/search?q=repo%3Aooni%2Fbackend+metrics.timer&type=code>

  Where possible, timers have the same name as the function being timed e.g.
  <https://github.com/search?q=repo%3Aooni%2Fbackend+clickhouse_upsert_summary&type=code>

  See [Conventions](#conventions)&thinsp;💡 for patterns around component naming.


### Metrics list
  This subsection provides a list of the most important application metrics as they
  are shown in Grafana. The names are autogenerated by Netdata based on the
  metric name used in StatsD.

  For example a `@metrics.timer("generate_test_list")` Python decorator is used at:
  <https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/prio.py#L162>.
  Such timer will be processed by Netdata and appear in Grafana as:
  ```
  netdata_statsd_timer_ooni_api_generate_test_list_milliseconds_average
  ```

  The metrics always start with `netdata_statsd` and end with:

  * `_milliseconds_average`
  * `_events_persec_average`
  * `_value_average`

  Also see <https://blog.netdata.cloud/introduction-to-statsd/>

  TIP: StatsD collectors (like Netdata or others) preprocess datapoints by calculating average/min/max values etc.

  Run this to locate where in the backend codbase application metrics
  are being generated:

  ```bash
  find ~ -name '*.py' -exec grep 'metrics\.' -H "{}" \;
  ```

  Metrics for [ASN metadata updater](#asn-metadata-updater)&thinsp;⚙.
  See the [ASN metadata updater dashboard](#asn-metadata-updater-dashboard)&thinsp;📊:

```
netdata_statsd_asnmeta_updater_asnmeta_tmp_len_gauge_value_average
netdata_statsd_asnmeta_updater_asnmeta_update_progress_gauge_value_average
netdata_statsd_asnmeta_updater_fetch_data_timer_milliseconds_average
netdata_statsd_gauge_asnmeta_updater_asnmeta_tmp_len_value_average
netdata_statsd_gauge_asnmeta_updater_asnmeta_update_progress_value_average
netdata_statsd_timer_asnmeta_updater_fetch_data_milliseconds_average
```


Metrics for [CitizenLab test list updater](#citizenlab-test-list-updater)&thinsp;⚙

```
netdata_statsd_citizenlab_test_lists_updater_citizenlab_test_list_len_gauge_value_average
netdata_statsd_citizenlab_test_lists_updater_fetch_citizen_lab_lists_timer_milliseconds_average
netdata_statsd_citizenlab_test_lists_updater_update_citizenlab_table_timer_milliseconds_average
netdata_statsd_gauge_citizenlab_test_lists_updater_citizenlab_test_list_len_value_average
netdata_statsd_gauge_citizenlab_test_lists_updater_rowcount_value_average
netdata_statsd_timer_citizenlab_test_lists_updater_fetch_citizen_lab_lists_milliseconds_average
netdata_statsd_timer_citizenlab_test_lists_updater_rebuild_citizenlab_table_from_citizen_lab_lists_milliseconds_average
netdata_statsd_timer_citizenlab_test_lists_updater_update_citizenlab_table_milliseconds_average
```

Metrics for the [Database backup tool](#database-backup-tool)&thinsp;⚙.
See the [Database backup dashboard](#database-backup-dashboard)&thinsp;📊 on Grafana:

```
netdata_statsd_db_backup_run_export_timer_milliseconds_average
netdata_statsd_db_backup_status_gauge_value_average
netdata_statsd_db_backup_table_fastpath_backup_time_ms_gauge_value_average
netdata_statsd_db_backup_table_jsonl_backup_time_ms_gauge_value_average
netdata_statsd_db_backup_uploaded_bytes_tot_gauge_value_average
netdata_statsd_db_backup_upload_to_s3_timer_milliseconds_average
netdata_statsd_gauge_db_backup_status_value_average
netdata_statsd_gauge_db_backup_table_fastpath_backup_time_ms_value_average
netdata_statsd_gauge_db_backup_table_jsonl_backup_time_ms_value_average
netdata_statsd_gauge_db_backup_uploaded_bytes_tot_value_average
netdata_statsd_timer_db_backup_run_backup_milliseconds_average
netdata_statsd_timer_db_backup_run_export_milliseconds_average
netdata_statsd_timer_db_backup_upload_to_s3_milliseconds_average
netdata_statsd_gauge_db_backup_status_value_average
netdata_statsd_gauge_db_backup_table_citizenlab_byte_count_value_average
netdata_statsd_gauge_db_backup_table_fastpath_backup_time_ms_value_average
netdata_statsd_gauge_db_backup_table_fastpath_byte_count_value_average
netdata_statsd_gauge_db_backup_table_jsonl_backup_time_ms_value_average
netdata_statsd_gauge_db_backup_table_jsonl_byte_count_value_average
netdata_statsd_gauge_db_backup_uploaded_bytes_tot_value_average
netdata_statsd_timer_db_backup_backup_table_citizenlab_milliseconds_average
netdata_statsd_timer_db_backup_backup_table_fastpath_milliseconds_average
netdata_statsd_timer_db_backup_backup_table_jsonl_milliseconds_average
```


Metrics for the [social media blocking event detector](#social-media-blocking-event-detector)&thinsp;⚙:

```
netdata_statsd_gauge_detector_blocking_events_tblsize_value_average
netdata_statsd_gauge_detector_blocking_status_tblsize_value_average
netdata_statsd_timer_detector_run_detection_milliseconds_average
```


Metrics for the [Fastpath](#fastpath)&thinsp;⚙. Used in various dashboards,
primarily [API and fastpath](#api-and-fastpath)&thinsp;📊 dashboard.

```
netdata_statsd_timer_fastpath_db_clickhouse_upsert_summary_milliseconds_average
netdata_statsd_timer_fastpath_db_fetch_fingerprints_milliseconds_average
netdata_statsd_timer_fastpath_full_run_milliseconds_average
netdata_statsd_gauge_fastpath_recent_measurement_count_value_average
```


Metrics [Fingerprint updater](#fingerprint-updater)&thinsp;⚙
See the [Fingerprint updater dashboard](#fingerprint-updater-dashboard)&thinsp;📊 on Grafana.

```
netdata_statsd_timer_fingerprints_updater_fetch_csv_milliseconds_average
netdata_statsd_gauge_fingerprints_updater_fingerprints_dns_tmp_len_value_average
netdata_statsd_gauge_fingerprints_updater_fingerprints_http_tmp_len_value_average
netdata_statsd_gauge_fingerprints_updater_fingerprints_update_progress_value_average
```

Metrics from Nginx caching of the aggregation API.
See [Aggregation cache monitoring](#aggregation-cache-monitoring)&thinsp;🐍

```
netdata_statsd_gauge_nginx_aggregation_cache_EXPIRED_value_average
netdata_statsd_gauge_nginx_aggregation_cache_HIT_value_average
netdata_statsd_gauge_nginx_aggregation_cache_MISS_value_average
netdata_statsd_gauge_nginx_aggregation_cache_UPDATING_value_average
```

Metrics for the [API](#api)&thinsp;⚙.

```
netdata_statsd_counter_ooni_api_geoip_asn_differs_events_persec_average
netdata_statsd_counter_ooni_api_geoip_cc_differs_events_persec_average
netdata_statsd_counter_ooni_api_geoip_ipaddr_found_events_persec_average
netdata_statsd_counter_ooni_api_geoip_ipaddr_not_found_events_persec_average
netdata_statsd_counter_ooni_api_gunicorn_request_status_
netdata_statsd_counter_ooni_api_probe_cc_asn_match_events_persec_average
netdata_statsd_counter_ooni_api_probe_cc_asn_nomatch_events_persec_average
netdata_statsd_counter_ooni_api_probe_legacy_login_successful_events_persec_average
netdata_statsd_counter_ooni_api_probe_login_successful_events_persec_average
netdata_statsd_counter_ooni_api_receive_measurement_count_events_persec_average
netdata_statsd_counter_ooni_api_receive_measurement_discard_asn_
netdata_statsd_counter_ooni_api_receive_measurement_discard_cc_zz_events_persec_average
netdata_statsd_counter_ooni_api_uploader_msmt_count_events_persec_average
netdata_statsd_counter_ooni_api_uploader_postcan_count_events_persec_average
netdata_statsd_gauge_ooni_api_check_in_test_list_count_value_average
netdata_statsd_gauge_ooni_api_spool_post_count_value_average
netdata_statsd_gauge_ooni_api_test_list_urls_count_value_average
netdata_statsd_timer_ooni_api_apicall___api__v
netdata_statsd_timer_ooni_api_citizenlab_lock_time_milliseconds_average
netdata_statsd_timer_ooni_api_citizenlab_repo_init_milliseconds_average
netdata_statsd_timer_ooni_api_citizenlab_repo_pull_milliseconds_average
netdata_statsd_timer_ooni_api_fetch_citizenlab_data_milliseconds_average
netdata_statsd_timer_ooni_api_fetch_reactive_url_list_milliseconds_average
netdata_statsd_timer_ooni_api_generate_test_list_milliseconds_average
netdata_statsd_timer_ooni_api_get_aggregated_milliseconds_average
netdata_statsd_timer_ooni_api_get_measurement_meta_clickhouse_milliseconds_average
netdata_statsd_timer_ooni_api_get_measurement_meta_milliseconds_average
netdata_statsd_timer_ooni_api_get_raw_measurement_milliseconds_average
netdata_statsd_timer_ooni_api_get_torsf_stats_milliseconds_average
netdata_statsd_timer_ooni_api_gunicorn_request_duration_milliseconds_average
netdata_statsd_timer_ooni_api_open_report_milliseconds_average
netdata_statsd_timer_ooni_api_open_report_milliseconds_averageopen_report
netdata_statsd_timer_ooni_api_receive_measurement_milliseconds_average
netdata_statsd_timer_ooni_api_uploader_fill_jsonl_milliseconds_average
netdata_statsd_timer_ooni_api_uploader_fill_postcan_milliseconds_average
netdata_statsd_timer_ooni_api_uploader_total_run_time_milliseconds_average
netdata_statsd_timer_ooni_api_uploader_update_db_table_milliseconds_average
netdata_statsd_timer_ooni_api_uploader_upload_measurement_milliseconds_average
```

Metrics for the [GeoIP downloader](#geoip-downloader)&thinsp;⚙.

```
netdata_statsd_gauge_ooni_download_geoip_geoip_asn_epoch_value_average
netdata_statsd_gauge_ooni_download_geoip_geoip_asn_node_cnt_value_average
netdata_statsd_gauge_ooni_download_geoip_geoip_cc_epoch_value_average
netdata_statsd_gauge_ooni_download_geoip_geoip_cc_node_cnt_value_average
netdata_statsd_timer_ooni_download_geoip_download_geoip_milliseconds_average
```

Metrics for the [test helper rotation](#test-helper-rotation)&thinsp;⚙.

```
netdata_statsd_timer_rotation_create_le_do_ssl_cert_milliseconds_average
netdata_statsd_timer_rotation_deploy_ssl_cert_milliseconds_average
netdata_statsd_timer_rotation_destroy_drained_droplets_milliseconds_average
netdata_statsd_timer_rotation_end_to_end_test_milliseconds_average
netdata_statsd_timer_rotation_run_time_milliseconds_average
netdata_statsd_timer_rotation_scp_file_milliseconds_average
netdata_statsd_timer_rotation_setup_nginx_milliseconds_average
netdata_statsd_timer_rotation_setup_vector_milliseconds_average
netdata_statsd_timer_rotation_spawn_new_droplet_milliseconds_average
netdata_statsd_timer_rotation_ssh_reload_nginx_milliseconds_average
netdata_statsd_timer_rotation_ssh_restart_netdata_milliseconds_average
netdata_statsd_timer_rotation_ssh_restart_nginx_milliseconds_average
netdata_statsd_timer_rotation_ssh_restart_vector_milliseconds_average
netdata_statsd_timer_rotation_ssh_wait_droplet_warmup_milliseconds_average
netdata_statsd_timer_rotation_update_dns_records_milliseconds_average
```


## Prometheus
Prometheus <https://prometheus.io/> is a popular monitoring system and
runs on [monitoring.ooni.org](#monitoring.ooni.org)&thinsp;🖥

It is deployed and configured by [Ansible](#ansible)&thinsp;🔧 using the
following playbook:
<https://github.com/ooni/sysadmin/blob/master/ansible/deploy-monitoring.yml>

Most of the metrics are collected by scraping Prometheus endpoints,
Netdata, and using node exporter. The web UI is accessible at
<https://prometheus.ooni.org>

### Blackbox exporter
Blackbox exporter is part of Prometheus. It's a daemon that performs HTTP
probing against other hosts without relying on local agents (hence the name Blackbox)
and feeds the generated datapoints into Promethous.

See <https://github.com/prometheus/blackbox_exporter>

It is deployed by
[Ansible](#tool:ansible) on the [monitoring.ooni.org](#monitoring.ooni.org)&thinsp;🖥

See
[Updating Blackbox Exporter runbook](#updating-blackbox-exporter-runbook)&thinsp;📒


## Grafana dashboards
There is a number of dashboards on [Grafana](#grafana)&thinsp;🔧 at
<https://grafana.ooni.org/>

[Grafana](#grafana)&thinsp;🔧 is deployed on the
[monitoring.ooni.org](#monitoring.ooni.org)&thinsp;🖥 host. See
[Monitoring deployment runbook](#monitoring-deployment-runbook)&thinsp;📒 for deployment.

The dashboards are used for:

 * Routinely reviewing the general health of the backend infrastructure

 * Predicting long-term scaling requirements, i.e.

 * increasing disk space for the database

 * increasing CPU and memory requirements

 * Investigating alerts and troubleshooting incidents


### Alerting
Alerts from [Grafana](#tool:grafana) and [Prometheus](#prometheus)&thinsp;🔧
are sent to the [#ooni-bots](#topic:oonibots) [Slack](#slack)&thinsp;🔧
channel by a bot.

[Slack](#slack)&thinsp;🔧 can be configured to provide desktop notification
from browsers and audible notifications on smartphones.

Alert flow:

![Diagram](https://kroki.io/blockdiag/svg/eNp1jUEKwjAQRfc9xTBd9wSioBtxV3ApIpNmYktjJiQpCuLdTbvQIDirP7zH_8pKN-qBrvCsQLOhyaZL7MkzrCHI5DRrJY9VBW2QG6eepwinTqyELGDN-YzBcxb2gQw5-kOxFnFDoyRFLBVjZmlRioVm86nLEY-WuhG27QGXt6z6YvIef4dmugtyjxwye70BaPFK1w==)

<!--
blockdiag {
 default_shape = roundedbox;
 Prometheus [color = "#eeeeff"];
 Grafana [color = "#eeeeff"];
 "#ooni-bots" [color = "#ffeeee"];
 Prometheus -> Grafana -> "Slack API" -> "#ooni-bots" -> "Slack app";
 "#ooni-bots" -> "Browser";
}
-->

The diagram does not explicitly include alertmanager. It is part of Prometheus and receives alerts and routes them to Slack.

More detailed diagram:

```mermaid
flowchart LR
    P(Prometheus) -- datapoints --> G(Grafana)
    G --> A(Alertmanager)
    A --> S(Slack API) --> O(#ooni-bots)
    P --> A
    O --> R(Browser / apps)
    J(Jupyter notebook) --> A
    classDef box fill:#eeffee,stroke:#777,stroke-width:2px;
    class P,G,A,S,O,R,J box;
```

In the diagram Prometheus receives, stores and serves datapoints and has some alert rules to trigger alerts.
Grafana acts as a UI for Prometheus and also triggers alerts based on alert rules configured in Grafana itself.

Alertmanager is pretty simple - receives alerts and sends notification to Slack.

The alert rules are listed at <https://grafana.ooni.org/alerting/list>
The list also shows which alerts are firing at the moment, if any. There
is also a handful of alerts configured in [Prometheus](#prometheus)&thinsp;🔧
using [Ansible](#ansible)&thinsp;🔧.

The silences list shows if any alert has been temporarily silenced:
<https://grafana.ooni.org/alerting/silences>

See [Grafana editing](#grafana-editing)&thinsp;📒 and
[Managing Grafana alert rules](#managing-grafana-alert-rules)&thinsp;📒 for details.

There are also many dashboards and alerts configured in
[Jupyter Notebook](#jupyter-notebook)&thinsp;🔧. These are meant for metrics that require more
complex algorithms, predictions and SQL queries that cannot be
implemented using [Grafana](#grafana)&thinsp;🔧 e.g. when using machine learning or Pandas.
See [Ooniutils microlibrary](#ooniutils-microlibrary)&thinsp;💡 for details.

On many dashboards you can set the averaging timespan and the target
hostname using fields on the top left.

Here is an overview of the most useful dashboards:


### API and fastpath
<https://grafana.ooni.org/d/l-MQSGonk/api-and-fastpath-multihost?orgId=1&var-avgspan=1h&var-host=backend-fsn.ooni.org>

This is the most important dashboard showing metrics of the
[API](#comp:api) and the [Fastpath](#fastpath)&thinsp;⚙.


### Test-list repository in the API
<https://grafana.ooni.org/d/siWZslSVk/api-test-list-repo?orgId=1>

This dashboard shows timings around the git repository checked out by the
[API](#api)&thinsp;⚙ that contains the test lists.


### Measurement uploader dashboard
<https://grafana.ooni.org/d/ma3Q6GzVz/api-uploader?orgId=1>

This dashboard shows metrics, timing and amounts of data transferred by the
[Measurement uploader](#measurement-uploader)&thinsp;⚙


### Fingerprint updater dashboard
<https://grafana.ooni.org/d/JNlK8ox4z/fingerprints>

This dashboard shows metrics and timing from the
[Fingerprint updater](#fingerprint-updater)&thinsp;⚙


### ClickHouse dashboard
<https://grafana.ooni.org/d/thEkJB_Mz/clickhouse?orgId=1>

This dashboards show ClickHouse-specific performance metrics.
It can be used for optimizations.

For investigating slow queries also see the [ClickHouse queries notebook](#clickhouse-queries-notebook)&thinsp;📔.


### HaProxy dashboard
<https://grafana.ooni.org/d/ba33e4df-d686-4459-b37d-3966af14ad00/haproxy>

Basic metrics from [HaProxy](#haproxy)&thinsp;⚙ load balancers. Used for
[OONI bridges](#ooni-bridges)&thinsp;⚙.


### TLS certificate dashboard
<https://grafana.ooni.org/d/-1mr7sWMk/ssl-certificates>

Certificate expiration times. There are alerts configured in
[Grafana](#grafana)&thinsp;🔧 to alert on expiring certificates.


### Test helpers dashboard
<https://grafana.ooni.org/d/Dn1R7QEnz/test-helpers>

Status, uptime and load metrics from the
[Test helpers](#test-helpers)&thinsp;⚙.


### Database backup dashboard
<https://grafana.ooni.org/d/aQjQYhoGz/db-backup>

Metrics, timing and data transferred by
[Database backup tool](#database-backup-tool)&thinsp;⚙

By looking at the last 24 hours of run you should be able to see the backup
being run
<https://grafana.ooni.org/d/aQjQYhoGz/db-backup?orgId=1&from=now-24h&to=now>

The "Status" chart shows the running status.
"Uploaded bytes in total" and "Backup time" should be self explanatory.

TIP: If the backup time or size grows too much it could be worth alerting and considering implementing incremental backups.


### Event detector dashboard
<https://grafana.ooni.org/d/FH2TmwFVz/event-detection?orgId=1&refresh=1m>

Basic metrics from the
[social media blocking event detector](#social-media-blocking-event-detector)&thinsp;⚙


### GeoIP MMDB database dashboard
<https://grafana.ooni.org/d/0e6eROj7z/geoip?orgId=1&from=now-7d&to=now>

Age and size of the GeoIP MMDB database. Also, a chart showing
discrepancies between the lookup performed by the probes VS the one in
the API, used to gauge the benefits of using a centralized solution.

Also see [Geolocation script](#geolocation-script)&thinsp;🐍

See [GeoIP downloader](#geoip-downloader)&thinsp;⚙


### Host clock offset dashboard
<https://grafana.ooni.org/d/9dLa-RSnk/host-clock-offset?orgId=1>

Measures NTP clock sync and alarms on big offsets


### Netdata-specific dashboard
<https://grafana.ooni.org/d/M1rOa7CWz/netdata?orgId=1&var-instance=backend-fsn.ooni.org:19999>

Shows all the metrics captured by [Netdata](#netdata)&thinsp;🔧 - useful for
in-depth performance investigation.


### ASN metadata updater dashboard
<https://grafana.ooni.org/d/XRihZL-Vk/ansmeta-update?orgId=1&from=now-7d>

Progress, runtime and table size of the [ASN metadata updater](#asn-metadata-updater)&thinsp;⚙

See [Metrics list](#metrics-list)&thinsp;💡


## Netdata
Netdata <https://www.netdata.cloud/> is a monitoring agent that runs
locally on the backend servers. It exports host and
[Application metrics](#topic:appmetrics) to [Prometheus](#prometheus)&thinsp;🔧.

It also provides a web UI that can be accessed on port 19999. It can be
useful during development, performance optimization and debugging as it
provides metrics with higher time granularity (1 second) and almost no
delay.

Netdata is not exposed on the Internet for security reasons and can be
accessed only when nededed by setting up port forwarding using SSH. For
example:

```bash
ssh ams-pg-test.ooni.org -L 19998:127.0.0.1:19999
```

Netdata can also be run on a development desktop and be accessed locally
in order to explore application metrics without having to deploy
[Prometheus](#tool:prometheus) and [Grafana](#grafana)&thinsp;🔧.

See [Netdata-specific dashboard](#netdata-specific-dashboard)&thinsp;📊 of an example of native
Netdata metrics.


# Log management
All components of the backend are designed to output logs to Systemd's
journald. They usually log using the component name as Systemd unit
name.

Sometimes you might have to use `--identifier <name>` instead for
scripts that are not run as Systemd units.

Journald automatically indexes logs by time, unit name and other items.
This allows to quickly filter logs during troubleshooting, for example:

```bash
sudo journalctl -u ooni-api --since '10 m ago'
```

Or follow live logs using e.g.:

```bash
sudo journalctl -u nginx -f
```

Sometimes it is useful to show milliseconds in the timestamps:

```bash
sudo journalctl -f -u ooni-api -o short-precise
```

The logger used in Python components also sets additional fields,
notably CODE_FUNC and CODE_LINE

Available fields can be listed using:

```bash
sudo journalctl -f -u ooni-api  -N | sort
```

It is possible to filter by those fields. It comes very handy for
debugging e.g.:

```bash
sudo journalctl -f -u ooni-api CODE_FUNC=open_report
```

Every host running backend services also sends host to
monitoring.ooni.org using [Vector](#vector)&thinsp;🔧.

![Diagram](https://kroki.io/blockdiag/svg/eNrFks9qwzAMxu95CpNel_gYWOlgDEqfYJdRiv_IiYltBccphdF3n5yyNellt01H6ZO_nyxJh6rXVrTss2AajJhcOo2dGIDtWMQpaNASL9uCvQ6Ds0oki4FVL-wdVMJYjUCKuEhEUGDPt9QbNfQHnEZ46P9Q6DCSQ7kxBijKIynWTy40WWFM-cS6CGaXU11Kw_jMeWtTN8laoeeIwXIpVE_tlUY1eQhptuPSoeRe2PBdP63qtdeb8-y9xPgZ5N9A7t_3xwwqG3fZOHMUrKVDGPKBUCzWuF1vjIivD-LfboLCCQkuT-EJmcQ2tHWmrzG25U1yn71p9vumKWen6xdypu8x)

<!--
blockdiag {
 default_shape = roundedbox;
 Application -> Vector-sender -> Vector-receiver -> ClickHouse;
 Application [color = "#ffeeee"];
 Vector-sender [color = "#eeeeff", href= = "@@vector"];
 Vector-receiver [color = "#eeeeff", href= = "@@vector"];
 ClickHouse [color = "#eeeeff", href= = "@@clickhouse"];

 group {
    Application; Vector-sender;
 }

 group {
    Vector-receiver -> ClickHouse;
    label = "monitoring.ooni.org";
    color = "#77FF77";
 }
}
-->
There is a dedicated ClickHouse instance on monitoring.ooni.org used to
collect logs. See the [ClickHouse instance for logs](#clickhouse-instance-for-logs)&thinsp;⚙.
This is done to avoid adding unnecessary load to the production database
on FSN that contains measurements and also keep a copy of FSN's logs on
a different host.

The receiving [Vector](#vector)&thinsp;🔧 instance and ClickHouse are
deployed and configured by [Ansible](#ansible)&thinsp;🔧 using the following
playbook:
<https://github.com/ooni/sysadmin/blob/master/ansible/deploy-monitoring.yml>

See [Logs from FSN notebook](#logs-from-fsn-notebook)&thinsp;📔 and
[Logs investigation notebook](#logs-investigation-notebook)&thinsp;📔


## Slack
[Slack](https://slack.com/) is used for team messaging and automated
alerts at the following instance: <https://openobservatory.slack.com/>


### #ooni-bots
`#ooni-bots` is a [Slack](#slack)&thinsp;🔧 channel used for automated
alerts: <https://app.slack.com/client/T37Q8EGUU/C38EJ0CET>


# Systemd timers
Some backend components like the API and Fastpath run as daemons. Many
other run as Systemd timers at various intervals.

The latter approach ensures that a component will start again at the
next time interval even if the previous run crashed out. This provides a
moderate reliability benefit at the expense of having to perform
initialization and shutdown at every run.

To show the existing timers and their next start time run:

```bash
systemctl list-timers
```


## Summary of timers
Here is a summary of the most important timers used in the backend:

    UNIT                           ACTIVATES
    dehydrated.timer               dehydrated.service
    detector.timer                 detector.service
    ooni-api-uploader.timer        ooni-api-uploader.service
    ooni-db-backup.timer           ooni-db-backup.service
    ooni-download-geoip.timer      ooni-download-geoip.service
    ooni-rotation.timer            ooni-rotation.service
    ooni-update-asn-metadata.timer ooni-update-asn-metadata.service
    ooni-update-citizenlab.timer   ooni-update-citizenlab.service
    ooni-update-fingerprints.timer ooni-update-fingerprints.service

Ooni-developed timers have a matching unit file with .service extension.

To show the existing timers and their next start time run:

```bash
systemctl list-timers
```

This can be useful for debugging.


## Dehydrated timer
Runs the Dehydrated ACME tool, see [Dehydrated](#dehydrated)&thinsp;⚙

is a simple script that provides ACME support for Letsencrypt. It's
integrated with Nginx or HaProxy with custom configuration or a small
script as \"glue\".

[Source](https://github.com/ooni/sysadmin/blob/master/ansible/roles/dehydrated/templates/dehydrated.timer)


## Detector timer
Runs the [social media blocking event detector](#social-media-blocking-event-detector)&thinsp;⚙. It is
installed by the [detector package](#detector-package)&thinsp;📦.


## ooni-api-uploader timer
Runs the [Measurement uploader](#measurement-uploader)&thinsp;⚙. It is installed by the
[analysis package](#analysis-package)&thinsp;📦. Runs `/usr/bin/ooni_api_uploader.py`

[Source](https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/debian/ooni-api-uploader.timer)


## ooni-db-backup timer
Runs the [Database backup tool](#database-backup-tool)&thinsp;⚙ as
`/usr/bin/ooni-db-backup` Also installed by the
[analysis package](#analysis-package)&thinsp;📦.

[Source](https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/analysis/debian/ooni-db-backup.timer)


## ooni-download-geoip timer
Fetches GeoIP databases, installed by the [ooni-api](#api)&thinsp;⚙. Runs
`/usr/bin/ooni_download_geoip.py`

Monitored with the [GeoIP dashboard](#geoip-mmdb-database-dashboard)&thinsp;📊

See [GeoIP downloader](#geoip-downloader)&thinsp;⚙

[Source](https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/debian/ooni-download-geoip.timer)


## ooni-rotation timer
Runs the test helper rotation script, installed by the
[analysis package](#analysis-package)&thinsp;📦. Runs `/usr/bin/rotation`

[Source](https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/analysis/debian/ooni-rotation.timer)


## ooni-update-asn-metadata timer
Fetches [ASN](#asn)&thinsp;💡 metadata, installed by the
[analysis package](#analysis-package)&thinsp;📦. Runs `/usr/bin/analysis --update-asnmeta`

[Source](https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/analysis/debian/ooni-update-asn-metadata.timer)


## ooni-update-citizenlab
Fetches CitizenLab data from GitHub, installed by the
[analysis package](#analysis-package)&thinsp;📦. Runs `/usr/bin/analysis --update-citizenlab`

[Source](https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/analysis/debian/ooni-update-citizenlab.timer)


## ooni-update-fingerprints
Fetches fingerprints from GitHub, installed by the
[analysis package](#analysis-package)&thinsp;📦. Runs `/usr/bin/analysis --update-fingerprints`

[Source](https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/analysis/debian/ooni-update-fingerprints.timer)


# Nettest names
Nettest specifications are kept at
<https://github.com/ooni/spec/tree/master/nettests>


## Web connectivity test
Nettest for HTTP/HTTPS connectivity named `web_connectivity`.
See <https://github.com/ooni/spec/blob/master/nettests/ts-017-web-connectivity.md>


## Signal test
Nettest for [Signal Private Messenger](https://signal.org/) named `signal`
See <https://github.com/ooni/spec/blob/master/nettests/ts-029-signal.md>


# API
The API provides entry points used by the probes,
[Explorer](#ui:explorer) [Test List Editor](#test-list-editor)&thinsp;🖱 and other UIs, and
researchers.

Entry points under `/api/v1/` are meant for public consumption and
versioned. Those under `/api/_/` are for internal use.

The API is versioned. Access is rate limited based on source IP address
and access tokens. See [Rate limiting and quotas](#rate-limiting-and-quotas)&thinsp;🐝 for
details.

due to the computational cost of running heavy queries on the database.
The API entry points are documented at
[apidocs](https://api.ooni.io/apidocs/) using
[flasgger](https://flasgger.pythonanywhere.com/). A swagger JSON
specification is published at <https://api.ooni.io/apispec_1.json>

The file is also tracked at
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/docs/apispec.json>
It is checked for consistency by CI in the
[API end-to-end test](#api-end-to-end-test)&thinsp;💡, see
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/.github/workflows/test_new_api.yml#L27>

To regenerate the spec file when implementing changes to the API use:
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/tools/check_apispec_changes>

Before diving into the API codebase it's worth glancing at commonly used
functions:

URL parameter parsing utilities at
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/urlparams.py>

Caching functions `cachedjson` and `nocachejson` at
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/utils.py#L18>

Various database access functions `query_click`, `insert_click` at
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/database.py#L73>

API routes are mounted at:
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/database.py#L73>

Functions related to initialization of the service and configurating
rate limiting:
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/app.py>

> **note**
> Caching can be a source of bugs: enabling or disabling it explicitly in
> the codebase (instead of relying on defaults in Nginx/HaProxy) improves
> readability.

> **important**
> Various queries are designed to support active/standby or active/active
> database setups. See [Overall design](#overall-design)&thinsp;💡 for details.


## API cache
The API uses cacheing functions provided by [Nginx](#nginx)&thinsp;⚙.

Caching functions `cachedjson` and `nocachejson` are defined at
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/utils.py#L18>


## ASN
Autonomous System Number, described at
<https://en.wikipedia.org/wiki/Autonomous_system_(Internet>) It is
stored as `probe_asn` in measurements, and as `probe_asn` column in the
[fastpath table](#fastpath-table)&thinsp;⛁. Used as a search term in
[Searching for measurements](#api:list_msmts) and [Aggregation and MAT](#aggregation-and-mat)&thinsp;🐝

The lookup system in the API is updated by the
[ASN metadata updater](#asn-metadata-updater)&thinsp;⚙. See
[ASN metadata updater dashboard](#asn-metadata-updater-dashboard)&thinsp;📊 and [ooni-update-asn-metadata
timer](#timer:asnmeta_updater).


## Geolocation
The API and implements
[geolocation](https://en.wikipedia.org/wiki/Internet_geolocation) in
order to identify the [ASN](#asn)&thinsp;💡


## Auth
This module implements browser authentication and user accounts. See
[Probe services](#probe-services)&thinsp;🐝 for probe authentication.

It is designed to fit the following requirements:

 * Never store users email address centrally nor IP addresses nor
    passwords

 * Verify email to limit spambots. Do not use CAPCHAs or other 3rd
    party services

 * Support multiple sessions and multiple devices, ability to register
    multiple times

 * Do not leak the existence of absence of accounts for a given email
    address

Workflow:

 * To register the UIs call
    <https://api.ooni.io/apidocs/#/default/post_api_v1_user_register>
    using an email address and the user receives a temporary login link
    by email

 * Upon clicking on the link the UIs call
    <https://api.ooni.io/apidocs/#/default/get_api_v1_user_login> and
    receive a long-lived JWT in a cookie

 * The UIs call any API entry point sending the JWT cookie

 * The UIs call
    <https://api.ooni.io/apidocs/#/default/get_api_v1_user_refresh_token>
    as needed to refresh the JWT

The API als provides entry points to:

 * Get account metadata
    <https://api.ooni.io/apidocs/#/default/get_api___account_metadata>

 * Get role for an existing account
    <https://api.ooni.io/apidocs/#/default/get_api_v1_get_account_role__email_address_>

 * Set account roles
    <https://api.ooni.io/apidocs/#/default/post_api_v1_set_account_role>

 * Expunge sessions (see below)
    <https://api.ooni.io/apidocs/#/default/post_api_v1_set_session_expunge>

Browsers sessions can be expunged to require users to log in again. This
can be used if an account role needs to be downgraded or terminated
urgently.

> **important**
> Account IDs are not the same across test and production instances.

This is due to the use of a configuration variable
`ACCOUNT_ID_HASHING_KEY` in the hashing of the email address. The
parameter is read from the API configuration file. The values are
different across deployment stages as a security feature.

Also see [Creating admin API accounts](#creating-admin-api-accounts)&thinsp;📒 for more
details.

[Sources](https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/auth.py)


## Measurements
This module primarily provides entry points to access measurements,
typically used by Explorer and sometimes directly by users.

Mounted under `/api/v1/measurement/`

[Sources](https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/measurements.py)


### Searching for measurements
The entry point
<https://api.ooni.io/apidocs/#/default/get_api_v1_measurements> provides
measurement searching and listing.

It is primarily used by <https://explorer.ooni.org/search>


### Getting measurement bodies
Various API entry points allow accessing measurement bodies. Typically
the lookup is done by `measurement_uid`:

 * <https://api.ooni.io/apidocs/#/default/get_api_v1_measurement__measurement_uid_>

 * <https://api.ooni.io/apidocs/#/default/get_api_v1_raw_measurement>

 * <https://api.ooni.io/apidocs/#/default/get_api_v1_measurement_meta>

For legacy reasons measurements can also be accessed by `report_id` +
`input` instead of `measurement_uid`

> **important**
> Accessing measurements by `report_id` + `input` does not guarantee
> uniqueness.

The lookup process can access measurement bodies from multiple
locations. The lookup is performed in different order for different
measurements based on the likelihood of success:

 * Scan the local spool directory
    `/var/lib/ooniapi/measurements/incoming/` for fresh measurements

 * Scan other API hosts (if any) for fresh measurements. This is
    required to support active/active backend deployments.

 * Lookup the measurement data in [jsonl table](#jsonl-table)&thinsp;⛁ and then
    fetch the required [JSONL files](#jsonl-files)&thinsp;💡 from
    [S3 data bucket](#s3-data-bucket)&thinsp;💡 and extract the content.


#### Performance considerations
Fetching data from the [S3 data bucket](#s3-data-bucket)&thinsp;💡 bucket can be
resource-intensive. However:

 * Very recent measurements are likely to be found in the local on-disk
    queue instead of having to fetch them from S3. See
    [Measurement uploader](#measurement-uploader)&thinsp;⚙ for details.

 * Frequently accessed measurements benefit from the [API cache](#api-cache)&thinsp;💡.

 * Measurement bodies are rarely accessed. The overall amount of
    measurements is too large for users to explore a significant
    fraction through the web UIs.

Possible improvements are:

 * Compress JSONL files using <https://github.com/facebook/zstd> with
    high compression rates

 * Use a seekable format and store the measurement location in the
    JSONL file in the [jsonl table](#jsonl-table)&thinsp;⛁ expressed in bytes. See
    <https://github.com/facebook/zstd/blob/dev/contrib/seekable_format/README.md>

<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/measurements.py>


### Measurement feedback
This part of the API is used to collect and serve user feedback on
measurements. It uses [msmt_feedback table](#msmt_feedback-table)&thinsp;⛁ and
provides:

 * Getting feedback for an existing measurement
    <https://api.ooni.io/apidocs/#/default/get_api___measurement_feedback__measurement_uid_>

 * Submitting new feedback
    <https://api.ooni.io/apidocs/#/default/post_api___measurement_feedback>

All users can access feedbacks but only authenticated ones can submit
their feedbacks.

Users can submit only one feedback for each measurement. When the
submission entry point is called a second time for the same measurements
the previous feedback is overwritten using database row deduplication.

Valid feedback statuses are:

    blocked
    blocked.blockpage
    blocked.blockpage.http
    blocked.blockpage.dns
    blocked.blockpage.server_side
    blocked.blockpage.server_side.captcha
    blocked.dns
    blocked.dns.inconsistent
    blocked.dns.nxdomain
    blocked.tcp
    blocked.tls
    ok
    down
    down.unreachable
    down.misconfigured

[Sources](https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/measurements.py)


## Aggregation and MAT
The aggregation API leverages the OLAP features of
[ClickHouse](#clickhouse)&thinsp;⚙ to provide summaries and statistics on
measurements. It is primarily used by the
[MAT](https://explorer.ooni.org/chart/mat). It can also be used to
implement other statistics in Explorer or accessed directly by
researchers to extract data.

[Aggregation entry
point](https://api.ooni.io/apidocs/#/default/get_api_v1_aggregation)

[Sources](https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/aggregation.py)

> **important**
> Caching of this entry point should be implemented carefully as new
> measurements are constantly being inserted and old measurements might be
> occasionally reprocessed.

Also see [Aggregation cache monitoring](#aggregation-cache-monitoring)&thinsp;🐍 and
[Investigating heavy aggregation queries runbook](#investigating-heavy-aggregation-queries-runbook)&thinsp;📒.


## Probe services
This part of the API is implemented in the `probe_services.py` module.
It provides entry points that are meant to be used exclusively by
probes.

[Sources](https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/probe_services.py)


### Check-in
This entry point is the preferred way for probes to:

 * Geo-resolve their IP address to [ASN](#asn)&thinsp;💡 and network name.
    See

 * Receive a list of URLs for [Web connectivity test](#web-connectivity-test)&thinsp;Ⓣ

 * Receive a list of test helpers

 * Set flags to implement incremental rollouts and A/B testing new
    features

See <https://api.ooni.io/apidocs/#/default/post_api_v1_check_in>

Test lists are prioritized based on the country code and
[ASN](#asn)&thinsp;💡 of the probes, as well as flags indicating if the
probe is connected to WiFi and the battery is being charged.


## Tor targets
Tor targets are served: at path `/api/v1/test-list/tor-targets`. See
<https://api.ooni.io/apidocs/#/default/get_api_v1_test_list_tor_targets>

They are read from a configuration file. The path is set in the main
configuration file under `TOR_TARGETS_CONFFILE`. It usually is
`/etc/ooni/tor_targets.json`.

To make changes in the Tor targets see the runbook
[Updating tor targets](#updating-tor-targets)&thinsp;📒


## Test helpers list
This entry point provides a list of test helpers to the probes:
<https://api.ooni.io/apidocs/#/default/get_api_v1_test_helpers>

> **important**
> Test helpers addresses are served with a load-balancing algorithm. The
> amount requests per second they receive should be consistent across
> hosts, except for `0.th.ooni.org`.

`0.th.ooni.org` is treated differently from other test helpers:
it receives less traffic to allow testing new releases with lower impact.

See
<https://github.com/ooni/backend/blob/86c6c7e1d297fb8361a162f6081e5e138731e492/api/ooniapi/probe_services.py#L480>


### Miscellaneous probe configuration data
Various endpoints provide data to configure the probe:

 * <https://api.ooni.io/apidocs/#/default/get_api_v1_collectors>

 * <https://api.ooni.io/apidocs/#/default/get_api_v1_test_list_psiphon_config>

 * <https://api.ooni.io/apidocs/#/default/post_bouncer_net_tests>


### Probe registration and login
Two entry points provide probe registration and login. The mechanism and
the accounts are legacy and completely independent from
[Auth](#auth)&thinsp;🐝.

The workflows follow these steps:

 * A new probe registers and receives a `client_id` token using
    <https://api.ooni.io/apidocs/#/default/post_api_v1_register>

 * The token is stored permanently on the probe

 * The probe calls
    <https://api.ooni.io/apidocs/#/default/post_api_v1_login> when
    needed and receives a temporary token

 * The probe calls check-in supplying the temporary token

On [backend-fsn.ooni.org](#backend-fsn.ooni.org)&thinsp;🖥 the two entry points are currently
being redirected to a different host implementing
<https://orchestrate.ooni.io/> while other backend hosts are exposing
the endpoints in the API.

> **important**
> The probe authentication implemented in the API is not backward
> compatible with credentials already generated by Orchestrate and stored
> by existing probes.


### Measurement submission
The probe services module also provides entry points to submit
measurements. The submission is done in steps:

1.  The probe opens a new report at
    <https://api.ooni.io/apidocs/#/default/post_report>

2.  The probe submit one or more measurements with one HTTP POST each at
    <https://api.ooni.io/apidocs/#/default/post_report__report_id_>

3.  The probe optionally closes the report using
    <https://api.ooni.io/apidocs/#/default/post_report>*report_id*close
    Closing reports is currently unnecessary.


### Robots.txt
Probe services also serve the `robots.txt` file at
<https://api.ooni.io/robots.txt>
<https://api.ooni.io/apidocs/#/default/get_robots_txt>

This is use to block or throttle search engines and other bots that in
the past caused significant load on the API.

> **note**
> some aggressive bots might ignore `robots.txt`. See
> [Limiting scraping](#limiting-scraping)&thinsp;📒

<https://api.ooni.io/apidocs/#/default/get_stats>


### Incident management
The incident management module implements ways for users to create,
update and list incidents.

Related:
<https://docs.google.com/document/d/1TdMBWD45j3bx7GRMIriMvey72viQeKrx7Ad6DyboLwg/>

Accounts with \"admin\" role can perform the same actions as regolar
users and additionally can publish, unpublish and delete incidents.

All the routes related to this module are mounted under
`/api/v1/incidents/`:

 * Search and list incidents:
    <https://api.ooni.io/apidocs/#/default/get_api_v1_incidents_search>

 * Show an incident:
    <https://api.ooni.io/apidocs/#/default/get_api_v1_incidents_show__incident_id_>

 * Create or update an incident:
    <https://api.ooni.io/apidocs/#/default/post_api_v1_incidents__action_>
    Search/list incidents with:

 * Filtering by domain/cc/asn/creator id/ and so on

 * Sort by creation/edit date, event date, and so on

Users can only update/delete incidents created by themselves. Admins can
update/delete everything.

Incidents are stored in the [incidents table](#incidents-table)&thinsp;⛁

See
[Sources](https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/incidents.py)


### Prioritization
This module implements reactive prioritization for URLs in the test list
served to the probes.

`/api/v1/check-in` and `/api/v1/test-list/urls` provide dynamic URL
tests lists for [Web connectivity test](#web-connectivity-test)&thinsp;Ⓣ based on the
CitizenLab URL list and the measurements count from the last 7 days.

The `prio.py` module is used mainly by the [Probe services](#probe-services)&thinsp;🐝 API
and secondarily by the `private_api_check_in` method in the
[Private entry points](#private-entry-points)&thinsp;🐝.

For changing prioritization rules see
[Prioritization rules UI](#ui:priomgm) and [Prioritization management](#prioritization-management)&thinsp;🐝

![Diagram](https://kroki.io/blockdiag/svg/eNrNU01Lw0AUvPdXLOnVNPeKgkiEgmixvYmE_XjbLN3sC7svIoj_3axpY0MTb6J7nZllZpgnLMq9MnzH3mdMgeaNpSKUvAZ2xTw2ToES-HY5Y8nao4CQsGeJFn0LJ3OtoX3JS4Q1D1RzKhlxYaGlHX8Ba00d4IKVHnSUlUR1WGbZzlDZiIXEKkN0JhNc7sGpVKFsKnDEyaDLhEWRVdy4I14M8EWl5rL1SeBDwYMrCAIV1gRKOyNf5hpvi9ob9IYMhD-wODRwam3c_D_r72a9WjIPEswrsCpUNJhBHIHWHfPuMIMxwi9GOK7vxG6se8pmt2WWXo9Hs1yAjZr142Y72UBUn8QdEX2jkXt2Ib1i9bDJn7bjdxSVkxvpf-AN4c976rMeeumlm_w-v92eFRf5_cn3ZFmC3KfGRfrHJwZAcmE=)

<!--
blockdiag {
 default_shape = roundedbox;
 "Probes" [color = "#ffeeee"];
 "fastpath table" [shape = ellipse, href = "@@counters_asn_test_list-table"];
 "url_priorities table" [shape = ellipse, href = "@@url_priorities-table"];
 "counters_asn_test_list" [shape = ellipse, href = "@@counters_asn_test_list-table"];
 "API: receive msmt" [color = "#eeeeff"];
 "Fastpath" [color = "#eeeeff", href = "@@fastpath"];
 "API: prio" [color = "#eeeeff"];
 Probes -> "API: receive msmt" [label = "POST"];
 "API: receive msmt" -> "Fastpath" [label = "POST"];
 "Fastpath" -> "fastpath table" [label = "INSERT"];
 "fastpath table" -> "counters_asn_test_list" [label = "auto"];
 "counters_asn_test_list" -> "API: prio" [label = "SELECT"];
 "API: prio" -> "Probes" [label = "check-in"];
}
-->
Ellipses represent data; rectangles represent processes. Purple
components belong to the backend. Click on the image and then click on
each shape to see related documentation.

In the diagram arrows show information flow.

The prioritization system implements a feedback mechanism to provide
efficient coverage of URLs in [Web connectivity test](#web-connectivity-test)&thinsp;Ⓣ in
[ASN](#asn)&thinsp;💡 with low coverage.

Measurements from probes are received by the [API](#api)&thinsp;⚙, sent to
the [Fastpath](#fastpath)&thinsp;⚙ and then stored in the
[fastpath table](#tbl:fastpath). [ClickHouse](#clickhouse)&thinsp;⚙ automatically
updates the [counters_asn_test_list table](#counters_asn_test_list-table)&thinsp;⛁
in real time. See the link for details on the table contents.

Later on probes call API entry points like
<https://api.ooni.io/apidocs/#/default/post_api_v1_check_in> and receive
new URLs (inputs) for [Web connectivity test](#web-connectivity-test)&thinsp;Ⓣ. The URLs
are ordered according to the priorities set in the
[url_priorities table](#url_priorities-table)&thinsp;⛁ and the amount of measurements gathered in
the past days from probes in the same [ASN](#asn)&thinsp;💡.

[prio.py
sources](https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/prio.py)

[private API
sources](https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/private.py)

[probe services
sources](https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/probe_services.py)

For debugging see
<https://api.ooni.io/apidocs/#/default/get_api___debug_prioritization>
and [Test list prioritization monitoring](#test-list-prioritization-monitoring)&thinsp;🐍


### Priorities and weights
URLs have priorities based on the rules from the
[url_priorities table](#url_priorities-table)&thinsp;⛁.

Prioritization rules can be viewed and edited by accounts with `admin`
rights on <https://test-lists.ooni.org/prioritization>

The
[compute_priorities](https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/prio.py#L102)
function calculates priority and weight for each supplied URL.

Priorities are calculated by matching all the prioritization rules to
each URL in the [citizenlab table](#citizenlab-table)&thinsp;⛁. They do not depend
on the amount of past measurements.

Priorities values are relative, e.g. if one URL has a priority of 800
and another one has 200 the first should be measured 4 times more often
than the latter.

The URLs sent to the probes are ordered from the most urgent to the
least urgent by calculating weights as `priority / measurement count`.
This is done with a granularity of a single country code +
[ASN](#asn)&thinsp;💡 pair.

Probes start performing [Web connectivity test](#web-connectivity-test)&thinsp;Ⓣ from the
top of the list.

You can inspect generated priorities with the
[Priorities and weights notebook](#priorities-and-weights-notebook)&thinsp;📔 or using the API at
<https://api.ooni.io/apidocs/>*/default/get_api_v1_test_list_urls or
<https://api.ooni.io/apidocs/>*/default/get_api\_\_\_debug_prioritization
e.g.:

    $ curl -s 'https://api.ooni.io/api/v1/test-list/urls?country_code=IT&probe_asn=3269&debug=True' | jq -S | less

    $ curl -s 'https://ams-pg-test.ooni.org/api/_/debug_prioritization?probe_cc=IT&probe_asn=3269&limit=9999' | jq -S | less


## Private entry points
The `private.py` module provides many entry points not meant for public
consumption. They are not versioned, mounted under `/api/_` and used
exclusively by:

 * [Explorer](#explorer)&thinsp;🖱

 * [Test List Editor](#test-list-editor)&thinsp;🖱

Statistics and summaries, mainly for Explorer:

 * <https://api.ooni.io/apidocs/#/default/get_api___asn_by_month>

 * <https://api.ooni.io/apidocs/#/default/get_api___circumvention_runtime_stats>

 * <https://api.ooni.io/apidocs/#/default/get_api___circumvention_stats_by_country>

 * <https://api.ooni.io/apidocs/#/default/get_api___countries>

 * <https://api.ooni.io/apidocs/#/default/get_api___countries_by_month>

 * <https://api.ooni.io/apidocs/#/default/get_api___country_overview>

 * <https://api.ooni.io/apidocs/#/default/get_api___domain_metadata>

 * <https://api.ooni.io/apidocs/#/default/get_api___domains>

 * <https://api.ooni.io/apidocs/#/default/get_api___global_overview>

 * <https://api.ooni.io/apidocs/#/default/get_api___global_overview_by_month>

 * <https://api.ooni.io/apidocs/#/default/get_api___im_networks>

 * <https://api.ooni.io/apidocs/#/default/get_api___im_stats>

 * <https://api.ooni.io/apidocs/#/default/get_api___network_stats>

 * <https://api.ooni.io/apidocs/#/default/get_api___networks>

 * <https://api.ooni.io/apidocs/#/default/get_api___test_coverage>

 * <https://api.ooni.io/apidocs/#/default/get_api___test_names>

 * <https://api.ooni.io/apidocs/#/default/get_api___vanilla_tor_stats>

 * <https://api.ooni.io/apidocs/#/default/get_api___website_networks>

 * <https://api.ooni.io/apidocs/#/default/get_api___website_stats>

 * <https://api.ooni.io/apidocs/#/default/get_api___website_urls>

Misc functions:

 * [ASN](#asn)&thinsp;💡 metadata
    <https://api.ooni.io/apidocs/#/default/get_api___asnmeta>

 * Check uploaded reports
    <https://api.ooni.io/apidocs/#/default/get_api___check_report_id>

For debugging:
<https://api.ooni.io/apidocs/#/default/get_api___quotas_summary> See
[Rate limiting and quotas](#rate-limiting-and-quotas)&thinsp;🐝 for details.

> **note**
> There are other entry points under `/api/_` that are not part of this
> module, e.g. [OONI Run](#ooni-run)&thinsp;🐝

[Sources](https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/private.py)


## Rate limiting and quotas
The API is provided with rate limiting functions and traffic quotas to
provide fair use and protect the database from overloading. It was
initially implemented to protect PostgreSQL used in the past.

The rate limiting is based on multiple usages quotas with monthly,
weekly and daily limits. The limit are applied to `/24` subnets where
HTTP connections are coming from by default, or with a token system for
authenticated accounts. Quotas are stored in
[LMDB](http://www.lmdb.tech/doc/) in order to track the values
consistently across API processes with minimal increase in CPU and I/O
load.

Resource usage can vary widely between different API entry points and
query parameters. In order to account resource consumption both in terms
of CPU and disk I/O quotas are consumed based on the wallclock time
taken to to process each API call. This means that i.e. an API call that
takes 2 seconds consumes 20 times more quota than a call that takes 100
ms.

When any of the monthly, weekly and daily quotas are exceeded users
receive HTTP 429 (Too Many Requests) until quotas are incremented again.
Increments happen every hour.

There's an API call to get a summary of used quotas:
<https://api.ooni.io/api/_/quotas_summary>
See [Investigating heavy aggregation queries runbook](#investigating-heavy-aggregation-queries-runbook)&thinsp;📒 for usage examples.

Configuration for rate limiting is at:
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/app.py>

[Sources](https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/rate_limit_quotas.py)


## OONI Run
This module implements management of OONI Run links.

All the routes related to this module are mounted under
`/api/v1/ooni_run/`:

 * List OONIRun descriptors
    <https://api.ooni.io/apidocs/#/default/get_api___ooni_run_list>

 * Archive an OONIRun descriptor and all its past versions
    <https://api.ooni.io/apidocs/#/default/post_api>**ooni_run_archive*ooni_run_link_id*

 * Create a new oonirun link or a new version for an existing one
    <https://api.ooni.io/apidocs/#/default/post_api___ooni_run_create>

 * Fetch OONIRun descriptor by creation time or the newest one
    <https://api.ooni.io/apidocs/#/default/get_api>**ooni_run_fetch*ooni_run_link_id*

Specifications are published at:
<https://github.com/ooni/spec/blob/master/backends/bk-005-ooni-run-v2.md>

OONI Run links can be updated by sending new translations and new
versions. Each entry is stored as a new database row. The creation entry
point detects if the new submission contains only translation changes.
In that case it only updates `translation_creation_time`. Otherwise it
also updates `descriptor_creation_time`. The two values are then used by
the probe to access either the latest translation for a given
`descriptor_creation_time`, or the latest version overall.

[Sources](https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/oonirun.py)


## CitizenLab
The `citizenlab.py` module contains entry points related to managing
both the [CitizenLab Test List](#citizenlab-test-list)&thinsp;💡 and
[Prioritization management](#prioritization-management)&thinsp;🐝.

This subchapter describes the first part.

The following entry points allow authenticated users to propose changes
to the CitizenLab repository. It is a private API used by
[Test List Editor](#test-list-editor)&thinsp;🖱. The API manages an internal clone of the CitizenLab
repository for each user that goes through the workflow.

Entry points:

 * Fetch Citizenlab URL list and additional metadata
   <https://api.ooni.io/apidocs/#/default/get_api___url_submission_test_list__country_code_>

 * Create/update/delete a CitizenLab URL entry. Changes are stored in a
    temporary git repository in the API
    <https://api.ooni.io/apidocs/#/default/post_api_v1_url_submission_update_url>

 * Submit changes by opening a pull request on the CitizenLab
    repository
    <https://api.ooni.io/apidocs/#/default/post_api_v1_url_submission_submit>

The repository goes through the following state machine:

```mermaid

stateDiagram-v2
    direction LR
    classDef green fill:#eeffee,black,font-weight:bold,stroke-width:2px

    [*] --> CLEAN: checkout
    CLEAN:::green --> IN_PROGRESS: make changes
    IN_PROGRESS:::green --> PR_OPEN: open PR
    PR_OPEN:::green --> IN_PROGRESS: close PR
    PR_OPEN --> CLEAN: PR merged/rejected
```


Description of the states:

 * ● - the local repository does not exist yet

 * CLEAN - the local repository has no changes and it is in sync with
    the CitizenLab public repository

 * IN_PROGRESS - there are some changes in the working tree but they
    have not been pushed to the public repository's pull request branch

 * PR_OPEN - a pull request is open

Users can open a pull request and close it to make further changes. The
\"PR merged/rejected\" edge in the state machine diagram the only
transition that is not started by the user.

See [CitizenLab test list updater](#citizenlab-test-list-updater)&thinsp;⚙
for a description of the data flow.

[Sources](https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/citizenlab.py)

See [Metrics list](#metrics-list)&thinsp;💡 for application metrics.


## Prioritization management
This part of the API is used by the OONI team to manage prioritization
rules for URLs used by [Web connectivity test](#web-connectivity-test)&thinsp;Ⓣ. It lives
in the `citizenlab.py` module.

The UI is at <https://test-lists.ooni.org/prioritization> and it is part
of the [Test List Editor](#test-list-editor)&thinsp;🖱. It is available to accounts with
`admin` role only.

See [Prioritization](#prioritization)&thinsp;🐝 for the prioritization rules logic.

There are two entry points:

 * List URL priority rules
    <https://api.ooni.io/apidocs/#/default/get_api___url_priorities_list>

 * Add/update/delete an URL priority rule
    <https://api.ooni.io/apidocs/#/default/post_api___url_priorities_update>

The changes are immediately applied to the
[url_priorities table](#tbl:url_priorities) and used by [Prioritization](#prioritization)&thinsp;🐝.


# Public and private web UIs

## Explorer
UI to display measurements and aggregated data to the users
<https://explorer.ooni.org/>

Fetches data from the [API](#api)&thinsp;⚙


## CitizenLab Test List
A list of URLs for [Web connectivity test](#web-connectivity-test)&thinsp;Ⓣ managed by
the [CitizenLab project](https://citizenlab.ca/).

The repository lives at <https://github.com/citizenlab/test-lists>

It is fetched automatically by the
[CitizenLab test list updater](#citizenlab-test-list-updater)&thinsp;⚙.


## Test List Editor
UI to allow authenticated users to submit or edit URLs in the CitizenLab
Test List <https://test-lists.ooni.org/>

Documented at <https://ooni.org/support/github-test-lists/>

Fetches data from the [CitizenLab](#citizenlab)&thinsp;🐝 API.


## Prioritization rules UI
UI for internal use to manage prioritization rules. It is available at
<https://test-lists.ooni.org/prioritization> and it is part of the
[Test List Editor](#test-list-editor)&thinsp;🖱.

See [Prioritization management](#prioritization-management)&thinsp;🐝 for details.


# Fastpath
The fastpath is a data processing pipeline designed to process incoming
measurements in real time.

It also supports processing old measurements by fetching them from the
[S3 data bucket](#s3-data-bucket)&thinsp;💡.

 * Generate scoring metadata and flag them as `confirmed`, `anomaly` as
    needed.

 * Detect invalid measurements (e.g. from the future) and flag them as
    `failed`.

 * Extract metadata from measurements e.g. `engine_version`

 * Write scoring and metadata to the [fastpath table](#fastpath-table)&thinsp;⛁

 * Extract OpenVPN observations into [obs_openvpn table](#obs_openvpn-table)&thinsp;⛁

Data flow diagram:

```mermaid

flowchart LR

      A(Probes) --> B(API) --> C(Fastpath HTTP)
      C --> D(queue) --> E(Fastpath worker)
      E --> F(fa:fa-database fastpath)

      style A fill:#ffeeee
      style B fill:#eeffee
      style C fill:#eeffee
      style D fill:#eeffee
      style E fill:#eeffee
      style F fill:#eeffee
```


![Diagram](https://kroki.io/blockdiag/svg/eNq1kctqwzAQRff-isHZNhFpdg0tdNldoMsQgh7jWLGkUfWAlNJ_rxwS1y54WW0kzbnM444wJDul-Qm-KlDY8GzSMbbcIzxDoOwUKkGXbQX1-wbOkZypYX8XoDHaRzzcsKeYJHdxRqF07OAjY8YZwTU9JC7MjGIXSGCEvctWYEBV6LqPv-7eJsHHB2gDNuVVtyn5-MTYSac2i5Uky4icZoLLDp1aKpLZoks8aXJMGBLMcu3u_DjhK6sW3Ou6r5m9Ia4wTApv_rFwsSPQ5bMeGbF8uY5erom55T9017Nhc-O2b2DY2V82Xsa2-vVekqHQD7hoGiyn7-f7B2qbw7M=)

<!--
blockdiag {
 default_shape = roundedbox;
 "S3 jsonl" [shape = ellipse];
 "S3 postcans" [shape = ellipse];
 "disk queue" [shape = ellipse];
 "jsonl table" [shape = ellipse];
 Probes [numbered = 1];
 API [numbered = 2, href = "@@api"];
 uploader [numbered = 3, href = "@@haproxy"];
 Probes -> API -> "disk queue" -> uploader -> "S3 jsonl";
 uploader -> "S3 postcans";
 uploader -> "jsonl table";

 Probes [color = "#ffeeee"];
}
-->
Ellipses represent data; rectangles represent processes. Click on the
image and then click on each shape to see related documentation.

The fastpath has been designed around a set of goals:

 * Resiliency: the processing pipeline is lenient towards measurements
    with missing or incorrect fields to mitigate the impact of known and
    unknown bugs in the incoming measurements. Also it is designed to
    minimize the risk of crashing out, blocking or deadlocking.
    Ancillary data sources e.g.
    [Fingerprint updater](#fingerprint-updater)&thinsp;⚙ have failure modes that do not
    block the fastpath.

 * Horizontal scalability: Measurement processing is stateless and
    supports lockless multiprocessing. Measurement collection, spooling
    and processing can scale horizontally on multiple hosts.

 * Security: measurements are received from a public API and treated as
    untrusted input. All backend components are built on a secure SBOM
    and are sandboxed using minimum privileges.

 * Maintainability: all data flows in one direction from the API
    through a simple queue system to the database. The only information
    flowing in the opposite direction is backpressure to prevent
    stalling or unbounded RAM usage when the CPUs are saturated. The
    code style favors simplicity and readability over speed and feature
    richness.

 * Support unit, functional, end-to-end integration testing, CI/CD.


## Core logic
Python module:
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/fastpath/fastpath/core.py>

Implement the main logic:

 * Parse CLI options and configuration files and manage local
    directories.

 * Fetch and prepare HTTP and DNS fingerprints from the
    [fingerprints_dns table](#fingerprints_dns-table)&thinsp;⛁ and
    [fingerprints_http table](#fingerprints_http-table)&thinsp;⛁. See
    [Fingerprint updater](#fingerprint-updater)&thinsp;⚙.

 * Spawn a local HTTP server to receive fresh measurements from the
    API. See `localhttpfeeder.py`

 * Spawn and manage a worker pool to scale out horizontally on
    available CPU cores.

 * Alternatively, feed measurements from the [S3 data bucket](#s3-data-bucket)&thinsp;💡.

 * Process incoming measurements, see the `process_measurement`
    function.

 * Score each measurement individually, see the `score_measurement`
    function. The scoring functions e.g. `score_measurement_telegram`
    are meant to focus only on test-specific data, be stateless and
    unit-testable in isolation.

 * Generate anomaly, confirmed and failure flag consistently with
    [Explorer](#explorer)&thinsp;🖱 and the batch pipeline used before.

 * Extract metadata and upsert each measurement into
    [fastpath table](#fastpath-table)&thinsp;⛁ in `clickhouse_upsert_summary`

The fastpath also supports buffering writes into large batches to avoid
single-record insert queries in ClickHouse. This provides a 25x speedup
when [Reprocessing measurements](#reprocessing-measurements)&thinsp;📒 from
[S3 data bucket](#s3-data-bucket)&thinsp;💡.

This is not meant to be used for real-time measurement scoring as it
would create risk of losing multiple records in case of failed query,
crash, etc and also increase latency.

> **note**
> Batching writes can also be implemented in ClickHouse using
> [Buffer Table Engine](https://clickhouse.com/docs/en/engines/table-engines/special/buffer)
> or
> [async insert](https://clickhouse.com/docs/en/optimize/asynchronous-inserts)


## Database module
Python module:
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/fastpath/fastpath/db.py>

Implements all the database-related functions. The rest of the codebase
is database-agnostic.

> **note**
> Upserts are based on the MergeTree table engine, see
> [Overall design](#overall-design)&thinsp;💡.


## S3 Feeder
Python module:
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/fastpath/fastpath/s3feeder.py>

Implements the fetching of measurements from
[S3 data bucket](#s3-data-bucket)&thinsp;💡. The rest of the codebase is agnostic of S3.

It supports new and legacy cans and JSON and YAML formats.

See [Feed fastpath from JSONL](#feed-fastpath-from-jsonl)&thinsp;🐞


## YAML normalization
Python module:
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/fastpath/fastpath/normalize.py>

Transforms legacy measurement format in YAML into JSON. YAML form is
legacy and not used for new measurements.


# Test helpers
Test helpers are hosts that provide the test helper `oohelperd` service
to probes. They are deployed by
[Test helper rotation](#test-helper-rotation)&thinsp;⚙ and tracked in
[test_helper_instances table](#test_helper_instances-table)&thinsp;⛁.

They have names and DNS entries `<number>.th.ooni.org`. See
[Test helper rotation](#test-helper-rotation)&thinsp;⚙ for details on the deployment
process.

Test helpers send metrics to [Prometheus](#prometheus)&thinsp;🔧 and send
logs to [monitoring.ooni.org](#monitoring.ooni.org)&thinsp;🖥.

See [Test helpers dashboard](#test-helpers-dashboard)&thinsp;📊 for metrics and
alarming and [Test helpers failure runbook](#test-helpers-failure-runbook)&thinsp;📒 for
troubleshooting.

The address of the test helpers are provided to the probes by the API in
[Test helpers list](#test-helpers-list)&thinsp;🐝.
`0.th.ooni.org` is treated differently from other test helpers.



## Analysis
Miscellaneous scripts, services and tools. It contains ancillary
components that are not updated often and might not justify a dedicated
Debian package for each of them.

Deployed using the [analysis package](#analysis-package)&thinsp;📦

<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/analysis/>

Data flows from various updaters:

![Diagram](https://kroki.io/blockdiag/svg/eNrFVF1LwzAUfd-vCN2rW94VhaGoAxmF7W3IuGlu17A0N6SpiuJ_t1tXSLuOobDZt_vRc87N_RCako1UsGZfAyYxhVL7VZGBRXbLHJVGohT0cTNg0WQ-Yw4tRWx0V1ulleDR1Q4oTI4emAehceeaxNP2f8sGGLVWtsDXbfgJaRoHwLUt6d1oAtmg8zdwXCvBiYwCq0KCEKGX4l559YnmBUTAEzhbdQT-g1IOgE7R7RG6aVcsc5hWdpR5b4trztfKZ6UYJ5TvKuQCkg0aOZKUlDkaD16R4UKT4Dko08RXrfg4l8OkJtcgRjX5TtLDbM5SZdbo7Jh5oS8qaU_slPHFSpoiFPYIhbfgs0rQ-fgbjpoxUBOMQ8vdGojDt6u8je4_IT4vFvGvIXtHrQfpvxq7RQ8727kHF5S1Z26NWW8vlglpclsNQ6y-NI26-3sqtXUFj-QcHrRjYPG0L3bOl6oOaXcNL8kfbub3D9DuO-g=)

<!--
blockdiag {
 default_shape = roundedbox;
 "ASN repo" -> "ASN updater" -> "asnmeta table" -> API;
 "ASN repo" [shape = ellipse];
 "GeoIP repo" -> "GeoIP downloader" -> "/var/lib/ooniapi" -> API;
 "GeoIP repo" [shape = ellipse];
 "CitizenLab repo" -> "CitizenLab updater" -> "CitizenLab table" -> API;
 "CitizenLab repo" [shape = ellipse];
 "CitizenLab table" [shape = ellipse, href = "@@citizenlab-table"];
 "DNS fingerp. tbl" [shape = ellipse, href = "@@fingerprints_dns-table"];
 "Fastpath" [href = "@@fastpath"];
 "Fingerprints repo" -> "Fingerprints updater" -> "DNS fingerp. tbl" -> Fastpath;
 "Fingerprints repo" -> "Fingerprints updater" -> "HTTP fingerp. tbl" -> Fastpath;
 "Fingerprints repo" [shape = ellipse];
 "HTTP fingerp. tbl" [shape = ellipse, href = "@@fingerprints_http-table"];
 "asnmeta table" [shape = ellipse, href = "@@asnmeta-table"];
 "Fingerprints updater" [color = "#eeeeff"];
 "CitizenLab updater" [color = "#eeeeff"];
 "ASN updater" [color = "#eeeeff"];
 "GeoIP downloader" [color = "#eeeeff"];
 "API" [color = "#eeeeff", href = "@@api"];
 "Fastpath" [color = "#eeeeff", href = "@@fastpath"];
}
-->

Ellipses represent data; rectangles represent processes. Purple
components belong to the backend. Click on the image and then click on
each shape to see related documentation.

See the following subchapters for details:


### CitizenLab test list updater
This component fetches the test lists from
[CitizenLab Test List](#citizenlab-test-list)&thinsp;💡 and populates the
[citizenlab table](#citizenlab-table)&thinsp;⛁ and
[citizenlab_flip table](#citizenlab_flip-table)&thinsp;⛁.

```mermaid

flowchart LR

    A(Github repository) --> B(updater)
    B --> C(fa:fa-database citizenlab) --> E(API)
    B --> D(fa:fa-database citizenlab_flip)
    E --> TLE(Test List Editor)

    style B fill:#ffeeee
    style A fill:#eeffee
    style C fill:#eeffee
    style D fill:#eeffee
    style E fill:#eeffee
```


The git repository <https://github.com/citizenlab/test-lists.git> is
cloned as an unauthenticated user.

Database writes are performed as the `citizenlab` user.

The tables have few constraints on the database side: most of the
validation is done in the script and it is meant to be strict. The
updater overwrites [citizenlab_flip table](#citizenlab_flip-table)&thinsp;⛁ and
then swaps it with [citizenlab table](#citizenlab-table)&thinsp;⛁ atomically. In
case of failure during git cloning, verification and table overwrite the
final swap does not happen, leaving the `citizenlab` table unaltered.

It is deployed using the [analysis package](#analysis-package)&thinsp;📦 and started
by the [ooni-update-citizenlab](#ooni-update-citizenlab)&thinsp;⏲ Systemd timer.

Logs are generated as the `analysis.citizenlab_test_lists_updater` unit.

Also it generates the following metrics with the
`citizenlab_test_lists_updater` prefix:

| Metric name | Type | Description |
| -------- | ------- | ------ |
| `fetch_citizen_lab_lists` | timer | Fetch duration |
| `update_citizenlab_table` | timer | Update duration |
| `citizenlab_test_list_len` | gauge | Table size |

The updater lives in one file:
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/analysis/analysis/citizenlab_test_lists_updater.py>

To run the updater manually during development:

    PYTHONPATH=analysis ./run_analysis --update-citizenlab --dry-run --stdout


### Fingerprint updater
This component fetches measurement fingerprints as CSV files from
<https://github.com/ooni/blocking-fingerprints> and populates
[fingerprints_dns table](#fingerprints_dns-table)&thinsp;⛁,
[fingerprints_dns_tmp table](#fingerprints_dns_tmp-table)&thinsp;⛁,
[fingerprints_http table](#fingerprints_http-table)&thinsp;⛁ and
[fingerprints_http_tmp table](#fingerprints_http_tmp-table)&thinsp;⛁.

The tables without `_tmp` are used by the [Fastpath](#fastpath)&thinsp;⚙.

The CSV files are fetched directly without git-cloning.

Database writes are performed as the `api` user, configured in
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/analysis/analysis/analysis.py#L64>

The tables have no constraints on the database side and basic validation
is performed by the script:
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/analysis/analysis/fingerprints_updater.py#L91>

The updater overwrites the tables ending with `_tmp` and then swaps them
with the \"real\" tables atomically. In case of failure the final swap
does not happen, leaving the \"real\" tables unaltered.

It is deployed using the [analysis package](#analysis-package)&thinsp;📦 and started
by the [ooni-update-citizenlab](#ooni-update-citizenlab)&thinsp;⏲ Systemd timer.

Logs are generated as the `analysis.fingerprints_updater` unit.

Also it generates the following metrics with the `fingerprints_updater`
prefix:

| Metric name | Type | Description |
| -------- | ------- | ------ |
| `fetch_csv` | timer | CSV fetch duration |
| `fingerprints_update_progress` | gauge | Update progress |
| `fingerprints_dns_tmp_len` | gauge | DNS table size |
| `fingerprints_http_tmp_len` | gauge | HTTP table size |

See the [Fingerprint updater dashboard](#fingerprint-updater-dashboard)&thinsp;📊 on Grafana.

The updater lives primarily in
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/analysis/analysis/fingerprints_updater.py>
and it's called by the `analysis.py` script

To run the updater manually during development:

    PYTHONPATH=analysis ./run_analysis --update-citizenlab --dry-run --stdout


### ASN metadata updater
This component fetches ASN metadata from
<https://archive.org/download/ip2country-as> (generated via:
<https://github.com/ooni/historical-geoip>)

It populates the [asnmeta table](#asnmeta-table)&thinsp;⛁ and
[asnmeta_tmp table](#asnmeta_tmp-table)&thinsp;⛁.

[asnmeta table](#tbl:asnmeta) is used by the private [API](#api)&thinsp;⚙,
see:
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/private.py#L923>
and <https://api.ooni.io/apidocs/#/default/get_api___asnmeta>

Database writes are performed as the `api` user, configured in
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/analysis/analysis/analysis.py#L64>

The table has no constraints on the database side and basic validation
is performed by the script:
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/analysis/analysis/asnmeta_updater.py#L95>

Logs are generated as the `analysis.asnmeta_updater` unit.

Also it generates the following metrics with the `asnmeta_updater`
prefix:

| Metric name | Type | Description |
| -------- | ------- | ------ |
| `fetch_data` | timer | Data fetch duration |
| `asnmeta_update_progress` | gauge | Update progress |
| `asnmeta_tmp_len` | gauge | table size |

See the [ASN metadata updater dashboard](#asn-metadata-updater-dashboard)&thinsp;📊 on
Grafana.

To run the updater manually during development:

    PYTHONPATH=analysis ./run_analysis --update-asnmeta --stdout


### GeoIP downloader
Fetches GeoIP databases, installed by the [ooni-api](#api)&thinsp;⚙. Started
by the [ooni-download-geoip timer](#ooni-download-geoip-timer)&thinsp;⏲ on
[backend-fsn.ooni.org](#backend-fsn.ooni.org)&thinsp;🖥, see.

Lives at
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/>
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/debian/ooni_download_geoip.py>

Updates `asn.mmdb` and `cc.mmdb` in `/var/lib/ooniapi/`

Can be monitored with the [GeoIP MMDB database dashboard](#geoip-mmdb-database-dashboard)&thinsp;📊
and by running:

    sudo journalctl --identifier ooni_download_geoip


### Database backup tool
The backup tool is a service that regularly backs up
[ClickHouse](#clickhouse)&thinsp;⚙ tables to S3. It also exports tables in
`CSV.zstd` format for public consumption.

Contrarily to similar tools, it is designed to:

 * extract data in chunks and upload it without creating temporary
    files

 * without requiring transaction support in the database (not available
    in ClickHouse)

 * without requiring transactional filesystems or interrupting the
    database workload

It is configured by [Ansible](#ansible)&thinsp;🔧 using the
`/etc/ooni/db-backup.conf` file. Runs as a SystemD service, see
[ooni-db-backup timer](#ooni-db-backup-timer)&thinsp;⏲

It compresses data using <https://facebook.github.io/zstd/> during the
upload.

The tool chunks tables as needed and add sleeps as needed to prevent a
query backlog impacting the database performance.

Logs are generated as the `ooni-db-backup` unit.

Also it generates the following metrics with the `db-backup` prefix:

| Metric name | Type | Description |
| -------- | ------- | ------ |
| `upload_to_s3` | timer | Data upload duration |
| `run_export` | timer | Data export duration |
| `table_{tblname}_backup_time_ms` | timer | Table backup time |

See the [Database backup dashboard](#database-backup-dashboard)&thinsp;📊 on Grafana and
[Metrics list](#metrics-list)&thinsp;💡 for application metrics.


Monitor with:

    sudo journalctl -f --identifier ooni-db-backup

Future improvements:

 * [private/public backups](https://github.com/ooni/backend/issues/766)

 * [safer table backup workflow](https://github.com/ooni/backend/issues/767)

 * [database schema backup](https://github.com/ooni/backend/issues/765).
   For extracting the schema see [Database schema check](#database-schema-check)&thinsp;💡


### Ancillary modules
`analysis/analysis/analysis.py` is the main analysis script and acts as
a wrapper to other components.

`analysis/analysis/metrics.py` is a tiny wrapper for the Statsd Python
library.


## Social media blocking event detector
Blocking event detector currently under development. Documented in
<https://docs.google.com/document/d/1WQ6_ybnPbO_W6Tq-xKuHQslG1dSPr4jUbZ3jQLaMqdw/edit>

Deployed by the [detector package](#detector-package)&thinsp;📦.

See [Monitor blocking event detections notebook](#monitor-blocking-event-detections-notebook)&thinsp;📔
[Event detector dashboard](#dash:detector) [Detector timer](#detector-timer)&thinsp;⏲


## OONI bridges
OONI bridges are a new design for handling the connectivity between
Probes and the backend components.

The provide a set of benefits compared to the previous architecture:

 * Circumvention: the entry point for the API accepts all FDQN allowing
    mitigation for DPI-based blocking.

 * Circumvention 2: bridges are designed to be deployed on both
    publicly known and \"static\" IP addresses as well as ephemeral,
    less visible addresses and/or lesser known hosting providers.

    -   Bridges are stateless and could be deployed by
        [Test helper rotation](#test-helper-rotation)&thinsp;⚙.

    -   Test helper VMs can run HaProxy and be used as bridges without
        impacting their ability to run test helpers as well.

 * Faster connectivity: probes use the same HTTPS connection to a
    bridge for both traffic to the API and to the test helper.

 * Resiliency: Test helpers are load-balanced using stateful
    connections. Anything affecting the test helpers is not going to
    impact the Probes, including: test helper rotation, CPU overload,
    network weather or datacenter outages.

The current configuration is based on [HaProxy](#haproxy)&thinsp;⚙ being run
as a load balancer in front of [Test helpers](#test-helpers)&thinsp;⚙ and
the previously configured [Nginx](#nginx)&thinsp;⚙ instance.

The configuration is stored in
<https://github.com/ooni/sysadmin/blob/master/ansible/roles/ooni-backend/templates/haproxy.cfg>

The following diagram shows the load balancing topology:

In the diagram caching for the API and proxying for various services is
still done by Nginx for legacy reasons but can be moved to HaProxy to
simplify configuration management and troubleshooting.

Bridges are deployed as:

 * [ams-pg-test.ooni.org](#ams-pg-test.ooni.org)&thinsp;🖥
    <https://ams-pg-test.ooni.org:444/__haproxy_stats>

 * [backend-hel.ooni.org](#backend-hel.ooni.org)&thinsp;🖥
    <https://backend-hel.ooni.org:444/__haproxy_stats>

 * [bridge-greenhost.ooni.org](#bridge-greenhost.ooni.org)&thinsp;🖥
    <https://bridge-greenhost.ooni.org:444/__haproxy_stats>


## Test helper rotation
The test helper rotation script is responsible for spawning and
deploying VMs on Digital Ocean to be used as
[Test helpers](#test-helpers)&thinsp;⚙.

The main goals for this architecture are:

 * Incremental rollout: the deployment of a new version of the test
    helper (`oohelperd`) is performed incrementally over 4 weeks without
    requiring manual intervention. This allow detecting changes in:

    -   Percentages of `anomaly` / `confirmed` and `failure`
        measurements

    -   Test execution time, CPU and memory usage.

 * Resiliency: traffic between probes and test helpers could be blocked
    by 3rd parties. The rotation system mitigates this risk by cycling
    countries, datacenters and IP addresses across a very large pool.
    The tool is designed to allow future extensions to support other
    hosting providers in order to increase the address pool.

 * Trust: test helpers could be treated differently than other hosts by
    censors e.g. allowing test helpers to reach websites otherwise
    blocked. By rotating test helpers this risk is also mitigated.

 * Failure resolution: in case of failure of a test helper due to
    crash, hardware or network issues, DoS etc. the impacted host can be
    replaced within 5 minutes.

 * Future extension: The ability to rotate country / datacenter / IP
    address can also be used for [OONI bridges](#ooni-bridges)&thinsp;⚙ in
    future.

The primary functions are:

 * Select datacenters and spawn VMs. This allows having helpers to live
    in many countries, datacenters and subnets making attempts at
    blocking them more difficult.

 * Runs a setup script on the host at first boot

 * Keeps a list of live and old hosts in a dedicated database table

 * Create SSL certificates using the Digital Ocean API

 * Performs end-to-end test on newly created VMs to ensure the test
    helper service is running

 * Update DNS to publish new services

 * Drain and destroy old VMs

A summary of the sequence to deploy, drain and destroy a test helper VM:

```mermaid

sequenceDiagram
    Rotation->>Digital Ocean: Spawn VM
    Digital Ocean->>+TH: Spawn
    Rotation-->>+TH: Poll for readiness
    TH-->>-Rotation: Ready
    Rotation->>+Digital Ocean: Create TLS cert
    Digital Ocean->>-Rotation: TLS cert
    Rotation->>TH: Install TLS cert
    Rotation->>TH: Start service
    Rotation->>TH: Run TH test
    Rotation->>Digital Ocean: Update DNS: publish
    Rotation->>Digital Ocean: Update DNS: drain
    Rotation->>Digital Ocean: Terminate VM
    Digital Ocean->>TH: Terminate
    TH->>-Digital Ocean: Terminates
```


DNS propagation time affects the probe's ability to move to a newly
deployed test helper quickly. This is why the rotation script includes a
generous draining period and destroys old VMs after a week.

> **note**
> When triggering rotations manually monitor the change in traffic and
> give probes enough time to catch up.

It is designed to be extended:

 * Support multiple cloud services. The database tables already contain
    columns to track VMs on different cloud providers.

 * Support deploying [OONI bridges](#ooni-bridges)&thinsp;⚙. This can
    provide frequently changing \"entry point\" IP addresses for probes.

The script is deployed on [backend-fsn.ooni.org](#backend-fsn.ooni.org)&thinsp;🖥 using the
[analysis deb package](#analysis-package)&thinsp;📦

The configuration is deployed using [Ansible](#ansible)&thinsp;🔧:

 * `/etc/ooni/rotation.conf`: main configuration file deployed using
    <https://github.com/ooni/sysadmin/blob/master/ansible/roles/ooni-backend/tasks/main.yml>

 * `/etc/ooni/rotation_setup.sh`: this script is executed on the VMs
    for configuration, see
    <https://github.com/ooni/sysadmin/blob/master/ansible/roles/ooni-backend/templates/rotation_nginx_conf>

 * `/etc/ooni/rotation_nginx_conf`: configuration for Nginx to be
    deployed on the VMs, see
    <https://github.com/ooni/sysadmin/blob/master/ansible/roles/ooni-backend/templates/rotation_setup.sh>

Test helpers are named `<number>.th.ooni.org`. This is required to
generate `*.th.ooni.org` certificates.

To troubleshoot rotation see
the [test helper rotation runbook](#test-helper-rotation-runbook)&thinsp;📒


### Internals
The tool reads /etc/ooni/rotation.conf

It uses the following APIs:

 * Digital Ocean API: DNS A/AAAA records

 * Digital Ocean API: Live droplets

Other stateful data exists only in:

 * test_helper_instances database table

 * \"Let's Encrypt\" SSL certificates for `*.th.ooni.org` temporarily
    stored on local host and pushed to the test helpers

For the database table setup see
[test_helper_instances table](#test_helper_instances-table)&thinsp;⛁.

Example of /etc/ooni/rotation.conf

    [DEFAULT]
    token = CHANGEME
    active_droplets_count = 4
    size_slug = s-1vcpu-1gb
    image_name = debian-10-x64
    draining_time_minutes = 240
    dns_zone = th.ooni.org

Simple example for /etc/ooni/rotation_setup.sh:

    #!/bin/bash
    # Configure test-helper droplet
    # This script is run as root and with CWD=/
    set -euo pipefail
    exec 1>setup.log 2>&1
    echo "deb [trusted=yes] https://ooni-internal-deb.s3.eu-central-1.amazonaws.com unstable main" > /etc/apt/sources.list.d/ooni.list
    apt-key adv --verbose --keyserver hkp://keyserver.ubuntu.com --recv-keys 'B5A08F01796E7F521861B449372D1FF271F2DD50'
    apt-get update
    apt-get upgrade -qy
    echo > /etc/motd
    apt-get install -qy oohelperd
    apt-get install -qy oohelperd nginx-light

It is activated by the [ooni-rotation timer](#ooni-rotation-timer)&thinsp;⏲ Systemd
timer Generates metrix prefixed as `rotation` and logs as a journald
unit named `rotation`.

Related files in the backend repository:

    analysis/rotation.py
    analysis/tests/test_rotation.py

> **warning**
> The script has no access to security-critical credentials for services
> like Namecheap, however it has the ability to spawn VMs and control
> Digital Ocean's DNS.


# Infrastructure
This part describes tools used to manage the infrastructure.


## Hosts
This section provides a summary of the backend hosts described in the
rest of the document.

A full list is available at
<https://github.com/ooni/sysadmin/blob/master/ansible/inventory.yml> -
also see [Ansible](#ansible)&thinsp;🔧


### backend-fsn.ooni.org
Public-facing production backend host, receiving the deployment of the
packages:

 * [ooni-api](#ooni-api-package)&thinsp;📦

 * [fastpath](#fastpath-package)&thinsp;📦

 * [analysis](#analysis-package)&thinsp;📦

 * [detector](#detector-package)&thinsp;📦


### backend-hel.ooni.org
Standby / pre-production backend host. Runs the same software stack as
[backend-fsn.ooni.org](#backend-fsn.ooni.org)&thinsp;🖥, plus the
[OONI bridges](#ooni-bridges)&thinsp;⚙


### bridge-greenhost.ooni.org
Runs a [OONI bridges](#ooni-bridges)&thinsp;⚙ in front of the production API
and production [Test helpers](#test-helpers)&thinsp;⚙.


### ams-pg-test.ooni.org
Testbed backend host. Runs the same software stack as
[backend-fsn.ooni.org](#backend-fsn.ooni.org)&thinsp;🖥. Database tables are not backed up and
incoming measurements are not uploaded to S3. All data is considered
ephemeral.


### monitoring.ooni.org
Runs the internal monitoring stack, including
[Jupyter Notebook](#tool:jupyter), [Prometheus](#prometheus)&thinsp;🔧,
[Vector](#vector)&thinsp;🔧 and
[ClickHouse instance for logs](#clickhouse-instance-for-logs)&thinsp;⚙


## The Sysadmin repository
This is a git repository living at <https://github.com/ooni/sysadmin/>
for internal use. It primarily contains:

 * Playbooks for [Ansible](#ansible)&thinsp;🔧

 * The [debops-ci tool](#debops-ci-tool)&thinsp;🔧

 * Scripts and tools including diagrams for
    [DNS and Domains](#dns-and-domains)&thinsp;💡


## Ansible
Ansible is used to configure the OSes on the backend hosts and manage
the configuration of backend components. The playbooks are kept at
<https://github.com/ooni/sysadmin/tree/master/ansible>

This manual supersedes
<https://github.com/ooni/sysadmin/blob/master/README.md>


### Installation and setup
Install Ansible using a OS packages or a Python virtualenv. Ensure the
same major+minor version is used across the team.

Secrets are stored in vaults using the `ansible/vault` script as a
wrapper for `ansible-vault`. Store encrypted variables with a `vault_`
prefix to allow using grep: <http://docs.ansible.com/ansible/playbooks_best_practices.html#best-practices-for-variables-and-vaults>
and link location of the variable using same name without prefix in
corresponding `vars.yml`.

In order to access secrets stored inside of the vault, you will need a
copy of the vault password encrypted with your PGP key. This file should
be stored inside of `~/.ssh/ooni-sysadmin.vaultpw.gpg`.

The file should be provided by other teammates and GPG-encrypted for your own GPG key.


### SSH Configuration
You should configure your `~/.ssh/config` with the following:

    IdentitiesOnly yes
    ServerAliveInterval 120
    UserKnownHostsFile ~/.ssh/known_hosts ~/REPLACE_ME/sysadmin/ext/known_hosts

    host *.ooni.io
      user YOUR_USERNAME

    host *.ooni.nu
      user YOUR_USERNAME

    host *.ooni.org
      user YOUR_USERNAME

Replace `~/REPLACE_ME/sysadmin/ext/known_hosts` to where you have cloned
the `ooni/sysadmin` repo. This will ensure you use the host key
fingeprints from this repo instead of just relying on TOFU.

You should replace `YOUR_USERNAME` with your username from `adm_login`.

On MacOS you may want to also add:

    host *
        UseKeychain yes

To use the Keychain to store passwords.


## Ansible playbooks summary
Usage:

    ./play deploy-<component>.yml -l <hostname> --diff -C
    ./play deploy-<component>.yml -l <hostname> --diff

> **warning**
> any minor error in configuration files or ansible's playbooks can be
> destructive for the backend infrastructure. Always test-run playbooks
> with `--diff` and `-C` at first and carefully verify configuration
> changes. After verification run the playbook without `-C` and verify
> again the applied changes.

> **note**
> [Etckeeper](#etckeeper)&thinsp;🔧 can be useful to verify configuration
> changes from a different point of view.

Some notable parts of the repository:

A list of the backend hosts lives at
<https://github.com/ooni/sysadmin/blob/master/ansible/inventory.yml>

The backend deployment playbook lives at
<https://github.com/ooni/sysadmin/blob/master/ansible/deploy-backend.yml>

Many playbooks depend on roles that configure the OS, named
`base-<os_version>`, for example:
<https://github.com/ooni/sysadmin/blob/master/ansible/roles/base-bookworm>
for Debian Bookworm and
<https://github.com/ooni/sysadmin/tree/master/ansible/roles/base-bullseye>
for Debian Bullseye

The nftables firewall is configured to read every `.nft` file under
`/etc/ooni/nftables/` and `/etc/ooni/nftables/`. This allows roles to
create small files to open a port each and keep the configuration as
close as possible to the ansible step that deploys a service. For
example:
<https://github.com/ooni/sysadmin/blob/master/ansible/roles/base-bookworm/tasks/main.yml#L110>

> **note**
> Ansible announces its runs on [ooni-bots](##ooni-bots)&thinsp;💡 unless running with `-C`.

### The root account

Runbooks use ssh to log on the hosts using your own account and leveraging `sudo` to act as root.

The only exception is when a new host is being deployed - in that case ansible will log in as root to create
individual accounts and lock out the root user.

When running the entire runbook ansible might try to run it as root.
This can be avoided by selecting only the required tags using `-t <tagname>`.

Ideally the root user should be disabled after succesfully creating user accounts.


### Roles layout
Ansible playbooks use multiple roles (see
[example](https://github.com/ooni/sysadmin/blob/master/ansible/deploy-backend.yml#L46))
to deploy various components.

Few roles use the `meta/main.yml` file to depend on other roles. See
[example](https://github.com/ooni/sysadmin/blob/master/ansible/roles/ooni-backend/meta/main.yml)

> **note**
> The latter method should be used sparingly because ansible does not
> indicate where each task in a playbook is coming from.

A diagram of the role dependencies for the deploy-backend.yml playbook:

```mermaid

flowchart LR
        A(deploy-backend.yml) --> B(base-bullseye)
        B -- meta --> G(adm)
        A --> F(nftables)
        A --> C(nginx-buster)
        A --> D(dehydrated)
        D -- meta --> C
        E -- meta --> F
        A --> E(ooni-backend)
        style B fill:#eeffee
        style C fill:#eeffee
        style D fill:#eeffee
        style E fill:#eeffee
        style F fill:#eeffee
        style G fill:#eeffee
```


A similar diagram for deploy-monitoring.yml:

```mermaid

flowchart LR
        B -- meta --> G(adm)
        M(deploy-monitoring.yml) --> B(base-bookworm)
        M --> O(ooca-cert)
        M --> F(nftables)
        M --> D(dehydrated) -- meta --> N(nginx-buster)
        M --> P(prometheus)
        M --> X(blackbox-exporter)
        M --> T(alertmanager)
        style B fill:#eeffee
        style D fill:#eeffee
        style F fill:#eeffee
        style G fill:#eeffee
        style N fill:#eeffee
        style O fill:#eeffee
        style P fill:#eeffee
        style T fill:#eeffee
        style X fill:#eeffee
```

> **note**
> When deploying files or updating files already existing on the hosts it can be useful to add a note e.g. "Deployed by ansible, see <role_name>".
> This helps track down how files on the host were modified and why.

## Creating new playbooks runbook
This runbook describe how to add new runbooks or modify existing runbooks to support new hosts.

When adding a new host to an existing group, if no customization is required it is enough to modify `inventory.yml`
and insert the hostname in the same locations as its peers.

If the host requires small customization e.g. a different configuration file for the <<comp:api>>:

1. add the hostname to `inventory.yml` as described above
2. create "custom" blocks in `tasks/main.yml` to adapt the deployment steps to the new host using the `when:` syntax.

For an example see: <https://github.com/ooni/sysadmin/blob/adb22576791baae046827c79e99b71fc825caae0/ansible/roles/ooni-backend/tasks/main.yml#L65>

NOTE: Complex `when:` rules can lower the readability of `main.yml`

When adding a new type of backend component that is different from anything already existing a new dedicated role can be created:

1. add the hostname to `inventory.yml` as described above
2. create a new playbook e.g. `ansible/deploy-newcomponent.yml`
3. copy files from an existing role into a new `ansible/roles/newcomponent` directory:
  * `ansible/roles/newcomponent/meta/main.yml`
  * `ansible/roles/newcomponent/tasks/main.yml`
  * `ansible/roles/newcomponent/templates/example_config_file`
4. run `./play deploy-newcomponent.yml -l newhost.ooni.org --diff -C` and review the output
5. run `./play deploy-newcomponent.yml -l newhost.ooni.org --diff` and review the output

Example: <https://github.com/ooni/sysadmin/commit/50271b9f5a8fd96dad5531c01fcfdd08bac98fe9>

TIP: To ensure playbooks are robust and idemponent it can be beneficial to develop and test tasks incrementally by running the deployment commands often.


### Monitoring deployment runbook
The monitoring stack is deployed and configured by
[Ansible](#tool:ansible) on the [monitoring.ooni.org](#monitoring.ooni.org)&thinsp;🖥
host using the following playbook:
<https://github.com/ooni/sysadmin/blob/master/ansible/deploy-monitoring.yml>

It includes:

 * [Grafana](#grafana)&thinsp;🔧 at <https://grafana.ooni.org>

 * [Jupyter Notebook](#jupyter-notebook)&thinsp;🔧 at <https://jupyter.ooni.org>

 * [Vector](#tool:vector) (see [Log management](#log-management)&thinsp;💡)

 * local [Netdata](#tool:netdata), [Blackbox exporter](#blackbox-exporter)&thinsp;🔧, etc

 * [Prometheus](#prometheus)&thinsp;🔧 at <https://prometheus.ooni.org>

It also configures the FQDNs:

 * loghost.ooni.org

 * monitoring.ooni.org

 * netdata.ooni.org

This also includes the credentials to access the Web UIs. They are
deployed as `/etc/nginx/monitoring.htpasswd` from
`ansible/roles/monitoring/files/htpasswd`

Steps:

1.  Review [Ansible playbooks summary](#ansible-playbooks-summary)&thinsp;📒,
    [Deploying a new host](#run:newhost) [Grafana dashboards](#grafana-dashboards)&thinsp;💡.

2.  Run `./play deploy-monitoring.yml -l monitoring.ooni.org --diff -C`
    and review the output

3.  Run `./play deploy-monitoring.yml -l monitoring.ooni.org --diff` and
    review the output


### Updating Blackbox Exporter runbook
This runbook describes updating [Blackbox exporter](#blackbox-exporter)&thinsp;🔧.

The `blackbox_exporter` role in ansible is pulled in by the `deploy-monitoring.yml`
runbook.

The configuration file is at `roles/blackbox_exporter/templates/blackbox.yml.j2`
together with `host_vars/monitoring.ooni.org/vars.yml`.

To add a simple HTTP[S] check, for example, you can copy the "ooni website" block.

Edit it and run the deployment of the monitoring stack as described in the previous subchapter.


## Etckeeper
Etckeeper <https://etckeeper.branchable.com/> is deployed on backend
hosts and keeps the `/etc` directory under git version control. It
commits automatically on package deployment and on timed runs. It also
allows doing commits manually.

To check for history of the /etc directory:

```bash
sudo -i
cd /etc
git log --raw
```

And `git diff` for unmerged changes.

Use `etckeeper commit <message>` to commit changes.

:::tip
Etckeeper commits changes automatically when APT is used or on daily basis, whichever comes first.
:::


## Team credential repository
A private repository <https://github.com/ooni/private> contains team
credentials, including username/password tuples, GPG keys and more.

> **warning**
> The credential file is GPG-encrypted as `credentials.json.gpg`. Do not
> commit the cleartext `credentials.json` file.

> **note**
> The credentials are stored in a JSON file to allow a flexible,
> hierarchical layout. This allow storing metadata like descriptions on
> account usage, dates of account creations, expiry, and credential
> rotation time.

The tool checks JSON syntax and sorts keys automatically.


### Listing file contents
    git pull
    make show


### Editing contents
    git pull
    make edit
    git commit credentials.json.gpg -m "<message>"
    git push


### Extracting a credential programmatically:
    git pull
    ./extract 'grafana.username'

> **note**
> this can be used to automate credential retrieval from other tools, e.g.
> [Ansible](#ansible)&thinsp;🔧


### Updating users allowed to decrypt the credentials file
Edit `makefile` to add or remove recipients (see `--recipient`)

Then run:

    git pull
    make decrypt encrypt
    git commit makefile credentials.json.gpg
    git push


## Deploying a new host
To deploy a new host:

1.  Choose a FQDN like \$name.ooni.org based on the
    [DNS naming policy](#dns-naming-policy)&thinsp;💡

2.  Deploy the physical host or VM using Debian Stable

3.  Create `A` and `AAAA` records for the FQDN in the Namecheap web UI

4.  Follow [Updating DNS diagrams](#updating-dns-diagrams)&thinsp;📒

5.  Review the `inventory.yml` file and git-commit it

6.  Deploy the required stack. Run ansible it test mode first. For
    example this would deploy a backend host:

        ./play deploy-backend.yml --diff -l <name>.ooni.org -C
        ./play deploy-backend.yml --diff -l <name>.ooni.org

7.  Update [Prometheus](#prometheus)&thinsp;🔧 by following
    [Monitoring deployment runbook](#monitoring-deployment-runbook)&thinsp;📒

8.  git-push the commits

Also see [Monitoring deployment runbook](#monitoring-deployment-runbook)&thinsp;📒 for an
example of deployment.


# DNS and Domains
The primary domains used by the backend are:

 * ooni.org

 * ooni.io

 * ooni.nu


## DNS naming policy
Public-facing HTTPS services are named `<service>.ooni.org` or
`$<service>.ooni.io` (legacy). Public-facing means the FQDNs are used
directly by external users, services, or embedded in the probes. They
cannot be changed or retired without causing outages.

Give hosts private names and use only those in `inventory_hostname` to
ease migrations.

VMs should have FQDN like `<name>-<location>[-<number>].ooni.org`. VMs
can provide one or more public-facing services that can change over
time. The name should be as descriptive as possible e.g. the type of
services or the most important service being run.

[Test helpers](#test-helpers)&thinsp;⚙ have a special naming policy and are
deployed by [Test helper rotation](#test-helper-rotation)&thinsp;⚙

Various legacy names should be cleaned up during re-deploying VMs with
newer base OS version.


## DNS diagrams

### A:
See
<https://raw.githubusercontent.com/ooni/sysadmin/master/ext/dnsgraph.A.svg>

The image is not included here due to space constraints.


### CNAME:
![CNAME](https://raw.githubusercontent.com/ooni/sysadmin/master/ext/dnsgraph.CNAME.svg)


### MX:
![MX](https://raw.githubusercontent.com/ooni/sysadmin/master/ext/dnsgraph.MX.svg)


### NS:
![NS](https://raw.githubusercontent.com/ooni/sysadmin/master/ext/dnsgraph.NS.svg)


### TXT:
![TXT](https://raw.githubusercontent.com/ooni/sysadmin/master/ext/dnsgraph.TXT.svg)


### HTTP Moved Permanently (HTTP code 301):
![URL301](https://raw.githubusercontent.com/ooni/sysadmin/master/ext/dnsgraph.URL301.svg)


### HTTP Redirects:
![URL](https://raw.githubusercontent.com/ooni/sysadmin/master/ext/dnsgraph.URL.svg)


### Updating DNS diagrams
To update the diagrams use the sysadmin repository:

Update the `./ext/dns.json` file:

    cd ansible
    ./play ext-inventory.yml -t namecheap
    cd ..

Then run <https://github.com/ooni/sysadmin/blob/master/scripts/dnsgraph>
to generate the charts:

    ./scripts/dnsgraph

It will generate SVG files under the `./ext/` directory. Finally, commit
and push the dns.json and SVG files.


# Operations
This section contains howtos and runbooks on how to manage and update
the backend.


## Build, deploy, rollback
Host deployments are done with the
[sysadmin repo](https://github.com/ooni/sysadmin)

For component updates a deployment pipeline is used:

Look at the \[Status
dashboard\](<https://github.com/ooni/backend/wiki/Backend>) - be aware
of badge image caching


## The deployer tool
Deployments can be performed with a tool that acts as a frontend for
APT. It implements a simple Continuous Delivery workflow from CLI. It
does not require running a centralized CD pipeline server (e.g. like
<https://www.gocd.org/>)

The tool is hosted on the backend repository together with its
configuration file for simplicity:
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/deployer>

At start time it traverses the path from the current working directory
back to root until it finds a configuration file named deployer.ini This
allows using different deployment pipelines stored in configuration
files across different repositories and subdirectories.

The tool connects to the hosts to perform deployments and requires sudo
rights. It installs Debian packages from repositories already configured
on the hosts.

It runs `apt-get update` and then `apt-get install …​` to update or
rollback packages. By design, it does not interfere with manual
execution of apt-get or through tools like [Ansible](#ansible)&thinsp;🔧.
This means operators can log on a host to do manual upgrade or rollback
of packages without breaking the deployer tool.

The tool depends only on the `python3-apt` package.

Here is a configuration file example, with comments:

``` ini
[environment]
# Location on the path where SVG badges are stored
badges_path = /var/www/package_badges


# List of packages that are handled by the deployer, space separated
deb_packages = ooni-api fastpath analysis detector


# List of deployment stage names, space separated, from the least to the most critical
stages = test hel prod


# For each stage a block named stage:<stage_name> is required.
# The block lists the stage hosts.


# Example of an unused stage (not list under stages)
[stage:alpha]
hosts = localhost

[stage:test]
hosts = ams-pg-test.ooni.org

[stage:hel]
hosts = backend-hel.ooni.org

[stage:prod]
hosts = backend-fsn.ooni.org
```

By running the tool without any argument it will connect to the hosts
from the configuration file and print a summary of the installed
packages, for example:

``` bash
$ deployer

     Package               test                   prod
ooni-api               1.0.79~pr751-194       1.0.79~pr751-194
fastpath               0.81~pr748-191     ►►  0.77~pr705-119
analysis               1.9~pr659-61       ⚠   1.10~pr692-102
detector               0.3~pr651-98           0.3~pr651-98
```

The green arrows between two package versions indicates that the version
on the left side is higher than the one on the right side. This means
that a rollout is pending. In the example the fastpath package on the
\"prod\" stage can be updated.

A red warning sign indicates that the version on the right side is
higher than the one on the left side. During a typical continuous
deployment workflow version numbers should always increment The rollout
should go from left to right, aka from the least critical stage to the
most critical stage.

Deploy/rollback a given version on the \"test\" stage:

``` bash
./deployer deploy ooni-api test 0.6~pr194-147
```

Deploy latest build on the first stage:

``` bash
./deployer deploy ooni-api
```

Deploy latest build on a given stage. This usage is not recommended as
it deploys the latest build regardless of what is currently running on
previous stages.

``` bash
./deployer deploy ooni-api prod
```

The deployer tool can also generate SVG badges that can then served by
[Nginx](#nginx)&thinsp;⚙ or copied elsewhere to create a status dashboard.

Example:

![badge](images/badge.png)

Update all badges with:

``` bash
./deployer refresh_badges
```


## Adding new tests
This runbook describes how to add support for a new test in the
[Fastpath](#fastpath)&thinsp;⚙.

Review [Backend code changes](#backend-code-changes)&thinsp;📒, then update
[fastpath core](https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/fastpath/fastpath/core.py)
to add a scoring function.

See for example `def score_torsf(msm: dict) → dict:`

Also add an `if` block to the `def score_measurement(msm: dict) → dict:`
function to call the newly created function.

Finish by adding a new test to the `score_measurement` function and
adding relevant integration tests.

Run the integration tests locally.

Update the
[api](https://github.rom/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/measurements.py#L491)
if needed.

Deploy on [ams-pg-test.ooni.org](#ams-pg-test.ooni.org)&thinsp;🖥 and run end-to-end tests
using real probes.


## Adding support for a new test key
This runbook describes how to modify the [Fastpath](#fastpath)&thinsp;⚙
and the [API](#api)&thinsp;⚙ to extract, process, store and publish a new measurement
field.

Start with adding a new column to the [fastpath table](#fastpath-table)&thinsp;⛁
by following [Adding a new column to the fastpath](#adding-a-new-column-to-the-fastpath)&thinsp;📒.

Add the column to the local ClickHouse instance used for tests and
[ams-pg-test.ooni.org](#ams-pg-test.ooni.org)&thinsp;🖥.

Update <https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/tests/integ/clickhouse_1_schema.sql> as described in
[Continuous Deployment: Database schema changes](#continuous-deployment:-database-schema-changes)&thinsp;💡

Add support for the new field in the fastpath `core.py` and `db.py` modules
and related tests.
See https://github.com/ooni/backend/pull/682 for a comprehensive example.

Run tests locally, then open a draft pull request and ensure the CI tests are
running successfully.

If needed, the current pull request can be reviewed and deployed without modifying the API to expose the new column. This allows processing data sooner while the API is still being worked on.

Add support for the new column in the API. The change depends on where and how the
new value is to be published.
See <https://github.com/ooni/backend/commit/ae2097498ec4d6a271d8cdca9d68bd277a7ac19d#diff-4a1608b389874f2c35c64297e9c676dffafd49b9ba80e495a703ba51d2ebd2bbL359> for a generic example of updating an SQL query in the API and updating related tests.

Deploy the changes on test and pre-production stages after creating the new column in the database.
See [The deployer tool](#the-deployer-tool)&thinsp;🔧 for details.

Perform end-to-end tests with real probes and [Public and private web UIs](#public-and-private-web-uis)&thinsp;💡 as needed.

Complete the pull request and deploy to production.


## Adding new fingerprints
This is performed on <https://github.com/ooni/blocking-fingerprints>

Updates are fetched automatically by
[Fingerprint updater](#fingerprint-updater)&thinsp;⚙

Also see [Fingerprint updater dashboard](#fingerprint-updater-dashboard)&thinsp;📊.


## Backend code changes
This runbook describes making changes to backend components and
deploying them.

Summary of the steps:

1.  Check out the backend repository.

2.  Create a dedicated branch.

3.  Update `debian/changelog` in the component you want to monify. See
    [Package versioning](#package-versioning)&thinsp;💡 for details.

4.  Run unit/functional/integ tests as needed.

5.  Create a pull request.

6.  Ensure the CI workflows are successful.

7.  Deploy the package on the testbed [ams-pg-test.ooni.org](#ams-pg-test.ooni.org)&thinsp;🖥
    and verify the change works as intended.

8.  Add a comment the PR with the deployed version and stage.

9.  Wait for the PR to be approved.

10. Deploy the package to production on
    [backend-fsn.ooni.org](#backend-fsn.ooni.org)&thinsp;🖥. Ensure it is the same version
    that has been used on the testbed. See [API runbook](#api-runbook)&thinsp;📒 for
    deployment steps.

11. Add a comment the PR with the deployed version and stage, then merge
    the PR.

When introducing new metrics:

1.  Create [Grafana](#grafana)&thinsp;🔧 dashboards, alerts and
    [Jupyter Notebook](#jupyter-notebook)&thinsp;🔧 and link them in the PR.

2.  Collect and analize metrics and logs from the testbed stages before
    deploying to production.

3.  Test alarming by simulating incidents.


## Backend component deployment
This runbook provides general steps to deploy backend components on
production hosts.

Review the package changelog and the related pull request.

The amount of testing and monitoring required depends on:

1.  the impact of possible bugs in terms of number of users affected and
    consequences

2.  the level of risk involved in rolling back the change, if needed

3.  the complexity of the change and the risk of unforeseen impact

Monitor the [API and fastpath](#api-and-fastpath)&thinsp;📊 and dedicated . Review past
weeks for any anomaly before starting a deployment.

Ensure that either the database schema is consistent with the new
deployment by creating tables and columns manually, or that the new
codebase is automatically updating the database.

Quickly check past logs.

Follow logs with:

``` bash
sudo journalctl -f --no-hostname
```

While monitoring the logs, deploy the package using the
[The deployer tool](#the-deployer-tool)&thinsp;🔧 tool. (Details on the tool subchapter)


## API runbook
This runbook describes making changes to the [API](#api)&thinsp;⚙ and
deploying it.

Follow [Backend code changes](#backend-code-changes)&thinsp;📒 and
[Backend component deployment](#backend-component-deployment)&thinsp;📒.

In addition, monitor logs from Nginx and API focusing on HTTP errors and
failing SQL queries.

Manually check [Explorer](#explorer)&thinsp;🖱 and other
[Public and private web UIs](#public-and-private-web-uis)&thinsp;💡 as needed.


### Managing feature flags
To change feature flags in the API a simple pull request like
<https://github.com/ooni/backend/pull/776> is enough.

Follow [Backend code changes](#backend-code-changes)&thinsp;📒 and deploy it after
basic testing on [ams-pg-test.ooni.org](#ams-pg-test.ooni.org)&thinsp;🖥.


## Running database queries
This subsection describes how to run queries against
[ClickHouse](#clickhouse)&thinsp;⚙. You can run queries from
[Jupyter Notebook](#jupyter-notebook)&thinsp;🔧 or from the CLI:

```bash
    ssh <backend_host>
    $ clickhouse-client
```

Prefer using the default user when possible. To log in as admin:

```bash
    $ clickhouse-client -u admin --password <redacted>
```

> **note**
> Heavy queries can impact the production database. When in doubt run them
> on the CLI interface in order to terminate them using CTRL-C if needed.

> **warning**
> ClickHouse is not transactional! Always test queries that mutate schemas
> or data on testbeds like [ams-pg-test.ooni.org](#ams-pg-test.ooni.org)&thinsp;🖥

For long running queries see the use of timeouts in
[Fastpath deduplication](#fastpath-deduplication)&thinsp;📒

Also see [Dropping tables](#dropping-tables)&thinsp;📒,
[Investigating table sizes](#investigating-table-sizes)&thinsp;📒


### Modifying the fastpath table
This runbook show an example of changing the contents of the
[fastpath table](#fastpath-table)&thinsp;⛁ by running a \"mutation\" query.

> **warning**
> This method creates changes that cannot be reproduced by external
> researchers by [Reprocessing measurements](#reprocessing-measurements)&thinsp;📒. See
> [Reproducibility](#reproducibility)&thinsp;💡

In this example [Signal test](#signal-test)&thinsp;Ⓣ measurements are being
flagged as failed due to <https://github.com/ooni/probe/issues/2627>

Summarize affected measurements with:

``` sql
SELECT test_version, msm_failure, count()
FROM fastpath
WHERE test_name = 'signal' AND measurement_start_time > '2023-11-06T16:00:00'
GROUP BY msm_failure, test_version
ORDER BY test_version ASC
```

> **important**
> `ALTER TABLE …​ UPDATE` starts a
> [mutation](https://clickhouse.com/docs/en/sql-reference/statements/alter#mutations)
> that runs in background.

Check for any running or stuck mutation:

``` sql
SELECT * FROM system.mutations WHERE is_done != 1
```

Start the mutation:

``` sql
ALTER TABLE fastpath
UPDATE
  msm_failure = 't',
  anomaly = 'f',
  scores = '{"blocking_general":0.0,"blocking_global":0.0,"blocking_country":0.0,"blocking_isp":0.0,"blocking_local":0.0,"accuracy":0.0,"msg":"bad test_version"}'
WHERE test_name = 'signal'
AND measurement_start_time > '2023-11-06T16:00:00'
AND msm_failure = 'f'
```

Run the previous `SELECT` queries to monitor the mutation and its
outcome.


## Updating tor targets
See [Tor targets](#tor-targets)&thinsp;🐝 for a general description.

Review the [Ansible](#ansible)&thinsp;🔧 chapter. Checkout the repository and
update the file `ansible/roles/ooni-backend/templates/tor_targets.json`

Commit the changes and deploy as usual:

    ./play deploy-backend.yml --diff -l ams-pg-test.ooni.org -t api -C
    ./play deploy-backend.yml --diff -l ams-pg-test.ooni.org -t api

Test the updated configuration, then:

    ./play deploy-backend.yml --diff -l backend-fsn.ooni.org -t api -C
    ./play deploy-backend.yml --diff -l backend-fsn.ooni.org -t api

git-push the changes.

Implements [Document Tor targets](#document-tor-targets)&thinsp;🐞


## Creating admin API accounts
See [Auth](#auth)&thinsp;🐝 for a description of the API entry points related
to account management.

The API provides entry points to:

 * [get role](https://api.ooni.io/apidocs/#/default/get_api_v1_get_account_role__email_address_)

 * [set role](https://api.ooni.io/apidocs/#/default/post_api_v1_set_account_role).

The latter is implemented
[here](https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/auth.py#L437).

> **important**
> The default value for API accounts is `user`. For such accounts there is
> no need for a record in the `accounts` table.

To change roles it is required to be authenticated and have a role as
`admin`.

It is also possible to create or update roles by running SQL queries
directly on [ClickHouse](#clickhouse)&thinsp;⚙. This can be necessary to
create the initial `admin` account on a new deployment stage.

A quick way to identify the account ID an user is to extract logs from
the [API](#api)&thinsp;⚙ either from the backend host or using
[Logs from FSN notebook](#logs-from-fsn-notebook)&thinsp;📔

```bash
sudo journalctl --since '5 min ago' -u ooni-api | grep 'SELECT role FROM accounts WHERE account_id' -C5
```

Example output:

    Nov 09 16:03:00 backend-fsn ooni-api[1763457]: DEBUG Query: SELECT role FROM accounts WHERE account_id = '<redacted>'

Then on the database test host:

```bash
clickhouse-client
```

Then in the ClickHouse shell insert a record to give\`admin\` role to
the user. See [Running database queries](#running-database-queries)&thinsp;📒:

```sql
INSERT INTO accounts (account_id, role) VALUES ('<redacted>', 'admin')
```

`accounts` is an EmbeddedRocksDB table with `account_id` as primary key.
No record deduplication is necessary.

To access the new role the user has to log out from web UIs and login
again.

> **important**
> Account IDs are not the same across test and production instances.

This is due to the use of a configuration variable
`ACCOUNT_ID_HASHING_KEY` in the hashing of the email address. The
parameter is read from the API configuration file. The values are
different across deployment stages as a security feature.


## Fastpath runbook

### Fastpath code changes and deployment
Review [Backend code changes](#backend-code-changes)&thinsp;📒 and
[Backend component deployment](#backend-component-deployment)&thinsp;📒 for changes and deployment of the
backend stack in general.

Also see [Modifying the fastpath table](#modifying-the-fastpath-table)&thinsp;📒

In addition, monitor logs and [Grafana dashboards](#grafana-dashboards)&thinsp;💡
focusing on changes in incoming measurements.

You can use the [The deployer tool](#the-deployer-tool)&thinsp;🔧 tool to perform
deployment and rollbacks of the [Fastpath](#fastpath)&thinsp;⚙.

> **important**
> the fastpath is configured **not** to restart automatically during
> deployment.

Always monitor logs and restart it as needed:

```bash
sudo systemctl restart fastpath
```


### Fastpath manual deployment
Sometimes it can be useful to run APT directly:

```bash
ssh <host>
sudo apt-get update
apt-cache show fastpath | grep Ver | head -n5
sudo apt-get install fastpath=<version>
```


### Reprocessing measurements
Reprocess old measurement by running the fastpath manually. This can be
done without shutting down the fastpath instance running on live
measurements.

You can run the fastpath as root or using the fastpath user. Both users
are able to read the configuration file under `/etc/ooni`. The fastpath
will download [Postcans](#postcans)&thinsp;💡 in the local directory.

`fastpath -h` generates:

    usage:
    OONI Fastpath

    See README.adoc

     [-h] [--start-day START_DAY] [--end-day END_DAY]
                                             [--devel] [--noapi] [--stdout] [--debug]
                                             [--db-uri DB_URI]
                                             [--clickhouse-url CLICKHOUSE_URL] [--update]
                                             [--stop-after STOP_AFTER] [--no-write-to-db]
                                             [--keep-s3-cache] [--ccs CCS]
                                             [--testnames TESTNAMES]

    options:
      -h, --help            show this help message and exit
      --start-day START_DAY
      --end-day END_DAY
      --devel               Devel mode
      --noapi               Process measurements from S3 and do not start API feeder
      --stdout              Log to stdout
      --debug               Log at debug level
      --clickhouse-url CLICKHOUSE_URL
                            ClickHouse url
      --stop-after STOP_AFTER
                            Stop after feeding N measurements from S3
      --no-write-to-db      Do not insert measurement in database
      --ccs CCS             Filter comma-separated CCs when feeding from S3
      --testnames TESTNAMES
                            Filter comma-separated test names when feeding from S3 (without
                            underscores)

To run the fastpath manually use:

    ssh <host>
    sudo sudo -u fastpath /bin/bash

    fastpath --help
    fastpath --start-day 2023-08-14 --end-day 2023-08-19 --noapi --stdout

The `--no-write-to-db` option can be useful for testing.

The `--ccs` and `--testnames` flags are useful to selectively reprocess
measurements.

After reprocessing measurements it's recommended to manually deduplicate
the contents of the `fastpath` table. See
[Fastpath deduplication](#fastpath-deduplication)&thinsp;📒

> **note**
> it is possible to run multiple `fastpath` processes using
> <https://www.gnu.org/software/parallel/> with different time ranges.
> Running the reprocessing under `byobu` is recommended.

The fastpath will pull [Postcans](#postcans)&thinsp;💡 from S3. See
[Feed fastpath from JSONL](#feed-fastpath-from-jsonl)&thinsp;🐞 for possible speedup.


### Fastpath monitoring
The fastpath pipeline can be monitored using the
[Fastpath dashboard](#dash:api_fp) and [API and fastpath](#api-and-fastpath)&thinsp;📊.

Also follow real-time process using:

    sudo journalctl -f -u fastpath


## Android probe release runbook
This runbook is meant to help coordinate Android probe releases between
the probe and backend developers and public announcements. It does not
contain detailed instructions for individual components.

Also see the [Measurement drop runbook](#measurement-drop-tutorial)&thinsp;📒.


Roles: \@probe, \@backend, \@media


### Android pre-release
\@probe: drive the process involving the other teams as needed. Create
calendar events to track the next steps. Run the probe checklist
<https://docs.google.com/document/d/1S6X5DqVd8YzlBLRvMFa4RR6aGQs8HSXfz8oGkKoKwnA/edit>

\@backend: review
<https://jupyter.ooni.org/view/notebooks/jupycron/autorun_android_probe_release.html>
and
<https://grafana.ooni.org/d/l-MQSGonk/api-and-fastpath-multihost?orgId=1&refresh=5s&var-avgspan=8h&var-host=backend-fsn.ooni.org&from=now-30d&to=now>
for long-term trends


### Android release
\@probe: release the probe for early adopters

\@backend: monitor
<https://jupyter.ooni.org/view/notebooks/jupycron/autorun_android_probe_release.html>
frequently during the first 24h and report any drop on
[Slack](#slack)&thinsp;🔧

\@probe: wait at least 24h then release the probe for all users

\@backend: monitor
<https://jupyter.ooni.org/view/notebooks/jupycron/autorun_android_probe_release.html>
daily for 14 days and report any drop on [Slack](#slack)&thinsp;🔧

\@probe: wait at least 24h then poke \@media to announce the release

(<https://github.com/ooni/backend/wiki/Runbooks:-Android-Probe-Release>


## CLI probe release runbook
This runbook is meant to help coordinate CLI probe releases between the
probe and backend developers and public announcements. It does not
contain detailed instructions for individual components.

Roles: \@probe, \@backend, \@media


### CLI pre-release
\@probe: drive the process involving the other teams as needed. Create
calendar events to track the next steps. Run the probe checklist and
review the CI.

\@backend: review
\[jupyter\](<https://jupyter.ooni.org/view/notebooks/jupycron/autorun_cli_probe_release.html>)
and
\[grafana\](<https://grafana.ooni.org/d/l-MQSGonk/api-and-fastpath-multihost?orgId=1&refresh=5s&var-avgspan=8h&var-host=backend-fsn.ooni.org&from=now-30d&to=now>)
for long-term trends


### CLI release
\@probe: release the probe for early adopters

\@backend: monitor
\[jupyter\](<https://jupyter.ooni.org/view/notebooks/jupycron/autorun_cli_probe_release.html>)
frequently during the first 24h and report any drop on
[Slack](#slack)&thinsp;🔧

\@probe: wait at least 24h then release the probe for all users

\@backend: monitor
\[jupyter\](<https://jupyter.ooni.org/view/notebooks/jupycron/autorun_cli_probe_release.html>)
daily for 14 days and report any drop on [Slack](#slack)&thinsp;🔧

\@probe: wait at least 24h then poke \@media to announce the release


## Investigating heavy aggregation queries runbook
In the following scenario the [Aggregation and MAT](#aggregation-and-mat)&thinsp;🐝 API is
experiencing query timeouts impacting users.

Reproduce the issue by setting a large enough time span on the MAT,
e.g.:
<https://explorer.ooni.org/chart/mat?test_name=web_connectivity&axis_x=measurement_start_day&since=2023-10-15&until=2023-11-15&time_grain=day>

Click on the link to JSON, e.g.
<https://api.ooni.io/api/v1/aggregation?test_name=web_connectivity&axis_x=measurement_start_day&since=2023-01-01&until=2023-11-15&time_grain=day>

Review the [backend-fsn.ooni.org](#backend-fsn.ooni.org)&thinsp;🖥 metrics on
<https://grafana.ooni.org/d/M1rOa7CWz/netdata?orgId=1&var-instance=backend-fsn.ooni.org:19999>
(see [Netdata-specific dashboard](#netdata-specific-dashboard)&thinsp;📊 for details)

Also review the [API and fastpath](#api-and-fastpath)&thinsp;📊 dashboard, looking at
CPU load, disk I/O, query time, measurement flow.

Also see [Aggregation cache monitoring](#aggregation-cache-monitoring)&thinsp;🐍

Refresh and review the charts on the [ClickHouse queries notebook](#clickhouse-queries-notebook)&thinsp;📔.

In this instance frequent calls to the aggregation API are found.

Review the summary of the API quotas. See
[Calling the API manually](#calling-the-api-manually)&thinsp;📒 for details:

    $ http https://api.ooni.io/api/_/quotas_summary Authorization:'Bearer <mytoken>'

Log on [backend-fsn.ooni.org](#backend-fsn.ooni.org)&thinsp;🖥 and review the logs:

    backend-fsn:~$ sudo journalctl --since '5 min ago'

Summarize the subnets calling the API:

    backend-fsn:~$ sudo journalctl --since '5 hour ago' -u ooni-api -u nginx | grep aggreg | cut -d' ' -f 8 | sort | uniq -c | sort -nr | head

    807 <redacted subnet>
    112 <redacted subnet>
     92 <redacted subnet>
     38 <redacted subnet>
     16 <redacted subnet>
     15 <redacted subnet>
     11 <redacted subnet>
     11 <redacted subnet>
     10 <redacted subnet>

To block IP addresses or subnets see [Nginx](#nginx)&thinsp;⚙ or
[HaProxy](#haproxy)&thinsp;⚙, then configure the required file in
[Ansible](#ansible)&thinsp;🔧 and deploy.

Also see [Limiting scraping](#limiting-scraping)&thinsp;📒.


## Aggregation cache monitoring
To monitor cache hit/miss ratio using StatsD metrics the following
script can be run as needed.

See [Metrics list](#metrics-list)&thinsp;💡.

``` python
import subprocess

import statsd
metrics = statsd.StatsClient('localhost', 8125)

def main():
    cmd = "sudo journalctl --since '5 min ago' -u nginx | grep 'GET /api/v1/aggregation' | cut -d ' ' -f 10 | sort | uniq -c"
    out = subprocess.check_output(cmd, shell=True)
    for line in out.splitlines():
        cnt, name = line.strip().split()
        name = name.decode()
        metrics.gauge(f"nginx_aggregation_cache_{name}", int(cnt))

if __name__ == '__main__':
    main()
```


## Limiting scraping
Aggressive bots and scrapers can be limited using a combination of
methods. Listed below ordered starting from the most user-friendly:

1.  Reduce the impact on the API (CPU, disk I/O, memory usage) by
    caching the results.

2.  [Rate limiting and quotas](#rate-limiting-and-quotas)&thinsp;🐝 already built in the API. It
    might need lowering of the quotas.

3.  Adding API entry points to [Robots.txt](#robots.txt)&thinsp;🐝

4.  Adding specific `User-Agent` entries to [Robots.txt](#robots.txt)&thinsp;🐝

5.  Blocking IP addresses or subnets in the [Nginx](#nginx)&thinsp;⚙ or
    [HaProxy](#haproxy)&thinsp;⚙ configuration files

To add caching to the API or increase the expiration times:

1.  Identify API calls that cause significant load. [Nginx](#nginx)&thinsp;⚙
    is configured to log timing information for each HTTP request. See
    [Logs investigation notebook](#logs-investigation-notebook)&thinsp;📔 for examples. Also see
    [Logs from FSN notebook](#logs-from-fsn-notebook)&thinsp;📔 and
    [ClickHouse instance for logs](#clickhouse-instance-for-logs)&thinsp;⚙. Additionally,
    [Aggregation cache monitoring](#aggregation-cache-monitoring)&thinsp;🐍 can be tweaked for the present use-case.

2.  Implement caching or increase expiration times across the API
    codebase. See [API cache](#api-cache)&thinsp;💡 and
    [Purging Nginx cache](#purging-nginx-cache)&thinsp;📒.

3.  Monitor the improvement in terms of cache hit VS cache miss ratio.

> **important**
> Caching can be applied selectively for API requests that return rapidly
> changing data VS old, stable data. See [Aggregation and MAT](#aggregation-and-mat)&thinsp;🐝
> for an example.

To update the quotas edit the API here
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/app.py#L187>
and deploy as usual.

To update the `robots.txt` entry point see [Robots.txt](#robots.txt)&thinsp;🐝 and
edit the API here
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/pages/>*init*.py#L124
and deploy as usual

To block IP addresses or subnets see [Nginx](#nginx)&thinsp;⚙ or
[HaProxy](#haproxy)&thinsp;⚙, then configure the required file in
[Ansible](#ansible)&thinsp;🔧 and deploy.


## Calling the API manually
To make HTTP calls to the API manually you'll need to extact a JWT from
the browser, sometimes with admin rights.

In Firefox, authenticate against <https://test-lists.ooni.org/> , then
open Inspect \>\> Storage \>\> Local Storage \>\> Find
`{"token": "<mytoken>"}`

Extract the token ascii-encoded string without braces nor quotes.

Call the API using [httpie](https://httpie.io/) with:

    $ http https://api.ooni.io/<path> Authorization:'Bearer <mytoken>'

E.g.:

    $ http https://api.ooni.io/api/_/quotas_summary Authorization:'Bearer <mytoken>'

> **note**
> Do not leave whitespaces after \"Authorization:\"


## Debian packages
This section lists the Debian packages used to deploy backend
components. They are built by [GitHub CI workflows](#github-ci-workflows)&thinsp;💡
and deployed using [The deployer tool](#the-deployer-tool)&thinsp;🔧. See
[Debian package build and publish](#debian-package-build-and-publish)&thinsp;💡.


### ooni-api package
Debian package for the [API](#api)&thinsp;⚙


### fastpath package
Debian package for the [Fastpath](#fastpath)&thinsp;⚙


### detector package
Debian package for the
[Social media blocking event detector](#social-media-blocking-event-detector)&thinsp;⚙


### analysis package
The `analysis` Debian package contains various tools and runs various of
systemd timers, see [Systemd timers](#systemd-timers)&thinsp;💡.


### Analysis deployment
See [Backend component deployment](#backend-component-deployment)&thinsp;📒


# Measurement uploader
This component uploads fresh measurements from
[backend-fsn.ooni.org](#host:FSN) to [S3 data bucket](#s3-data-bucket)&thinsp;💡
after compressing them into [Postcans](#postcans)&thinsp;💡 and .jsonl
files.

It inserts records in the [jsonl table](#jsonl-table)&thinsp;⛁ using the `api`
database user.

The uploader runs hourly. The measurement batching process is designed
to avoid data loss in case of interruption or crash:

 * Scan for raw measurements from the spool directory, typically
    `/var/lib/ooniapi/measurements/incoming/`

 * Generate one [Postcans](#postcans)&thinsp;💡 and
    [JSONL files](#jsonl-files)&thinsp;💡 in a different directory

 * Delete the raw measurements

 * Upload the postcan and jsonl files to
    [S3 data bucket](#s3-data-bucket)&thinsp;💡

 * Insert new records in [jsonl table](#jsonl-table)&thinsp;⛁ with fields
    `report_id`, `input`, `s3path`, `linenum`, `measurement_uid`

The jsonl table is used by the API to look up measurement bodies. There
is one line per measurement. The `s3path` column identifies the key on
[S3 data bucket](#s3-data-bucket)&thinsp;💡 containing the compressed JSONL file
with the measurement data. The `linenum` column contains the line number
in such file where the measurement is found. See
[Measurements](#measurements)&thinsp;🐝

Reads the `/etc/ooni/api.conf` file. The file itself is deployed by
[Ansible](#ansible)&thinsp;🔧.

Also see the [Measurement uploader dashboard](#measurement-uploader-dashboard)&thinsp;📊,
[uploader timer](#timer:uploader) and [Main data flows](#main-data-flows)&thinsp;💡

[Sources](https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooni_api_uploader.py)


## Postcans
A "postcan" is tarball containing measurements as they are uploaded by
the probes, optionally compressed. Postcans are meant for internal use.


# S3 data bucket
The `ooni-data-eu-fra` Amazon S3 bucket contains the whole OONI dataset.
It is accessible with the S3 protocol and also over HTTPS:
<https://ooni-data-eu-fra.s3.eu-central-1.amazonaws.com/>

It uses a dedicated [Open Data](https://aws.amazon.com/opendata/)
account providing free hosting for public data. Details on the OONI
account used for this are in the
[Team credential repository](#team-credential-repository)&thinsp;💡.

> **note**
> All data on the bucket has to be kept publicly accessible to comply with
> the Open Data requirements. Do not run other AWS services using the Open
> Data account.


# S3 measurement files layout
Probes usually upload multiple measurements on each execution.
Measurements are stored temporarily and then batched together,
compressed and uploaded to the S3 bucket once every hour. To ensure
transparency, incoming measurements go through basic content validation
and the API returns success or error; once a measurement is accepted it
will be published on S3.

Specifications of the raw measurement data can be found inside of the
`ooni/spec` repository.


## JSONL files
File paths in the S3 bucket in JSONL format.

Contains a JSON document for each measurement, separated by newline and
compressed, for faster processing. The JSONL format is natively
supported by various data science tools and libraries.

The path structure allows to easily select, identify and download data
based on the researcher's needs.

In the path template:

 * `cc` is an uppercase 2 letter country code

 * `testname` is a test name where underscores are removed

 * `timestamp` is a YYYYMMDD timestamp

 * `name` is a unique filename


### Compressed JSONL from measurements before 20201021
The path structure is:
`s3://ooni-data-eu-fra/jsonl/<testname>/<cc>/<timestamp>/00/<name>.jsonl.gz`

Example:

    s3://ooni-data-eu-fra/jsonl/webconnectivity/IT/20200921/00/20200921_IT_webconnectivity.l.0.jsonl.gz

You can list JSONL files with:

    s3cmd ls s3://ooni-data-eu-fra/jsonl/
    s3cmd ls s3://ooni-data-eu-fra/jsonl/webconnectivity/US/20201021/00/


### Compressed JSONL from measurements starting from 20201020
The path structure is:

    s3://ooni-data-eu-fra/raw/<timestamp>/<hour>/<cc>/<testname>/<ts2>_<cc>_<testname>.<host_id>.<counter>.jsonl.gz

Example:

    s3://ooni-data-eu-fra/raw/20210817/15/US/webconnectivity/2021081715_US_webconnectivity.n0.0.jsonl.gz

Note: The path will be updated in the future to live under `/jsonl/`

You can list JSONL files with:

    s3cmd ls s3://ooni-data-eu-fra/raw/20210817/15/US/webconnectivity/


### Raw "postcans" from measurements starting from 20201020
Each HTTP POST is stored in the tarball as
`<timestamp>_<cc>_<testname>/<timestamp>_<cc>_<testname>_<hash>.post`

Example:

    s3://ooni-data-eu-fra/raw/20210817/11/GB/webconnectivity/2021081711_GB_webconnectivity.n0.0.tar.gz

Listing postcan files:

    s3cmd ls s3://ooni-data-eu-fra/raw/20210817/
    s3cmd ls s3://ooni-data-eu-fra/raw/20210817/11/GB/webconnectivity/


# Other backend components

## Nginx
Nginx <https://www.nginx.com/> is used across various servers in the
backend, primarily as a reverse proxy. It's worth summarizing the main
different uses here:

 * Reverse proxy for the [API](#api)&thinsp;⚙, also providing **caching**
    from many API methods

 * Serving local measurements from disk from the backend hosts

 * Serving Atom/RSS feeds from disk from the backend hosts

 * Serving ACME challenge text files for [Dehydrated](#dehydrated)&thinsp;⚙

 * Reverse proxy for the test helpers

 * Reverse proxy for deb.ooni.org

 * Reverse proxy for internal or ancillary services e.g.
    [Prometheus](#tool:prometheus) scraping, [Grafana](#grafana)&thinsp;🔧
    etc

Nginx configuration files are stored in
<https://github.com/ooni/sysadmin/tree/master/ansible>

Most of the proxying functionalities of Nginx can be replaced with
[HaProxy](#haproxy)&thinsp;⚙ to benefit from load balancing and active
checks.

Caching could be provided by Varnish <https://varnish-cache.org/> as it
provides the ability to explicitly purge caches. This would be useful
when testing the API.


### Purging Nginx cache
While testing the API it can be useful to purge the cache provide by
Nginx.

This selectively removes the cache files used for the API:

    rm /var/cache/nginx/ooni-api/* -rf

> **note**
> This method is not natively supported by Nginx. It's recommended to use
> it only on the backend testbed.


## HaProxy
HaProxy runs on the [OONI bridges](#ooni-bridges)&thinsp;⚙ and works as a
load balancer for the test helpers and the APIs on
[backend-hel.ooni.org](#host:HEL), [ams-pg-test.ooni.org](#ams-pg-test.ooni.org)&thinsp;🖥
and the bridge on [bridge-greenhost.ooni.org](#bridge-greenhost.ooni.org)&thinsp;🖥.

Contrasted to [Nginx](#nginx)&thinsp;⚙ it's focused on load balancing rather
than serving files and provides:

 * dashboards showing the current status of the web services and the
    load balancing targets

 * flexible active healthchecks and failover

 * more detailed metrics

 * more flexible routing policies that allow implementing better
    incremental rollouts, A/B testing etc

HaProxy is currently deployed on:

 * [ams-pg-test.ooni.org](#ams-pg-test.ooni.org)&thinsp;🖥
    <https://ams-pg-test.ooni.org:444/__haproxy_stats>

 * [backend-hel.ooni.org](#backend-hel.ooni.org)&thinsp;🖥
    <https://backend-hel.ooni.org:444/__haproxy_stats>

 * [bridge-greenhost.ooni.org](#bridge-greenhost.ooni.org)&thinsp;🖥
    <https://bridge-greenhost.ooni.org:444/__haproxy_stats>

An example of the built-in dashboard:

![haproxy](images/haproxy.png)

Also see [HaProxy dashboard](#haproxy-dashboard)&thinsp;📊.

When providing load balancing for the [Test helpers](#test-helpers)&thinsp;⚙
it uses a stateful algorithm based on the source IP address to ensure
that every given probe reaches the same test helper. This is meant to
help troubleshooting. Yet, in case a test helper becomes unreachable
probe traffic is sent to the remaining test helpers. This affects
exclusively the probes that were using the unreachable test helper. The
probes that were reaching other test helpers are not shuffled around.


## Dehydrated
Dehydrated provides Let's Encrypt certificate handling using ACME. It
replaces certbot with a simpler and more reliable implementation.

Dehydrated is configured in [Ansible](#ansible)&thinsp;🔧, see
<https://github.com/ooni/sysadmin/tree/master/ansible/roles/dehydrated>

For monitoring see [TLS certificate dashboard](#tls-certificate-dashboard)&thinsp;📊. There are
alerts configured in [Grafana](#grafana)&thinsp;🔧 to alert on expiring
certificates, see [Alerting](#alerting)&thinsp;💡.


## Jupyter Notebook
There is an instance of Jupyter Notebook <https://jupyter.org/> deployed
on the [monitoring.ooni.org](#monitoring.ooni.org)&thinsp;🖥 available for internal
use at <https://jupyter.ooni.org/tree/notebooks>

It is used primarily for:

 * Performing research and data analysis using data science tools like
    [Pandas](https://pandas.pydata.org/) and
    [Altair](https://altair-viz.github.io/).

 * Generating automatic dashboards using [Jupycron](#jupycron)&thinsp;🔧 and
    sending alerts.

 * Analyzing logs from the
    [ClickHouse instance for logs](#clickhouse-instance-for-logs)&thinsp;⚙

> **important**
> There in no user account support in Jupyter Notebook. The instance is
> protected by HTTP basic auth using team credentials. To clarify
> ownership of notebooks put your account name as part of the notebook
> name. To prevent data loss do not modify notebooks owned by other users.


### Ooniutils microlibrary
The following notebook is often used as a library in other notebooks:
<https://jupyter.ooni.org/notebooks/notebooks/ooniutils.ipynb>

It can be imported in other notebooks by adding this at the top:

    %run ooniutils.ipynb

> **important**
> be careful when making changes to it because it could break many
> notebooks including the ones automatically run by
> [Jupycron](#jupycron)&thinsp;🔧

Running the notebook imports commonly used libraries, including Pandas
and Altair, configures Jupyter Notebook and provides some convenience
functions:

 * `click_query_fsn(query, **params)` to run queries against ClickHouse
    on [backend-fsn.ooni.org](#backend-fsn.ooni.org)&thinsp;🖥. Returns a Pandas dataframe.

 * `alertmanager_fire_alert(summary, alertname, job="", instance="", annotations={}, duration_min=1)`
    to send an alert through alertmanager ([Grafana](#grafana)&thinsp;🔧).

 * `send_slack_msg(title, msg, color="3AA3E3")` to send messages
    directly to [Slack](#slack)&thinsp;🔧.

 * `send_alert_through_ntfy(title, msg, priority="urgent", tags="warning")`
    to send alerts directly using <https://ntfy.sh/> - see
    [Redundant notifications](#redundant-notifications)&thinsp;🔧 for details.

Confusingly, `alertmanager_fire_alert` needs an alarm duration to be set
when called.

> **note**
> `send_slack_msg` can be used in addition to provide more details and
> subsequent updates to an existing alert.

Additionally, `send_slack_msg` can deliver clickable links.

> **note**
> When creating new alerts it is helpful to include full links to the
> automated notebook generating the alert and its HTML output.

See [Jupycron](#jupycron)&thinsp;🔧 for details.


### Jupycron
Jupycron is a Python script that runs Jupyter notebooks automatically.

Various notebooks are used to perform analysing, reporting and alarming
using data science tools that are more powerful than
[Grafana](#tool:grafana) and [Prometheus](#prometheus)&thinsp;🔧 internal
query language. An example is the use of
[scikit-learn](https://scikit-learn.org)\'s machine learning for
predicting incoming measurement flow.

It is internally developed and hosted on
[github](https://github.com/ooni/jupycron.git). It is deployed by
[Ansible](#tool:ansible) on [monitoring.ooni.org](#monitoring.ooni.org)&thinsp;🖥.

It runs every minute and scans the existing notebooks at
`/var/lib/jupyter/notebooks/`. It parses only notebooks that have the
word `autorun` in the filename. (see example below) It then scans the
content of the notebook looking for a code cell that contains a
commented line like:

    # jupycron: {"every": "30 minute"}

If such line is found it executes the notebook according to the required
time interval and stores the output as an HTML file at
`/var/lib/jupyter/notebooks/jupycron`

Execution intervals can be specified using keywords:

    "min", "hour", "day", "week", "month"

> **note**
> The `AUTORUN` environment variable is set when a notebook is run under
> jupycron. Also [Ooniutils microlibrary](#ooniutils-microlibrary)&thinsp;💡 sets the
> `autorun` Python variable to `True`. This can be useful to send alerts
> only when notebooks are being run automatically.

Jupycron also provides an HTML
[summary](https://jupyter.ooni.org/view/notebooks/jupycron/summary.html)
of the existing automated notebooks.

The status column indicates the outcome of the previous run, if any:

 * 🟢: successful run

 * 🔴: failed run

 * ⌛: never executed before

 * 🛇: disabled notebook: the `# jupycron: {…​}` line was not found

> **note**
> notebooks are executed by `jupyter-nbconvert` under `systemd-run` with
> memory limits to protect the [monitoring.ooni.org](#monitoring.ooni.org)&thinsp;🖥
> host. The limit can be changed by setting a `MaxMem` key in the
> configuration line, in megabytes.

Debugging tip: Jupycron stores the history of notebook executions in
`/var/lib/jupyter/notebooks/jupycron/.history.json`.

For an example of automated notebook that sends alarm see
[Test helper failure rate notebook](#test-helper-failure-rate-notebook)&thinsp;📔

> **note**
> When a notebook is run automatically by Jupycron only the HTML output is updated.
> The notebook itself is not.


### Test helper failure rate notebook
This automated notebook performs a correlation of test failures and the
location of [Test helpers](#test-helpers)&thinsp;⚙.

It sends alerts directly to [Slack](#slack)&thinsp;🔧.

Notebook:
<https://jupyter.ooni.org/notebooks/notebooks/autorun_test_helper_failure_rate_alarm.ipynb>

Output:
<https://jupyter.ooni.org/view/notebooks/jupycron/autorun_test_helper_failure_rate_alarm.html>

Also see the [test helpers notebook](#test-helpers-notebook)&thinsp;📔,
the [test helper rotation runbook](#test-helper-rotation-runbook)&thinsp;📒 and
the [test helpers failure runbook](#test-helpers-failure-runbook)&thinsp;📒


### Test helpers notebook
This notebook provides tables and charts to investigate the general
status of the [Test helpers](#test-helpers)&thinsp;⚙

It provides a summary of the live and rotated test helpers:

![notebook](images/test_helpers_notebook.png)

See <https://jupyter.ooni.org/notebooks/notebooks/test%20helpers.ipynb>
for investigation

Also see the [test helper rotation runbook](#test-helper-rotation-runbook)&thinsp;📒 and
the [test helpers failure runbook](#test-helpers-failure-runbook)&thinsp;📒


### Android probe release notebook
This automated notebook is used to compare changes in incoming
measurements across different versions of the Android probe.

It is used in the [Android probe release runbook](#android-probe-release-runbook)&thinsp;📒

Notebook:
<https://jupyter.ooni.org/notebooks/notebooks/autorun_android_probe_release.ipynb>

Output:
<https://jupyter.ooni.org/view/notebooks/jupycron/autorun_android_probe_release.html>


### iOS probe release notebook
This automated notebook is used to compare changes in incoming
measurements across different versions of the iOS probe.

Notebook:
<https://jupyter.ooni.org/notebooks/notebooks/autorun_ios_probe_release.ipynb>

Output:
<https://jupyter.ooni.org/view/notebooks/jupycron/autorun_ios_probe_release.html>


### CLI probe release notebook
This automated notebook performs Used to compare changes in incoming
measurements across different versions of the CLI probe.

It is used in the [CLI probe release runbook](#cli-probe-release-runbook)&thinsp;📒

Notebook:
<https://jupyter.ooni.org/notebooks/notebooks/autorun_cli_probe_release.ipynb>

Output:
<https://jupyter.ooni.org/view/notebooks/jupycron/autorun_cli_probe_release.html>


### Duplicate test-list URLs notebook
This automated notebook shows duplicate URLs across global and
per-country test lists

Notebook:
<https://jupyter.ooni.org/notebooks/notebooks/autorun_duplicate_test_list_urls.ipynb>

Output:
<https://jupyter.ooni.org/view/notebooks/jupycron/autorun_duplicate_test_list_urls.html>


### Monitor blocking event detections notebook
This automated notebook monitor the
[Social media blocking event detector](#social-media-blocking-event-detector)&thinsp;⚙ creating a summary table with clickable links
to events. It also sends notification messages on [Slack](#slack)&thinsp;🔧.

Notebook:
<https://jupyter.ooni.org/notebooks/notebooks/autorun_event_detector_alerts.ipynb>

Output:
<https://jupyter.ooni.org/view/notebooks/jupycron/autorun_event_detector_alerts.html>


### Logs from FSN notebook
This automated notebook provides summaries and examples of log analysis.
See [ClickHouse instance for logs](#clickhouse-instance-for-logs)&thinsp;⚙ for an overview.

Notebook:
<https://jupyter.ooni.org/notebooks/notebooks/autorun_fsn_logs.ipynb>

Output:
<https://jupyter.ooni.org/view/notebooks/jupycron/autorun_fsn_logs.html>


### Logs investigation notebook
This notebook provides various examples of log analysis. See
[ClickHouse instance for logs](#clickhouse-instance-for-logs)&thinsp;⚙ for an overview.

<https://jupyter.ooni.org/notebooks/notebooks/centralized_logs.ipynb>


### Incoming measurements prediction and alarming notebook
This automated notebook uses Sklearn to implement predictions of the
incoming measurement flow using a linear regression. It generates alarms
if incoming traffic drops below a given threshold compared to the
predicted value.

The purpose of the notebook is to detect major outages or any event
affecting ooni's infrastructure that causes significant loss in incoming
traffic.

Predictions and runs work on a hourly basis and process different probe
types independently.

Notebook:
<https://jupyter.ooni.org/notebooks/notebooks/autorun_incoming_measurements_prediction_alarming.ipynb>

Output:
<https://jupyter.ooni.org/view/notebooks/jupycron/autorun_incoming_measurements_prediction_alarming.html>

An example of measurement flow prediction:

![prediction](images/autorun_incoming_measurements_prediction_alarming.html.png)


### Incoming measurements prediction-by-country notebook
This automated notebook is similar to the previous one but evaluates
each country independently.

The main purpose is to detect countries encountering significant
Internet access disruption or blocking of ooni's network traffic.

The notebook is currently work-in-progress and therefore monitoring few
countries.

Notebook:
<https://jupyter.ooni.org/notebooks/notebooks/autorun_incoming_measurements_prediction_alarming_per_country.ipynb>

Output:
<https://jupyter.ooni.org/view/notebooks/jupycron/autorun_incoming_measurements_prediction_alarming_per_country.html>


### Long term measurements prediction notebook
This automated notebook runs long-term predictions of incoming
measurement flows and alarms on significant drops.

The purpose is to detect **slow** decreases in measurements over time
that have gone unnoticed in the past in similar situations.

Notebook:
<https://jupyter.ooni.org/notebooks/notebooks/autorun_incoming_measurements_prediction_long_term_alarming.ipynb>

Output:
<https://jupyter.ooni.org/view/notebooks/jupycron/autorun_incoming_measurements_prediction_long_term_alarming.html>


### Incoming measurements notebook
This automated notebook provides a dashboard of incoming measurement
flows grouped by probe type, from a multiple-year point of view.

Its purpose is to provide charts to be reviewed by the team to discuss
long term trends.

Notebook:
<https://jupyter.ooni.org/notebooks/notebooks/autorun_incoming_msmts.ipynb>

Output:
<https://jupyter.ooni.org/view/notebooks/jupycron/autorun_incoming_msmts.html>


### ClickHouse queries notebook
This is a non-automated notebook used to summarize heavy queries in
[ClickHouse](#clickhouse)&thinsp;⚙

<https://jupyter.ooni.org/notebooks/notebooks/2023%20%5Bfederico%5D%20clickhouse%20query%20log.ipynb>

> **note**
> The `system.query_log` table grows continuously and might be trimmed or
> emptied using `TRUNCATE` to free disk space.


### Priorities and weights notebook
Notebooks to investigate prioritization. See
[Priorities and weights](#priorities-and-weights)&thinsp;💡

<https://jupyter.ooni.org/notebooks/notebooks/2022%20test-list%20URL%20input%20prioritization%20dashboard.ipynb>

<https://jupyter.ooni.org/notebooks/notebooks/2022%20test-list%20URL%20input%20prioritization%20experiments.ipynb>


### Campaign investigation notebook
A test campaign has been monitored with the following notebook. It could
be tweaked and improved for other campaigns.
To reuse it copy it to a new notebook and update the queries. Rerun all cells.
The notebook will show how the measurement quantity and coverage increased.

<https://jupyter.ooni.org/notebooks/notebooks/2023-05-18%20TL%20campaign.ipynb>


### Easy charting notebook
This notebook contains examples that can be used as building blocks for various
investigations and dashboards.

It provides the `easy_msm_count_chart` helper function.
Such functions shows measurement counts over time, grouped over a dimension.

By default it shows maximum 6 different values (color bands).
This means you are not seeing the total amount of measurements in the charts.

You can add the following parameters, all optional:
- `title`: free-form title
- `since`: when to start, defaults to "3 months". You can use day, week, month, year
- `until`: when to stop, defaults to now. You can use day, week, month, year
- `granularity`: how big is each time slice: day, week, month. Defaults to day
- `dimension`: what dimension to show as different color bands. Defaults to `software_version`
- `dimension_limit`: how many values to show as different color bands. **Defaults to 6.**

The following parameters add filtering. The names should be self explanatory.
The default value is no filter (count everything).
- `filter_software_name`: looks only at measurement for a given software name, e.g. "ooniprobe-android".
- `filter_domain`
- `filter_engine_name`
- `filter_input`
- `filter_probe_asn`
- `filter_probe_cc`
- `filter_test_name`

<https://jupyter.ooni.org/notebooks/notebooks/easy_charts.ipynb>



## Grafana
Grafana <https://grafana.com/> is a popular platform for monitoring and
alarming.

It is deployed on [monitoring.ooni.org](#monitoring.ooni.org)&thinsp;🖥 by
[Ansible](#ansible)&thinsp;🔧 and lives at <https://grafana.ooni.org/>

See [Grafana backup runbook](#grafana-backup-runbook)&thinsp;📒 and
[Grafana editing](#grafana-editing)&thinsp;📒


## ClickHouse instance for logs
There is an instance of ClickHouse deployed on
[monitoring.ooni.org](#monitoring.ooni.org)&thinsp;🖥 that receives logs. See
[Log management](#log-management)&thinsp;💡 for details on logging in general.

See [Logs from FSN notebook](#logs-from-fsn-notebook)&thinsp;📔 and
[Logs investigation notebook](#logs-investigation-notebook)&thinsp;📔 for examples on how to query the `logs` table to
extract logs and generate charts.

> **note**
> The `logs` table on [monitoring.ooni.org](#monitoring.ooni.org)&thinsp;🖥 is indexed
> by `__REALTIME_TIMESTAMP`. It can be used for fast ordering and
> filtering by time.

An example of a simple query to show recent logs:

``` sql
SELECT timestamp, _SYSTEMD_UNIT, message
FROM logs
WHERE host = 'backend-fsn'
AND __REALTIME_TIMESTAMP > toUnixTimestamp(NOW('UTC') - INTERVAL 1 minute) * 1000000)
ORDER BY __REALTIME_TIMESTAMP DESC
LIMIT 10
```


## Vector
Vector <https://vector.dev> is a log (and metric) management tool. See
[Log management](#log-management)&thinsp;💡 for details.

> **important**
> Vector is not packaged in Debian yet. See
> <https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=1019316> Vector is
> currently installed using a 3rd party APT archive.


## ClickHouse
ClickHouse is main database that stores measurements and many other
tables. It accessed primarily by the [API](#api)&thinsp;⚙ and the
[Fastpath](#fastpath)&thinsp;⚙.

It is an OLAP, columnar database. For documentation see
<https://clickhouse.com/docs/en/intro>

The database schema required by the API is stored in:
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/tests/integ/clickhouse_1_schema.sql>

ClickHouse is deployed by [Ansible](#ansible)&thinsp;🔧 as part of the
deploy-backend.yml playbook. It is installed using an APT archive from
the developers and

Related: [Database backup tool](#database-backup-tool)&thinsp;⚙


### Overall design
The backend uses both the native ClickHouse table engine MergeTree and
\"foreign\" database engines, for example in EmbeddedRocksDB tables.

The MergeTree family is column-oriented, **eventually consistent**,
**non-transactional** and optimized for OLAP workloads. It is suitable
for very large number of records and columns.

The existing tables using MergeTree are meant to support replication in
future by switching to ReplicatedMergeTree. This is crucial to implement
active/standby and even active/active setups both for increased
reliability and performance.

This choice is reflected in how record insertion and deduplication are
performed: where possible, the API codebase assumes that multiple API
instances can insert and fetch records without leading to inconsistent
results. This requires special care in the implementation because
MergeTree tables do not support **immediate** consistency nor
transactions.

EmbeddedRocksDB uses <https://rocksdb.org/> - a key-value, row-oriented,
low-latency engine. Some backend workloads are more suitable for this
engine. EmbeddedRocksDB **does not support replication**.

> **important**
> Some workloads would better suited for a transactional and immediately
> consistent database. E.g. [OONI Run](#ooni-run)&thinsp;🐝 and
> [Incident management](#incident-management)&thinsp;🐝. See
> <https://clickhouse.com/docs/en/engines/table-engines/special/keeper-map>

To get an overview of the existing tables and engines use:

    SELECT engine, name FROM system.tables WHERE database = 'default' ORDER BY engine, name

An overview of the more important tables:

 * [accounts table](#accounts-table)&thinsp;⛁ EmbeddedRocksDB

 * [asnmeta table](#asnmeta-table)&thinsp;⛁ MergeTree

 * [citizenlab table](#citizenlab-table)&thinsp;⛁ ReplacingMergeTree

 * [citizenlab_flip table](#citizenlab_flip-table)&thinsp;⛁ ReplacingMergeTree

 * [counters_asn_test_list table](#counters_asn_test_list-table)&thinsp;⛁
    MaterializedView

 * [counters_test_list table](#counters_test_list-table)&thinsp;⛁ MaterializedView

 * [fastpath table](#fastpath-table)&thinsp;⛁ ReplacingMergeTree

 * [fingerprints_dns table](#fingerprints_dns-table)&thinsp;⛁ EmbeddedRocksDB

 * [fingerprints_dns_tmp table](#fingerprints_dns_tmp-table)&thinsp;⛁
    EmbeddedRocksDB

 * [fingerprints_http table](#fingerprints_http-table)&thinsp;⛁ EmbeddedRocksDB

 * [fingerprints_http_tmp table](#fingerprints_http_tmp-table)&thinsp;⛁
    EmbeddedRocksDB

 * [incidents table](#incidents-table)&thinsp;⛁ ReplacingMergeTree

 * [jsonl table](#jsonl-table)&thinsp;⛁ MergeTree

 * [msmt_feedback table](#msmt_feedback-table)&thinsp;⛁ ReplacingMergeTree

 * [oonirun table](#oonirun-table)&thinsp;⛁ ReplacingMergeTree

 * [session_expunge table](#session_expunge-table)&thinsp;⛁ EmbeddedRocksDB

 * [test_groups table](#test_groups-table)&thinsp;⛁ Join

 * [url_priorities table](#url_priorities-table)&thinsp;⛁ CollapsingMergeTree

> **note**
> As ClickHouse does not support transactions, there are some workarounds
> to implement atomic updates of whole tables.

One way is to use two tables with the same schema, where one table
receive updates and another one is used for reading, and swap them once
the writes are completed. This is used by the [API](#api)&thinsp;⚙,
[Test helper rotation](#test-helper-rotation)&thinsp;⚙ and other components. The
SQL syntax is:

```sql
EXCHANGE TABLES <a> AND <b>
```


### accounts table
Used for authentication. Assignes roles to accounts (by account id). The
default value for accounts not present in the table is \"user\". As
such, the table is currently tracking only admin roles.

Schema:

```sql
CREATE TABLE default.accounts
(
    `account_id` FixedString(32),
    `role` String,
    `update_time` DateTime DEFAULT now()
)
ENGINE = EmbeddedRocksDB
PRIMARY KEY account_id
```

To create and update account roles see:

[Creating admin API accounts](#creating-admin-api-accounts)&thinsp;📒


### asnmeta table
Contains [ASN](#asn)&thinsp;💡 lookup data used by the API

Schema:

``` sql
CREATE TABLE default.asnmeta
(
    `asn` UInt32,
    `org_name` String,
    `cc` String,
    `changed` Date,
    `aut_name` String,
    `source` String
)
ENGINE = MergeTree
ORDER BY (asn, changed)
SETTINGS index_granularity = 8192
```


### asnmeta_tmp table
Temporary table, see [asnmeta table](#asnmeta-table)&thinsp;⛁

Schema:

``` sql
CREATE TABLE default.blocking_events
(
    `test_name` String,
    `input` String,
    `probe_cc` String,
    `probe_asn` Int32,
    `status` String,
    `time` DateTime64(3),
    `detection_time` DateTime64(0) MATERIALIZED now64()
)
ENGINE = ReplacingMergeTree
ORDER BY (test_name, input, probe_cc, probe_asn, time)
SETTINGS index_granularity = 4
```

Schema:

``` sql
CREATE TABLE default.blocking_status
(
    `test_name` String,
    `input` String,
    `probe_cc` String,
    `probe_asn` Int32,
    `confirmed_perc` Float32,
    `pure_anomaly_perc` Float32,
    `accessible_perc` Float32,
    `cnt` Float32,
    `status` String,
    `old_status` String,
    `change` Float32,
    `stability` Float32,
    `update_time` DateTime64(0) MATERIALIZED now64()
)
ENGINE = ReplacingMergeTree
ORDER BY (test_name, input, probe_cc, probe_asn)
SETTINGS index_granularity = 4
```


### citizenlab table
Contains data from the
[CitizenLab URL testing list repository](https://github.com/citizenlab/test-lists).

Schema:

``` sql
CREATE TABLE default.citizenlab
(
    `domain` String,
    `url` String,
    `cc` FixedString(32),
    `category_code` String
)
ENGINE = ReplacingMergeTree
ORDER BY (domain, url, cc, category_code)
SETTINGS index_granularity = 4
```

Receive writes from [CitizenLab test list updater](#citizenlab-test-list-updater)&thinsp;⚙

Used by [CitizenLab](#citizenlab)&thinsp;🐝


### citizenlab_flip table
Temporary table. See [CitizenLab test list updater](#citizenlab-test-list-updater)&thinsp;⚙


### counters_asn_test_list table
A `MATERIALIZED VIEW` table that, despite the name, is updated
continuously by ClickHouse as new measurements are inserted in the
`fastpath` table.

It contains statistics on the incoming measurement flow, grouped by
week, `probe_cc`, `probe_asn` and `input`. It is used by
[Prioritization](#prioritization)&thinsp;🐝.

Schema:

``` sql
CREATE MATERIALIZED VIEW default.counters_asn_test_list
(
    `week` DateTime,
    `probe_cc` String,
    `probe_asn` UInt64,
    `input` String,
    `msmt_cnt` UInt64
)
ENGINE = SummingMergeTree
ORDER BY (probe_cc, probe_asn, input)
SETTINGS index_granularity = 8192 AS
SELECT
    toStartOfWeek(measurement_start_time) AS week,
    probe_cc,
    probe_asn,
    input,
    count() AS msmt_cnt
FROM default.fastpath
INNER JOIN default.citizenlab ON fastpath.input = citizenlab.url
WHERE (measurement_start_time < now()) AND (measurement_start_time > (now() - toIntervalDay(8))) AND (test_name = \'web_connectivity\')
GROUP BY week, probe_cc, probe_asn, input
```


### counters_test_list table
Similar to [counters_asn_test_list table](#counters_asn_test_list-table)&thinsp;⛁ -
the main differences are that this table has daily granularity and does
not discriminate by `probe_asn`

Schema:

``` sql
CREATE MATERIALIZED VIEW default.counters_test_list
(
    `day` DateTime,
    `probe_cc` String,
    `input` String,
    `msmt_cnt` UInt64
)
ENGINE = SummingMergeTree
PARTITION BY day
ORDER BY (probe_cc, input)
SETTINGS index_granularity = 8192 AS
SELECT
    toDate(measurement_start_time) AS day,
    probe_cc,
    input,
    count() AS msmt_cnt
FROM default.fastpath
INNER JOIN default.citizenlab ON fastpath.input = citizenlab.url
WHERE (measurement_start_time < now()) AND (measurement_start_time > (now() - toIntervalDay(8))) AND (test_name = \'web_connectivity\')
GROUP BY day, probe_cc, input
```


### fastpath table
This table stores the output of the [Fastpath](#fastpath)&thinsp;⚙. It is
usually the largest table in the database and receives the largest
amount of read and write traffic.

It is used by multiple entry points in the [API](#api)&thinsp;⚙, primarily
for measurement listing and presentation and by the
[Aggregation and MAT](#aggregation-and-mat)&thinsp;🐝

Schema:

``` sql
CREATE TABLE default.fastpath
(
    `measurement_uid` String,
    `report_id` String,
    `input` String,
    `probe_cc` LowCardinality(String),
    `probe_asn` Int32,
    `test_name` LowCardinality(String),
    `test_start_time` DateTime,
    `measurement_start_time` DateTime,
    `filename` String,
    `scores` String,
    `platform` String,
    `anomaly` String,
    `confirmed` String,
    `msm_failure` String,
    `domain` String,
    `software_name` String,
    `software_version` String,
    `control_failure` String,
    `blocking_general` Float32,
    `is_ssl_expected` Int8,
    `page_len` Int32,
    `page_len_ratio` Float32,
    `server_cc` String,
    `server_asn` Int8,
    `server_as_name` String,
    `update_time` DateTime64(3) MATERIALIZED now64(),
    `test_version` String,
    `architecture` String,
    `engine_name` LowCardinality(String),
    `engine_version` String,
    `test_runtime` Float32,
    `blocking_type` String,
    `test_helper_address` LowCardinality(String),
    `test_helper_type` LowCardinality(String),
    INDEX fastpath_rid_idx report_id TYPE minmax GRANULARITY 1,
    INDEX measurement_uid_idx measurement_uid TYPE minmax GRANULARITY 8
)
ENGINE = ReplacingMergeTree(update_time)
ORDER BY (measurement_start_time, report_id, input, measurement_uid)
SETTINGS index_granularity = 8192
```

See [Fastpath deduplication](#fastpath-deduplication)&thinsp;📒 for deduplicating the table
records.


### fingerprints_dns table
Stores measurement DNS fingerprints. The contents are used by the
[Fastpath](#fastpath)&thinsp;⚙ to detect `confirmed` measurements.

It is updated by the [Fingerprint updater](#fingerprint-updater)&thinsp;⚙

Schema:

``` sql
CREATE TABLE default.fingerprints_dns
(
    `name` String,
    `scope` Enum8(\'nat\' = 1, \'isp\' = 2, \'prod\' = 3, \'inst\' = 4, \'vbw\' = 5, \'fp\' = 6),
    `other_names` String,
    `location_found` String,
    `pattern_type` Enum8(\'full\' = 1, \'prefix\' = 2, \'contains\' = 3, \'regexp\' = 4),
    `pattern` String,
    `confidence_no_fp` UInt8,
    `expected_countries` String,
    `source` String,
    `exp_url` String,
    `notes` String
)
ENGINE = EmbeddedRocksDB
PRIMARY KEY name
```


### fingerprints_dns_tmp table
Temporary table. See [Fingerprint updater](#fingerprint-updater)&thinsp;⚙


### fingerprints_http table
Stores measurement HTTP fingerprints. The contents are used by the
[Fastpath](#fastpath)&thinsp;⚙ to detect `confirmed` measurements.

It is updated by the [Fingerprint updater](#fingerprint-updater)&thinsp;⚙

Schema:

``` sql
CREATE TABLE default.fingerprints_http
(
    `name` String,
    `scope` Enum8(\'nat\' = 1, \'isp\' = 2, \'prod\' = 3, \'inst\' = 4, \'vbw\' = 5, \'fp\' = 6, \'injb\' = 7, \'prov\' = 8),
    `other_names` String,
    `location_found` String,
    `pattern_type` Enum8(\'full\' = 1, \'prefix\' = 2, \'contains\' = 3, \'regexp\' = 4),
    `pattern` String,
    `confidence_no_fp` UInt8,
    `expected_countries` String,
    `source` String,
    `exp_url` String,
    `notes` String
)
ENGINE = EmbeddedRocksDB
PRIMARY KEY name
```


### fingerprints_http_tmp table
Temporary table. See [Fingerprint updater](#fingerprint-updater)&thinsp;⚙


### incidents table
Stores incidents. See [Incident management](#incident-management)&thinsp;🐝.

Schema:

``` sql
CREATE TABLE default.incidents
(
    `update_time` DateTime DEFAULT now(),
    `start_time` DateTime DEFAULT now(),
    `end_time` Nullable(DateTime),
    `creator_account_id` FixedString(32),
    `reported_by` String,
    `id` String,
    `title` String,
    `text` String,
    `event_type` LowCardinality(String),
    `published` UInt8,
    `deleted` UInt8 DEFAULT 0,
    `CCs` Array(FixedString(2)),
    `ASNs` Array(UInt32),
    `domains` Array(String),
    `tags` Array(String),
    `links` Array(String),
    `test_names` Array(String),
    `short_description` String,
    `email_address` String,
    `create_time` DateTime DEFAULT now()
)
ENGINE = ReplacingMergeTree(update_time)
ORDER BY id
SETTINGS index_granularity = 1
```


### jsonl table
This table provides a method to look up measurements in
[JSONL files](#topic:jsonl) stored in [S3 data bucket](#s3-data-bucket)&thinsp;💡 buckets.

It is written by the [Measurement uploader](#measurement-uploader)&thinsp;⚙ when
[Postcans](#topic:postcans) and [JSONL files](#jsonl-files)&thinsp;💡 just after
measurements are uploaded to the bucket.

It is used by multiple entry points in the [API](#api)&thinsp;⚙, primarily
by `get_measurement_meta`.

<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/ooniapi/measurements.py#L470>

Schema:

``` sql
CREATE TABLE default.jsonl
(
    `report_id` String,
    `input` String,
    `s3path` String,
    `linenum` Int32,
    `measurement_uid` String,
    `date` Date,
    `source` String,
    `update_time` DateTime64(3) MATERIALIZED now64()
)
ENGINE = ReplacingMergeTree(update_time)
ORDER BY (report_id, input, measurement_uid)
SETTINGS index_granularity = 8192
```


### msmt_feedback table
Used for [Measurement feedback](#measurement-feedback)&thinsp;🐝

Schema:

``` sql
CREATE TABLE default.msmt_feedback
(
    `measurement_uid` String,
    `account_id` String,
    `status` String,
    `update_time` DateTime64(3) MATERIALIZED now64(),
    `comment` String
)
ENGINE = ReplacingMergeTree
ORDER BY (measurement_uid, account_id)
SETTINGS index_granularity = 4
```


### oonirun table
Used for [OONI Run](#ooni-run)&thinsp;🐝

Schema:

``` sql
CREATE TABLE default.oonirun
(
    `ooni_run_link_id` UInt64,
    `descriptor_creation_time` DateTime64(3),
    `translation_creation_time` DateTime64(3),
    `creator_account_id` FixedString(32),
    `archived` UInt8 DEFAULT 0,
    `descriptor` String,
    `author` String,
    `name` String,
    `short_description` String,
    `icon` String
)
ENGINE = ReplacingMergeTree(translation_creation_time)
ORDER BY (ooni_run_link_id, descriptor_creation_time)
SETTINGS index_granularity = 1
```


### session_expunge table
Used for authentication. It stores

Schema:

``` sql
CREATE TABLE default.session_expunge
(
    `account_id` FixedString(32),
    `threshold` DateTime DEFAULT now()
)
ENGINE = EmbeddedRocksDB
PRIMARY KEY account_id
```


### obs_openvpn table
Table used by OpenVPN tests. Written by the [Fastpath](#fastpath)&thinsp;⚙
and read by the [API](#api)&thinsp;⚙

Schema:

``` sql
CREATE TABLE default.obs_openvpn
(
    `anomaly` Int8,
    `bootstrap_time` Float32,
    `confirmed` Int8,
    `error` String,
    `failure` String,
    `input` String,
    `last_handshake_transaction_id` Int32,
    `measurement_start_time` DateTime,
    `measurement_uid` String,
    `minivpn_version` String,
    `obfs4_version` String,
    `obfuscation` String,
    `platform` String,
    `probe_asn` Int32,
    `probe_cc` String,
    `probe_network_name` String,
    `provider` String,
    `remote` String,
    `report_id` String,
    `resolver_asn` Int32,
    `resolver_ip` String,
    `resolver_network_name` String,
    `software_name` String,
    `software_version` String,
    `success` Int8,
    `success_handshake` Int8,
    `success_icmp` Int8,
    `success_urlgrab` Int8,
    `tcp_connect_status_success` Int8,
    `test_runtime` Float32,
    `test_start_time` DateTime,
    `transport` String
)
ENGINE = ReplacingMergeTree
ORDER BY (measurement_start_time, report_id, input)
SETTINGS index_granularity = 8
```


### test_groups table
Contains the definition of test groups. Updated manually and read by the
[API](#comp:api), mainly to show grouping in [Explorer](#explorer)&thinsp;🖱.

Schema:

``` sql
CREATE TABLE default.test_groups
(
    `test_name` String,
    `test_group` String
)
ENGINE = Join(ANY, LEFT, test_name)
```


### test_helper_instances table
List of live, draining and destroyed test helper instances. Used by
[Test helper rotation](#test-helper-rotation)&thinsp;⚙ internally.

Schema:

``` sql
CREATE TABLE default.test_helper_instances
(
    `rdn` String,
    `dns_zone` String,
    `name` String,
    `provider` String,
    `region` String,
    `ipaddr` IPv4,
    `ipv6addr` IPv6,
    `draining_at` Nullable(DateTime(\'UTC\')),
    `sign` Int8,
    `destroyed_at` Nullable(DateTime(\'UTC\'))
)
ENGINE = CollapsingMergeTree(sign)
ORDER BY name
SETTINGS index_granularity = 8
```


### url_priorities table
This table stores rules to compute priorities for URLs used in
[Web connectivity test](#web-connectivity-test)&thinsp;Ⓣ.

See [Prioritization](#prioritization)&thinsp;🐝,
[Prioritization management](#api:priomgm) and [Prioritization rules UI](#prioritization-rules-ui)&thinsp;🖱

Schema:

``` sql
CREATE TABLE default.url_priorities
(
    `sign` Int8,
    `category_code` String,
    `cc` String,
    `domain` String,
    `url` String,
    `priority` Int32
)
ENGINE = CollapsingMergeTree(sign)
ORDER BY (category_code, cc, domain, url, priority)
SETTINGS index_granularity = 1024
```


## ClickHouse system tables
ClickHouse has many system tables that can be used for monitoring
performance and debugging.

Tables matching the following names (wildcard) can grow in size and also
cause unnecessary I/O load:

    system.asynchronous_metric_log*
    system.metric_log*
    system.query_log*
    system.query_thread_log*
    system.trace_log*


### Fastpath deduplication
ClickHouse does not deduplicate all records deterministically for
performance reasons. Full deduplication can be performed with the
following command and it is often required after running the fastpath to
reprocess old measurements. Deduplication is CPU and IO-intensive.

    time clickhouse-client -u admin --password admin --receive_timeout 9999 --send_timeout 9999 --query 'OPTIMIZE TABLE fastpath FINAL'


### Dropping tables
ClickHouse has a protection against table drops. Use this to allow
dropping a table once. After use the flag file is automatically removed:

    sudo touch '/var/lib/clickhouse/flags/force_drop_table' && sudo chmod 666 '/var/lib/clickhouse/flags/force_drop_table'


### Investigating table sizes
To monitor ClickHouse's performance and table size growth there's a
[dashboard](#dash:clickhouse) and the [ClickHouse queries notebook](#clickhouse-queries-notebook)&thinsp;📔

To investigate table and index sizes the following query is useful:

```sql
SELECT
    concat(database, '.', table) AS table,
    formatReadableSize(sum(bytes)) AS size,
    sum(rows) AS rows,
    max(modification_time) AS latest_modification,
    sum(bytes) AS bytes_size,
    any(engine) AS engine,
    formatReadableSize(sum(primary_key_bytes_in_memory)) AS primary_keys_size
FROM system.parts
WHERE active
GROUP BY database, table
ORDER BY bytes_size DESC
```

> **important**
> The system tables named `asynchronous_metric_log`, `query_log` and
> `query_thread_log` can be useful for debugging and performance
> optimization but grow over time and create additional I/O traffic. Also
> see [ClickHouse system tables](#clickhouse-system-tables)&thinsp;💡.

Possible workarounds are:

 * Drop old records.

 * Implement sampling writes that write only a percentage of the
    records.

 * Disable logging for specific users or queries.

 * Disable logging in the codebase running the query. See
    <https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/fastpath/fastpath/db.py#L177>
    for an example of custom settings.

 * Disable logging altogether.

Also see [Disable unnecessary ClickHouse system tables](#disable-unnecessary-clickhouse-system-tables)&thinsp;🐞


### Investigating database performance
If needed a simple script can be run to generate additional metrics and
logs to identify slow queries.

``` bash
while true;
do
  # Extract longest running query time and send it as a statsd metric
  v=$(clickhouse-client -q 'SELECT elapsed FROM system.processes ORDER BY elapsed DESC LIMIT 1')
  echo "clickhouse_longest_query_elapsed_seconds:$v|g" | nc -w 1 -u 127.0.0.1 8125

  # Extract count of running queries and send it as a statsd metric
  v=$(clickhouse-client -q 'SELECT count() FROM system.processes')
  echo "clickhouse_running_queries_count:$v|g" | nc -w 1 -u 127.0.0.1 8125

  # Log long running queries into journald
  clickhouse-client -q 'SELECT query FROM system.processes WHERE elapsed > 10' | sed 's/\\n//g' | systemd-cat -t click-queries

  sleep 1
done
```


## Continuous Deployment: Database schema changes
The database schema required by the API is stored in
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/tests/integ/clickhouse_1_schema.sql>

In a typical CD workflows it is possible to deploy database schema
changes in a way that provides safety against outages and requires no
downtimes.

Any component that can be deployed independently from others can be made
responsible for creating and updating the tables it needs. If multiple
components access the same table it's important to coordinate their code
updates and deployments during schema changes.

> **important**
> typical CD requires deployments to be performed incrementally to speed
> up and simplify rollouts. It does not guarantees that package versions
> can be rolled forward or back by skipping intermediate versions.

To support such use-case more complex database scheme versioning would
be required. Due to hardware resource constraints and time constraints
we need to be able to manually create new tables (especially on the test
stage) and perform analysis work, run performance and feature tests,
troubleshooting etc. As such, at this time the backend components are
not checking the database schema against a stored \"template\" at
install or start time.

> **note**
> A starting point for more automated database versioning existed at
> <https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/database_upgrade_schema.py>

> **note**
> in future such check could be implemented and generate a warning to be
> enabled only on non-test CD stages.

> **note**
> one way to make database changes at package install time is to use the
> `debian/postinst` script. For example see
> <https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/fastpath/debian/postinst>

> **important**
> in writing SQL queries explicitly set the columns you need to read or
> write and access them by name: any new column will be ignored.

Some examples of database schema change workflows:


### Adding a new column to the fastpath
The following workflow can be tweaked or simplified to add columns or
tables to other components.

> **note**
> multiple deployments or restarts should not trigger failures: always add
> columns using
> `ALTER TABLE <name> ADD COLUMN IF NOT EXISTS <name> <other parameters>`

The new column is added either manually or by deploying a new version of
the fastpath package that adds the column at install or start time.
Ensure the rollout reaches all the CD stages.

Prepare, code-review, merge and deploy a new version of the fastpath
that writes data to the new column. Ensure the rollout reaches all the
CD stages.

Verify the contents of the new column e.g. using a Jupyter Notebook

Prepare, code-review, merge and deploy a new version of the API that
reads data from the new column. Ensure the rollout reaches all the CD
stages.

This workflow guarantees that the same schema and codebase has been
tested on all stages before reaching production.


### Renaming a column or table
This is a more complex workflow especially when multiple components
access the same table or column. It can also be used for changing a
column type and other invasive changes.

For each of the following steps ensure the rollout reaches all the CD
stages.

 * Create a new column or table.

 * Prepare, code-review, merge and deploy a new version of all the
    components that write data. Write all data to both the old and new
    column/table.

 * If required, run one-off queries that migrate existing data from the
    old column/table to the new one. Using `IF NOT EXISTS` can be
    useful.

 * Prepare, code-review, merge and deploy a new version of all the
    components that reads data. Update queries to only read from the new
    column/table.

 * Prepare, code-review, merge and deploy a new version of all the
    components that write data. Write data only to the new column/table.

 * Run one-off queries to delete the old column/table. Using
    `IF EXISTS` can be useful for idempotent runs.


### Database schema check
To compare the database schemas across backend hosts you can use:

```bash
for hn in ams-pg-test backend-hel backend-fsn; do
  hn=${hn}.ooni.org
  echo $hn
  > "schema_${hn}"
  for tbl in $(ssh $hn 'clickhouse-client -q "SHOW TABLES FROM default"' | grep -v inner_id ); do
    echo "  ${tbl}"
    ssh $hn "clickhouse-client -q \"SHOW CREATE TABLE default.${tbl}\"" | sed 's/\\n/\n/g' >> "schema_${hn}"
  done
done
```

The generated files can be compared more easily using `meld`.

Also related: [Database backup tool](#database-backup-tool)&thinsp;⚙


# GitHub CI workflows
There are 5 main workflows that a run by GitHub Actions when new pull
requests are created or updated the backend repository.

They are configured using YAML files in the repository, plus secret keys
and other variables that can be edited on
<https://github.com/ooni/backend/settings/secrets/actions>

The workflow files are:

 * .github/workflows/build_deb_packages.yml

 * .github/workflows/docgen.yaml

 * .github/workflows/mypy.yml

 * .github/workflows/test_fastpath.yaml

 * .github/workflows/test_new_api.yml

> **warning**
> the workflows handle sensitive credentials. GitHub provides security
> features to prevent 3rd party contributors from running CI actions
> through pull requests that might expose credentials.


## Debian package build and publish
Builds one or more .deb files from the backend repository and uploads
them to Debian archives on S3.

The configuration file is:

    .github/workflows/build_deb_packages.yml

The workflow installs the required dependencies as Debian packages and
then fetches the [debops-ci tool](#debops-ci-tool)&thinsp;🔧 from the [sysadmin repository](https://github.com/ooni/sysadmin/tree/master/tools)


### debops-ci tool
A Python script that automates package builds and CI/CD workflows:

 * Detects `debian/` directories in the current repository. It inspects
    file changes tracked by git to identify which projects are being
    modified.

 * Bumps up version numbers according to the [package versioning](#package-versioning)&thinsp;💡 criteria described below.

 * Builds Debian packages as needed.

 * Generates a Debian archive on S3. Uploads .deb files into it to make
    them available for deployment.

The script has been designed to address the following requirements:

 * Remove dependencies from 3rd party services that might become expensive or be shut down.
 * Serverless: do not require a dedicated CI/CD server
 * Stateless: allow the script to run in CI and on developer desktops without requiring a local database, cache etc.
 * Compatible: generate a package archive that can be used by external users to install ooniprobe on their hosts using standard tooling.
 * CI-friendly: minimize the amount of data that needs to be deployed on servers to update a backend component or probe.
 * Reproducible: given the same sources the generated output should be virtually identical across different systems. I.e. it should not depend on other software installed locally.


It is used in the CI workflow as:

```bash
./debops-ci --show-commands ci --bucket-name ooni-internal-deb
```


#### debops-ci details
The tool requires the following environment variables for automated ci
runs:

 * `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` in order to write to
    S3 buckets.

 * `DEB_GPG_KEY` or `DEB_GPG_KEY_BASE64` to GPG-sign packages. The
    first variable can contain a PEM-encoded private key, while the
    second is base64-encoded.

Other configuration parameters can be set from the command line. See the
`-h` CLI option:

    usage:
    Version 2023-10-20.1

    Builds deb packages and uploads them to S3-compatible storage.
    Works locally and on GitHub Actions and CircleCI
    Detects which package[s] need to be built.
    Support "release" and PR/testing archives.

    scan              - scan the current repository for packages to be built
    build             - locate and build packages
    upload <filename> - upload one package to S3
    ci                - detect CircleCI PRs, build and upload packages

    Features:
     - Implement CI/CD workflow using package archives
     - Support adding packages to an existing S3 archive without requiring a
       local mirror
     - Support multiple packages in the same git repository
     - GPG signing
     - Support multiple architectures
     - Update changelogs automatically
     - Easy to debug
     - Phased updates (rolling deployments)

    positional arguments:
      {upload,scan,ci,build}

    options:
      -h, --help            show this help message and exit
      --bucket-name BUCKET_NAME
                            S3 bucket name
      --distro DISTRO       Debian distribution name
      --origin ORIGIN       Debian Origin name
      --arch ARCH           Debian architecture name
      --gpg-key-fp GPG_KEY_FP
                            GPG key fingerprint
      --show-commands       Show shell commands

When running with the `ci` or `upload` commands, the tool creates the
required directories and files on S3 to publish an archive suitable for
APT. It uploads any newly built package and then appends the package
description to the `Packages` file. It then generates a new `InRelease`
file and adds a GPG signature at the bottom of the file.

See
[Packages](https://ooni-internal-deb.s3.eu-central-1.amazonaws.com/dists/unstable/main/binary-amd64/Packages)
and
[InRelease](https://ooni-internal-deb.s3.eu-central-1.amazonaws.com/dists/unstable/InRelease)
as examples.

The current OONI package archives are:

 * <https://ooni-internal-deb.s3.eu-central-1.amazonaws.com/> - for
    internal use

 * <http://deb.ooni.org/> also available as <https://deb.ooni.org/> -
    for publicly available packages

Documentation on how to use the public archive:
<https://ooni.org/install/cli/ubuntu-debian>

The tool can also be used manually e.g. for debugging using actions
`scan`, `build` and `upload`.

Unlike other CI/CD tools, debops-ci is designed to be stateless in order
to run in ephemeral CI worker containers. Due to limitations around
transactional file uploads, a pair of lockfiles are used, named
`.debrepos3.lock` and `.debrepos3.nolock`

> **note**
> If the github actions CI workflow for package build is restarted
> manually (that is, without git-pushing a new commit) debops-ci will
> refuse to overwrite the .deb package already present on S3. This is
> expected.


### Package versioning
The CI workflows builds Debian packages with the following versioning
scheme:

    <semver_version>~pr<pull_request_number>-<build_number>

For example:

    1.0.79~pr751-194

This format has the benefits of:

 * Providing a user-configured [semantic versioning](https://semver.org/) part.

 * Automatically inserting a pull request number. This prevents version
    conflicts in case the first component has not been updated when
    opening a new pull request.

 * Automatically inserting an incremental build number. This allows
    incremental deployments of commits on testbeds.

Once a package at a given version is uploaded the CI tool will refuse to
overwrite it in future CI runs. This is meant to prevent changing the
contents of published packages.

It is recommended to update the `debian/changelog` file in the first
commit of each new pull request by adding a new semver number and a
short changelog. This helps keeping track of what changes are being
deployed on test, integration and production servers.

Sometimes multiple pull requests for the same package are open at the
same time and are rebased or merged in an order that is different from
the pull request number. In this case manually bumping up the semver
number after each rebase is beneficial to ensure that the rollout is
done incrementally.

For example:

 * Pull request 751 builds a package versioned 1.0.79\~pr751-14. The
    package is deployed on the testbed.

 * Pull request 752, opened later, is rebased on top of 751 and builds
    a package versioned 1.0.80\~pr752-51. The package is deployed on the
    testbed.

 * The need to reorder the two PRs arises. PR 751 is now rebased on top
    of 752. The changelog file is manually updated to version 1.0.81 The
    CI now generates 1.0.81\~pr751-43 and the packa

See the subchapter on the [The deployer tool](#the-deployer-tool)&thinsp;🔧


## Code documentation generation
The configuration lives at:

    .github/workflows/docgen.yaml

The workflow runs `./build_docs.py` to generate documentation from the
Python files. It is a script that extracts MarkDown and AsciiDoc
docstrings from the codebase in the backend repository. It also supports
graphviz diagrams using kroki.io It generates a set of HTML files that
are published as GitHub pages as part of the CI workflow.

The outputs is lives at <https://ooni.github.io/backend/>

At the time of writing it is superseded by the present document.


## Mypy
The configuration lives at:

    .github/workflows/mypy.yml

It runs mypy to check Python typing and makes the CI run fail if errors
are detected.


## Fastpath test
The configuration lives at:

    .github/workflows/test_fastpath.yaml

Runs the following unit and functional tests against the fastpath:

    fastpath/tests/test_functional.py
    fastpath/tests/test_functional_nodb.py
    fastpath/tests/test_unit.py

These are not end-to-end tests for the backend e.g. the API is not being
tested.

The CI action creates a dedicated container and installs the required
dependencies as Debian packages. It provides a code coverage report in
HTML format.


## API end-to-end test
The configuration lives at:

    .github/workflows/test_new_api.yml

This CI action performs the most comprehensive test of the backed. In
sequence:

 * Creates a dedicated container and installs the required dependencies
    as Debian packages.

 * It starts a ClickHouse container

 * Creates the required database tables, see
    <https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/tests/integ/clickhouse_1_schema.sql>

 * Initializes the database tables with test data, see
    <https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/api/tests/integ/clickhouse_2_fixtures.sql>

 * Runs the fastpath against data from a bucket on S3 in order to
    populate the `fastpath` table with real data. The measurements come
    from both:

    -   A fixed time range in the past

    -   Recent data

 * It runs unit, functional and integration tests against the API.
    Integration tests require the API to run queries against multiple
    tables in ClickHouse.

> **important**
> the presence of recent data makes the content of the `fastpath` database
> table non deterministic between CI runs. This is necessary to perform
> integration tests against API entry points that need fresh data. To
> prevent flaky CI runs write integration tests with to handle variable
> data. E.g. use broad thresholds like asserting that the number of
> measurements collected in a day is more than 1.

> **note**
> prefer unit tests where possible

> **note**
> To implement precise functional tests, mock out the database.


# Incident response

## On-call preparation
Review [Alerting](#alerting)&thinsp;💡 and check
[Grafana dashboards](#grafana-dashboards)&thinsp;💡

On Android devices the following apps can be used:

 * [Slack](#slack)&thinsp;🔧 app with audible notifications from the
    #ooni-bots channel

 * [Grafana](#grafana)&thinsp;🔧 viewer
    <https://play.google.com/store/apps/details?id=it.ksol.grafanaview>

## Tiers and severities

When designing architecture of backend components or handling incidents it can be useful to have
defined severities and tiers.

A set of guidelines are described at <https://google.github.io/building-secure-and-reliable-systems/raw/ch16.html#establish_severity_and_priority_models>
This section presets a simplified approach to prioritizing incident response.

In this case there is no distinction between severity and priority. Impact and response time are connected.

Incidents and alarms from monitoring can be classified by severity levels based on their impact:

 - 1: Serious security breach or data loss; serious loss of privacy impacting users or team members; legal risks.
 - 2: Downtime impacting service usability for a significant fraction of users; Serious security vulnerability.
      Examples: probes being unable to submit measurements
 - 3: Downtime or poor performance impacting secondary services; anything that can cause a level 2 event if not addressed within 24h; outages of monitoring infrastructure
 - 4: Every other event that requires attention within 7 days

Based on the set of severities, components can be classified in tier as follows:

 - tier 1: Anything that can cause a severity 1 (or less severe) event.
 - tier 2: Anything that can cause a severity 2 (or less severe) event but not a severity 1.
 - tier 3: Anything that can cause a severity 3 (or less severe) event but not a severity 1 or 2.
 - ...and so on

### Relations and dependencies between services

Tiers are useful during design and deployment as a way to minimize risk of outages and avoid unexpected cascading failures.

Having a low tier value should not be treated as a sign of "importance" for a component, but a liability.

Pre-production deployment stages (e.g. testbed) have tier level >= 5

In this context a component can be a service as a whole, or a running process (daemon), a host, a hardware device, etc.
A component can contain other components.

A component "A" is said to "hard depend" on another component "B" if an outage of B triggers an outage of A.

It can also "soft depend" on another component if an outage of the latter triggers only a failure of a subsystem, or an ancillary feature or a reasonably short downtime.

Regardless of tiers, components at a higher stage, (e.g. production) cannot depend and/or receive data from lower stages. The opposite is acceptable.

Components can only hard-depend on other components at the same tier or with lower values.
E.g. a Tier 2 component can depend on a Tier 1 but not the other way around.
If it happens, the Tier 2 component should be immediatly re-classified as Tier 1 and treated accordingly (see below).

E.g. anything that handles real-time failover for a service should be treated at the same tier (or lower value) as the service.

Redundant components follow a special rule. For example, the "test helper" service provided to the probes, as a whole, should be considered tier 2 at least,
as it can impact all probes preventing them from running tests succesfully.
Yet, test helper processes and VMs can be considered tier 3 or even 4 if they sit behind a load balancer that can move traffic away from a failing host reliably
and with no significant downtime.

Example: An active/standby database pair provides a tier 2 service. An automatic failover tool is triggered by a simple monitoring script.
Both have to be labeled tier 2.


### Handling incidents

Depending on the severity of an event a different workflow can be followed.

An example of incident management workflow can be:

| Severity | Response time | Requires conference call | Requires call leader | Requires postmortem | Sterile |
| -------- | ------- | ------ | -------- | ------- | ------ |
| 1 | 2h | Yes | Yes | Yes | Yes |
| 2 | 8h | Yes | No | Yes | Yes |
| 3 | 24h | No | No | No | Yes |
| 4 | 7d | No | No | No | No |

The term "sterile" is named after <https://en.wikipedia.org/wiki/Sterile_flight_deck_rule> - during the investigation the only priority should be to solve the issue at hand.
Other investigations, discussions, meetings should be postponed.

When in doubt around the severity of an event, always err on the safe side.


### Regular operations

Based on the tier of a component, development and operation can follow different rules.

An example of incident management workflow can be:

| Tier | Require architecture review | Require code review | Require 3rd party security review | Require Change Management |
| -------- | ------- | ------ | -------- | ------- |
| 1 | Yes | Yes | Yes | Yes |
| 2 | Yes | Yes | No | No |
| 3 | No | Yes | No | No |
| 4 | No | No | No | No |

"Change Management" refers to planning operational changes in advance and having team members review the change to be deployed in advance.

E.g. scheduling a meeting to perform a probe release, have 2 people reviewing the metrics before and after the change.


## Redundant notifications
If needed, a secondary channel for alert notification can be set up
using <https://ntfy.sh/>

Ntfy can host a push notification topic for free.

For example <https://ntfy.sh/ooni-7aGhaS> is currently being used to
notify the outcome of CI runs from
<https://github.com/ooni/backend/blob/0ec9fba0eb9c4c440dcb7456f2aab529561104ae/.github/workflows/test_new_api.yml>

An Android app is available:
<https://f-droid.org/en/packages/io.heckel.ntfy/>

[Grafana](#grafana)&thinsp;🔧 can be configured to send alerts to ntfy.sh
using a webhook.


## Test helper rotation runbook
This runbook provides hints to troubleshoot the rotation of test
helpers. In this scenario test helpers are not being rotated as expected
and their TLS certificates might be at risk of expiring.

Steps:

1.  Review [Test helpers](#comp:test_helpers), [Test helper rotation](#comp:test_helper_rotation) and [Test helpers notebook](#test-helpers-notebook)&thinsp;📔

2.  Review the charts on [Test helpers dashboard](#test-helpers-dashboard)&thinsp;📊.
    Look at different timespans:

    a.  The uptime of the test helpers should be staggered by a week
        depending on [Test helper rotation](#test-helper-rotation)&thinsp;⚙.

3.  A summary of the live and last rotated test helper can be obtained
    with:

```sql
SELECT rdn, dns_zone, name, region, draining_at FROM test_helper_instances ORDER BY name DESC LIMIT 8
```

4.  The rotation tool can be started manually. It will always pick the
    oldest host for rotation. ⚠️ Due to the propagation time of changes
    in the DNS rotating many test helpers too quickly can impact the
    probes.

    a.  Log on [backend-fsn.ooni.org](#backend-fsn.ooni.org)&thinsp;🖥

    b.  Check the last run using
        `sudo systemctl status ooni-rotation.timer`

    c.  Review the logs using `sudo journalctl -u ooni-rotation`

    d.  Run `sudo systemctl restart ooni-rotation` and monitor the logs.

5.  Review the charts on [Test helpers dashboard](#test-helpers-dashboard)&thinsp;📊
    during and after the rotation.


## Test helpers failure runbook
This runbook presents a scenario where a test helper is causing probes
to fail their tests sporadically. It describes how to identify the
affected host and mitigate the issue but can also be used to investigate
other issues affecting the test helpers.

It has been chosen because such kind of incidents can impact the quality
of measurements and can be relatively difficult to troubleshoot.

For investigating glitches in the
[test helper rotation](#test-helper-rotation)&thinsp;⚙ see
[test helper rotation runbook](#test-helper-rotation-runbook)&thinsp;📒.

In this scenario either an alert has been sent to the
[#ooni-bots](#topic:oonibots) [Slack](#slack)&thinsp;🔧 channel by
the [test helper failure rate notebook](#test-helper-failure-rate-notebook)&thinsp;📔 or something
else caused the investigation.
See [Alerting](#alerting)&thinsp;💡 for details.

Steps:

1.  Review [Test helpers](#test-helpers)&thinsp;⚙

2.  Review the charts on [Test helpers dashboard](#test-helpers-dashboard)&thinsp;📊.
    Look at different timespans:

    a.  The uptime of the test helpers should be staggered by a week
        depending on [Test helper rotation](#test-helper-rotation)&thinsp;⚙.

    b.  The in-flight requests and requests per second should be
        consistent across hosts, except for `0.th.ooni.org`. See
        [Test helpers list](#test-helpers-list)&thinsp;🐝 for details.

    c.  Review CPU load, memory usage and run duration percentiles.

3.  Review [Test helper failure rate notebook](#test-helper-failure-rate-notebook)&thinsp;📔

4.  For more detailed investigation there is also a [test helper notebook](https://jupyter.ooni.org/notebooks/notebooks/2023%20%5Bfederico%5D%20test%20helper%20metadata%20in%20fastpath.ipynb)

5.  Log on the hosts using
    `ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -Snone root@0.th.ooni.org`

6.  Run `journalctl --since '1 hour ago'` or review logs using the query
    below.

7.  Run `top`, `strace`, `tcpdump` as needed.

8.  The rotation tool can be started at any time to rotate away failing
    test helpers. The rotation script will always pick the oldest host
    for rotation. ⚠️ Due to the propagation time of changes in the DNS
    rotating many test helpers too quickly can impact the probes.

    a.  Log on [backend-fsn.ooni.org](#backend-fsn.ooni.org)&thinsp;🖥

    b.  Check the last run using
        `sudo systemctl status ooni-rotation.timer`

    c.  Review the logs using `sudo journalctl -u ooni-rotation`

    d.  Run `sudo systemctl restart ooni-rotation` and monitor the logs.

9.  Review the charts on [Test helpers dashboard](#test-helpers-dashboard)&thinsp;📊
    during and after the rotation.

10. Summarize traffic hitting a test helper using the following commands:

    Top 10 miniooni probe IP addresses (Warning: this is sensitive data)

    `tail -n 100000 /var/log/nginx/access.log | grep miniooni | cut -d' ' -f1|sort|uniq -c|sort -nr|head`

    Similar, with anonimized IP addresses:

    `grep POST /var/log/nginx/access.log | grep miniooni | cut -d'.' -f1-3 | head -n 10000 |sort|uniq -c|sort -nr|head`

    Number of requests from miniooni probe in 10-minutes buckets:

    `grep POST /var/log/nginx/access.log | grep miniooni | cut -d' ' -f4 | cut -c1-17 | uniq -c`

    Number of requests from miniooni probe in 1-minute buckets:

    `grep POST /var/log/nginx/access.log | grep miniooni | cut -d' ' -f4 | cut -c1-18 | uniq -c`

    Number of requests grouped by hour, cache HIT/MISS/etc, software name and version

    `head -n 100000 /var/log/nginx/access.log | awk '{print $4, $6, $13}' | cut -c1-15,22- | sort | uniq -c | sort -n`

To extract data from the centralized log database
on [monitoring.ooni.org](#monitoring.ooni.org)&thinsp;🖥 you can use:

``` sql
SELECT message FROM logs
WHERE SYSLOG_IDENTIFIER = 'oohelperd'
ORDER BY __REALTIME_TIMESTAMP DESC
LIMIT 10
```

> **note**
> The table is indexed by `__REALTIME_TIMESTAMP`. Limiting the range by time can significantly increase query performance.


See [Selecting test helper for rotation](#selecting-test-helper-for-rotation)&thinsp;🐞


## Measurement drop tutorial
This tutorial provides examples on how to investigate a drop in measurements.
It is based on an incident where a drop in measurement was detected and the cause was not immediately clear.

It is not meant to be a step-by-step runbook but rather give hints on what data to look for, how to generate charts and identify the root cause of an incident.

A dedicated issue can be used to track the incident and the investigation effort and provide visibility:
https://github.com/ooni/sysadmin/blob/master/.github/ISSUE_TEMPLATE/incident.md
The issue can be filed during or after the incident depending on urgency.

Some of the examples below come from
https://jupyter.ooni.org/notebooks/notebooks/android_probe_release_msm_drop_investigation.ipynb
During an investigation it can be good to create a dedicated Jupyter notebook.

We started with reviewing:

 * <https://jupyter.ooni.org/view/notebooks/jupycron/autorun_android_probe_release.html>
   No issues detected as the charts show a short timespan.
 * The charts on [Test helpers dashboard](#test-helpers-dashboard)&thinsp;📊.
   No issues detected here.
 * The [API and fastpath](#api-and-fastpath)&thinsp;📊 dashboard.
   No issues detected here.
 * The [Long term measurements prediction notebook](#long-term-measurements-prediction-notebook)&thinsp;📔
   The decrease was clearly showing.

Everything looked OK in terms of backend health. We then generated the following charts.

The chunks of Python code below are meant to be run in
[Jupyter Notebook](#jupyter-notebook)&thinsp;🔧 and are mostly "self-contained".
To be used you only need to import the
[Ooniutils microlibrary](#ooniutils-microlibrary)&thinsp;💡:

``` python
%run ooniutils.ipynb
```

The "t" label is commonly used on existing notebooks to refer to hour/day/week time slices.

We want to plot how many measurements we are receiving from Ooniprobe Android in unattended runs, grouped by day and by `software_version`.

The last line generates an area chart using Altair. Notice that the `x` and `y` and `color` parameters match the 3 columns extracted by the `SELECT`.

The `GROUP BY` is performed on 2 of those 3 columns, while `COUNT(*)` is counting how many measurements exist in each t/software_version "bucket".

The output of the SQL query is just a dataframe with 3 columns. There is no need to pivot or reindex it as Altair does the data transformation required.

> **note**
> Altair refuses to process dataframes with more than 5000 rows.

``` python
x = click_query("""
    SELECT
      toStartOfDay(toStartOfWeek(measurement_start_time)) AS t,
      software_version,
      COUNT(*) AS msm_cnt
    FROM fastpath
    WHERE measurement_start_time > today() - interval 3 month
    AND measurement_start_time < today()
    AND software_name = 'ooniprobe-android-unattended'
    GROUP BY t, software_version
""")
alt.Chart(x).mark_area().encode(x='t', y='msm_cnt', color='software_version').properties(width=1000, height=200, title="Android unattended msm cnt")
```

The generated chart was:

![chart](images/msm_drop_investigation_1.png)

From the chart we concluded that the overall number of measurements have been decreasing since the release of a new version.
We also re-ran the plot by filtering on other `software_name` values and saw no other type of probe was affected.

> **note**
> Due to a limitation in Altair, when grouping time by week use
> `toStartOfDay(toStartOfWeek(measurement_start_time)) AS t`

Then we wanted to measure how many measurements are being collected during each `web_connectivity` test run.
This is to understand if probes are testing less measurements in each run.

The following Python snippet uses nested SQL queries. The inner query groups measurements by time, `software_version` and `report_id`,
and counts how many measurements are related to each `report_id`.
The outer query "ignores" the `report_id` value and `quantile()` is used to extract the 50 percentile of `msm_cnt`.

> **note**
> The use of double `%%` in `LIKE` is required to escape the `%` wildcard. The wildcard is used to match any amount of characters.

``` python
x = click_query("""
    SELECT
        t,
        quantile(0.5)(msm_cnt) AS msm_cnt_p50,
        software_version
    FROM (
        SELECT
            toStartOfDay(toStartOfWeek(measurement_start_time)) AS t,
            software_version,
            report_id,
            COUNT(*) AS msm_cnt
        FROM fastpath
        WHERE measurement_start_time > today() - interval 3 month
        AND test_name = 'web_connectivity'
        AND measurement_start_time < today()
        AND software_name = 'ooniprobe-android-unattended'
        AND software_version LIKE '3.8%%'
        GROUP BY t, software_version, report_id
    ) GROUP BY t, software_version
""")
alt.Chart(x).mark_line().encode(x='t', y='msm_cnt_p50', color='software_version').properties(width=1000, height=200, title="Android unattended msmt count per report")
```

We also compared different version groups and different `software_name`.
The output shows that indeed the number of measurements for each run is significantly lower for the newly released versions.

![chart](images/msm_drop_investigation_4.png)

To update the previous Python snippet to group measurements by a different field, change `software_version` into the new column name.
For example use `probe_cc` to show a chart with a breakdown by probe country name. You should change `software_version` once in each SELECT part,
then in the last two `GROUP BY`, and finally in the `color` line at the bottom.

We did such change to confirm that all countries were impacted in the same way. (The output is not included here as not remarkable)

Also, `mark_line` on the bottom line is used to create line charts. Switch it to `mark_area` to generate *stacked* area charts.
See the previous two charts as examples.

We implemented a change to the API to improve logging the list of tests returned at check-in: <https://github.com/ooni/backend/pull/781>
and reviewed monitored the logs using `sudo journalctl -f -u ooni-api`.

The output showed that the API is very often returning 100 URLs to probes.

We then ran a similar query to extract the test duration time by calculating
`MAX(measurement_start_time) - MIN(measurement_start_time) AS delta` for each `report_id` value:

``` python
x = click_query("""
    SELECT t, quantile(0.5)(delta) AS deltaq, software_version
    FROM (
        SELECT
            toStartOfDay(toStartOfWeek(measurement_start_time)) AS t,
            software_version,
            report_id,
            MAX(measurement_start_time) - MIN(measurement_start_time) AS delta
        FROM fastpath
        WHERE measurement_start_time > today() - interval 3 month
        AND test_name = 'web_connectivity'
        AND measurement_start_time < today()
        AND software_name = 'ooniprobe-android-unattended'
        AND software_version LIKE '3.8%%'
        GROUP BY t, software_version, report_id
    ) GROUP BY t, software_version
""")
alt.Chart(x).mark_line().encode(x='t', y='deltaq', color='software_version').properties(width=1000, height=200, title="Android unattended test run time")
```

![chart](images/msm_drop_investigation_2.png)

The chart showed that the tests are indeed running for a shorter amount of time.

> **note**
> Percentiles can be more meaningful then averages.
> To calculate quantiles in ClickHouse use `quantile(<fraction>)(<column_name>)`.

Example:

``` sql
quantile(0.1)(delta) AS deltaq10
```

Wondering if the slowdown was due to slower measurement execution or other issues, we also generated a table as follows.

> **note**
> Showing color bars allows to visually inspect tables more quickly. Setting the axis value to `0`, `1` or `None` helps readability:
> `y.style.bar(axis=None)`

Notice the `delta / msmcnt AS seconds_per_msm` calculation:

``` python
y = click_query("""
    SELECT
        quantile(0.1)(delta) AS deltaq10,
        quantile(0.3)(delta) AS deltaq30,
        quantile(0.5)(delta) AS deltaq50,
        quantile(0.7)(delta) AS deltaq70,
        quantile(0.9)(delta) AS deltaq90,

        quantile(0.5)(seconds_per_msm) AS seconds_per_msm_q50,
        quantile(0.5)(msmcnt) AS msmcnt_q50,

    software_version, software_name
    FROM (
        SELECT
            software_version, software_name,
            report_id,
            MAX(measurement_start_time) - MIN(measurement_start_time) AS delta,
            count(*) AS msmcnt,
            delta / msmcnt AS seconds_per_msm
        FROM fastpath
        WHERE measurement_start_time > today() - interval 3 month
        AND test_name = 'web_connectivity'
        AND measurement_start_time < today()
        AND software_name IN ['ooniprobe-android-unattended', 'ooniprobe-android']
        AND software_version LIKE '3.8%%'
        GROUP BY software_version, report_id, software_name
    ) GROUP BY software_version, software_name
    ORDER by software_version, software_name ASC
""")
y.style.bar(axis=None)
```

![chart](images/msm_drop_investigation_3.png)

In the table we looked at the `seconds_per_msm_q50` column: the median time for running each test did not change significantly.

To summarize:
 * The backend appears to deliver the same amount of URLs to the Probes as usual.
 * The time required to run each test is rougly the same.
 * Both the number of measurements per run and the run time decreased in the new releases.


## Weekly measurements review runbook
On a daily or weekly basis the following dashboards and Jupyter notebooks can be reviewed to detect unexpected patterns in measurements focusing on measurement drops, slowdowns or any potential issue affecting the backend infrastructure.

When browsing the dashboards expand the time range to one year in order to spot long term trends.
Also zoom in to the last month to spot small glitches that could otherwise go unnoticed.

Review the [API and fastpath](#api-and-fastpath)&thinsp;📊 dashboard for the production backend host[s] for measurement flow, CPU and memory load,
timings of various API calls, disk usage.

Review the [Incoming measurements notebook](#incoming-measurements-notebook)&thinsp;📔 for unexpected trends.

Quickly review the following dashboards for unexpected changes:

 * [Long term measurements prediction notebook](#long-term-measurements-prediction-notebook)&thinsp;📔
 * [Test helpers dashboard](#test-helpers-dashboard)&thinsp;📊
 * [Test helper failure rate notebook](#test-helper-failure-rate-notebook)&thinsp;📔
 * [Database backup dashboard](#database-backup-dashboard)&thinsp;📊
 * [GeoIP MMDB database dashboard](#geoip-mmdb-database-dashboard)&thinsp;📊
 * [GeoIP dashboard](#geoip-mmdb-database-dashboard)&thinsp;📊
 * [Fingerprint updater dashboard](#fingerprint-updater-dashboard)&thinsp;📊
 * [ASN metadata updater dashboard](#asn-metadata-updater-dashboard)&thinsp;📊

Also check <https://jupyter.ooni.org/view/notebooks/jupycron/summary.html> for glitches like notebooks not being run etc.


## Grafana backup runbook
This runbook describes how to back up dashboards and alarms in Grafana.
It does not include backing up datapoints stored in
[Prometheus](#prometheus)&thinsp;🔧.

The Grafana SQLite database can be dumped by running:

```bash
sqlite3 -line /var/lib/grafana/grafana.db '.dump' > grafana_dump.sql
```

Future implementation is tracked in:
[Implement Grafana dashboard and alarms backup](#implement-grafana-dashboard-and-alarms-backup)&thinsp;🐞


## Grafana editing
This runbook describes adding new dashboards, panels and alerts in
[Grafana](#grafana)&thinsp;🔧

To add a new dashboard use this
<https://grafana.ooni.org/dashboard/new?orgId=1>

To add a new panel to an existing dashboard load the dashboard and then
click the \"Add\" button on the top.

Many dashboards use variables. For example, on
<https://grafana.ooni.org/d/l-MQSGonk/api-and-fastpath-multihost?orgId=1>
the variables `$host` and `$avgspan` are set on the top left and used in
metrics like:

    avg_over_time(netdata_disk_backlog_milliseconds_average{instance="$host:19999"}[$avgspan])


### Managing Grafana alert rules
Alert rules can be listed at <https://grafana.ooni.org/alerting/list>

> **note**
> The list also shows which alerts are currently alarming, if any.

Click the arrow on the left to expand each alerting rule.

The list shows:

![editing_alerts](images/grafana_alerts_editing.png)

> **note**
> When creating alerts it can be useful to add full URLs linking to
> dashboards, runbooks etc.

To stop notifications create a \"silence\" either:

1.  by further expanding an alert rule (see below) and clicking the
    \"Silence\" button

2.  by inputting it in <https://grafana.ooni.org/alerting/silences>

Screenshot:

![adding_silence](images/grafana_alerts_silence.png)

Additionally, the \"Show state history\" button is useful especially
with flapping alerts.


## Geolocation script
The following script can be used to compare the geolocation reported by
the probes submitting measurements compared to the geolocation of the
`/24` subnet the probe is coming from. It is meant to be run on
[backend-fsn.ooni.org](#backend-fsn.ooni.org)&thinsp;🖥.

``` python
#!/usr/bin/env python3

from time import sleep

import systemd.journal
import geoip2.database  # type: ignore

asnfn = "/var/lib/ooniapi/asn.mmdb"
ccfn = "/var/lib/ooniapi/cc.mmdb"
geoip_asn_reader = geoip2.database.Reader(asnfn)
geoip_cc_reader = geoip2.database.Reader(ccfn)


def follow_journal():
    journal = systemd.journal.Reader()
    #journal.seek_tail()
    journal.get_previous()
    journal.add_match(_SYSTEMD_UNIT="nginx.service")
    while True:
        try:
            event = journal.wait(-1)
            if event == systemd.journal.APPEND:
                for entry in journal:
                    yield entry["MESSAGE"]
        except Exception as e:
            print(e)
            sleep(0.1)


def geolookup(ipaddr: str):
    cc = geoip_cc_reader.country(ipaddr).country.iso_code
    asn = geoip_asn_reader.asn(ipaddr).autonomous_system_number
    return cc, asn


def process(rawmsg):
    if ' "POST /report/' not in rawmsg:
        return
    msg = rawmsg.strip().split()
    ipaddr = msg[2]
    ipaddr2 = msg[3]
    path = msg[8][8:]
    tsamp, tn, probe_cc, probe_asn, collector, rand = path.split("_")
    geo_cc, geo_asn = geolookup(ipaddr)
    proxied = 0
    probe_type = rawmsg.rsplit('"', 2)[-2]
    if "," in probe_type:
        return
    if ipaddr2 != "0.0.0.0":
        proxied = 1
        # Probably CloudFront, use second ipaddr
        geo_cc, geo_asn = geolookup(ipaddr2)

    print(f"{probe_cc},{geo_cc},{probe_asn},{geo_asn},{proxied},{probe_type}")


def main():
    for msg in follow_journal():
        if msg is None:
            break
        try:
            process(msg)
        except Exception as e:
            print(e)
            sleep(0.1)


if __name__ == "__main__":
    main()
```


## Test list prioritization monitoring
The following script monitors prioritized test list for changes in URLs
for a set of countries. Outputs StatsS metrics.

> **note**
> The prioritization system has been modified to work on a granularity of
> probe_cc + probe_asn rather than whole countries.

Country-wise changes might be misleading. The script can be modified to
filter for a set of CCs+ASNs.

``` python
#!/usr/bin/env python3

from time import sleep
import urllib.request
import json

import statsd  # debdeps: python3-statsd

metrics = statsd.StatsClient("127.0.0.1", 8125, prefix="test-list-changes")

CCs = ["GE", "IT", "US"]
THRESH = 100


def peek(cc, listmap) -> None:
    url = f"https://api.ooni.io/api/v1/test-list/urls?country_code={cc}&debug=True"
    res = urllib.request.urlopen(url)
    j = json.load(res)
    top = j["results"][:THRESH]  # list of dicts
    top_urls = set(d["url"] for d in top)

    if cc in listmap:
        old = listmap[cc]
        changed = old.symmetric_difference(top_urls)
        tot_cnt = len(old.union(top_urls))
        changed_ratio = len(changed) / tot_cnt * 100
        metrics.gauge(f"-{cc}", changed_ratio)

    listmap[cc] = top_urls


def main() -> None:
    listmap = {}
    while True:
        for cc in CCs:
            try:
                peek(cc, listmap)
            except Exception as e:
                print(e)
            sleep(1)
        sleep(60 * 10)


if __name__ == "__main__":
    main()
```

## Recompressing postcans on S3
The following script can be used to compress .tar.gz files in the S3 data bucket.
It keeps a copy of the original files locally as a backup.
It terminates once a correctly compressed file is found.
Running the script on an AWS host close to the S3 bucket can significantly
speed up the process.

Tested with the packages:

  * python3-boto3  1.28.49+dfsg-1
  * python3-magic  2:0.4.27-2

Set the ACCESS_KEY and SECRET_KEY environment variables.
Update the PREFIX variable as needed.

```python
#!/usr/bin/env python3
from os import getenv, rename
from sys import exit
import boto3
import gzip
import magic

BUCKET_NAME = "ooni-data-eu-fra-test"
# BUCKET_NAME = "ooni-data-eu-fra"
PREFIX = "raw/2021"

def fetch_files():
    s3 = boto3.client(
        "s3",
        aws_access_key_id=getenv("ACCESS_KEY"),
        aws_secret_access_key=getenv("SECRET_KEY"),
    )
    cont_token = None
    while True:
        kw = {} if cont_token is None else dict(ContinuationToken=cont_token)
        r = s3.list_objects_v2(Bucket=BUCKET_NAME, Prefix=PREFIX, **kw)
        cont_token = r.get("NextContinuationToken", None)
        for i in r.get("Contents", []):
            k = i["Key"]
            if k.endswith(".tar.gz"):
                fn = k.rsplit("/", 1)[-1]
                s3.download_file(BUCKET_NAME, k, fn)
                yield k, fn
        if cont_token is None:
            return

def main():
    s3res = session = boto3.Session(
        aws_access_key_id=getenv("ACCESS_KEY"),
        aws_secret_access_key=getenv("SECRET_KEY"),
    ).resource("s3")
    for s3key, fn in fetch_files():
        ft = magic.from_file(fn)
        if "tar archive" not in ft:
            print(f"found {ft} at {s3key}")
            # continue   # simply ignore already compressed files
            exit()      # stop when compressed files are found
        tarfn = fn[:-3]
        rename(fn, tarfn)  # keep the local file as a backup
        with open(tarfn, "rb") as f:
            inp = f.read()
            comp = gzip.compress(inp, compresslevel=9)
        ratio = len(inp) / len(comp)
        del inp
        print(f"uploading {s3key}   compression ratio {ratio}")
        obj = s3res.Object(BUCKET_NAME, s3key)
        obj.put(Body=comp)
        del comp

main()
```

# Github issues

## Selecting test helper for rotation
See <https://github.com/ooni/backend/issues/721>


## Document Tor targets
See <https://github.com/ooni/backend/issues/761>


## Disable unnecessary ClickHouse system tables
See <https://github.com/ooni/backend/issues/779>


## Feed fastpath from JSONL
See <https://github.com/ooni/backend/issues/778>


## Implement Grafana dashboard and alarms backup
See <https://github.com/ooni/backend/issues/770>


