# Glue Data Catalog

> **MCP Validated:** 2026-05-12
> **Purpose**: Glue Data Catalog as the metadata store for Athena+Iceberg — databases, tables, registration, IAM
> **Confidence**: 0.95
> **Sources**: [AWS Athena Iceberg tables](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg-creating-tables.html), [AWS prescriptive guidance pyiceberg](https://docs.aws.amazon.com/prescriptive-guidance/latest/apache-iceberg-on-aws/iceberg-pyiceberg.html)

## Overview

AWS Glue Data Catalog is the **only supported catalog backend** for Athena+Iceberg tables. It stores:
- Database definitions (namespaces)
- Table metadata: schema, partition spec, S3 location, `metadata_location` pointer to the current Iceberg metadata JSON
- Table properties (TBLPROPERTIES key-value pairs)

The Glue catalog is per-AWS-account and per-region. The default catalog in Athena SQL is `AwsDataCatalog`.

## Hierarchy

```
AwsDataCatalog (Glue catalog — 1 per account/region)
└── database (e.g. invoices_dev, invoices_prod)
    └── table  (e.g. extracted_invoices)
        ├── columns + data types
        ├── partition spec
        ├── S3 data location
        └── metadata_location → s3://…/metadata/v3.metadata.json
```

## Creating a Database

```sql
-- Athena SQL (also creates in Glue automatically)
CREATE DATABASE IF NOT EXISTS invoices_dev
COMMENT 'Invoice pipeline warehouse — dev'
LOCATION 's3://invoices-warehouse-dev/';
```

Or via Terraform:

```hcl
resource "aws_glue_catalog_database" "invoices" {
  name        = "invoices_${var.env}"
  description = "Invoice pipeline warehouse"
  location_uri = "s3://invoices-warehouse-${var.env}/"
}
```

## Table Registration

Tables created via Athena DDL (`CREATE TABLE ... TBLPROPERTIES ('table_type'='ICEBERG')`) are **automatically registered** in Glue. No manual registration needed.

Tables created via `pyiceberg` with `GlueCatalog` are also automatically registered — pyiceberg calls the Glue API (`CreateTable` / `UpdateTable`) directly.

To register an existing Iceberg table (e.g. migrated from another catalog), use a Glue Crawler pointed at the S3 `metadata/` directory, or manually create with `ALTER TABLE SET LOCATION`.

## IAM Permissions Required

The Lambda execution role needs:

```json
{
  "Effect": "Allow",
  "Action": [
    "glue:GetDatabase",
    "glue:GetDatabases",
    "glue:GetTable",
    "glue:GetTables",
    "glue:CreateTable",
    "glue:UpdateTable",
    "glue:DeleteTable",
    "glue:GetPartitions"
  ],
  "Resource": [
    "arn:aws:glue:REGION:ACCOUNT:catalog",
    "arn:aws:glue:REGION:ACCOUNT:database/invoices_*",
    "arn:aws:glue:REGION:ACCOUNT:table/invoices_*/*"
  ]
}
```

`glue:UpdateTable` is required for every Iceberg write — the catalog pointer (`metadata_location`) is updated atomically on each commit.

## Glue Iceberg REST Endpoint (alternative to direct GlueCatalog)

Since 2024, Glue exposes an Iceberg REST catalog endpoint:

```
https://glue.{region}.amazonaws.com/iceberg
```

This endpoint is compatible with any REST-catalog-aware Iceberg client (pyiceberg, Spark, Trino). Advantages:
- No direct Glue API calls needed; standard REST protocol
- Supports SigV4 authentication via IAM Role
- Required for S3 Tables (Amazon S3 native Iceberg tables)

For the invoice pipeline, either `GlueCatalog` (direct boto3) or REST endpoint work. The REST endpoint is preferred for `pyiceberg` >= 0.7.0.

## Naming Conventions

| Object | Convention | Example |
|--------|-----------|---------|
| Database | `{domain}_{env}` | `invoices_dev`, `invoices_prod` |
| Table | snake_case, descriptive | `extracted_invoices` |
| S3 prefix | matches table name | `s3://invoices-warehouse-dev/extracted_invoices/` |

## Related

- [athena-engine-v3](athena-engine-v3.md)
- [iceberg-on-athena](iceberg-on-athena.md)
- [athena-iam-policy spec](../specs/athena-iam-policy.json)
