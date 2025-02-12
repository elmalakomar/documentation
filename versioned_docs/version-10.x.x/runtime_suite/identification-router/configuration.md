---
id: configuration
title: Configuration
sidebar_label: Configuration
---
The `Flow Manager Router` needs the following environment variables to work propertly:
- SUB_FLOW_CONFIGURATION_FILE_PATH path to configuration file with sub flow rules
- MAIN_FLOW_MANAGER_URL url to the `Main Flow Manager` service
- MAIN_SAGA_CRUD_URL url to the `CRUD Service` with the related subflow collection
- EXTERNAL_SERVICE_URL optional url to an external service to call when the `/notify` endpoint is called
- MAX_RETRIES  max number of retries performed by the service
- RETRIES_DELAY_MS delay in milliseconds between two retries

## SubFlows Configuration File

The configuration file has the following schema:

```json
{
  "type": "array",
  "items": {
    "type": "object",
    "required": [
      "id",
      "info",
      "rules"
    ],
    "properties": {
      "id": {
        "type": "string"
      },
      "info": {
        "type": "object",
        "properties": {
          "flowManagerUrl": {
            "type": "string"
          },
          "crudServiceUrl": {
            "type": "string"
          }
        }
      },
      "rules": {
        "type": "object"
      }
    }
  }
}
```

Below, an example of a valid configuration file:
```json
[
  {
    "id": "customRule",
    "info": {
      "flowManagerUrl": "http://custom-flow-manager-service",
      "crudServiceUrl": "http://crud-service/custom-saga-collection/"
    },
    "rules": {
      "someKey": "someValue",
      "nestedObject": {
        "nestedKey": "nestedValue"
      }
    }
  },
  {
    "id": "default",
    "info": {
      "flowManagerUrl": "http://default-flow-manager-service",
      "crudServiceUrl": "http://crud-service/default-saga-collection/"
    },
    "rules": {}
  }
]
```

## Kafka Configuration File

In order to enable Kafka usage, the following environment variables are required:
- MODE=`KAFKA`
- KAFKA_CONFIGURATION_FILE_PATH path to configuration file with kafka config

The configuration file has the following schema:

```json
{
  "type": "object",
  "required": [
    "brokers",
    "authMechanism",
    "username",
    "password",
    "consumerConfig",
    "producerConfig"
  ],
  "properties": {
    "brokers": {
      "type": "string"
    },
    "authMechanism": {
      "type": "string"
    },
    "username": {
      "type": "string"
    },
    "password": {
      "type": "string"
    },
    "consumerConfig": {
      "type": "object",
      "required": [
        "groupId",
        "eventsTopic",
        "notificationsTopic"
      ],
      "properties": {
        "groupId": {
          "type": "string"
        },
        "eventsTopic": {
          "type": "string"
        },
        "notificationsTopic": {
          "type": "string"
        }
      }
    },
    "producerConfig": {
      "type": "object",
      "required": [
        "mainFlowTopic"
      ],
      "properties": {
        "mainFlowTopic": {
          "type": "string"
        }
      }
    }
  }
}
```

Below, an example of a valid configuration file:
```json
{
  "brokers": "broker:9093",
  "authMechanism": "PLAIN",
  "username": "username",
  "password": "password",
  "consumerConfig": {
    "groupId": "router-0",
    "eventsTopic": "router-events",
    "notificationsTopic": "router-notification"
  },
  "producerConfig": {
    "mainFlowTopic": "main-flow"
  }
}
```
