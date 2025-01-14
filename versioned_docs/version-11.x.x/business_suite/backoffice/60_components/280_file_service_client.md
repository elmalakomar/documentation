---
id: file_service_client
title: File Service Client
sidebar_label: File Service Client
---
<!--
WARNING:
This file is automatically generated. Please edit the 'README' file of the corresponding component and run `yarn copy:docs`
-->

[files-service]: /runtime_suite/files-service/configuration.mdx
[query-params]: /runtime_suite/files-service/usage.mdx#query-parameters

[file-management]: ../80_flows/10_file_management.md

[bk-pdf-viewer]: ./470_pdf_viewer.md

[upload-file]: ../70_events.md#upload-file
[uploaded-file]: ../70_events.md#uploaded-file
[error]: ../70_events.md#error
[download-file]: ../70_events.md#download-file
[downloaded-file]: ../70_events.md#downloaded-file
[delete-file]: ../70_events.md#delete-file
[deleted-file]: ../70_events.md#deleted-file
[fetch-files]: ../70_events.md#fetch-files
[fetched-file]: ../70_events.md#fetched-files
[success]: ../70_events.md#success
[show-in-viewer]: ../70_events.md#show-in-viewer



```html
<bk-file-client></bk-file-client>
```

The File Service Client manages http requests towards an instance of [Mia Files Service][files-service] to upload, download, store files.

Further details on how Back-Kit components can be composed to handle file fields are available in the [specific section][file-management].

## How to configure

To configure the File Service Client, property `basePath` should be specified. `basePath` is the base path of the URL to which HTTP requests are sent, i.e. the endpoint targeting the Files Service.

```json
{
  "tag": "bk-file-client",
  "properties": {
    "basePath": "/files"
  }
}
```

### Download and Preview File

When a component signals the need to download a file (that is, emits a [download-file] event), the File Service Client by default performs a GET request to `/download/<file-name>`.
Upon successful download, an [deleted-file] event is emitted, else an [error] event.

However, if the meta field of the `download-file` event has key `showInViewer` set to true, the File Service Client attempts to fetch the file and, in case of success, notifies via [show-in-viewer] event that the file needs to be previewed, else notifies the failure with an [error] event. A component such as the [Pdf Viewer][bk-pdf-viewer] could pick up the `show-in-viewer` request.
If meta field `showInViewer` in `download-file` meta is set to "skip-checks", the File Service Client does not attempt to download the file, but rather immediately signals the request for the file to be previewed.

#### Query Parameters Tuning

Property `queryParams` handles how the file is downloaded, in compliance with [Files Service specifications][query-params]

```json
{
  "download": 1,
  "downloadWithOriginalName": 1,
  "useOriginalName": 1
}
```

Parameter `download` and `downloadWithOriginalName` are used when downloading the file, while parameter `useOriginalName` is used to fetch a file which should be previewed rather than downloaded.

### Upload and Delete Files

The File Service Client acts as a proxy between rendering components and the Files Service, facilitating the communication and interaction between the two.
It is designed to handle various types of events that may be received from other components, leading to HTTP requests being made to the Files Service. These events include:

- **Upload File**
  When a [upload-file][upload-file] event is received, the File Service Client sends a POST request to its base path
  with a multipart body containing the file form the payload of the event, aiming at uploading the file to the file storage service.
  In case the meta field of the event includes extra information (`metaData`), this is also included in the body of the POST request.
  Upon successful upload, an [uploaded-file] event and a [success] event are emitted, else an [error] event.

- **Delete File**
  When a [delete-file][delete-file] event is received, the File Service Client sends a DELETE request to `/<file-name>`,
  where the name of the file is taken from the payload of the event, aiming at removing the file from the file storage service.
  Upon successful deletion, an [deleted-file] event and a [success] event are emitted, else an [error] event.


### Fetch all Files

Beginning from Files Service version 2.7.0, route `/files` is available. Reaching this route with a GET request allows to retrieve the list of files metadata handled by the Files Service. Data fetched in this way are paginated.

The following query parameters can be appended to the GET
  - `limit`: number of items to be fetched per page
  - `page`: number of pages to skip
  - `dateFrom`: filters out all files that were updated afterwards

Upon receiving a [fetch-files] event, the File Service Client triggers a GET to `/files`, appending query parameters as specified in the payload of the event.


## Examples

### Example: Download File

A File Service Client configured like the following:
```json
{
  "tag": "bk-file-client",
  "properties": {
    "basePath": "/files"
  }
}
```

upon listening to a `download-file` event with payload
```json
{
  "file": "avatar.jpg"
}
```

triggers a GET call to `/files/download/avatar.jpg` with query parameters
```json
{
  "download": 1,
  "downloadWithOriginalName": 1
}
```

### Example: Query Parameters

