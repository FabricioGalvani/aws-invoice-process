# PyIceberg Alternative Write Path

> **MCP Validated:** 2026-05-12
> **Purpose**: Writing to Iceberg directly from Lambda via pyiceberg — no Athena query needed for writes
> **Confidence**: 0.95
> **Sources**: [AWS blog: pyiceberg + Lambda + Glue REST](https://aws.amazon.com/blogs/big-data/accelerate-lightweight-analytics-using-pyiceberg-with-aws-lambda-and-an-aws-glue-iceberg-rest-endpoint/), [pyiceberg GlueCatalog reference](https://py.iceberg.apache.org/reference/pyiceberg/catalog/glue/), [AWS prescriptive guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/apache-iceberg-on-aws/iceberg-pyiceberg.html)

## Overview

The `warehouse-writer` Lambda has two write paths for inserting extracted invoice data into the Iceberg table:

| Path | Mechanism | When to Use |
|------|-----------|-------------|
| **Athena SQL** (`start_query_execution` + polling) | Async SQL `INSERT INTO` via Athena | Default; simpler; async-friendly |
| **pyiceberg direct** | Python SDK writes directly to S3 + updates Glue catalog | Better for sub-second writes; avoids Athena async complexity |

The pyiceberg path is recommended when:
- Lambda payloads are small (1-10 rows per invocation) — avoids Athena cold start overhead
- Synchronous write confirmation is needed within the Lambda execution
- Athena service quotas (concurrent queries) are a concern at scale

## How pyiceberg Interacts with Glue

pyiceberg calls Glue APIs directly:
1. `glue:GetTable` → read current `metadata_location`
2. Writes Parquet data file(s) to S3 via `s3:PutObject`
3. Writes new metadata JSON + manifest files to S3
4. `glue:UpdateTable` → atomically sets new `metadata_location` (optimistic lock via Glue version ID)

No Athena service involved. Tables written by pyiceberg are immediately queryable by Athena because they share the same Glue catalog.

## Catalog Configuration Options

### Option A: GlueCatalog (direct Glue API)

```python
from pyiceberg.catalog.glue import GlueCatalog

catalog = GlueCatalog(
    "invoices",
    **{
        "glue.region": "us-east-1",
        # In Lambda: credentials come from execution role automatically
        # No explicit key/secret needed when running inside Lambda
    }
)
```

### Option B: Glue REST Endpoint (recommended for pyiceberg >= 0.7.0)

```python
from pyiceberg.catalog import load_catalog

catalog = load_catalog(
    "invoices",
    **{
        "type": "rest",
        "uri": "https://glue.us-east-1.amazonaws.com/iceberg",
        "s3.region": "us-east-1",
        "rest.sigv4-enabled": "true",
        "rest.signing-name": "glue",
        "rest.signing-region": "us-east-1",
    }
)
```

The REST endpoint supports the same Glue catalog but uses the standard Iceberg REST protocol. Credentials are inferred from the Lambda execution role via boto3 default credential chain.

## Concurrency and Retry

pyiceberg uses **optimistic concurrency control** — if two Lambda invocations try to commit simultaneously, one will fail with a `CommitFailedException`. The SQS trigger for `warehouse-writer` should set:

```
MaximumConcurrency = 1  # on SQS event source mapping
```

This serializes invocations, eliminating commit conflicts. At 3,500 invoices/month the throughput is trivially within a single-concurrency Lambda.

Alternative: implement exponential backoff retry on `CommitFailedException` (3 retries, 500ms base delay).

## IAM Requirements

Same as Athena path plus direct S3 writes:

```json
{
  "Action": [
    "glue:GetTable",
    "glue:UpdateTable",
    "glue:GetDatabase",
    "s3:PutObject",
    "s3:GetObject",
    "s3:ListBucket",
    "s3:DeleteObject"
  ]
}
```

No `athena:*` permissions needed for pyiceberg-only write path.

## Related

- [pyiceberg-write pattern](../patterns/pyiceberg-write.md)
- [iceberg-on-athena](iceberg-on-athena.md)
- [lambda-insert-batch pattern](../patterns/lambda-insert-batch.md)
- [athena-iam-policy spec](../specs/athena-iam-policy.json)
