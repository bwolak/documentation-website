---
layout: default
title: Star Tree
nav_order: 61
has_children: false
parent: Supported field types
---
# Star-tree field type

This is an experimental feature and is not recommended for use in a production environment. For updates on the progress the feature or if you want to leave feedback, join the discussion on the [OpenSearch forum](https://forum.opensearch.org/).    
{: .warning}

[Star-tree index](https://docs.pinot.apache.org/basics/indexing/star-tree-index) pre-computes aggregations, accelerating the performance of aggregation queries. 
If a star-tree index is configured as part of an index mapping, the star-tree index is created and maintained as data is ingested in real time.

OpenSearch will automatically use the star-tree index to optimize aggregations if the fields queried are part of star-tree index dimension fields and the aggregations are on star-tree index metrics fields. No changes are required in the query syntax or the request parameters.

For more information, see [Star Tree index]({{site.url}}{{site.baseurl}}/search-plugins/star-tree-index/)

## Prerequisites

To use the star-tree index, fulfill the following prerequisites: 

- Set the feature flag `opensearch.experimental.feature.composite_index.star_tree.enabled` to `true`. For more information about enabling and disabling feature flags, see [Enabling experimental features]({{site.url}}{{site.baseurl}}/install-and-configure/configuring-opensearch/experimental/).
- Set the `indices.composite_index.star_tree.enabled` setting to `true`. For instructions on how to configure OpenSearch, see [configuring settings]({{site.url}}{{site.baseurl}}/install-and-configure/configuring-opensearch/index/#static-settings).
- Set the `index.composite_index` index setting to `true` during index creation.
- Enable `doc_values` : Ensure that the `doc_values` is enabled for the [dimensions](#ordered-dimensions) and [metrics](#metrics) fields used in your star-tree mapping.

## Limitations

The star-tree index feature has the following limitations:

- A star-tree index should only be used on indexes whose data is not updated or deleted, as standard updates and deletions are not accounted in the star-tree index.
- Currently, only `one` star-tree index can be created per index. Support for multiple star-trees will be added in a future release.

## Examples

The following examples show how to use a star-tree index

### Star0tree index mapping

Define Star Tree mapping under the `composite` section in `mappings`. 

The following example creates a corresponding star-tree index for all `request_aggs`. To compute metric aggregations for `request_size` and `latency` fields with queries on `port` and `status` fields, configure the following mappings:

```json
PUT logs
{
  "settings": {
    "index.number_of_shards": 1,
    "index.number_of_replicas": 0,
    "index.composite_index": true
  },
  "mappings": {
    "composite": {
      "request_aggs": {
        "type": "star_tree",
        "config": {
          "max_leaf_docs": 10000,
          "skip_star_node_creation_for_dimensions": [
            "port"
          ],
          "ordered_dimensions": [
            {
              "name": "status"
            },
            {
              "name": "port"
            }
          ],
          "metrics": [
            {
              "name": "request_size",
              "stats": [
                "sum",
                "value_count",
                "min",
                "max"
              ]
            },
            {
              "name": "latency",
              "stats": [
                "sum",
                "value_count",
                "min",
                "max"
              ]
            }
          ]
        }
      }
    },
    "properties": {
      "status": {
        "type": "integer"
      },
      "port": {
        "type": "integer"
      },
      "request_size": {
        "type": "integer"
      },
      "latency": {
        "type": "scaled_float",
        "scaling_factor": 10
      }
    }
  }
}
```



## Star tree mapping parameters

Specify any star-tree configuration mapping options under the `config` section. All parameters are final and cannot be modified without reindexing documents.

The star-tree `config` section supports the following property.

| Parameter  | Required/Optional | Description  | 
| :--- | :--- | :--- |
| `name` | Required | The name of the field. The field name should be present in the `properties` section as part of index `mapping`. Ensure that the `doc_values` setting is `enabled` for any associated fields.

### Ordered dimensions

The `ordered_dimensions` parameter are fields based on which metrics will be aggregated in a star-tree index. The star-tree index will be picked for querying only if all the fields in the query are part of the `ordered_dimensions`. 

When using the `ordered_dimesions` parameter, remember the following best practices:

- The order of dimensions matter. You can define the dimensions ordered from the highest cardinality to the lowest cardinality for efficient storage and query pruning. 
- Avoid high cardinality fields as dimensions. High cardinality fields adversely affect storage space, indexing throughput, and query performance.
- Currently, supported fields for `ordered_dimensions` are all [numeric field types](https://opensearch.org/docs/latest/field-types/supported-field-types/numeric/) with the exception of `unsigned_long`. For more information, see [GitHub issue #15231](https://github.com/opensearch-project/OpenSearch/issues/15231). 
- Support for other field_types such as `keyword`, `ip`  will be released in future versions. For more information, see [GitHub issue #16232](https://github.com/opensearch-project/OpenSearch/issues/16232).
- A minimum of `2` and maximum of `10` dimensions are supported per star-tree index.

`ordered_dimensions` supports the following property

| Parameter  | Required/Optional | Description  | 
| :--- | :--- | :--- |
| `name` | Required | The name of the field. The field name should be present in the `properties` section as part of index `mapping`. Ensure that the `doc_values` setting is `enabled` for any associated fields. |


### Metrics

Configure any metric fields for which you need to perform aggregations. `Metrics` are required as part of a star-tree configuration.

When using `metrics`, remember the following best practices: 

- Currently, supported fields for `metrics` are all [numeric field types](https://opensearch.org/docs/latest/field-types/supported-field-types/numeric/) with the exception of `unsigned_long`. For more information, see [GitHub issue #15231](https://github.com/opensearch-project/OpenSearch/issues/15231). 
- Supported metric aggregations include `Min`, `Max`, `Sum`, `Avg` and `Value_count`. 
    - `Avg` is a derived metric based on `Sum` and `Value_count` and is not indexed and is derived on query time. Rest of the base metrics are indexed.
- A maximum of `100` base metrics are supported per star-tree index.

If `Min`, `Max`, `Sum` and `Value_count` are defined as `metrics` for each field then up to 25 such fields can be configured, as shown in the following example:

```json
{
  "metrics": [
    {
      "name": "field1",
      "stats": [
        "sum",
        "value_count",
        "min",
        "max"
      ],
      ...,
      ...,
      "name": "field25",
      "stats": [
        "sum",
        "value_count",
        "min",
        "max"
      ]
    }
  ]
}
```


#### Properties

The `metrics` parameter supports the following properties:

| Parameter   | Required/Optional | Description  | 
| :--- | :--- | :--- |
| `name` | Required | The name of the field. The field name should be present in the `properties` section as part of index `mapping`. Ensure that the `doc_values` setting is `enabled` for any associated fields. |
| `stats` | Optional | A list of metric aggregations computed for each field. You can choose between `Min`, `Max`, `Sum`, `Avg`, and `Value Count`.<br/>Default is `Sum` and `Value_count`.<br/>`Avg` is a derived metric stat which will automatically be supported in queries if `Sum` and `Value_Count` are present as part of metric `stats`.

### Star tree configuration parameters

The following optional parameters can be configured with a star-tree index. These are final and cannot be modified post index creation.

| Parameter  | Description   | 
| :--- | :--- |
| `max_leaf_docs` | The maximum number of star-tree documents a leaf node can point to. After the maximum number is reached post which the nodes will be split to the next dimension. Default is `10000`. A lower value will use more storage but result in faster query performance. Inversely, a higher value will use less storage but result in slower query performance.  For more information, see [star tree indexing structure]({{site.url}}{{site.baseurl}}/search-plugins/star-tree-index/#star-tree-index-structure)  |
| `skip_star_node_creation_for_dimensions`  | A list of dimensions for which a star-tree index will skip creating the star node. When `true`, this reduces storage size at the expense of query performance. Default is `false`. For more information on star nodes, see [star-tree indexing structure]({{site.url}}{{site.baseurl}}/search-plugins/star-tree-index/#star-tree-index-structure) |

## Supported queries and aggregations

For more details on supported queries and aggregations, see [supported query and aggregations for Star Tree index]({{site.url}}{{site.baseurl}}/search-plugins/star-tree-index/#supported-query-and-aggregations)
