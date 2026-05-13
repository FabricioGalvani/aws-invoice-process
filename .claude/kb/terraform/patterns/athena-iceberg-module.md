# AWS Athena + Iceberg Warehouse Module

> **MCP Validated:** 2026-05-12
> **Purpose**: Reusable module for provisioning a Glue Catalog database, an Iceberg-format Glue table, and an Athena workgroup with a results S3 location

## When to Use

The `warehouse-writer` Lambda writes extracted invoice fields to an Iceberg table stored on S3, replacing the original BigQuery target. Athena provides serverless, pay-per-query SQL access over the same data. This module provisions the Glue Catalog objects (database + Iceberg table via `open_table_format_input`) and an Athena workgroup whose query results go to a dedicated S3 prefix. It also includes an optional named query resource for the most frequent ad-hoc validation queries.

> **Alternative path**: If invoice writes happen exclusively from Lambda using the `pyiceberg` library, the Glue catalog table is not needed for writes — `pyiceberg` manages its own metadata. In that case, create only the `aws_glue_catalog_database` and `aws_athena_workgroup` from this module and omit `aws_glue_catalog_table`. Athena can still query pyiceberg-managed tables via the Glue catalog once `pyiceberg` has registered them.

## Module Structure

```hcl
# modules/athena-iceberg/variables.tf
variable "database_name"   { type = string }
variable "table_name"      { type = string }
variable "data_bucket_arn" { type = string; description = "ARN of the S3 bucket storing Iceberg data files" }
variable "data_bucket_name" { type = string; description = "Name of the S3 bucket (used to build the table location URI)" }
variable "results_bucket_name" { type = string; description = "S3 bucket for Athena query results" }
variable "results_prefix"  { type = string; default = "athena-results/" }
variable "partition_columns" {
  type    = list(object({ name = string; type = string }))
  default = []
}
variable "schema_columns" {
  type    = list(object({ name = string; type = string; comment = optional(string, "") }))
  default = []
}
variable "workgroup_name" { type = string; default = "invoice-pipeline" }
variable "bytes_scanned_cutoff" {
  type    = number
  default = 1073741824  # 1 GB guard rail per query
}
variable "named_queries" {
  type    = map(string)
  default = {}
  description = "Map of query_name => SQL string for saved named queries"
}
variable "tags" { type = map(string); default = {} }
```

```hcl
# modules/athena-iceberg/main.tf
resource "aws_glue_catalog_database" "db" {
  name        = var.database_name
  description = "Invoice pipeline lakehouse database"
}

resource "aws_glue_catalog_table" "iceberg" {
  name          = var.table_name
  database_name = aws_glue_catalog_database.db.name

  # Iceberg tables require EXTERNAL_TABLE + open_table_format_input
  table_type = "EXTERNAL_TABLE"

  open_table_format_input {
    iceberg_input {
      metadata_operation = "CREATE"
      version            = "2"
    }
  }

  storage_descriptor {
    location = "s3://${var.data_bucket_name}/${var.table_name}/"

    dynamic "columns" {
      for_each = var.schema_columns
      content {
        name    = columns.value.name
        type    = columns.value.type
        comment = columns.value.comment
      }
    }
  }

  dynamic "partition_keys" {
    for_each = var.partition_columns
    content {
      name = partition_keys.value.name
      type = partition_keys.value.type
    }
  }
}

resource "aws_athena_workgroup" "wg" {
  name  = var.workgroup_name
  state = "ENABLED"
  tags  = var.tags

  configuration {
    enforce_workgroup_configuration    = true
    publish_cloudwatch_metrics_enabled = true
    bytes_scanned_cutoff_per_query     = var.bytes_scanned_cutoff

    result_configuration {
      output_location = "s3://${var.results_bucket_name}/${var.results_prefix}"

      encryption_configuration {
        encryption_option = "SSE_S3"
      }
    }
  }
}

resource "aws_athena_named_query" "queries" {
  for_each  = var.named_queries
  name      = each.key
  database  = aws_glue_catalog_database.db.name
  workgroup = aws_athena_workgroup.wg.id
  query     = each.value
}
```

