# data-lineage-playground
Data lineage playground using OpenLineage

Deploy Marquez
```
git clone https://github.com/MarquezProject/marquez.git && cd marquez

./docker/up.sh

# If virtual memory areas is too low
sudo sysctl -w vm.max_map_count=262144
```

Start a run

```
curl -X POST http://localhost:5000/api/v1/lineage \
  -i -H 'Content-Type: application/json' \
  -d '{
        "eventType": "START",
        "eventTime": "2020-12-28T19:52:00.001+10:00",
        "run": {
          "runId": "0176a8c2-fe01-7439-87e6-56a1a1b4029f"
        },
        "job": {
          "namespace": "my-namespace",
          "name": "my-job"
        },
        "inputs": [{
          "namespace": "my-namespace",
          "name": "my-input"
        }],  
        "producer": "https://github.com/OpenLineage/OpenLineage/blob/v1-0-0/client",
        "schemaURL": "https://openlineage.io/spec/1-0-5/OpenLineage.json#/definitions/RunEvent"
      }'
```

Complete a run

```
curl -X POST http://localhost:5000/api/v1/lineage \
  -i -H 'Content-Type: application/json' \
  -d '{
        "eventType": "COMPLETE",
        "eventTime": "2020-12-28T20:52:00.001+10:00",
        "run": {
          "runId": "0176a8c2-fe01-7439-87e6-56a1a1b4029f"
        },
        "job": {
          "namespace": "my-namespace",
          "name": "my-job"
        },
        "outputs": [{
          "namespace": "my-namespace",
          "name": "my-output",
          "facets": {
            "schema": {
              "_producer": "https://github.com/OpenLineage/OpenLineage/blob/v1-0-0/client",
              "_schemaURL": "https://github.com/OpenLineage/OpenLineage/blob/v1-0-0/spec/OpenLineage.json#/definitions/SchemaDatasetFacet",
              "fields": [
                { "name": "a", "type": "VARCHAR"},
                { "name": "b", "type": "VARCHAR"}
              ]
            }
          }
        }],     
        "producer": "https://github.com/OpenLineage/OpenLineage/blob/v1-0-0/client",
        "schemaURL": "https://openlineage.io/spec/1-0-5/OpenLineage.json#/definitions/RunEvent"
      }'
```

Example of script to send a full pipeline data lineage to Marquez

```
#!/bin/bash

ENDPOINT="http://localhost:5000/api/v1/lineage"

echo "=== Job 1 ==="
curl -X POST $ENDPOINT \
  -H "Content-Type: application/json" \
  -d '{
    "eventType": "COMPLETE",
    "eventTime": "2026-04-01T14:00:00Z",
    "producer": "nifi://my-nifi/flow",
    "run": { "runId": "run-nifi-job1-001" },
    "job": { "namespace": "nifi", "name": "extract-oracle-to-avro" },
    "inputs": [
      {
        "namespace": "oracle://source-oracle",
        "name": "schema.table_oracle",
        "facets": {
          "schema": {
            "_producer": "manual",
            "_schemaURL": "http://openlineage.io/schema/1-0-0",
            "fields": [{ "name": "nom_client", "type": "STRING" }]
          }
        }
      }
    ],
    "outputs": [
      {
        "namespace": "gs://cloud-storage",
        "name": "extraction/table_oracle.avro",
        "facets": {
          "schema": {
            "_producer": "manual",
            "_schemaURL": "http://openlineage.io/schema/1-0-0",
            "fields": [{ "name": "nom_client", "type": "STRING" }]
          }
        }
      }
    ]
  }'

echo
echo "=== Job 2 ==="
curl -X POST $ENDPOINT \
  -H "Content-Type: application/json" \
  -d '{
    "eventType": "COMPLETE",
    "eventTime": "2026-04-01T14:20:00Z",
    "producer": "airflow://gcp/ingestion-pipeline",
    "run": { "runId": "run-airflow-job2-001" },
    "job": {
      "namespace": "airflow",
      "name": "avro-to-bigquery-ingestion"
    },
    "inputs": [
      {
        "namespace": "gs://cloud-storage",
        "name": "extraction/table_oracle.avro",
        "facets": {
          "schema": {
            "_producer": "manual",
            "_schemaURL": "http://openlineage.io/schema/1-0-0",
            "fields": [{ "name": "nom_client", "type": "STRING" }]
          }
        }
      }
    ],
    "outputs": [
      {
        "namespace": "bigquery",
        "name": "dataset.table",
        "facets": {
          "schema": {
            "_producer": "manual",
            "_schemaURL": "http://openlineage.io/schema/1-0-0",
            "fields": [{ "name": "nom_client", "type": "STRING" }]
          },
          "columnLineage": {
            "_producer": "manual",
            "_schemaURL": "http://openlineage.io/columnLineage/1-0-0",
            "fields": {
              "nom_client": {
                "inputFields": [
                  {
                    "namespace": "gs://cloud-storage",
                    "name": "extraction/table_oracle.avro",
                    "field": "nom_client"
                  }
                ],
                "transformationDescription": "identity"
              }
            }
          }
        }
      }
    ]
  }'

echo
echo "=== Job 3 ==="
curl -X POST $ENDPOINT \
  -H "Content-Type: application/json" \
  -d '{
    "eventType": "COMPLETE",
    "eventTime": "2026-04-01T14:40:00Z",
    "producer": "airflow://gcp/transformation-pipeline",
    "run": { "runId": "run-airflow-job3-001" },
    "job": {
      "namespace": "airflow",
      "name": "rename-client-column"
    },
    "inputs": [
      {
        "namespace": "bigquery",
        "name": "dataset.table",
        "facets": {
          "schema": {
            "_producer": "manual",
            "_schemaURL": "http://openlineage.io/schema/1-0-0",
            "fields": [{ "name": "nom_client", "type": "STRING" }]
          }
        }
      }
    ],
    "outputs": [
      {
        "namespace": "bigquery",
        "name": "dataset.dest",
        "facets": {
          "schema": {
            "_producer": "manual",
            "_schemaURL": "http://openlineage.io/schema/1-0-0",
            "fields": [{ "name": "client_name", "type": "STRING" }]
          },
          "columnLineage": {
            "_producer": "manual",
            "_schemaURL": "http://openlineage.io/columnLineage/1-0-0",
            "fields": {
              "client_name": {
                "inputFields": [
                  {
                    "namespace": "bigquery",
                    "name": "dataset.table",
                    "field": "nom_client"
                  }
                ],
                "transformationDescription": "rename"
              }
            }
          }
        }
      }
    ]
  }'

echo
echo "=== DONE ==="
```
