---
title: InfluxDB API reference
description: >
  Use the InfluxDB API endpoints to run queries, write data, check server status, and troubleshoot by tracking HTTP client requests, collecting server statistics, and using Go "pprof" profiles.
aliases:
    - /enterprise_influxdb/v1/concepts/api/
menu:
  enterprise_influxdb_v1:
    name: InfluxDB API reference
    weight: 20
    parent: Tools
---

The InfluxDB API provides a simple way to interact with the database.
It uses HTTP response codes, authentication with username and password credentials or API tokens, and JSON-formatted response data.

The following sections assume your InfluxDB instance is running on `localhost`
port `8086` and HTTPS is not enabled.
Those settings [are configurable](/enterprise_influxdb/v1/administration/configure/config-data-nodes/#http-endpoints-settings).

- [InfluxDB 2.x API compatibility endpoints](#influxdb-20-api-compatibility-endpoints)
- [InfluxDB  1.x HTTP endpoints](#influxdb-1x-http-endpoints)

## InfluxDB 2.x API compatibility endpoints

InfluxDB 1.8.0 introduced forward compatibility APIs for InfluxDB 2.x.
There are multiple reasons for introducing these:

- [InfluxDB 2.x client libraries](/enterprise_influxdb/v1/tools/api_client_libraries/)
  are built for the InfluxDB `/api/v2` API and work with **InfluxDB 2.x** and **InfluxDB 1.8+**.
- InfluxDB Cloud is a generally available service across multiple cloud service providers and regions
  that is fully compatible with the InfluxDB 2.x client libraries.

If you are just getting started with InfluxDB 1.x today, we recommend adopting
the [latest InfluxDB 2.x client libraries](/enterprise_influxdb/v1/tools/api_client_libraries/).
They allow you to easily move from InfluxDB 1.x to InfluxDB OSS 2.x or InfluxDB Cloud
(when you are ready).

The following forward compatible APIs are available:

| Endpoint                                     | Description                                                                                                |
|:----------                                   |:----------                                                                                                 |
| [/api/v2/query](#apiv2query-http-endpoint) | Query data in InfluxDB 1.8.0+ using the InfluxDB 2.x API and [Flux](/flux/latest/)                         |
| [/api/v2/write](#apiv2write-http-endpoint) | Write data to InfluxDB 1.8.0+ using the InfluxDB 2.x API _(compatible with InfluxDB 2.x client libraries)_ |
| [/health](#health-http-endpoint)             | Check the health of your InfluxDB instance                                                                 |
| [/api/v2/buckets](#apiv2buckets-http-endpoint)    | Allows some client code using buckets to run against 1.X and 2.X without modification                                                             |
| [/api/v2/delete](#apiv2delete-http-endpoint)      | Supports deletion by tag value, timestamp and measurement using the InfluxDB 2.x API  

### `/api/v2/query/` HTTP endpoint

The `/api/v2/query` endpoint accepts `POST` HTTP requests.
Use this endpoint to query data using [Flux](/enterprise_influxdb/v1/flux/) and [InfluxDB 2.x client libraries](/influxdb/v2.x/api-guide/client-libraries/).
 Flux is the primary language for working with data in InfluxDB 2.x.

**Include the following HTTP headers:**

- `Accept: application/csv`
- `Content-type: application/vnd.flux`
- If [authentication is enabled](/enterprise_influxdb/v1/administration/authentication_and_authorization),
  provide your InfluxDB username and password:  
  `Authorization: Token USERNAME:PASSWORD`

{{< code-tabs-wrapper >}}
{{% code-tabs %}}
[No Auth](#)
[Auth Enabled](#)
{{% /code-tabs %}}
{{% code-tab-content %}}
```bash
curl -XPOST localhost:8086/api/v2/query -sS \
  -H 'Accept:application/csv' \
  -H 'Content-type:application/vnd.flux' \
  -d 'from(bucket:"telegraf")
        |> range(start:-5m)
        |> filter(fn:(r) => r._measurement == "cpu")'
```
{{% /code-tab-content %}}
{{% code-tab-content %}}
```bash
curl -XPOST localhost:8086/api/v2/query -sS \
  -H 'Accept:application/csv' \
  -H 'Content-type:application/vnd.flux' \
  -H 'Authorization: Token USERNAME:PASSWORD' \
  -d 'from(bucket:"telegraf")
        |> range(start:-5m)
        |> filter(fn:(r) => r._measurement == "cpu")'
```
{{% /code-tab-content %}}
{{< /code-tabs-wrapper >}}

### `/api/v2/write/` HTTP endpoint

The `/api/v2/write` endpoint accepts `POST` HTTP requests.
Use this endpoint to write to an InfluxDB 1.8.0+ database using [InfluxDB 2.x client libraries](/influxdb/v2/api-guide/client-libraries/).

Both InfluxDB 1.x and 2.x APIs support the same line protocol format for raw time series data.
For the purposes of writing data, the APIs differ only in the URL parameters and request headers.
InfluxDB 2.x uses [organizations](/influxdb/v2/reference/glossary/#organization)
and [buckets](/influxdb/v2/reference/glossary/#bucket)
instead of databases and retention policies.
The `/api/v2/write` endpoint maps the supplied version 1.8 database and retention policy to a bucket.

**Include the following URL parameters:**

- `bucket`: Provide the database name and retention policy separated by a forward slash (`/`).
  For example: `database/retention-policy`.
  Empty retention policies map to the default retention policy.
  `database/weekly` maps to a database named "database" and retention policy named "weekly".
  `database/` and `database` map to a database named "database" and the default retention policy.
- `org`: In InfluxDB 1.x, there is no concept of organization. The `org` parameter is ignored and can be left empty.
- `precision`: Precision of timestamps in the line protocol.
  Accepts `ns` (nanoseconds), `us`(microseconds), `ms` (milliseconds) and `s` (seconds).

**Include the following HTTP header:**

- `Authorization`: InfluxDB 2.x uses this header with the `Token` scheme and [API Tokens](/influxdb/v2/admin/tokens/)
  to authenticate each API request.
  InfluxDB v1.x uses username and password credentials for authenticating API requests.
  To provide InfluxDB 1.x credentials, use the `Token` scheme and include your username and password separated by a colon (`:`).

  - `Token` scheme with v1.x credentials:

    ```http
    Authorization: Token USERNAME:PASSWORD
    ```

{{< code-tabs-wrapper >}}
{{% code-tabs %}}
[No Auth](#)
[Auth Enabled](#)
{{% /code-tabs %}}
{{% code-tab-content %}}
```bash
curl -XPOST "localhost:8086/api/v2/write?bucket=db/rp&precision=s" \
  --data-raw "mem,host=host1 used_percent=23.43234543 1556896326"
```
{{% /code-tab-content %}}
{{% code-tab-content %}}
```bash
curl -XPOST "localhost:8086/api/v2/write?bucket=db/rp&precision=s" \
  -H 'Authorization: Token <username>:<password>' \
  --data-raw "mem,host=host1 used_percent=23.43234543 1556896326"
```
{{% /code-tab-content %}}
{{< /code-tabs-wrapper >}}

### `/health` HTTP endpoint
The `/health` endpoint accepts `Get` HTTP requests.
Use this endpoint to check the health of your InfluxDB instance.

```bash
curl -XGET "localhost:8086/health"
```

##### /health endpoint responses
| Response code | Health    | Message                        | Status |
|:------------- |:------    |:-------                        | ------:|
| 200           | Healthy   | `ready for queries and writes` | `pass` |
| 503           | Unhealthy |                                | `fail` |

---

### `/api/v2/buckets/` HTTP endpoint

The [/api/v2/buckets](/influxdb/v2/api/#tag/Buckets) endpoint accepts `GET`, `POST` and `DELETE` HTTP requests. Use this endpoint to [create](/influxdb/v2/api/#operation/PostBuckets), [delete](/influxdb/v2/api/#operation/DeleteBucketsID), [list](/influxdb/v2/api/#operation/GetBuckets), [update](/influxdb/v2/api/#operation/PatchBucketsID) and [retrieve](/influxdb/v2/api/#operation/GetBucketsID) buckets in your InfluxDB instance. Note that InfluxDB 2.x uses organizations and buckets instead of databases and retention policies.  

**Include the following URL parameters:**

- `bucket`: Provide the database name and retention policy separated by a forward slash (`/`).
  For example: `database/retention-policy`.
  Empty retention policies map to the default retention policy.
- `org`: In InfluxDB 1.x, there is no concept of organization. The `org` parameter is ignored and can be left empty.

**Include the following HTTP header:**

- `Authorization`: InfluxDB 2.x uses this header with the `Token` scheme and [API Tokens](/influxdb/v2/admin/tokens/)
  to authenticate each API request.
  InfluxDB v1.x uses username and password credentials for authenticating API requests.
  To provide InfluxDB 1.x credentials, use the `Token` scheme and include your username and password separated by a colon (`:`).

  - `Token` scheme with v1.x credentials:

    ```http
    Authorization: Token USERNAME:PASSWORD
    ```

The following example shows how to list all databases:

```bash
curl --request GET "http://localhost:8086/api/v2/buckets"   
  -H 'Authorization: Token <username>:<password>'
```

The following example shows how to delete a database named "test":

```bash
curl --request DELETE "http://localhost:8086/api/v2/buckets/test/autogen" 
  --header "Content-type: application/json"   
  -H 'Authorization: Token <username>:<password>'
```

### `/api/v2/delete/` HTTP endpoint

The [`/api/v2/delete`](/influxdb/v2/api/#tag/Delete) endpoint accepts `POST` HTTP requests. Use this endpoint to delete points from InfluxDB, including points with specific tag values, timestamps and measurements.  

**Include the following URL parameters:**

- `bucket`: Provide the database name and retention policy separated by a forward slash (`/`).
  For example: `database/retention-policy`.
- `precision`: Precision of timestamps in the line protocol.
  Accepts `ns` (nanoseconds), `us`(microseconds), `ms` (milliseconds) and `s` (seconds).

**Include the following HTTP header:**

- `Authorization`: InfluxDB 2.x uses this header with the `Token` scheme and [API Tokens](/influxdb/v2/admin/tokens/)
  to authenticate each API request.
  InfluxDB v1.x uses username and password credentials for authenticating API requests.
  To provide InfluxDB 1.x credentials, use the `Token` scheme and include your username and password separated by a colon (`:`).

  - `Token` scheme with v1.x credentials:

    ```http
    Authorization: Token USERNAME:PASSWORD
    ```

Delete all points in a specified time range:

```bash
curl --request POST "http://localhost:8086/api/v2/delete?bucket=exampleDB/autogen \
  --header 'Authorization: Token <username>:<password>' \
  --header 'Content-Type: application/json' \
  --data '{
    "start": "2020-03-01T00:00:00Z",
    "stop": "2020-11-14T00:00:00Z"
    }'
```

Delete points in a specific measurement with a specific tag value:

```bash
curl --request POST "http://localhost:8086/api/v2/delete?bucket=exampleDB/autogen \
  --header 'Authorization: Token <username>:<password>' \
  --header 'Content-Type: application/json' \
  --data '{
    "start": "2020-03-01T00:00:00Z",
    "stop": "2020-11-14T00:00:00Z",
    "predicate": "_measurement=\"example-measurement\" AND exampleTag=\"exampleTagValue\""
    }'
```

If you use the `predicate` option in your request, review [delete predicate syntax](/influxdb/v2/reference/syntax/delete-predicate/) and note its [limitations](/influxdb/v2/reference/syntax/delete-predicate/#limitations).

## InfluxDB 1.x HTTP endpoints
The following InfluxDB 1.x API endpoints are available:

| Endpoint                                        | Description                                                                    |
| :---------------------------------------------- | :----------------------------------------------------------------------------- |
| [/debug/pprof ](#debugpprof-http-endpoint)      | Generate profiles for troubleshooting                                          |
| [/debug/requests](#debugrequests-http-endpoint) | Track HTTP client requests to the `/write` and `/query` endpoints              |
| [/debug/vars](#debugvars-http-endpoint)         | Collect internal InfluxDB statistics                                           |
| [/ping](#ping-http-endpoint)                    | Check the status of your InfluxDB instance and your version of InfluxDB        |
| [/query](#query-http-endpoint)                  | Query data using **InfluxQL**, manage databases, retention policies, and users |
| [/write](#write-http-endpoint)                  | Write data to a database                                                       |
| [/shard-status](#shard-status-http-endpoint)    | Get information about a data node's shards                                     |

### `/debug/pprof` HTTP endpoint

InfluxDB supports the Go [`net/http/pprof`](https://golang.org/pkg/net/http/pprof/) HTTP endpoints, which are useful for troubleshooting. The `pprof` package serves runtime profiling data in the format expected by the _pprof_ visualization tool.

#### Definition

```
curl http://localhost:8086/debug/pprof/
```

The `/debug/pprof/` endpoint generates an HTML page with a list of built-in Go profiles and hyperlinks for each.

| Profile | Description
| :---------------- | :-------------------- |
| block | Stack traces that led to blocking on synchronization primitives. |
| goroutine  | Stack traces of all current goroutines.  |
| heap  | Sampling of stack traces for heap allocations.  |
| mutex | Stack traces of holders of contended mutexes.  |
| threadcreate | Stack traces that led to the creation of new OS threads. |

To access one of the `/debug/pprof/` profiles listed above, use the following cURL request, substituting `<profile>` with the name of the profile. The resulting profile is output to a file specified in `<path/to/output-file>`.

```bash
curl -o <path/to/output-file>  http://localhost:8086/debug/pprof/<profile>
```

In the following example, the cURL command outputs the resulting heap profile to a file:

```bash
curl -o <path/to/output-file> http://localhost:/8086/debug/pprof/heap
```

You can also use the [Go `pprof` interactive tool](https://github.com/google/pprof) to access the InfluxDB `/debug/pprof/` profiles.
For example, to look at the heap profile of a InfluxDB instance using this tool, you would use a command like this:

```bash
go tool pprof http://localhost:8086/debug/pprof/heap
```

For more information about the Go `/net/http/pprof` package and the interactive _pprof_ analysis and visualization tool, see:

* [Package pprof (`net/http/pprof`)](https://golang.org/pkg/net/http/pprof/)
* [`pprof` analysis and visualization tool](https://github.com/google/pprof)
* [Profiling Go programs](https://blog.golang.org/profiling-go-programs)
* [Diagnostics - Profiling](https://golang.org/doc/diagnostics.html#profiling)

#### `/debug/pprof/all` HTTP endpoint

The `/debug/pprof/all` endpoint is a custom `/debug/pprof` profile intended primarily for use by InfluxData support.
This endpoint generates a `profile.tar.gz` archive containing text files with the standard Go profiling information and additional debugging data.
An optional CPU profile is generated when using the `cpu=` option followed by a duration in seconds (e.g., `cpu=30s`).

To create a `profile.tar.gz` archive, use the following cURL command to generate a `profile.tar.gz` file for sharing with InfluxData support.

```bash
curl -o profiles.tar.gz "http://localhost:8086/debug/pprof/all?cpu=30s"
```

The cURL output includes "Time Spent," the time elapsed (in seconds).

```bash
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  237k    0  237k    0     0   8025      0 --:--:--  0:00:30 --:--:-- 79588
```

After 30 seconds of data has been collected, the results are output to the specified file.

### `/debug/requests` HTTP endpoint

Use this endpoint to track HTTP client requests to the `/write` and `/query` endpoints.
The `/debug/requests` endpoint returns the number of writes and queries to InfluxDB per username and IP address.

#### Definition

```bash
curl http://localhost:8086/debug/requests
```

#### Query string parameters

| Query String Parameter | Optional/Required | Definition |
| :--------------------- | :---------------- |:---------- |
| seconds=\<integer\>      | Optional          | Sets the duration (in seconds) over which the client collects information. The default duration is ten seconds. |

#### Examples

##### Track requests over a ten-second interval

```bash
curl http://localhost:8086/debug/requests
```

The response body contains data in JSON format:

```JSON
{
"user1:123.45.678.91": {"writes":1,"queries":0},
}
```

The response shows that, over the past ten seconds, the `user1` user sent one request to the `/write` endpoint and no requests to the `/query` endpoint from the `123.45.678.91` IP address.

##### Track requests over a one-minute interval

```bash
curl http://localhost:8086/debug/requests?seconds=60
```

The response body contains data in JSON format:

```JSON
{
"user1:123.45.678.91": {"writes":3,"queries":0},
"user1:000.0.0.0": {"writes":0,"queries":16},
"user2:xx.xx.xxx.xxx": {"writes":4,"queries":0}
}
```

The response shows that, over the past minute, `user1` sent three requests to the `/write` endpoint from `123.45.678.91`, `user1` sent 16 requests to the `/query` endpoint from `000.0.0.0`, and `user2` sent four requests to the `/write` endpoint from `xx.xx.xxx.xxx`.

### `/debug/vars` HTTP endpoint

InfluxDB exposes statistics and information about its runtime through the `/debug/vars` endpoint, which can be accessed using the following cURL command:

```bash
curl http://localhost:8086/debug/vars
```

Server statistics and information are displayed in JSON format.
For information about InfluxDB HTTP server metrics, see the [`httpd` measurement](/platform/monitoring/influxdata-platform/tools/measurements-internal/#httpd).

>**Note:** The [InfluxDB input plugin](https://github.com/influxdata/telegraf/tree/master/plugins/inputs/influxdb) is available to collect metrics (using the `/debug/vars` endpoint) from specified Kapacitor instances. For a list of the measurements and fields, see the [InfluxDB input plugin README](https://github.com/influxdata/telegraf/tree/master/plugins/inputs/influxdb#readme).

### `/ping` HTTP endpoint

The ping endpoint accepts both `GET` and `HEAD` HTTP requests.
Use this endpoint to check the status of your InfluxDB instance and your version
of InfluxDB.

#### Definition

```
GET http://localhost:8086/ping
```

```
HEAD http://localhost:8086/ping
```

#### `verbose` option

Default value is `false`. By default, the `/ping` HTTP endpoint responds with HTTP status `204` and an empty response body to let the client know that the server is running. 
If set to `true` (`/ping?verbose=true`), the server responds with HTTP status `200` status and a response body that contains details.
The `verbose=true` option is required for [Google Cloud Load Balancing](https://cloud.google.com/load-balancing/docs/health-check-concepts) health checks.

#### Example

You can use the `/ping` endpoint to find the build and version of an InfluxDB instance.
The `X-Influxdb-Build` header field displays the InfluxDB build type, either `OSS` (open source) or `ENT` (Enterprise).
The `X-Influxdb-Version` header field displays the InfluxDB version.

```bash
~ curl -sl -I http://localhost:8086/ping

HTTP/1.1 204 No Content
Content-Type: application/json
Request-Id: 9c353b0e-aadc-11e8-8023-000000000000
X-Influxdb-Build: OSS
X-Influxdb-Version: v{{< latest-patch >}}
X-Request-Id: 9c353b0e-aadc-11e8-8023-000000000000
Date: Tue, 05 Nov 2018 16:08:32 GMT
```

#### Status Codes and Responses

The response body is empty.

| HTTP Status Code   | Description    |
| :----------------- | :------------- |
| 204      | Success! Your InfluxDB instance is up and running.      |

### `/query` HTTP endpoint

The `/query` endpoint accepts `GET` and `POST` HTTP requests.
Use this endpoint to query data and manage databases, retention policies,
and users.

#### Definition

```
GET http://localhost:8086/query
```

```
POST http://localhost:8086/query
```

#### Verb usage

| Verb  | Query Type |
| :---- | :--------- |
| GET   | Use for all queries that start with: <br><br> [`SELECT`](/enterprise_influxdb/v1/query_language/spec/#select)* <br><br> [`SHOW`](/enterprise_influxdb/v1/query_language/spec/#show-continuous-queries)   |
| POST  | Use for all queries that start with: <br><br> [`ALTER`](/enterprise_influxdb/v1/query_language/spec/#alter-retention-policy) <br><br> [`CREATE`](/enterprise_influxdb/v1/query_language/spec/#create-continuous-query) <br><br> [`DELETE`](/enterprise_influxdb/v1/query_language/spec/#delete) <br><br> [`DROP`](/enterprise_influxdb/v1/query_language/spec/#drop-continuous-query) <br><br> [`GRANT`](/enterprise_influxdb/v1/query_language/spec/#grant) <br><br> [`KILL`](/enterprise_influxdb/v1/query_language/spec/#kill-query) <br><br> [`REVOKE`](/enterprise_influxdb/v1/query_language/spec/#revoke) |

\* The only exceptions are `SELECT` queries that include an [`INTO` clause](/enterprise_influxdb/v1/query_language/explore-data/#the-into-clause).
Those `SELECT` queries require a `POST` request.

#### Examples

###### Query data with a `SELECT` statement

```bash
curl -G 'http://localhost:8086/query?db=mydb' --data-urlencode 'q=SELECT * FROM "mymeas"'
```

The response body contains data in JSON format:

```JSON
{"results":[{"statement_id":0,"series":[{"name":"mymeas","columns":["time","myfield","mytag1","mytag2"],"values":[["2017-03-01T00:16:18Z",33.1,null,null],["2017-03-01T00:17:18Z",12.4,"12","14"]]}]}]}
```

The `mymeas` [measurement](/enterprise_influxdb/v1/concepts/glossary/#measurement) has two points.
The first point has the [timestamp](/enterprise_influxdb/v1/concepts/glossary/#timestamp) `2017-03-01T00:16:18Z`, a `myfield` value of `33.1`, and no tag values for the `mytag1` and `mytag2` [tag keys](/enterprise_influxdb/v1/concepts/glossary/#tag-key).
The second point has the timestamp `2017-03-01T00:17:18Z`, a `myfield` value of `12.4`, a `mytag1` value of `12`, and a `mytag2` value of `14`.

The same query in the InfluxDB [Command Line Interface](/enterprise_influxdb/v1/tools/influx-cli/use-influx/) (CLI) returns the following table:

```sql
name: mymeas
time                  myfield  mytag1  mytag2
----                  -------  ------  ------
2017-03-01T00:16:18Z  33.1
2017-03-01T00:17:18Z  12.4     12      14
```

##### Query data with a `SELECT` statement and an `INTO` clause

```bash
curl -XPOST 'http://localhost:8086/query?db=mydb' --data-urlencode 'q=SELECT * INTO "newmeas" FROM "mymeas"'
```

The response body contains data in JSON format:

```JSON
{"results":[{"statement_id":0,"series":[{"name":"result","columns":["time","written"],"values":[["1970-01-01T00:00:00Z",2]]}]}]}
```

`SELECT` queries that include and [`INTO` clause](/enterprise_influxdb/v1/query_language/explore-data/#the-into-clause) require a `POST` request.

The response shows that InfluxDB writes two points to the `newmeas` [measurement](/enterprise_influxdb/v1/concepts/glossary/#measurement).
Note that the system uses epoch 0 (`1970-01-01T00:00:00Z`) as a [null timestamp equivalent](/enterprise_influxdb/v1/troubleshooting/frequently-asked-questions/#why-does-my-query-return-epoch-0-as-the-timestamp).

##### Create a database

```bash
curl -XPOST 'http://localhost:8086/query' --data-urlencode 'q=CREATE DATABASE "mydb"'
```

The response data is similar to the following:

```JSON
{"results":[{"statement_id":0}]}
```

A successful [`CREATE DATABASE` query](/enterprise_influxdb/v1/query_language/manage-database/#create-database) returns no additional information.

#### Query string parameters

| Query String Parameter | Optional/Required | Definition |
| :--------------------- | :---------------- |:---------- |
| chunked=[true \| \<number_of_points>] | Optional | Returns points in streamed batches instead of in a single response. If set to `true`, InfluxDB chunks responses by series or by every 10,000 points, whichever occurs first. If set to a specific value, InfluxDB chunks responses by series or by that number of points.*  |
| db=\<database_name> | Required for database-dependent queries (most [`SELECT`](/enterprise_influxdb/v1/query_language/spec/#select) queries and [`SHOW`](/enterprise_influxdb/v1/query_language/spec/#show-continuous-queries) queries require this parameter). | Sets the target [database](/enterprise_influxdb/v1/concepts/glossary/#database) for the query. |
| epoch=[ns,u,µ,ms,s,m,h] | Optional | Returns epoch timestamps with the specified precision. By default, InfluxDB returns timestamps in RFC3339 format with nanosecond precision. Both `u` and `µ` indicate microseconds. |
| p=\<password> | Optional if you haven't [enabled authentication](/enterprise_influxdb/v1/administration/authentication_and_authorization/#set-up-authentication). Required if you've enabled authentication.** | Sets the password for authentication if you've enabled authentication. Use with the query string parameter `u`. |
| pretty=true | Optional | Enables pretty-printed JSON output. While this is useful for debugging it is not recommended for production use as it consumes unnecessary network bandwidth. |
| q=\<query> | Required | InfluxQL string to execute.  See also [Request Body](/enterprise_influxdb/v1/tools/api/#request-body). |
| u=\<username> | Optional if you haven't [enabled authentication](/enterprise_influxdb/v1/administration/authentication_and_authorization/#set-up-authentication). Required if you've enabled authentication.* | Sets the username for authentication if you've enabled authentication. The user must have read access to the database. Use with the query string parameter `p`. |

\* InfluxDB does not truncate the number of rows returned for requests without the `chunked` parameter.
That behavior is configurable; see the [`max-row-limit`](/enterprise_influxdb/v1/administration/configure/config-data-nodes/#max-row-limit)
configuration option for more information.

\** The InfluxDB API also supports basic authentication.
Use basic authentication if you've [enabled authentication](/enterprise_influxdb/v1/administration/authentication_and_authorization/#set-up-authentication)
and aren't using the query string parameters `u` and `p`.
See below for an [example](#create-a-database-using-basic-authentication) of basic authentication.

#### Examples

##### Query data with a `SELECT` statement and return pretty-printed JSON

```bash
curl -G 'http://localhost:8086/query?db=mydb&pretty=true' --data-urlencode 'q=SELECT * FROM "mymeas"'

{
    "results": [
        {
            "statement_id": 0,
            "series": [
                {
                    "name": "mymeas",
                    "columns": [
                        "time",
                        "myfield",
                        "mytag1",
                        "mytag2"
                    ],
                    "values": [
                        [
                            "2017-03-01T00:16:18Z",
                            33.1,
                            null,
                            null
                        ],
                        [
                            "2017-03-01T00:17:18Z",
                            12.4,
                            "12",
                            "14"
                        ]
                    ]
                }
            ]
        }
    ]
}
```

##### Query data with a `SELECT` statement and return second precision epoch timestamps

```bash
curl -G 'http://localhost:8086/query?db=mydb&epoch=s' --data-urlencode 'q=SELECT * FROM "mymeas"'
```

The response body data is similar to the following:

```bash
{"results":[{"statement_id":0,"series":[{"name":"mymeas","columns":["time","myfield","mytag1","mytag2"],"values":[[1488327378,33.1,null,null],[1488327438,12.4,"12","14"]]}]}]}
```

##### Create a database using HTTP authentication

The following example shows how to authenticate with v1.x credentials in the query string and
create a database:

```bash
curl -XPOST 'http://localhost:8086/query?u=myusername&p=mypassword' --data-urlencode 'q=CREATE DATABASE "mydb"'
```

The response body contains the following:

```json
{"results":[{"statement_id":0}]}
```

A successful [`CREATE DATABASE` query](/enterprise_influxdb/v1/query_language/manage-database/#create-database) returns no additional information.

The following sample passes invalid credentials:

```bash
curl -XPOST 'http://localhost:8086/query?u=myusername&p=notmypassword' --data-urlencode 'q=CREATE DATABASE "mydb"'
```

The response body contains the following:

```json
{"error":"authorization failed"}
```

##### Create a database using basic authentication

The following example shows how to use Basic authentication with v1.x credentials and create a database.

```bash
curl -XPOST -u myusername:mypassword 'http://localhost:8086/query' --data-urlencode 'q=CREATE DATABASE "mydb"'

{"results":[{"statement_id":0}]}
```

A successful [`CREATE DATABASE` query](/enterprise_influxdb/v1/query_language/manage-database/#create-database) returns no additional information.

The following example uses invalid credentials.

```bash
curl -XPOST -u myusername:notmypassword 'http://localhost:8086/query' --data-urlencode 'q=CREATE DATABASE "mydb"'
```

The response body contains the following:

```json
{"error":"authorization failed"}
```

#### Request body

```sh
--data-urlencode "q=<InfluxQL query>"
```

All queries must be URL encoded and follow
[InfluxQL](/enterprise_influxdb/v1/query_language/) syntax.
Our example shows the `--data-urlencode` parameter from `curl`, which we use in all examples on this page.

#### Options

##### Request multiple queries

Delimit multiple queries with a semicolon `;`.

##### Submit queries from a file

The API supports submitting queries from a file using a multipart `POST`
request.
The queries in the file must be separated a semicolon (`;`).

Syntax:

```bash
curl -F "q=@<path_to_file>" -F "async=true" http://localhost:8086/query
```

##### Request query results in CSV format

Syntax:

```bash
curl -H "Accept: application/csv" -G 'http://localhost:8086/query' [...]
```

Note that when the request includes `-H "Accept: application/csv"`, the system returns timestamps in `epoch` format, not RFC3339 format.

##### Bind parameters

The API supports binding parameters to particular field values or tag values.
Use the syntax `$<placeholder_key>` as a placeholder in the query, and URL
encode the map of placeholder keys to placeholder values in the request body.
This allows all InfluxQL queries where the value is customizable---such as field
values, function names, or intervals---to be represented using bind parameters.

Query syntax:

```
--data-urlencode 'q= SELECT [...] WHERE [ <field_key> | <tag_key> ] = $<placeholder_key>'
```

Map syntax:

```
--data-urlencode 'params={"<placeholder_key>":[ <placeholder_float_field_value> | <placeholder_integer_field_value> | "<placeholder_string_field_value>" | <placeholder_boolean_field_value> | "<placeholder_tag_value>" ]}'
```

Delimit multiple placeholder key-value pairs with comma `,`.

#### Examples

##### Send multiple queries

```bash
curl -G 'http://localhost:8086/query?db=mydb&epoch=s' --data-urlencode 'q=SELECT * FROM "mymeas";SELECT mean("myfield") FROM "mymeas"'
```

The response body contains results for both queries:

```JSON
{"results":[{"statement_id":0,"series":[{"name":"mymeas","columns":["time","myfield","mytag1","mytag2"],"values":[[1488327378,33.1,null,null],[1488327438,12.4,"12","14"]]}]},{"statement_id":1,"series":[{"name":"mymeas","columns":["time","mean"],"values":[[0,22.75]]}]}]}
```

- The request includes two queries: `SELECT * FROM "mymeas"` and `SELECT mean("myfield") FROM "mymeas"'`.
- In the results, InfluxDB assigns a statement identifier to each query:
  - `"statement_id": 0`: the first query
  - `"statement_id": 1`: the second query 

##### Request query results in CSV format

To format results in CSV, specify `application/csv` in the HTTP `Accept` header--for example:

```bash
curl -H "Accept: application/csv" -G 'http://localhost:8086/query?db=mydb' --data-urlencode 'q=SELECT * FROM "mymeas"'
```

The response body contains data in CSV format:

```csv
name,tags,time,myfield,mytag1,mytag2
mymeas,,1488327378000000000,33.1,mytag1,mytag2
mymeas,,1488327438000000000,12.4,12,14
```

- In the sample data, the first point doesn't contain [tag values](/enterprise_influxdb/v1/concepts/glossary/#tag-value) for the `mytag1` and `mytag2` [tag keys](/enterprise_influxdb/v1/concepts/glossary/#tag-key).

##### Submit queries from a file

```bash
curl -F "q=@queries.txt" -F "async=true" 'http://localhost:8086/query'
```

A sample of the queries in `queries.txt`:

```sql
CREATE DATABASE mydb;
CREATE RETENTION POLICY four_weeks ON mydb DURATION 4w REPLICATION 1;
```

##### Bind a parameter in the `WHERE` clause to specific tag value

Use the `params` option to pass arguments for a parameterized query--for example:

```bash
curl -G 'http://localhost:8086/query?db=mydb' --data-urlencode 'q=SELECT * FROM "mymeas" WHERE "mytag1" = $tag_value' --data-urlencode 'params={"tag_value":"12"}'
```

The response data is similar to the following:

```JSON
{"results":[{"statement_id":0,"series":[{"name":"mymeas","columns":["time","myfield","mytag1","mytag2"],"values":[["2017-03-01T00:17:18Z",12.4,"12","14"]]}]}]}
```

- In the request, `params` maps `$tag_value` to `"12"`.
  Because InfluxDB stores [tag values](/enterprise_influxdb/v1/concepts/glossary/#tag-value) as strings, you must double-quote them in parameter values.
- During query execution, InfluxDB substitutes the parameter values for the associated keys in the query.

##### Bind a parameter in the `WHERE` clause to a numerical field value

```bash
curl -G 'http://localhost:8086/query?db=mydb' --data-urlencode 'q=SELECT * FROM "mymeas" WHERE "myfield" > $field_value' --data-urlencode 'params={"field_value":30}'
```

The response data is similar to the following:

```JSON
{"results":[{"statement_id":0,"series":[{"name":"mymeas","columns":["time","myfield","mytag1","mytag2"],"values":[["2017-03-01T00:16:18Z",33.1,null,null]]}]}]}
```

- In the request, `params` maps `$field_value` to `30`. Because `myfield` stores numerical [field values](/enterprise_influxdb/v1/concepts/glossary/#field-value), the parameter value `30` does not require double quotes.
- During query execution, InfluxDB substitutes the parameter values for the associated keys in the query.

##### Bind two parameters in the `WHERE` clause to a specific tag value and numerical field value

```bash
curl -G 'http://localhost:8086/query?db=mydb' --data-urlencode 'q=SELECT * FROM "mymeas" WHERE "mytag1" = $tag_value AND  "myfield" < $field_value' --data-urlencode 'params={"tag_value":"12","field_value":30}'
```

The response data is similar to the following:

```JSON
{"results":[{"statement_id":0,"series":[{"name":"mymeas","columns":["time","myfield","mytag1","mytag2"],"values":[["2017-03-01T00:17:18Z",12.4,"12","14"]]}]}]}
```

- In the request, `params` maps `$tag_value` to `12` and `$field_value` to `30`.
- During query execution, InfluxDB substitutes the parameter values for the associated keys in the query.

#### Status codes and responses

The API response body contains results or error messages in JSON format.
To pretty-print JSON for viewing, include the query string parameter `pretty=true`
or pipe the response to a JSON-processor, such as [**jq**](https://stedolan.github.io/jq/).

##### Summary table

| HTTP status code | Description |
| :--------------- | :---------- |
| `200 OK` | Success. Response body contains data in JSON format. |
| `400 Bad Request` | Unacceptable request. Can occur with a syntactically incorrect query. Response body contains an error message with additional information in JSON format. |
| `401 Unauthorized` | Unacceptable request. Can occur with invalid authentication credentials. |

#### Examples

##### A successful request that returns data

```bash
curl -i -G 'http://localhost:8086/query?db=mydb' --data-urlencode 'q=SELECT * FROM "mymeas"'
```

The response is HTTP status `200 OK` and the body contains data in JSON format:

```
HTTP/1.1 200 OK
Connection: close
Content-Type: application/json
Request-Id: [...]
X-Influxdb-Version: {{< latest-patch >}}
Date: Wed, 08 Nov 2017 19:22:54 GMT
Transfer-Encoding: chunked

{"results":[{"statement_id":0,"series":[{"name":"mymeas","columns":["time","myfield","mytag1","mytag2"],"values":[["2017-03-01T00:16:18Z",33.1,null,null],["2017-03-01T00:17:18Z",12.4,"12","14"]]}]}]}
```

##### A query that contains an error

```bash
curl -i -G 'http://localhost:8086/query?db=mydb1' --data-urlencode 'q=SELECT * FROM "mymeas"'

The response body contains details about the error:

```
HTTP/1.1 200 OK
Connection: close
Content-Type: application/json
Request-Id: [...]
X-Influxdb-Version: {{< latest-patch >}}
Date: Wed, 08 Nov 2017 19:23:48 GMT
Transfer-Encoding: chunked

{"results":[{"statement_id":0,"error":"database not found: mydb1"}]}
```

##### An incorrectly formatted query

```bash
curl -i -G 'http://localhost:8086/query?db=mydb' --data-urlencode 'q=SELECT *'
```

The response is HTTP status `400 Bad Request` and the body contains details
about the error:

```
HTTP/1.1 400 Bad Request
Content-Type: application/json
Request-Id: [...]
X-Influxdb-Version: {{< latest-patch >}}
Date: Wed, 08 Nov 2017 19:24:25 GMT
Content-Length: 76

{"error":"error parsing query: found EOF, expected FROM at line 1, char 9"}
```

##### A request with invalid authentication credentials

```bash
curl -i  -XPOST 'http://localhost:8086/query?u=myusername&p=notmypassword' --data-urlencode 'q=CREATE DATABASE "mydb"'
```

The response is HTTP status `401 Unauthorized` and the body contains the error message.

```
HTTP/1.1 401 Unauthorized
Content-Type: application/json
Request-Id: [...]
Www-Authenticate: Basic realm="InfluxDB"
X-Influxdb-Version: {{< latest-patch >}}
Date: Wed, 08 Nov 2017 19:11:26 GMT
Content-Length: 33

{"error":"authorization failed"}
```

### `/write` HTTP endpoint

The `/write` endpoint accepts `POST` HTTP requests.
Use this endpoint to write data to a pre-existing database.

#### Definition

```
POST http://localhost:8086/write
```

#### Query string parameters

| Query String Parameter | Optional/Required | Description |
| :--------------------- | :---------------- | :---------- |
| consistency=[any,one,quorum,all] | Optional, available with [InfluxDB Enterprise clusters](/enterprise_influxdb/v1/) only. | Sets the write consistency for the point. InfluxDB assumes that the write consistency is `one` if you do not specify `consistency`. See the [InfluxDB Enterprise documentation](/enterprise_influxdb/v1/concepts/clustering#write-consistency) for detailed descriptions of each consistency option. |
| db=\<database> | Required | Sets the target [database](/enterprise_influxdb/v1/concepts/glossary/#database) for the write. |
| p=\<password> | Optional if you haven't [enabled authentication](/enterprise_influxdb/v1/administration/authentication_and_authorization/#set-up-authentication). Required if you've enabled authentication.* | Sets the password for authentication if you've enabled authentication. Use with the query string parameter `u`. |
| precision=[ns,u,ms,s,m,h] | Optional | Sets the precision for the supplied Unix time values. InfluxDB assumes that timestamps are in nanoseconds if you do not specify `precision`.** |
| rp=\<retention_policy_name> | Optional | Sets the target [retention policy](/enterprise_influxdb/v1/concepts/glossary/#retention-policy-rp) for the write. InfluxDB writes to the `DEFAULT` retention policy if you do not specify a retention policy. |
| u=\<username> | Optional if you haven't [enabled authentication](/enterprise_influxdb/v1/administration/authentication_and_authorization/#set-up-authentication). Required if you've enabled authentication.* | Sets the username for authentication if you've enabled authentication. The user must have write access to the database. Use with the query string parameter `p`. |

\* The InfluxDB API also supports basic authentication.
Use basic authentication if you've [enabled authentication](/enterprise_influxdb/v1/administration/authentication_and_authorization/#set-up-authentication)
and aren't using the query string parameters `u` and `p`.
See below for an [example](#write-a-point-to-the-database-mydb-using-basic-authentication) of basic authentication.

\*\* We recommend using the least precise precision possible as this can result
in significant improvements in compression.

#### Examples

##### Write a point to the database `mydb` with a timestamp in seconds

```bash
curl -i -XPOST "http://localhost:8086/write?db=mydb&precision=s" --data-binary 'mymeas,mytag=1 myfield=90 1463683075'
```

A successful write returns HTTP status `204 No Content`--for example:

```
HTTP/1.1 204 No Content
Content-Type: application/json
Request-Id: [...]
X-Influxdb-Version: {{< latest-patch >}}
Date: Wed, 08 Nov 2017 17:33:23 GMT
```

##### Write a point to the database `mydb` and the retention policy `myrp`

```bash
curl -i -XPOST "http://localhost:8086/write?db=mydb&rp=myrp" --data-binary 'mymeas,mytag=1 myfield=90'
```

##### Write a point to the database `mydb` using HTTP authentication

Valid credentials:

```bash
curl -i -XPOST "http://localhost:8086/write?db=mydb&u=myusername&p=mypassword" --data-binary 'mymeas,mytag=1 myfield=91'
```

A successful write returns HTTP status `204 No Content`.

```
HTTP/1.1 204 No Content
```

Invalid credentials:

```bash
curl -i -XPOST "http://localhost:8086/write?db=mydb&u=myusername&p=notmypassword" --data-binary 'mymeas,mytag=1 myfield=91'
```

If the username or password is incorrect, the response status is `401 Unauthorized`
and the response body contains the error message--for example:

```
HTTP/1.1 401 Unauthorized
Content-Type: application/json
Request-Id: [...]
Www-Authenticate: Basic realm="InfluxDB"
X-Influxdb-Version: {{< latest-patch >}}
Date: Wed, 08 Nov 2017 17:40:30 GMT
Content-Length: 33

{"error":"authorization failed"}
```

##### Write a point to the database `mydb` using basic authentication

```bash
curl -i -XPOST -u myusername:mypassword "http://localhost:8086/write?db=mydb" --data-binary 'mymeas,mytag=1 myfield=91'
```

Invalid credentials:

```bash
curl -i -XPOST -u myusername:notmypassword "http://localhost:8086/write?db=mydb" --data-binary 'mymeas,mytag=1 myfield=91'
```

If the username or password is incorrect, the response status is `401 Unauthorized`
and the response body contains the error message--for example:

```
HTTP/1.1 401 Unauthorized
Content-Type: application/json
Request-Id: [...]
Www-Authenticate: Basic realm="InfluxDB"
X-Influxdb-Version: {{< latest-patch >}}
Date: Wed, 08 Nov 2017 17:46:40 GMT
Content-Length: 33

{"error":"authorization failed"}
```

#### Request body

```bash
--data-binary '<Data in InfluxDB line protocol format>'
```

Data to write must be binary encoded and in the
[InfluxDB line protocol](/enterprise_influxdb/v1/concepts/glossary/#influxdb-line-protocol) format.

Examples in this page use `curl` with the `--data-binary` parameter to encode
line protocol in the request. 
Using any encoding method other than `--data-binary` will likely lead to issues;
`-d`, `--data-urlencode`, and `--data-ascii` may strip out newlines or
introduce unintended formatting.

Options:

- Write several points to the database with one request by separating each point
by a new line.
- Write points from a file with the `@` flag.
The file should contain a batch of points in line protocol format.
Individual points must be on their own line and separated by newline characters
(`\n`).
Files containing carriage returns cause parser errors.

> [!Important]
> #### Batch writes for optimal performance
> Write points in batches of 5,000 to 10,000 points.
> Smaller batches, and more HTTP requests, will result in sub-optimal performance.

#### Examples

##### Write a point with a nanosecond timestamp to the `mydb` database

```bash
curl -i -XPOST "http://localhost:8086/write?db=mydb" --data-binary 'mymeas,mytag=1 myfield=90 1463683075000000000'
```

If successful, the response status is HTTP `204 No Content`.

```
HTTP/1.1 204 No Content
```

##### Write a point with the local server's nanosecond timestamp to the `mydb` database

```bash
curl -i -XPOST "http://localhost:8086/write?db=mydb" --data-binary 'mymeas,mytag=1 myfield=90'
```

##### Write several points to the database by separating points with a new line

```bash
curl -i -XPOST "http://localhost:8086/write?db=mydb" --data-binary 'mymeas,mytag=3 myfield=89 1463689152000000000
mymeas,mytag=2 myfield=34 1463689152000000000'
```

##### Write several points to the database `mydb` from the file `data.txt`

```bash
curl -i -XPOST "http://localhost:8086/write?db=mydb" --data-binary @data.txt
```

`data.txt` contains the following sample data:

```
mymeas,mytag1=1 value=21 1463689680000000000
mymeas,mytag1=1 value=34 1463689690000000000
mymeas,mytag2=8 value=78 1463689700000000000
mymeas,mytag3=9 value=89 1463689710000000000
```

#### Status codes and responses

In general, HTTP `2xx` status codes indicate success, `4xx` status codes indicate
that InfluxDB could not understand the request, and `5xx` status codes indicate that the
system is overloaded or significantly impaired.
The response body contains an error message in JSON format.

##### Summary table

| HTTP status code | Description    |
| :--------------- | :------------- |
| 204 No Content   | Success!      |
| 400 Bad Request  | Unacceptable request. Can occur with an InfluxDB line protocol syntax error or if a user attempts to write values to a field that previously accepted a different value type. The returned JSON offers further information. |
| 401 Unauthorized | Unacceptable request. Can occur with invalid authentication credentials.  |
| 404 Not Found    | Unacceptable request. Can occur if a user attempts to write to a database that does not exist. The returned JSON offers further information. |
| 413 Request Entity Too Large | Unaccetable request. It will occur if the payload of the POST request is bigger than the maximum size allowed. See [`max-body-size`](/enterprise_influxdb/v1/administration/configure/config-data-nodes/#max-body-size) parameter for more details.
| 500 Internal Server Error  | The system is overloaded or significantly impaired. Can occur if a user attempts to write to a retention policy that does not exist. The returned JSON offers further information. |

#### Examples

##### A successful write

```
HTTP/1.1 204 No Content
```

##### Write a point with an incorrect timestamp

```
HTTP/1.1 400 Bad Request
[...]
{"error":"unable to parse 'mymeas,mytag=1 myfield=91 abc123': bad timestamp"}
```

##### Write an integer to a field that previously accepted a float

```
HTTP/1.1 400 Bad Request
[...]
{"error":"field type conflict: input field \"myfield\" on measurement \"mymeas\" is type int64, already exists as type float"}
```

##### Write a point with invalid authentication credentials

```
HTTP/1.1 401 Unauthorized
[...]
{"error":"authorization failed"}
```

##### Write a point to a database that doesn't exist

```
HTTP/1.1 404 Not Found
[...]
{"error":"database not found: \"mydb1\""}
```

##### Send a request body that is too large

```
HTTP/2 413 Request Entity Too Large
[...]
{"error":"Request Entity Too Large"}
```

##### Write a point to a retention policy that doesn't exist

```
HTTP/1.1 500 Internal Server Error
[...]
{"error":"retention policy not found: myrp"}
```

### `/shard-status` HTTP endpoint

The `/shard-status` endpoint accepts HTTP `GET` requests.
Use this endpoint to get information about all shards for a given data node.

#### Response

Requests to `/shard-status` return the following information in JSON format:

- `id`: the shard ID
- `size`: the size on disk of the shard in bytes
- `is_hot`: whether the time range from the shard includes `now`
  {{% note %}}
An _idle_ shard is fully compacted and not receiving new (potentially historical) writes.
A hot shard may or may not be idle.
  {{% /note %}}
- `state`: the anti-entropy status of the shard can be one of the following:
  `healthy`,`restore pending`,`restoring`,`repairing`,`error processing`

#### Example

```
curl -q 'http://localhost:8086/shard-status' | jq
{
  "databases": [
    {
      "name": "_internal",
      "retention_policies": [
        {
          "name": "monitor",
          "replication_factor": 1,
          "shards": [
            {
              "id": 2,
              "size": 594491,
              "is_hot": true
            }
          ]
        }
      ]
    },
    {
      "name": "stress",
      "retention_policies": [
        {
          "name": "autogen",
          "replication_factor": 1,
          "shards": [
            {
              "id": 3,
              "is_hot": false
            },
            {
              "id": 6,
              "size": 1921,
              "is_hot": false
            },
            {
              "id": 7,
              "is_hot": false
            },
            {
              "id": 10,
              "size": 1920,
              "is_hot": false
            },
            {
              "id": 11,
              "is_hot": true
            }
          ]
        }
      ]
    }
  ]
}
```
{{% caption %}}
This example uses [`jq`](https://stedolan.github.io/jq/) to print the JSON object.
{{% /caption %}}