```hcl
# modules/athena-iceberg/outputs.tf
output "database_name"          { value = aws_glue_catalog_database.db.name }
output "table_arn"              { value = aws_glue_catalog_table.iceberg.arn }
output "workgroup_name"         { value = aws_athena_workgroup.wg.name }
output "query_results_location" {
  value = "s3://${var.results_bucket_name}/${var.results_prefix}"
}
```

## Example Terragrunt Invocation (dev)

```hcl
# infrastructure/environments/dev/athena-iceberg/terragrunt.hcl
terraform {
  source = "../../../../modules/athena-iceberg"
}

include "root" { path = find_in_parent_folders() }

inputs = {
  database_name        = "invoice_pipeline_dev"
  table_name           = "extracted_invoices"
  data_bucket_name     = "invoices-warehouse-dev"
  data_bucket_arn      = "arn:aws:s3:::invoices-warehouse-dev"
  results_bucket_name  = "invoices-athena-results-dev"
  workgroup_name       = "invoice-pipeline-dev"
  bytes_scanned_cutoff = 536870912  # 512 MB in dev

  partition_columns = [
    { name = "vendor_type", type = "string" },
    { name = "invoice_date", type = "date" }
  ]

  schema_columns = [
    { name = "invoice_id",     type = "string" },
    { name = "vendor_name",    type = "string" },
    { name = "total_amount",   type = "decimal(10,2)" },
    { name = "currency",       type = "string" },
    { name = "line_items",     type = "string" },   # JSON blob
    { name = "extracted_at",   type = "timestamp" },
    { name = "confidence",     type = "double" },
    { name = "model_id",       type = "string" }
  ]

  named_queries = {
    "recent_extractions" = "SELECT * FROM extracted_invoices WHERE invoice_date >= current_date - interval '7' day LIMIT 100"
  }

  tags = { env = "dev" }
}
```

## Inputs Reference

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `database_name` | Yes | — | Glue catalog database name |
| `table_name` | Yes | — | Glue catalog table name |
| `data_bucket_name` | Yes | — | S3 bucket for Iceberg data files |
| `data_bucket_arn` | Yes | — | ARN of the data bucket |
| `results_bucket_name` | Yes | — | S3 bucket for Athena query results |
| `results_prefix` | No | `athena-results/` | Prefix inside results bucket |
| `workgroup_name` | No | `invoice-pipeline` | Athena workgroup name |
| `partition_columns` | No | `[]` | Partition key definitions |
| `schema_columns` | No | `[]` | Column definitions (name + type) |
| `bytes_scanned_cutoff` | No | `1073741824` | Per-query scan cost guard (bytes) |
| `named_queries` | No | `{}` | Saved named queries map |

## Common Pitfalls

- **`table_type = "ICEBERG"` does not work**: As of provider v5, setting `table_type = "ICEBERG"` is not supported by `aws_glue_catalog_table`. Use `table_type = "EXTERNAL_TABLE"` combined with `open_table_format_input { iceberg_input {} }` as shown above.
- **Schema drift on re-apply**: Iceberg table schema changes via Terraform can cause destroy/recreate cycles. For schema evolution after initial provisioning, prefer Athena DDL (`ALTER TABLE ADD COLUMNS`) rather than modifying `schema_columns` in Terraform.
- **Lambda IAM needs Glue + Athena**: The `warehouse-writer` Lambda role must include `glue:GetTable`, `glue:GetDatabase`, `athena:StartQueryExecution`, `athena:GetQueryExecution`, and `s3:PutObject` on the data and results buckets.
- **Workgroup enforcement**: `enforce_workgroup_configuration = true` overrides per-query output location. All BI tools and ad-hoc queries must select this workgroup or they will be rejected.
- **pyiceberg path**: If you use `pyiceberg` for Lambda writes, remove the `aws_glue_catalog_table` resource from this module invocation. pyiceberg registers its own table metadata; running both causes conflicting table definitions.

## See Also

- [S3 Module](./s3-module.md) — warehouse S3 bucket provisioned separately
- [Lambda Module](./lambda-module.md) — `warehouse-writer` Lambda that writes to this table
- [Bedrock Module](./bedrock-module.md) — `data-extractor` upstream of warehouse-writer
