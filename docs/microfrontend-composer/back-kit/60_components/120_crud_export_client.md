---
id: crud_export_client
title: CRUD Export Client
sidebar_label: CRUD Export Client
---

<!--
WARNING: this file was automatically generated by Mia-Platform Doc Aggregator.
DO NOT MODIFY IT BY HAND.
Instead, modify the source file and run the aggregator to regenerate this file.
-->

<!--
WARNING:
This file is automatically generated. Please edit the 'README' file of the corresponding component and run `yarn copy:docs`
-->

[crud-service]: /runtime_suite/crud-service/10_overview_and_usage.md
[export-service]: /runtime_suite/export-service/10_overview.md

[resource]: https://jimmywarting.github.io/StreamSaver.js/mitm.html?version=2.0.0

[data-schema]: ../30_page_layout.md#data-schema

[bk-table]: ./510_table.md
[bk-gallery]: ./360_gallery.md
[bk-crud-client]: ./100_crud_client.md

[export-data]: ../70_events.md#export-data
[count-data]: ../70_events.md#count-data
[select-data-bulk]: ../70_events.md#select-data-bulk
[change-query]: ../70_events.md#change-query
[export-data/user-config]: ../70_events.md#export-data---user-config
[export-data/awaiting-config]: ../70_events.md#export-data---request-config
[success]: ../70_events.md#success
[error]: ../70_events.md#error



```html
<bk-export-client></bk-export-client>
```

Frontend client for [Mia Platform Export Service][export-service].

<!-- TODO link export flow -->

This component implements the Export Service interface on top of `fetch` http client on the browser.

It listens the specific events to record the current state of the page:

1. filters and query changes (that modify data shown by the [CRUD Client][bk-crud-client])
2. counts the currently filtered items
3. counts and records items selection (on components such as the [Table][bk-table] or the [Gallery][bk-gallery])

According with `Export Service` it provides 2 modes:

1. CSV
2. Excel

To open an export transaction it listens to an [export-data] event and return an [export-data/awaiting-config]
event which carries along the following payload

```typescript
export type AwaitUserConfig = {
  total?: number
  selected?: number
  columns: Option[]
}
```

where
- `total` is the last count of queried items,
- `selected` is the count of currently selected items and
- `columns` are selectable columns from the data-schema

a `Meta` contains the `transactionId` and must be re-cast when options are selected. An [export-data/user-config]
event must then follow with payload

```typescript
export type ExportUserConfig = {
  exportType: 'csv' | 'xlsx'
  csvSeparator?: 'COMMA' | 'SEMICOLON'
  filters: 'all' | 'filtered' | 'selected'
  columns: string[]
}
```

Once the config is received, the http client calls the Export Service and the download is completed natively by
a service worker registered within the browser. The UI is not locked.

## How to configure


To configure the CRUD Export Client, properties `basePath`, `exportInternalUrl`, `dataSchema` should be specified.

```json
{
  "tag": "bk-export-client",
  "properties": {
    "basePath": "/export",
    "exportInternalUrl": "http://crud-service/customers/export",
    "dataSchema": {
      "type": "object",
      "properties": {
        "_id": {"type": "string"},
        "__STATE__": {"type": "string"},
        "name": {"type": "string"}
      }
    }
  }
}
```

- `basePath` is the endpoint to target for triggering data export, i.e. the Export Service endpoint
- `exportInternalUrl` is the internal endpoint of the [CRUD Service][crud-service] collection to export
- `dataSchema` provides information on the structure of the data of the CRUD Service collection

Furthermore, a primary key should be specified for the targeted collection via property `primaryKey`, which defaults to `_id`.

### Stream Saver

In order to handle large sized file, a service worker is registered to perform local storage persistance operations instead of using large chunks of memory. To do so an external [resource] is needed. You can also use the same resource hosted with `back-kit` JS bundle available at `<back-kit endpoint>/export-service-worker.html`.
In the latter case set `streamSaverIFrameSrc` to the resource endpoint.


```json
{
  "tag": "bk-export-client",
  "properties": {
    "basePath": "/export",
    "exportInternalUrl": "http://crud-service/customers/export",
    "dataSchema": {
      "type": "object",
      "properties": {
        "_id": {"type": "string"},
        "__STATE__": {"type": "string"},
        "name": {"type": "string"}
      }
    },
    "streamSaverIFrameSrc": "/back-kit/{{BACK_KIT_VERSION}}/export-service-worker.html"
  }
}
```

<!-- ###
To allow notifications in case of failure an `error` event is triggered.
Add to `bk-notifications` the following error trigger
```json
{
  ...,
  "errorEventMap": {
    "export-data": {
      "title": "Error",
      "content": "An error occurred while exporting data"
    },
    ...
  }
}
``` -->

## API

### Properties & Attributes

| property               | attribute                 | type                                         | default | description                                                                       |
| ---------------------- | ------------------------- | -------------------------------------------- | ------- | --------------------------------------------------------------------------------- |
| `basePath`             | -                         | string                                       | -       | the URL base path to which to send HTTP requests                                  |
| `headers`              | -                         | {[key: string]: string}                      | -       | headers to add when an HTTP request is sent                                       |
| `credentials`          | -                         | 'include'\|'omit'\|'same-origin'             | -       | credentials policy to apply to HTTP requests                                      |
| `exportInternalUrl`    | `export-internal-url`     | string                                       | -       | url to be called internally to get `jsonl` formatted data                         |
| `primaryKey`           | `primary-key`             | string                                       | '_id'   | primary key to filter selected data when `selected only export` option is enabled |
| `streamSaverIFrameSrc` | `stream-saver-iframe-src` | string                                       | -       | location where stream saver service worker files are served                       |
| `dataSchema`           | -                         | [ExtendedJSONSchema7Definition][data-schema] | -       | data-schema describing the fields structure of the CRUD collection                |

### Listens to

| event                     | action                                                                           | emits                         | on error |
| ------------------------- | -------------------------------------------------------------------------------- | ----------------------------- | -------- |
| [export-data]             | opens a new export transaction                                                   | [export-data/awaiting-config] | -        |
| [export-data/user-config] | according to config, triggers an export                                          | [success]                     | [error]  |
| [count-data]              | notifies on how many items would be exported on `filtered` export option         | -                             | -        |
| [select-data-bulk]        | keeps track of items selections to prompt `selected` export option configuration | -                             | -        |
| [change-query]            | stores current collection filtering                                              | -                             | -        |

### Emits

| event                         | description                                         |
| ----------------------------- | --------------------------------------------------- |
| [export-data/awaiting-config] | registers a transaction and awaits for user configs |
| [success]                     | notifies successful data export                     |
| [error]                       | contains error message when something goes wrong    |
