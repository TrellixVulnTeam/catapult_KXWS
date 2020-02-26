# Chromeperf Datastore → BigQuery export pipeline.

This directory contains Apache Beam pipelines for exporting Anomaly and Row
entities from chromeperf's Datastore into BigQuery tables.

This README briefly describes how to run and deploy those pipelines.

## Set up a development environment

Follow the instructions at
https://cloud.google.com/dataflow/docs/quickstarts/quickstart-python#set-up-your-environment
to get a virtualenv with the Beam SDK installed.

## Deployment

### Upload job templates

In the virtualenv with the Beam SDK installed:

```
$ cd dashboard/
```

```
$ PYTHONPATH=$PYTHONPATH:"$(pwd)/bq_export" python bq_export/export_rows.py \
  --service_account_email=bigquery-exporter@chromeperf.iam.gserviceaccount.com \
  --runner=DataflowRunner
  --region=us-central1 \
  --experiments=use_beam_bq_sink  \
  --setup_file=bq_export/setup.py \
  --staging_location=gs://chromeperf-dataflow/staging \
  --template_location=gs://chromeperf-dataflow/templates/export_rows
```

```
$ PYTHONPATH=$PYTHONPATH:"$(pwd)/bq_export" python \
  bq_export/export_anomalies.py \
  --service_account_email=bigquery-exporter@chromeperf.iam.gserviceaccount.com \
  --runner=DataflowRunner
  --region=us-central1 \
  --experiments=use_beam_bq_sink  \
  --setup_file=bq_export/setup.py \
  --staging_location=gs://chromeperf-dataflow/staging \
  --template_location=gs://chromeperf-dataflow/templates/export_anomalies
```

There are Cloud Scheduler jobs configured to run
`gs://chromeperf-dataflow/templates/export_anomalies` and
`gs://chromeperf-dataflow/templates/export_rows` once every day, so updating
those job templates is all that is required to update those daily runs.  See the
Cloud Scheduler jobs at
https://console.cloud.google.com/cloudscheduler?project=chromeperf.

## Executing jobs

### Manually trigger jobs from templates

You can execute one-off jobs with the `gcloud` tool.  For example:

```
$ gcloud dataflow jobs run export-anomalies-example-job \
  --service-account-email=bigquery-exporter@chromeperf.iam.gserviceaccount.com \
  --gcs-location=gs://chromeperf-dataflow/templates/export_anomalies \
  --disable-public-ips \
  --max-workers=10 \
  --region=us-central1 \
  --staging-location=gs://chromeperf-dataflow-temp/export_anomalies \
  --parameters=experiments=shuffle_mode=service,subnetwork=regions/us-central1/subnetworks/dashboard-batch
```

To execute a manual backfill, specify the `end_date` and/or `num_days`
parameters.  For example, this will regenerate the anomalies for December 2019:

```
$ gcloud dataflow jobs run export-anomalies-backfill-example \
  --service-account-email=bigquery-exporter@chromeperf.iam.gserviceaccount.com \
  --gcs-location=gs://chromeperf-dataflow/templates/export_anomalies \
  --disable-public-ips \
  --max-workers=10 \
  --region=us-central1 \
  --staging-location=gs://chromeperf-dataflow-temp/export_anomalies \
  --parameters=experiments=shuffle_mode=service,subnetwork=regions/us-central1/subnetworks/dashboard-batch,end_date=20191231,num_days=31
```

**Tips:**

* When testing changes to the pipelines add `table_suffix=_test` to the
  parameters to write to `anomalies_test` or `rows_test` tables rather than the
  real tables.

* You can of also use the REST API instead of `gcloud`.  See [Using the REST
  API](https://cloud.google.com/dataflow/docs/guides/templates/running-templates#using-the-rest-api)
  in the Cloud Dataflow docs.  See the Cloud Scheduler jobs for some examples.

### Execute a job without a template

In the virtualenv with the Beam SDK you can run a job without creating a
template by specifying all the job execution parameters and omitting the
template parameters (i.e. omit `--staging_location` and `--template_location`).

For example:

```
$ PYTHONPATH=$PYTHONPATH:"$(pwd)/bq_export" python bq_export/export_rows.py \
  --service_account_email=bigquery-exporter@chromeperf.iam.gserviceaccount.com \
  --runner=DataflowRunner \
  --region=us-central1 \
  --temp_location=gs://chromeperf-dataflow-temp/export_rows_temp \
  --experiments=use_beam_bq_sink \
  --experiments=shuffle_mode=service \
  --autoscaling_algorithm=NONE \
  --num_workers=70 \
  --setup_file=bq_export/setup.py \
  --no_use_public_ips \
  --subnetwork=regions/us-central1/subnetworks/dashboard-batch
```