The File Service Client allows to configure the query parameters to be included in the HTTP call that is performed to download a file through property `queryParams`.

A File Service Client configured like the following
```json
{
  "tag": "bk-file-client",
  "properties": {
    "basePath": "/files",
    "queryParams": {
      "downloadWithOriginalName": 0
    }
  }
}
```

upon listening to a `download-file` event with payload
```json
{
  "file": "avatar.jpg"
}
```

triggers a GET call to `/files/download/avatar.jpg` with query parameters
```json
{
  "download": 1,
  "downloadWithOriginalName": 0
}
```

### Example: Show in viewer

A File Service Client configured like the following
```json
{
  "tag": "bk-file-client",
  "properties": {
    "basePath": "/files"
  }
}
```

upon listening to a `download-file` event with payload
```json
{
  "file": "avatar.jpg"
}
```
and meta
```json
{
  ...
  "showInViewer": true
}
```

attempts to download the file and, on success, emits a [show-in-viewer] event with payload
```json
{
  "url": "<page-domain>/files/download/avatar.jpg?useOriginalName=1"
}
```

If meta were:
```json
{
  ...
  "showInViewer": true
}
```

attempts to download the file and, on success, emits a [show-in-viewer] event with payload
```json
{
  ...
  "showInViewer": "skip-checks"
}
```

the File Service Client would directly emit the `show-in-viewer` event, without first attempting to download the file.

### Example: Fetch File List

```json
{
  "tag": "bk-file-client",
  "properties": {
    "basePath": "/file-service"
  }
}
```

upon listening to a [fetch-files] event with payload
```json
{
  "limit": 20,
  "page": 2
}
```

triggers a GET call to `/file-service/files/` with query parameters identical to the payload of the event
```json
{
  "limit": 20,
  "page": 2
}
```


### Example: Upload File

```json
{
  "tag": "bk-file-client",
  "properties": {
    "basePath": "/files"
  }
}
```

upon listening to a [upload-file] event with payload
```json
{
  "file": {
    ... // file to upload
  }
}
```
and meta
```json
{
  "metaData": {
    "fileOwner": "Adnrew"
  }
}
```

triggers a POST call to `/files/` with a multipart body translation to the payload of the event as well as the extra information inside `meta.metaData`
```json
{
  "file": {
    ... // file to upload
  },
  "fileOwner": "Andrew"
}
```

### Example: Delete File

```json
{
  "tag": "bk-file-client",
  "properties": {
    "basePath": "/files"
  }
}
```

upon listening to a [delete-file] event with payload
```json
{
  "file": "avatar.jpg"
}
```

triggers a DELETE call to `/avatar.jpg`


## API

### Properties & Attributes

| property      | attribute | type                             | default | description                                      |
| ------------- | --------- | -------------------------------- | ------- | ------------------------------------------------ |
| `basePath`    | -         | string                           | -       | the URL base path to which to send HTTP requests |
| `headers`     | -         | {[key: string]: string}          | -       | headers to add when an HTTP request is sent      |
| `credentials` | -         | 'include'\|'omit'\|'same-origin' | -       | credentials policy to apply to HTTP requests     |
| `basePath`    | -         | string                           | -       | the URL base path to which to send HTTP requests |
| `headers`     | -         | {[key: string]: string}          | -       | headers to add when an HTTP request is sent      |
| `credentials` | -         | 'include'\|'omit'\|'same-origin' | -       | credentials policy to apply to HTTP requests     |
| `queryParams` | -         | [QueryParams](#queryparams)      | {"download": 1,"downloadWithOriginalName": 1, "useOriginalName": 1} | query parameters to be passed to the Files Service, according to [its interface][query-params] |

#### QueryParams

```typescript
interface QueryParams {
  download: 0 | 1
  downloadWithOriginalName: 0 | 1
  useOriginalName: 0 | 1
}
```

### Listens to

| event           | action                                                     | emits                        | on error |
| --------------- | ---------------------------------------------------------- | ---------------------------- | -------- |
| [upload-file]   | sends a `POST` request to upload a file                    | [uploaded-file], [success]   | [error]  |
| [download-file] | sends a `GET` request to download/preview a file           | [downloaded-file], [success] | [error]  |
| [delete-file]   | sends a `DELETE` request to remova a file from storage     | [deleted-file], [success]    | [error]  |
| [fetch-files]   | sends a `GET` request to fetch uploaded files from storage | [fetched-file], [success]    | [error]  |


### Emits

| event             | description                                            |
| ----------------- | ------------------------------------------------------ |
| [uploaded-file]   | notifies successful file upload                        |
| [downloaded-file] | notifies successful file download                      |
| [deleted-file]    | notifies successful file deletion                      |
| [show-in-viewer]  | requests to preview a PDF file inside a viewer         |
| [success]         | notifies successful HTTP request                       |
| [error]           | contains http error messages when something goes wrong |
