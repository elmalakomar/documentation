---
id: export_modal
title: Export Modal
sidebar_label: Export Modal
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

[bk-export-client]: ./120_crud_export_client.md

[export-data]: ../70_events.md#export-data
[export-data/request-config]: ../70_events.md#export-data---request-config
[export-data/user-config]: ../70_events.md#export-data---user-config



```html
<bk-export-modal></bk-export-modal>
```

<!-- TODO add image -->

The Export Modal renders a form inside a modal with standard fields that allow user to configure a data export task.

The Export Modal opens whenever a component signals the need to specify configuration for an export task. That is, whenever a [export-data/request-config] event is received.

It is usually best to have this event emitted by a component like the [CRUD Export Client][bk-export-client], which uses it to open the modal with an associated `transactionId`.
For instance, the [CRUD Export Client][bk-export-client] emits a [export-data/request-config] event upon listening to an [export-data] event.
<!-- TODO add link to export flow -->

## How to configure

The Export Modal does not require any particular configuration.

```json
{
  "tag": "bk-export-modal"
}
```

The Export Modal allows the user to specify:
  - the format onto which data should be exported ("csv" or "xlsx")
  - what fiters to apply to the data to be exported. Options are to export the whole collection, to apply the same filters that have been applied to the current visualization of the collection, to export selected items only
  - what fields of the collection to include in the export result
  - if the specified format is "csv", what separator to use (comma or semicolon)

Upon submitting the form, the Export Modal signals the need to export data accordingly with the specified options.
A component like the [CRUD Export Client][bk-export-client] might pick up on the export request.

## API

### Properties & Attributes

None

### Listens to

| event                        | action                |
| ---------------------------- | --------------------- |
| [export-data/request-config] | prompts modal opening |


### Emits

| event                     | description                                               |
| ------------------------- | --------------------------------------------------------- |
| [export-data/user-config] | notifies the bus of user config for next export data task |
