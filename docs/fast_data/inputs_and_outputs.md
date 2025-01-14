---
id: inputs_and_outputs
title: Inputs and Outputs
sidebar_label: Inputs and Outputs
---

The Fast Data has an Event Driven Architecture, as such there are many events that make everything work. In this page we will explore these events in detail to understand better the whole Fast Data lifecycle.

## Projection

Here, we will discuss the inputs and outputs related to the Projection management.

### Ingestion Message

**Channel**: Apache Kafka

**Topic naming convention**: `<tenant>.<environment>.<source-system>.<projection>.ingestion`

Example: `test-tenant.PROD.system-name.test-projection.ingestion`

**Producer**: The CDC Connectors of the source databases

**Consumer**: Real-Time Updater

**Description**: The ingestion message is the message that allows us to mantain the Projections synchronized with the Source Databases since it contains the data of each record that gets inserted, updated or deleted.

When entering our systems, the message is read by the [Kafka Message Adapter](/fast_data/configuration/realtime_updater.md#kafka-adapters-kafka-messages-format) of the Real-Time Updater, which uses it to update the Projections.

Based on how the ingestion system is set up, the format can be one of three possible types:

* [IBM InfoSphere Data Replication for DB2](#ibm-infosphere-data-replication-for-db2)
* [Oracle Golden Gate](#oracle-golden-gate)
* [Debezium](#debezium)

You can also specify a [custom adapter](/fast_data/configuration/realtime_updater.md#custom) to handle any other message formats you need.
This format is always configurable in the System of Record page on the console, on the _Real-Time Updater_ tab.

Here's the AsyncApi specification of the message and some examples of the different formats.

#### IBM InfoSphere Data Replication for DB2

<details><summary>AsyncApi specification</summary>
<p>

```yaml
asyncapi: 2.6.0
info:
  title: Data Change API
  version: 1.0.0
channels:
  DB2:
    publish:
      message:
        name: DB2 data change message
        payload:
          type: object
          required:
            - key
            - value
          properties:
            key:
              type: object
              additionalProperties: true
              description: Record's primary keys
            value:
              type: object
              additionalProperties: true
              oneOf:
                - type: object
                  additionalProperties: true
                  description: Whole record (including primary and foreign keys) in case of *insert/update* operation
                - type: "null"
                  description: null in case of delete operation
          additionalProperties: false
```
</p>
</details>

Examples:

<details><summary>Upsert operation</summary>
<p>

```json
{
  "key": {
    "USER_ID": 123,
    "TAX_CODE": "ABCDEF12B02M100O"
  },
  "value": {
    "USER_ID": 123,
    "TAX_CODE": "ABCDEF12B02M100O",
    "NAME": 456
  }
}
```
</p>
</details>

<details><summary>Delete operation</summary>
<p>

```json
{
  "key": {
    "USER_ID": 123,
    "TAX_CODE": "ABCDEF12B02M100O"
  },
  "value": null
}
```
</p>
</details>

#### Oracle Golden Gate

<details><summary>AsyncApi specification</summary>
<p>

```yaml
asyncapi: 2.6.0
info:
  title: Data Change API
  version: "1.0.0"
channels:
  GoldenGate:
    publish:
      message:
        name: Golden Gate data change message
        payload:
          type: object
          required:
            - key
            - value
          properties:
            key: 
              type: string
              description: String of the primary keys values joined by underscores
              examples:
                - pkValue1_pkValue2
            value:
              type: object
              additionalProperties: false
              required:
                - op_type
                - pos
              properties:
                op_type:
                  type: string
                  description: Operation type. Can be *insert*, *update* or *delete*
                  enum:
                    - I
                    - U
                    - D
                before:
                  type: object
                  description: Whole record **before** the changes were applied (including the primary and foreign keys). This field is not defined with insert operation
                  additionalProperties: true
                after:
                  type: object
                  description: Whole record **after** the changes were applied (including the primary and foreign keys). This field is not defined with delete operation
                  additionalProperties: true
                pos:
                  type: integer
                  description: Position of the message, similar to the kafka message's offset
          additionalProperties: false
```
</p>
</details>

Examples:

<details><summary>Insert operation</summary>
<p>

```json
{
  "key": "123",
  "value": {
    "op_type": "I",
    "before": null,
    "after": {
      "USER_ID": 123,
      "TAX_CODE": "the-fiscal-code-123",
      "COINS": 300000000
    }
  }
}
```
</p>
</details>

<details><summary>Delete operation</summary>
<p>

```json
{
  "key": "123",
  "value": {
    "op_type": "D",
    "before": {
      "USER_ID": 123,
      "TAX_CODE": "the-fiscal-code-123",
      "COINS": 300000000
    },
    "after": null
  }
}
```
</p>
</details>

#### Debezium

<details><summary>AsyncApi specification</summary>
<p>

```yaml
asyncapi: 2.6.0
info:
  title: Data Change API
  version: 1.0.0
channels:
  Debezium:
    publish:
      message:
        name: Debezium data change message
        payload:
          type: object
          required:
            - key
            - value
          properties:
            key: 
              type: object
              description: Record's primary keys
              additionalProperties: true
            value:
              type: object
              required:
                - op
                - source
              properties:
                op:
                  type: string
                  description: Operation type. Can be *create/snapshot(r)*, *update* or *delete*
                  enum:
                    - c
                    - r
                    - u
                    - d
                before:
                  type: object
                  description: Whole record **before** the changes were applied (including the primary and foreign keys). This field is not defined with insert operation
                  additionalProperties: true
                after:
                  type: object
                  description: Whole record **after** the changes were applied (including the primary and foreign keys). This field is not defined with delete operation
                  additionalProperties: true
                source:
                  type: object
                  description: Metadata about the origin of the message (db, table, query...)
                  additionalProperties: true
              additionalProperties: false
          additionalProperties: false
```
</p>
</details>

Examples:

<details><summary>Insert operation</summary>
<p>

```json
{
  "key": {
    "USER_ID": 123,
    "TAX_CODE": "ABCDEF12B02M100O"
  },
  "value": {
    "op": "c",
    "before": null,
    "after": {
      "USER_ID": 123,
      "TAX_CODE": "the-fiscal-code-123",
      "COINS": 300000000
    }
  }
}
```
</p>
</details>

<details><summary>Delete operation</summary>
<p>

```json
{
  "key": {
    "USER_ID": 123,
    "TAX_CODE": "ABCDEF12B02M100O"
  },
  "value": {
    "op": "d",
    "before": {
      "USER_ID": 123,
      "TAX_CODE": "the-fiscal-code-123",
      "COINS": 300000000
    },
    "after": null
  }
}
```
</p>
</details>

### Projection Update Message

**Channel**: Apache Kafka

**Topic naming convention**: `<tenant>.<environment>.<mongo-database>.<collection>.pr-update`

Example: `test-tenant.PROD.restaurants-db.reviews-collection.pr-update`

**Producer**: Real-Time Updater

**Consumer**: Single View Trigger Generator or Single View Creator ([sv-patch](/fast_data/configuration/single_views.md#single-view-patch))

**Description**: The Projection Update or `pr-update` message informs the listener (typically the Single View Trigger Generator) that a Projection's record has been updated, inserted or deleted.

#### Message format v1.0.0

This version of Projection Record update is emitted by [Real-Time Updater](/fast_data/realtime_updater.md).

<details><summary>AsyncApi specification</summary>
<p>

```yaml
asyncapi: 2.6.0
info:
  title: Projection Update API
  version: 1.0.0
channels:
  pr-update:
    publish:
      message:
        name: Projection update message
        payload:
          type: object
          required:
            - header
            - key
            - value
          properties:
            headers:
              type: object
              required:
                - messageSchema
              properties:
                messageSchema:
                  type: object
                  required:
                    - type
                    - version
                  properties:
                    type:
                      type: string
                      description: Type of messsage (`pr-update`, `sv-update`, `sv-trigger`...)
                    version:
                      type: string
                      description: Version of the message format (v1.0.0)
            key:
              type: object
              description: Record's primary keys
              additionalProperties: true
            value:
              type: object
              required:
                - operationType
                - operationTimestamp
                - projectionName
                - source
                - primaryKeys
              properties:
                operationType:
                  type: string
                  enum: ["INSERT", "UPDATE", "DELETE", "UPSERT"]
                  description: Type of operation applied on the Projection's record
                operationTimestamp:
                  type: string
                  description: ISO 8601 String of the time at which the MongoDB operation on the projection's record has been carried out
                documentId:
                  description: Equals to the _id of the Projection's record on MongoDB
                  type: string
                projectionName:
                  description: Projection's name
                  type: string
                source:
                  description: Name of the System of Record
                  type: string
                primaryKeys:
                  description: Array of the primary key field names
                  type: array
                before:
                  type: object
                  description: Value of the MongoDB record **before** the changes have been applied. In case of a insert operation this field is not defined
                    **Note:** this field is always set to `null` when message adapter is set to `db2`
                  additionalProperties: true
                after:
                  type: object
                  description: Value of the MongoDB record **after** the changes have been applied. In case of a delete operation this field is not defined
                  additionalProperties: true
                __internal__kafkaInfo:
                  type: object
                  description: Metadata about the ingestion message that triggered the whole Fast Data flow
                  required:
                   - topic
                   - partition
                   - offset
                   - key
                   - timestamp
                  properties:
                    topic:
                      type: string
                      description: Ingestion topic's name
                    partition:
                      type: integer
                      description: Topic's partition
                    offset:
                      description: Message's offset
                      type: integer
                    key:
                      description: Message's key. The structure could be from any of the ingestion message key's formats
                    timestamp:
                      description: ISO 8601 date string of the kafka message timestamp
                      type: string
                  additionalProperties: false
              additionalProperties: false
          additionalProperties: false
```
</p>
</details>

Example:

<details><summary>Insert operation</summary>
<p>

```json
{
  "key": {
    "ID_USER": "ebc12dc8-939b-447e-88ef-6ef0b802a487"
  },
  "value": {
    "operationType":"INSERT",
    "operationTimestamp": "2022-05-20T10:25:56.401Z",
    "documentId": "62876cb2adb982a6195d26f9",
    "projectionName": "pr_registry",
    "source": "food-delivery",
    "primaryKeys":[
      "ID_USER"
    ],
    "after":{
      "ID_USER":"ebc12dc8-939b-447e-88ef-6ef0b802a487",
      "TAX_CODE":"tax_code",
      "NAME":"MARIO",
      "SURNAME":"ROSSI",
      "EMAIL":"email_mario",
      "ADDRESS":"address_1",
      "PHONE":"phone_number_1653042354472_last",
      "PROFESSION":"profession 1",
      "timestamp":"2022-05-20T10:25:56.323Z",
      "updatedAt":"2022-05-20T10:25:56.380Z",
      "__STATE__":"PUBLIC",
      "__internal__counter":467,
      "__internal__kafkaInfo":{
        "offset":"467",
        "partition":0,
        "timestamp":"2022-05-20T10:25:56.323Z",
        "topic":"kafka-topic-here",
        "key":{
          "ID_USER":"ebc12dc8-939b-447e-88ef-6ef0b802a487"
        }
      },
      "createdAt":"2022-05-20T10:25:56.380Z"
    },
    "__internal__kafkaInfo": {
      "offset": 467,
      "partition":0,
      "timestamp":"2022-05-20T10:25:56.323Z",
      "topic":"kafka-topic-here",
      "key":{
        "ID_USER":"ebc12dc8-939b-447e-88ef-6ef0b802a487"
      }
    }
  }
}
```
</p>
</details>

#### Message format v2.0.0

This version of Projection Record update is emitted by [Projection Storer](/fast_data/projection_storer.md).

:::note
Version `v2.0.0` of Projection Record update event is recognized by Single View Creator plugin (for SV Patch functionality) starting from version `v6.1.0`.
:::

<details><summary>AsyncApi specification</summary>
<p>

```yaml
asyncapi: 2.6.0
info:
  title: Projection Update Event
  version: 2.0.0
channels:
  pr-update:
    publish:
      message:
        name: Projection Record Update event
        payload:
          type: object
          required:
            - header
            - key
            - value
          properties:
            headers:
              type: object
              required:
                - messageSchema
              properties:
                messageSchema:
                  type: object
                  required:
                    - type
                    - version
                  properties:
                    type:
                      type: string
                      description: type of message
                      enum:
                        - pr-update
                        - sv-update
                    version:
                      const: v2.0.0
                      description: version of the message format (v2.0.0)
                context:
                  type: object
                  properties:
                    ingestion:
                      type: object
                      properties:
                        topic:
                          type: string
                          example: gt.fd-test.DEV.inventory.customers.ingestion
                        partitionID:
                          type: string
                          example: 3
                        offset:
                          type: number
                          example: 9001
                        timestamp:
                          type: string
                          format: date-time
                        pollTimestamp:
                          type: string
                          format: date-time
            key:
              type: object
              description: JSON object representation of the projection record identifier (fields that uniquely identify the records)
              additionalProperties: true
              example: {
                "customer_id": "24567",
                "fiscal_code": "99b81998-dd90-4bc8-8aea-62399f414d26"
              }
            value:
              type: object
              required:
                - operation
                - storageNamespace
                - primaryKeys
              properties:
                operation:
                  type: object
                  description: metadata regarding the operation that triggered this event
                  required:
                    - type
                    - timestamp
                  properties:
                    type:
                      type: string
                      description: the type of operation applied on the projection's record
                      enum:
                        - INSERT
                        - UPDATE
                        - DELETE
                    timestamp:
                      type: string
                      description: ISO8601 string marking the time when correspoding ingestion event has been processed
                      format: date-time
                source:
                  type: string
                  description: system of record name
                  example: inventory
                storageNamespace:
                  type: string
                  description: the projection name adopted on the storage system where the projection
                    record related to this has been stored - e.g. collection name on MongoDB database
                  example: pr_customers
                primaryKeys:
                  type: array
                  description: list of field names that compose the unique identifier of the projection
                    record
                  items:
                    type: string
                  example: [ "customer_id", "fiscal_code" ]
                before:
                  type: object
                  nullable: true
                  additionalProperties: true
                  description: the content as JSON object of the projection record before the operation
                    occurred - it may not be set
                    **Note:** this field is always set to `null` when message adapter is set to `db2`
                  example: null
                after:
                  type: object
                  nullable: true
                  additionalProperties: true
                  description: the content as JSON object of the projection record after the operation
                    occurred - it may not be set
                  example: {
                    "customer_id": "24567",
                    "fiscal_code": "99b81998-dd90-4bc8-8aea-62399f414d26",
                    "first_name": "Lara",
                    "last_name": "Croft",
                    "age": 28
                  }
```
</p>
</details>

Example:

<details><summary>Insert operation</summary>
<p>

```json
{
  "headers": {
    "messageSchema": {
      "type": "string",
      "version": "v2.0.0"
    },
    "context": {
      "ingestion": {
        "topic": "gt.fd-test.DEV.inventory.customers.ingestion",
        "partitionID": 3,
        "offset": 9001,
        "timestamp": "2019-08-24T14:15:22Z",
        "pollTimestamp": "2019-08-24T14:15:22Z"
      }
    }
  },
  "key": {
    "customer_id": "24567",
    "fiscal_code": "99b81998-dd90-4bc8-8aea-62399f414d26"
  },
  "value": {
    "operation": {
      "type": "INSERT",
      "timestamp": "2019-08-24T14:15:22Z"
    },
    "source": "inventory",
    "storageNamespace": "pr_customers",
    "primaryKeys": [
      "customer_id",
      "fiscal_code"
    ],
    "before": null,
    "after": {
      "customer_id": "24567",
      "fiscal_code": "99b81998-dd90-4bc8-8aea-62399f414d26",
      "first_name": "Lara",
      "last_name": "Croft",
      "age": 28
    }
  }
}
```
</p>
</details>

## Single View

This section covers the inputs and outputs concerning the Single View's aggregation.

### Projection Changes

**Channel**: MongoDB

**Producer**: Real-Time Updater or Single View Trigger Generator

**Consumer**: Single View Creator

**Description**: The Projection Changes or `pc` informs the listener (generally the Single View Creator) that a Single View should be updated.
This event is created as a result of a strategy execution.
It is stored on MongoDB and is very similar to the [Single View Trigger Message](#single-view-trigger-message) on Kafka.

<details><summary>JsonSchema specification</summary>
<p>

```json
{
  "type": "object",
  "required": [
    "identifier",
    "changes",
    "createdAt"
  ],
  "properties": {
    "type": {
      "type": "string",
      "description": "Identifier of for the Single View Creator service that should take care of the changes"
    },
    "identifier": {
      "type": "object",
      "description": "Identifier of the Projection Changes that should match with the Single View Keys fields or the identifierQueryMapping ones from the aggregation.json",
      "additionalProperties": true
    },
    "changes": {
      "type": "array",
      "description": "Array that keeps track of the changes requested for the Single View related to the identifier",
      "items": {
        "type": "object",
        "required": [
          "state"
        ],
        "properties": {
          "state": {
            "type": "string",
            "enum": [
              "NEW",
              "IN_PROGRESS",
              "ERROR"
            ],
            "description": "State of the change. State NEW means that the single view needs to be re-aggregated, state IN_PROGRESS means that the Single View Creator is already doing it, and ERROR means the Single View Creator encountered an error while trying to aggregate the Single Views"
          },
          "topic": {
            "type": "string",
            "description": "Ingestion topic that started the cycle"
          },
          "timestamp": {
            "type": "integer",
            "description": "Unix timestamp of the ingestion kafka message"
          },
          "partition": {
            "type": "integer",
            "description": "Partition number of the ingestion kafka message"
          },
          "offset": {
            "type": "integer",
            "description": "Offset of the ingestion kafka message"
          },
          "key": {
            "type": "object",
            "additionalProperties": true,
            "description": "Key of the ingestion kafka message"
          },
          "updatedAt": {
            "type": "object",
            "description": "MongoDB date object of the time the change has been updated",
            "additionalProperties": true
          },
          "inProgressAt": {
            "type": "object",
            "description": "MongoDB date object of the time the Single View Creator has started processing the change",
            "additionalProperties": true
          },
          "inErrorAt": {
            "type": "object",
            "description": "MongoDB date object of the time the Single View Creator encountered an error while aggregating",
            "additionalProperties": true
          }
        },
        "additionalProperties": true
      }
    },
    "createdAt": {
      "type": "object",
      "description": "MongoDB date object of the time the record has been created",
      "additionalProperties": true
    },
    "updatedAt": {
      "type": "object",
      "description": "MongoDB date object of the time the record has been updated, most of the times it means the moment when a change has been registered.",
      "additionalProperties": true
    },
    "doneAt": {
      "type": "object",
      "description": "MongoDB date object of the time that the last change was processed by the Single View Creator",
      "additionalProperties": true
    }
  }
}
```
</p>
</details>

Example:

<details><summary>MongoDB record</summary>
<p>

```json
{
  "_id": "627935df1810010012b0a328",
  "identifier": {
    "ID_USER": "ebc12dc8-939b-447e-88ef-6ef0b802a487"
  },
  "type": "sv_customers",
  "changes": [
    {
      "state": "NEW",
      "topic": "original-topic-2",
      "timestamp": 1234567,
      "partition": 0,
      "offset": 2,
      "key": {
        "originalKey2": "123",
      }
    }
  ],
  "createdAt": "2022-05-20T10:25:35.656Z",
  "updatedAt": "2022-05-20T10:25:35.656Z",
  "doneAt": "2022-05-20T10:25:35.656Z"
}
```
</p>
</details>

### Kafka Projection Changes

:::caution
This method is deprecated in favor of [sv-trigger](#single-view-trigger-message) or [Projection Changes](#projection-changes) and it will be removed in the next major release.
:::

**Channel**: Apache Kafka

**Producer**: Real-Time Updater

**Consumer**: Single View Creator

**Description**: Projection changes can also be sent to kafka when enabling the GENERATE_KAFKA_PROJECTION_CHANGES environment variable in the Real-Time Updater.

<details><summary>AsyncApi specification</summary>
<p>

```yaml
asyncapi: 2.6.0
info:
  title: Kafka Projection Changes API
  version: 1.0.0
channels:
  projectionChanges:
    subscribe:
      message:
        name: Kafka Projection Changes message
        payload:
          type: object
          required:
            - key
            - value
          properties:
            key:
              type: object
              description: Identifier of the Single View or the Projection
              additionalProperties: true
            value:
              type: object
              required:
                - identifier
                - __internal__kafkaInfo
              properties:
                identifier:
                  type: object
                  description: Identifier of the Single View or the Projection
                  additionalProperties: true
                __internal__kafkaInfo:
                  type: object
                  description: Metadata about the ingestion message that triggered the whole Fast Data flow
                  required:
                    - topic
                    - partition
                    - offset
                    - key
                    - timestamp
                  properties:
                    topic:
                      type: string
                      description: Ingestion topic's name
                    partition:
                      type: integer
                      description: Topic's partition
                    offset:
                      description: Message's offset
                      type: integer
                    key:
                      description: Message's key. The structure could be from any of the ingestion message key's formats
                    timestamp:
                      description: ISO 8601 date string of the ingestion Message
                      type: string
                  additionalProperties: false
          additionalProperties: false
```
</p>
</details>

Example:

<details><summary>Kafka PC message</summary>
<p>

```json
{
  "key": {
    "ID_USER": "ebc12dc8-939b-447e-88ef-6ef0b802a487"
  },
  "value": {
    "identifier": {
      "ID_USER": "ebc12dc8-939b-447e-88ef-6ef0b802a487"
    },
    "__internal__kafkaInfo": {
      "topic": "original-topic-2",
      "partition": 1,
      "offset": 2,
      "key": {
        "originalKey2": "123",
      },
      "timestamp": "1684290004852"
    }
  }
}
```
</p>
</details>

### Single View Trigger Message

**Channel**: Apache Kafka

**Topic naming convention**: `<tenant>.<environment>.<mongo-database>.<single-view-name>.sv-trigger`

Example: `test-tenant.PROD.restaurants-db.reviews-sv.sv-trigger`

**Producer**: Single View Trigger Generator

**Consumer**: Single View Creator

**Description**: The Single View Trigger Message or `sv-trigger` informs the listener that a Single View must be regenerated. This event is also created as a result of a strategy execution so you should configure your Fast Data system to generate either Single View Trigger Messages or [Projection Changes](#projection-changes).

<details><summary>AsyncApi specification</summary>
<p>

```yaml
asyncapi: 2.6.0
info:
  title: Single View Trigger API
  version: 1.0.0
channels:
  sv-trigger:
    subscribe:
      message:
        name: Single View Trigger message
        payload:
          type: object
          required:
            - key
            - value
          properties:
            key:
              type: object
              description: Identifier of the Single View
              additionalProperties: true
            value:
              type: object
              required:
                - singleViewIdentifier
                - __internal__kafkaInfo
                - change
              properties:
                singleViewName:
                  type: string
                  description: Name of the Single View
                singleViewIdentifier:
                  type: object
                  description: Identifier of the Single View just like the Projection Changes Identifier
                  additionalProperties: true
                retry:
                  type: object
                  properties:
                    lastError:
                      type: object,
                      description: Details of the last error that made the aggregation fail
                      properties:
                        type:
                          type: string
                        message:
                          type: string
                    attempts:
                      type: number
                      description: Number of times the aggreation of the Single View has been retried
                change:
                  type: object
                  description: Contains information about the projection record that triggered the strategy
                  additionalProperties: false
                  properties:
                    projectionName:
                      description: Name of the projection
                      type: string
                    projectionIdentifier:
                      type: object
                      description: Primary keys of the projection record
                      additionalProperties: true
                    data:
                      type: object
                      description: Data of the projection record after the changes were applied
                      additionalProperties: true
                __internal__kafkaInfo:
                  type: object
                  description: Metadata about the ingestion message that triggered the whole Fast Data flow
                  required:
                    - topic
                    - partition
                    - offset
                    - key
                    - timestamp
                  properties:
                    topic:
                      type: string
                      description: Ingestion topic's name
                    partition:
                      type: integer
                      description: Topic's partition
                    offset:
                      description: Message's offset
                      type: integer
                    key:
                      description: Message's key. The structure could be from any of the ingestion message key's formats
                    timestamp:
                      description: ISO 8601 date string of the ingestion Message
                      type: string
                  additionalProperties: false
          additionalProperties: false
```
</p>
</details>

Examples:

<details><summary>Trigger message</summary>
<p>

```json
{
  "key": {
    "bookId": "29EMA5BtaKhM6fipPIRDJWec"
  },
  "value": {
    "type": "aggregation",
    "singleViewName": "sv_books",
    "singleViewIdentifier": {
      "bookId": "29EMA5BtaKhM6fipPIRDJWec"
    },
    "change": {
      "data": {
        "__STATE__": "PUBLIC",
        "__internal__counter": 1685118744745,
        "__internal__counterType": "timestamp",
        "__internal__kafkaInfo": {
          "key": {
            "authorId": "7P3P9Pag59nxpOhfNMIweE0H"
          },
          "offset": "151",
          "partition": 0,
          "timestamp": "2023-05-26T16:32:24.745Z",
          "topic": "some.ingestion.topic"
        },
        "authorId": "7P3P9Pag59nxpOhfNMIweE0H",
        "bio": "episode lover, designer",
        "name": "Caitlyn",
        "surname": "Hettinger",
        "timestamp": "2023-05-26T16:32:24.745Z",
        "updatedAt": "2023-05-26T16:32:24.845Z"
      },
      "projectionIdentifier": {
        "authorId": "7P3P9Pag59nxpOhfNMIweE0H"
      },
      "projectionName": "authors"
    },
    "__internal__kafkaInfo": {
      "key": {
        "authorId": "7P3P9Pag59nxpOhfNMIweE0H"
      },
      "offset": "151",
      "partition": 0,
      "timestamp": "2023-05-26T16:32:24.745Z",
      "topic": "some.ingestion.topic"
    }
  }
}
```
</p>
</details>

<details><summary>Trigger message with retries</summary>
<p>

```json
{
  "key": {
    "bookId": "29EMA5BtaKhM6fipPIRDJWec"
  },
  "value": {
    "type": "aggregation",
    "singleViewName": "sv_books",
    "singleViewIdentifier": {
      "bookId": "29EMA5BtaKhM6fipPIRDJWec"
    },
    "retry": {
      "attempts": 5,
      "lastError": {
        "type": "SINGLE_VIEW_AGGREGATION_MAX_TIME",
        "message": "Aggregation exceeding configured time limit"
      }
    },
    "change": {
      "data": {
        "__STATE__": "PUBLIC",
        "__internal__counter": 1685118744745,
        "__internal__counterType": "timestamp",
        "__internal__kafkaInfo": {
          "key": {
            "authorId": "7P3P9Pag59nxpOhfNMIweE0H"
          },
          "offset": "151",
          "partition": 0,
          "timestamp": "2023-05-26T16:32:24.745Z",
          "topic": "some.ingestion.topic"
        },
        "authorId": "7P3P9Pag59nxpOhfNMIweE0H",
        "bio": "episode lover, designer",
        "name": "Caitlyn",
        "surname": "Hettinger",
        "timestamp": "2023-05-26T16:32:24.745Z",
        "updatedAt": "2023-05-26T16:32:24.845Z"
      },
      "projectionIdentifier": {
        "authorId": "7P3P9Pag59nxpOhfNMIweE0H"
      },
      "projectionName": "authors"
    },
    "__internal__kafkaInfo": {
      "key": {
        "authorId": "7P3P9Pag59nxpOhfNMIweE0H"
      },
      "offset": "151",
      "partition": 0,
      "timestamp": "2023-05-26T16:32:24.745Z",
      "topic": "some.ingestion.topic"
    }
  }
}
```
</p>
</details>

### Single View Update Message

**Channel**: Apache Kafka

**Topic naming convention**: `<tenant>.<environment>.<mongo-database>.<single-view-name>.sv-update`

Example: `test-tenant.PROD.restaurants-db.reviews-sv.sv-update`

**Producer**: Single View Creator

**Consumer**: Custom (whoever needs it)

**Description**: The Single View Update or `sv-update` informs the listener that a specific Single View record has been updated. This is used generally for statistical purposes, like knowing how many Single Views per minute our system can process, but it can also be used to keep a history of the changes since it can contain (although disabled by the default) the before and after values of the Single View record.

#### Message format v1.0.0

This version of Single View Update event is emitted by default in Single View Creator version `v6.x.x`.

<details><summary>AsyncApi specification</summary>
<p>

```yaml
asyncapi: 2.6.0
info:
  title: Single View Update API
  version: 1.0.0
channels:
  sv-update:
    subscribe:
      message:
        name: Single View Update message
        payload:
          type: object
          required:
            - key
            - value
          properties:
            key:
              type: object
              description: Primary keys of the updated Single View
              additionalProperties: true
            value:
              type: object
              required:
                - operationType
                - operationTimestamp
                - singleViewName
                - source
                - __internal__kafkaInfo
              properties:
                operationType:
                  type: string
                  description: Type of operation applied to the Single View record
                  enum:
                    - INSERT
                    - UPDATE
                    - DELETE
                operationTimestamp:
                  type: string
                  description: ISO 8601 date string of the ingestion Message
                documentId:
                  type: string
                  description: MongoDB _id of the Single View document
                singleViewName:
                  type: string
                  description: Single View name of the record
                source:
                  type: string
                  description: Equivalent to the SINGLE_VIEWS_PORTFOLIO_ORIGIN env var of the Single View Creator that took care of the Single View aggregation
                before:
                  type: object
                  description: Value of the Single View record **before** it was updated/deleted. In case of an insert the field won't be defined. Mind that both before and after values won't be defined by default so you need to configure the Single View Creator if you wish to include them
                  additionalProperties: true
                after:
                  type: object
                  description: Value of the Single View record **after** it was updated/inserted. In case of a delete the field won't be defined. Mind that both before and after values won't be defined by default so you need to configure the Single View Creator if you wish to include them
                  additionalProperties: true
                __internal__kafkaInfo:
                  type: object
                  description: Metadata about the ingestion message that triggered the whole Fast Data flow
                  required:
                    - topic
                    - partition
                    - offset
                    - key
                    - timestamp
                  properties:
                    topic:
                      type: string
                      description: Ingestion topic's name
                    partition:
                      type: integer
                      description: Topic's partition
                    offset:
                      description: Message's offset
                      type: integer
                    key:
                      description: Message's key. The structure could be from any of the ingestion message key's formats
                    timestamp:
                      description: ISO 8601 date string of the ingestion Message
                      type: string
                  additionalProperties: false
              additionalProperties: false   
          additionalProperties: false
```
</p>
</details>

Example:

<details><summary>Update message</summary>
<p>

```json
{
  "key": {
    "bookId": "9GQ4btTZk9L3bOKybn973Ph2"
  },
  "value": {
    "__internal__kafkaInfo": {
      "key": {
        "libraryId": "roG7Hrmobde8PImFo3KCIRBp"
      },
      "offset": 123,
      "partition": 0,
      "timestamp": "2023-06-01T10:05:51.442Z",
      "topic": "topic.libraries.ingestion"
    },
    "operationTimestamp": "2023-06-01T10:05:51.442Z",
    "operationType": "UPDATE",
    "singleViewName": "sv_books",
    "source": "library"
  }
}
```
</p>
</details>

#### Message format v2.0.0

:::note
Version `v2.0.0` of Single Update event is available starting from version `v6.3.0` of Single View Creator plugin. 
:::

This version of Single View Update event is emitted when service environment variable `SV_UPDATE_VERSION` is set to `v2.0.0`.
In future major releases, this will become the default message format.

An important feature of this new message format is the fact that it is compatible with Projection Update events. In this manner
`sv-update` events regarding the generation of a Single View can be employed to start computing the trigger for another Single View,
fundamentally enabling daisy-chaining the creation of two or more Single View.

<details><summary>AsyncApi specification</summary>
<p>

```yaml
asyncapi: 2.6.0
info:
  title: Single View Update API
  version: 2.0.0
channels:
  sv-update:
    publish:
      message:
        name: Single View Update event
        payload:
          type: object
          required:
            - header
            - key
            - value
          properties:
            headers:
              type: object
              required:
                - messageSchema
              properties:
                messageSchema:
                  type: object
                  required:
                    - type
                    - version
                  properties:
                    type:
                      type: string
                      description: type of messsage
                      enum:
                        - sv-update
                    version:
                      const: v2.0.0
                      description: version of the message format (v2.0.0)
                context:
                  type: object
                  properties:
                    ingestion:
                      type: object
                      properties:
                        topic:
                          type: string
                          example: gt.fd-test.DEV.customers.sv-update
                        partitionID:
                          type: string
                          example: 0
                        offset:
                          type: number
                          example: 9001
                        timestamp:
                          type: string
                          format: date-time
                        pollTimestamp:
                          type: string
                          format: date-time
            key:
              type: object
              description: JSON object representation of the Single View record identifier (fields that uniquely identify each record)
              additionalProperties: true
              example: {
                "customerId": "24567",
                "fiscalCode": "99b81998-dd90-4bc8-8aea-62399f414d26"
              }
            value:
              type: object
              required:
                - operation
                - storageNamespace
                - primaryKeys
              properties:
                operation:
                  type: object
                  description: metadata regarding the CDC operation that triggered this event
                  required:
                    - type
                    - timestamp
                  properties:
                    type:
                      type: string
                      description: the type of operation applied on the single view record
                      enum:
                        - INSERT
                        - UPDATE
                        - DELETE
                    timestamp:
                      type: string
                      description: ISO8601 string marking the time when correspoding ingestion event has been processed
                      format: date-time
                source:
                  type: string
                  description: system of record name
                  example: inventory
                storageNamespace:
                  type: string
                  description: the name of the namespace on the storage system where the Single View record is saved - e.g. MongoDB collection name
                  example: sv_customers
                primaryKeys:
                  type: array
                  description: list of field names that compose the Single View record unique identifier
                  items:
                    type: string
                  example: [ "customerId", "fiscalCode" ]
                before:
                  type: object
                  nullable: true
                  additionalProperties: true
                  description: the content as JSON object of the Single View record before the operation occurred - it may not be set
                  example: null
                after:
                  type: object
                  nullable: true
                  additionalProperties: true
                  description: the content as JSON object of the Single View record after the operation occurred - it may not be set
                  example: {
                    "customerId": "24567",
                    "fiscalCode": "99b81998-dd90-4bc8-8aea-62399f414d26",
                    "firstName": "Lara",
                    "lastName": "Croft",
                    "age": 28,
                    "orders": [
                      {
                        "name": "medi-kit",
                        "price": {
                          "amount": 10,
                          "currency": "USD"
                        },
                        "quantity": 5,
                        "date": "2023-11-27T10:23:00.579Z"
                      }
                    ]
                  }
```
</p>
</details>

Example:

<details><summary>Update message</summary>
<p>

```json
{
  "headers": {
    "messageSchema": {
      "type": "sv-update",
      "version": "v2.0.0"
    },
    "context": {
      "ingestion": {
        "topic": "gt.fd-test.DEV.customers.sv-update",
        "partitionID": 0,
        "offset": 9001,
        "timestamp": "2019-08-24T14:15:22Z",
        "pollTimestamp": "2019-08-24T14:15:22Z"
      }
    }
  },
  "key": {
    "customerId": "24567",
    "fiscalCode": "99b81998-dd90-4bc8-8aea-62399f414d26"
  },
  "value": {
    "operation": {
      "type": "INSERT",
      "timestamp": "2019-08-24T14:15:22Z"
    },
    "source": "inventory",
    "storageNamespace": "sv_customers",
    "primaryKeys": [
      "customerId",
      "fiscalCode"
    ],
    "before": null,
    "after": {
      "customerId": "24567",
      "fiscalCode": "99b81998-dd90-4bc8-8aea-62399f414d26",
      "firstName": "Lara",
      "lastName": "Croft",
      "age": 28,
      "orders": [
        {
          "name": "medi-kit",
          "price": {
            "amount": 10,
            "currency": "USD"
          },
          "quantity": 5,
          "date": "2023-11-27T10:23:00.579Z"
        }
      ]
    }
  }
}
```
</p>
</details>


### Single View Error

**Channel**: MongoDB

**Producer**: Single View Creator

**Consumer**: Custom (whoever needs it)

**Description**: A Single View Error is an event that warns us something went wrong with the aggregation of a Single View in the Single View Creator.

<details><summary>JsonSchema specification</summary>
<p>

```json
{
  "type": "object",
  "required": [
    "portfolioOrigin",
    "type",
    "errorType",
    "errorMessage",
    "identifier",
    "resolutionMethod"
  ],
  "properties": {
    "portfolioOrigin": {
      "type": "string",
      "description": "Equivalent to the SINGLE_VIEWS_PORTFOLIO_ORIGIN env var of the Single View Creator that generated the error"
    },
    "type": {
      "type": "string",
      "description": "Name of the Single View"
    },
    "identifier": {
      "type": "object",
      "description": "Identifier of the Projection Changes that should match with the Single View Keys fields or the identifierQueryMapping ones from the aggregation.json"
    },
    "errorType": {
      "type": "string",
      "enum": [
        "NO_SV_GENERATED",
        "VALIDATION_ERROR",
        "UNKNOWN_ERROR",
        "ERROR_SEND_SVC_EVENT",
        "SINGLE_VIEW_AGGREGATION_MAX_TIME"
      ],
      "description": "The cause of the error"
    },
    "errorMessage": {
      "type": "string",
      "description": "Further description of the error"
    },
    "resolutionMethod": {
      "type": "string",
      "enum": [
        "AGGREGATION",
        "PATCH"
      ],
      "description": "System that the Single View Creator was using to update the Single Views"
    },
    "createdAt": {
      "type": "object",
      "description": "MongoDB date object of the time the record has been created",
      "additionalProperties": true
    },
    "updatedAt": {
      "type": "object",
      "description": "MongoDB date object of the time the record has been updated",
      "additionalProperties": true
    },
  },
  "additionalProperties": false
}
```
</p>
</details>

Example:

<details><summary>MongoDB record</summary>
<p>

```json
{
  "_id": "64426177a879bbfec4eaed0f",
  "portfolioOrigin": "food-delivery",
  "type": "sv_customers",
  "identifier": {
    "ID_USER": "ebc12dc8-939b-447e-88ef-6ef0b802a487"
  },
  "errorType": "NO_SV_GENERATED",
  "errorMessage": "Unexpected error: No Single View record generated",
  "createdAt": "2022-05-20T10:25:35.656Z",
  "updatedAt": "2022-05-20T10:25:35.656Z",
  "resolutionMethod": "AGGREGATION"
}
```
</p>
</details>

### Single View Events Message

:::caution
This method is deprecated in favor of [sv-update](#single-view-update-message) and it will be removed in the next major release.
:::

**Channel**: Apache Kafka

**Topic naming convention**: `<tenant>.<environment>.<mongo-database>.<single-view-name>.svc-events`

Example: `test-tenant.PROD.restaurants-db.reviews-sv.svc-events`

**Producer**: Single View Creator

**Consumer**: Custom (whoever needs it)

**Description**: The Single View Events or `svc-events` informs the listener that a single view has been successfully updated.

<details><summary>AsyncApi specification</summary>
<p>

```yaml
asyncapi: 2.6.0
info:
  title: Single View Events API
  version: 1.0.0
channels:
  svc-events:
    subscribe:
      message:
        name: Single View Events message
        payload:
          type: object
          required:
            - key
            - headers
            - value
          properties:
            key:
              type: object
              description: Primary keys of the updated Single View
              additionalProperties: true
            headers:
              type: object
              required:
                - type
                - name
              properties:
                type:
                  type: string
                  enum:
                    - event
                name:
                  type: string
                  description: Operation type or outcome
                  enum:
                    - singleViewCreated
                    - singleViewUpdated
                    - singleViewDeleted
              additionalProperties: false
            value:
              type: object
              required:
                - type
                - portfolioOrigin
                - __internal__kafkaInfo
              properties:
                type:
                  type: string
                  description: The name of the Single View
                portfolioOrigin:
                  type: string
                  description: Equivalent to the SINGLE_VIEWS_PORTFOLIO_ORIGIN env var of the Single View Creator that generated the message
                __internal__kafkaInfo:
                  type: object
                  description: Metadata about the ingestion message that triggered the whole Fast Data flow
                  required:
                    - topic
                    - partition
                    - offset
                    - key
                    - timestamp
                  properties:
                    topic:
                      type: string
                      description: Ingestion topic's name
                    partition:
                      type: integer
                      description: Topic's partition
                    offset:
                      description: Message's offset
                      type: integer
                    key:
                      description: Message's key. The structure could be from any of the ingestion message key's formats
                    timestamp:
                      description: ISO 8601 date string of the kafka message timestamp
                      type: string
                  additionalProperties: false
              additionalProperties: false
          additionalProperties: false
```
</p>
</details>

Example:

<details><summary>Events example</summary>
<p>

```json
{
  "key": {
    "idCustomer": "ebc12dc8-939b-447e-88ef-6ef0b802a487"
  },
  "value": {
    "type": "sv_customers",
    "portfolioOrigin": "food-delivery",
    "__internal__kafkaInfo": {
      "topic": "kafka-topic-here",
      "partition": 0,
      "key": "Amatriciana_id",
      "offset": "466",
      "timestamp": "1653039238727",
    },
  }
}
```
</p>
</details>

### Single View Before After Message

:::caution
This method is deprecated in favor of [sv-update](#single-view-update-message) and it will be removed in the next major release.
:::

**Channel**: Apache Kafka

**Topic naming convention**: `<tenant>.<environment>.<mongo-database>.<single-view-name>.sv-before-after`

Example: `test-tenant.PROD.restaurants-db.reviews-sv.sv-before-after`

**Producer**: Single View Creator

**Consumer**: Custom (whoever needs it)

**Description**: The Single View Before After Message is an additional event used for debugging purposes, which contains both the previous and the current state of the Single View once it has been updated.

<details><summary>AsyncApi specification</summary>
<p>

```yaml
asyncapi: 2.6.0
info:
  title: Single View Before After API
  version: 1.0.0
channels:
  sv-before-after:
    subscribe:
      message:
        name: Single View Before After message
        payload:
          type: object
          required:
            - key
            - value
          additionalProperties: false
          properties:
            key:
              type: object
              description: Primary keys of the updated Single View
              additionalProperties: true
            value:
              type: object
              required:
                - key
                - type
                - opType
                - __internal__kafkaInfo
              properties:
                key:
                  type: object
                  description: Primary keys of the updated Single View
                  additionalProperties: true
                type:
                  type: string
                  description: Name of the Single View collection
                opType:
                  type: string
                  description: Operation performed by the Single View Creator
                  enum:
                    - NON_EXISTING_SV
                    - INSERT_SV
                    - DELETE_SV
                    - UPDATE_SV
                before:
                  type: object
                  description: The value of the Single View before the change occurred. Mind that in case of an insert this field won't be defined.
                  additionalProperties: true
                after:
                  type: object
                  description: The value of the Single View before the change occurred. Mind that in case of a delete this field won't be defined.
                  additionalProperties: true
                __internal__kafkaInfo:
                  type: object
                  description: Metadata about the ingestion message that triggered the whole Fast Data flow
                  required:
                    - topic
                    - partition
                    - offset
                    - key
                    - timestamp
                  properties:
                    topic:
                      type: string
                      description: Ingestion topic's name
                    partition:
                      type: integer
                      description: Topic's partition
                    offset:
                      description: Message's offset
                      type: integer
                    key:
                      description: Message's key. The structure could be from any of the ingestion message key's formats
                    timestamp:
                      description: ISO 8601 date string of the kafka message timestamp
                      type: string
                  additionalProperties: false
              additionalProperties: false
```
</p>
</details>

Example:

<details><summary>Update operation</summary>
<p>

```json
{
  "key": { 
    "idCustomer": "ebc12dc8-939b-447e-88ef-6ef0b802a487"
  },
  "value": {
    "key": {
      "idCustomer": "ebc12dc8-939b-447e-88ef-6ef0b802a487"
    },
    "before": {
      "_id": "6287a74d2931f4cc7356e505",
      "idCustomer": "ebc12dc8-939b-447e-88ef-6ef0b802a487",
      "taxCode": "tax_code",
      "name": "MARIO",
      "surname": "ROSSI",
      "email": "email_mario",
      "address": "address_1",
      "telephone": "phone_number_1653057355131_last",
      "updatedAt": "2022-05-20T14:35:58.943Z",
      "__STATE__": "PUBLIC",
      "__internal__kafkaInfo": {
        "topic": "kafka-topic-here",
        "partition": 0,
        "key": "Amatriciana_id",
        "offset": "475",
        "timestamp": "1653057358783"
      }
    },
    "after": {
      "idCustomer": "ebc12dc8-939b-447e-88ef-6ef0b802a487",
      "taxCode": "tax_code",
      "name": "MARIO",
      "surname": "ROSSI",
      "email": "email_mario",
      "address": "address_1",
      "telephone": "phone_number_1653057355131_last",
      "updatedAt": "2022-05-20T14:35:59.488Z",
      "__STATE__": "PUBLIC",
      "__internal__kafkaInfo": {
        "topic": "kafka-topic-here",
        "partition": 0,
        "key": "Amatriciana_id",
        "offset": "476",
        "timestamp": "1653057359355"
      }
    },
    "type": "sv_customers",
    "__internal__kafkaInfo": {
      "topic": "kafka-topic-here",
      "partition": 0,
      "key": "Amatriciana_id",
      "offset": "476",
      "timestamp": "1653057359355"
    },
    "opType": "UPDATE_SV"
  }
}
```
</p>
</details>
