---
title: Migrate data to InfluxDB Cloud Dedicated
description: >
  Migrate data from InfluxDB powered by TSM (OSS, Enterprise, or Cloud) to
  InfluxDB Cloud Dedicated.
menu:
  influxdb3_cloud_dedicated:
    name: Migrate data
    parent: Guides
weight: 104
aliases: 
  - /influxdb3/cloud-dedicated/write-data/migrate-data/
alt_links:
  cloud: /influxdb/cloud/write-data/migrate-data/
---

Migrate data to InfluxDB Cloud Dedicated from other
InfluxDB instances powered by TSM including InfluxDB OSS 1.x, 2.x,
InfluxDB Enterprise, and InfluxDB Cloud (TSM).

- [Should you migrate?](#should-you-migrate)
  - [Are you currently limited by series cardinality?](#are-you-currently-limited-by-series-cardinality)
  - [Do you want to use SQL to query your data?](#do-you-want-to-use-sql-to-query-your-data)
  - [Do you want better InfluxQL performance?](#do-you-want-better-influxql-performance)
  - [Do you depend on a specific cloud provider or region?](#do-you-depend-on-a-specific-cloud-provider-or-region)
  - [Are you reliant on Flux queries and Flux tasks?](#are-you-reliant-on-flux-queries-and-flux-tasks)
- [Before you migrate](#before-you-migrate)
- [Data migration guides](#data-migration-guides)

## Should you migrate?

There are important things to consider with migrating to InfluxDB Cloud Dedicated.
The following questions will help guide your decision to migrate.

#### Are you currently limited by series cardinality?

**Yes, you should migrate**. Series cardinality is a major limiting factor with
the InfluxDB TSM storage engine. The more unique series in your data, the less
performant your database.
The InfluxDB 3 storage engine supports limitless series cardinality and is, without
question, the better solution for high series cardinality workloads.

#### Do you want to use SQL to query your data?

**Yes, you should migrate**. InfluxDB Cloud Dedicated lets you query your time
series data with SQL. For more information about querying your data with SQL, see:

- [Query data with SQL](/influxdb3/cloud-dedicated/query-data/sql/)
- [InfluxDB SQL reference](/influxdb3/cloud-dedicated/reference/sql/)

#### Do you want better InfluxQL performance?

**Yes, you should migrate**. One of the primary goals when designing the InfluxDB
v3 storage engine was to enable performant implementations of both SQL and InfluxQL.
When compared to querying InfluxDB powered by TSM (InfluxDB OSS 1.x, 2.x, and Enterprise),
InfluxQL queries are more performant when querying InfluxDB powered by the v3 storage engine.

#### Do you depend on a specific cloud provider or region?

**You should maybe migrate**. InfluxDB Cloud Dedicated instances are available
from the following providers:

{{< cloud_regions type=iox-list >}}

If your deployment requires other cloud providers or regions, you may need to
wait until the v3 storage engine is available in a region that meets your requirements.
We are currently working to make InfluxDB 3 available on more providers and
in more regions around the world.

#### Are you reliant on Flux queries and Flux tasks?

**You should not migrate**. {{% product-name %}} doesn't support Flux.

---

## Before you migrate

Before you migrate from InfluxDB 1.x or 2.x to {{< product-name >}}, there
are schema design practices supported by the TSM storage engine that are not
supported in the InfluxDB 3 storage engine. Specifically, InfluxDB 3 enforces the following schema restrictions:

- You can't use duplicate names for tags and fields.
- By default, measurements can contain up to 250 columns where each column
  represents time, a field, or a tag.

_For more information, see [Schema restrictions](/influxdb3/cloud-serverless/write-data/best-practices/schema-design/#schema-restrictions)._

If your schema does not adhere to these restrictions, you must update your schema
before migrating to {{< product-name >}}.

---

## Data migration guides

{{< children >}}
