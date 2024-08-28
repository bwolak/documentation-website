---
layout: default
title: write_json
parent: Processors
grand_parent: Pipelines
nav_order: 56
---

# write_json


The `write_json` processor converts an object in an event into a JSON string. You can customize the processor to choose the source and target field names.

<!--
This table is autogenerated. Do not edit it.
- name: write_json
- pluginType: processor
- source: https://github.com/opensearch-project/data-prepper/blob/f0bd8d8e4773dc3d7318c19b8623c2213a099a86/data-prepper-plugins/write-json-processor/src/main/java/org/opensearch/dataprepper/plugins/processor/write_json/WriteJsonProcessorConfig.java
-->

Option | Description | Example
:--- | :--- | :---
source | Mandatory field that specifies the name of the field in the event containing the message or object to be parsed. | If `source` is set to `"message"` and the input is `{"message": {"key1":"value1", "key2":{"key3":"value3"}}}`, then the `write_json` processor outputs the event as `"{\"key1\":\"value1\",\"key2\":{\"key3\":\"value3\"}}"`.
target | An optional field that specifies the name of the field in which the resulting JSON string should be stored. If `target` is not specified, then the `source` field is used. | `key1`
