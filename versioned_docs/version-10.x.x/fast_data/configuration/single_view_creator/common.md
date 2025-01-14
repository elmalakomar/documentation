---
id: common
title: Single View Creator Common Configuration
sidebar_label: Common
---

# Single View Creator concepts

The Single View Creator is the service that keeps the Single View updated with data retrieved from Projections. 
This service is available as a plugin or as a template:
- plugin: it allows you to use the Single View Creator as a black-box. You just need to configure it through the Config Maps and environment variables
- [template](#template): it gives you access to the source code of the Single View Creator, which will be hosted on a Git repository. You will need to update its dependencies and maintain the code. 

We strongly recommend using the plugin. The template is supposed to be used only for advanced use cases where the plugin cannot be used. 

Single View Creator plugin can be used in two modes:
- [Low Code](/fast_data/configuration/single_view_creator/low_code.md): it allows you to perform aggregation through JSON files without writing any Javascript code. If you need a custom behavior on a piece of aggregation you can still write your own code.
- [Manual](/fast_data/configuration/single_view_creator/manual.md): it allows you to perform aggregation by writing your own Javascript code.

We recommend using the Low Code mode since it allows you to be faster and safer when aggregating your data. You will just need to think about the data and not the code for doing so.    
The Manual mode is supposed to be used only for cases where Low Code cannot be used, but this should rarely happen, since it is possible to write custom Javascript functions for specific pieces of aggregation also when using Low Code.

## Getting started

### Plugin

Go to the [Marketplace](/marketplace/overview_marketplace.md) and create a `Single View Creator` or `Single View Creator - Low Code` plugin. 
Go to the microservice page of the newly created Single View Creator and set the correct values to the environment variables containing a placeholder. 

## Template

Search in the [Marketplace](/marketplace/overview_marketplace.md) for a `Single View Creator - Template` and create it.
Then go to the microservice page of the newly created Single View Creator and set the correct values to the environment variables containing a placeholder. 
Click on the repository link in the microservice page and clone on your computer the repository.

:::info
You can use the template and all of Mia-Platform libraries **only under license**.
For further information contact your Mia Platform representative.
:::

### Code overview

The service starts in `index.js` file.
First, the template uses the [Custom Plugin Lib](/development_suite/api-console/api-design/plugin_baas_4.md) to instantiate a service.
Inside its callback, the `single-view-creator-lib` is initialized to deal with the complexity of the Fast Data components.

```js
const singleViewCreator = getSingleViewCreator(log, config, customMetrics)

await singleViewCreator.initEnvironment() // connect Mongo, Kafka and create the patient instance
service.decorate('patient', singleViewCreator.k8sPatient)
```

`config` is an object whose fields represent the [Microservice environment variables](/development_suite/api-console/api-design/services.md#environment-variable-configuration).

Some environment variables will be pre-compiled when you create the service from template, others won't, but they will still have a placeholder as value. Replace it with the correct value.

Now, we start the Single View Creator:

```js
const resolvedOnStop = singleViewCreator.startCustom({
  strategy: aggregatorBuilder(projectionsDB),
  mapper,
  validator,
  singleViewKeyGetter: singleViewKey,
  upsertSingleView: upsertFnSv(),
  deleteSingleView: deleteSV(),
})
```

- `strategy` is the function that performs the aggregation over the Projections
- `mapper` is the function that takes as input the raw aggregation result and maps the data to the final Single View
- `validator` is the validation function which determines if the Single View is valid (and thus inserted or updated in Mongo) or not (and thus deleted)
- `singleViewKeyGetter` is the function that, given the Projections changes identifier, returns the data used as selector to find the Single View document on Mongo to update or delete
- `upsertFnSv` is the function that updates or inserts the Single View to the Single Views collection on Mongo
- `deleteSingleView` is the function that deletes the Single View from the Single Views collection on Mongo. It uses the `deleteSV` function exported by the library.

:::note
The `deleteSV` function makes a *real delete* of the document on MongoDB. So, unlike the **Projections** deletion, it does *not* make a virtual delete.
:::

The value of `upsertFnSv` is based on the `UPSERT_STRATEGIES` environment variable. If its value is *update*, then the *updateOrInsertSV* function exported by the library is used, otherwise the function *replaceOrInsertSV* is used instead. The default upsert strategy is *replace*.

:::note
In the versions of the template prior to the `v3.1.0`, the UPSERT_STRATEGIES was missing, and it was used an alias function (*upsertSV*) of the *replaceOrInsertSV*.
:::

The Single View Creator needs to be stopped when the process is stopping. To do that, we use the `onClose` hook:

```js
service.addHook('onClose', async() => {
  log.fatal({ type: 'END' }, 'Single View Creator is stopping...')
  await singleViewCreator.stop()

  // this is a promise resolved when the infinite loop which processes the single views ends.
  // Here we wait for the resolving of the promise. You don't need to call it.
  await resolvedOnStop
  log.fatal({ type: 'END' }, 'Single View Creator stopped')
  await mongoClient.close()
})
```

## Environment Variables

<table>
<tr><th>Name</th><th>Required</th><th>Description</th><th>Default value</th></tr>
<tr><td>CONFIGURATION_FOLDER</td><td>false</td><td>Folder where configuration files are mounted</td><td>/home/node/app/src/</td></tr>
<tr><td>LOG_LEVEL</td><td>true</td><td>Level to use for logging</td><td>-</td></tr>
<tr><td>HTTP_PORT</td><td>false</td><td>Port exposed by the service</td><td>3000</td></tr>
<tr><td>TYPE</td><td>true</td><td>Identifies the type of Projection changes that needs to be read. It should be the same as the Single View name you want to update.</td><td>-</td></tr>
<tr><td>SCHEDULING_TIME</td><td>false</td><td>A quantity of time in milliseconds. Every X milliseconds the service wakes up and checks if there are some Projections changes in <code>NEW</code>state to work on. The service continues working until no more new Projections changes are found, then it goes to sleep for X milliseconds.</td><td>60000</td></tr>
<tr><td>PROJECTIONS_MONGODB_URL</td><td>true</td><td>MongoDB connection string where Projections are stored. Must be a valid URI.</td><td>-</td></tr>
<tr><td>SINGLE_VIEWS_MONGODB_URL</td><td>true</td><td>MongoDB connection string where Single View must be stored. Must be a valid URI.</td><td>-</td></tr>
<tr><td>PROJECTIONS_CHANGES_MONGODB_URL</td><td>false</td><td>The db from where Projections changes are read.</td><td>value of <code>PROJECTIONS_MONGODB_URL</code></td></tr>
<tr><td>PROJECTIONS_CHANGES_DATABASE</td><td>true</td><td>The db from where Projections changes are read.</td><td>-</td></tr>
<tr><td>PROJECTIONS_DATABASE</td><td>true</td><td>The db from where Projections are read.</td><td>value of <code>PROJECTIONS_CHANGES_DATABASE</code></td></tr>
<tr><td>PROJECTIONS_CHANGES_COLLECTION</td><td>false</td><td>If you have set a custom Projection change collection name from advanced, then set its name. Otherwise, it is <code>fd-pc-SYSTEM_ID</code>where <code>SYSTEM_ID</code>is the id of the System of Records this Single View Creator is responsible for.</td><td>-</td></tr>
<tr><td>SINGLE_VIEWS_DATABASE</td><td>true</td><td>The db from where Single Views are written.</td><td>-</td></tr>
<tr><td>SINGLE_VIEWS_COLLECTION</td><td>true</td><td>It must be equals to the Single View name the service is in charge of keeping updated.</td><td>-</td></tr>
<tr><td>SINGLE_VIEWS_PORTFOLIO_ORIGIN</td><td>true</td><td>Should be equals to the <code>SYSTEM_ID</code>you have set in <code>PROJECTIONS_CHANGES_COLLECTION</code></td><td>-</td></tr>
<tr><td>SINGLE_VIEWS_ERRORS_COLLECTION</td><td>true</td><td>Name of a MongoDB CRUD you want to use as collection for Single View errors.</td><td>-</td></tr>
<tr><td>KAFKA_CONSUMER_GROUP_ID</td><td>false</td><td><b>@deprecated</b> - in favor of <code>KAFKA_GROUP_ID</code>. The Kafka consumer group identifier</td><td>-</td></tr>
<tr><td>KAFKA_GROUP_ID</td><td>true</td><td>Defines the Kafka group id (it is suggested to use a syntax like <code>{'{tenant}.{environment}.{projectName}.{system}.{singleViewName}.single-view-creator'}</code>)</td><td>-</td></tr>
<tr><td>KAFKA_CLIENT_ID</td><td>false</td><td>The Kafka client identifier</td><td>-</td></tr>
<tr><td>KAFKA_BROKERS_LIST</td><td>false</td><td><b>@deprecated</b> - in favor of <code>KAFKA_BROKERS</code>. list of brokers the service needs to connect to</td><td>-</td></tr>
<tr><td>KAFKA_BROKERS</td><td>false</td><td>List of brokers the service needs to connect to</td><td>-</td></tr>
<tr><td>KAFKA_SASL_MECHANISM</td><td>false</td><td>The Kafka SASL mechanism to be used. Can be one of the following: "plain", "PLAIN", "scram-sha-256", "SCRAM-SHA-256", "scram-sha-512", "SCRAM-SHA-512"</td><td>plain</td></tr>
<tr><td>KAFKA_SASL_USERNAME</td><td>false</td><td>Username to use for logging into Kafka</td><td>-</td></tr>
<tr><td>KAFKA_SASL_PASSWORD</td><td>false</td><td>Password to use for logging into Kafka</td><td>-</td></tr>
<tr><td>KAFKA_SVC_EVENTS_TOPIC</td><td>false</td><td>Topic used to queue Single View Creator state changes (e.g. Single View creation). This feature is deprecated in favor of <code>KAFKA_SV_UPDATE_TOPIC</code>and it will be removed soon</td><td>-</td></tr>
<tr><td>SEND_BA_TO_KAFKA</td><td>false</td><td>True if you want to send to Kafka the <code>before-after</code>information about the update changes of the Single View. This feature is deprecated in favor of <code>ADD_BEFORE_AFTER_CONTENT</code>using the 'sv-update' event and it will be removed soon</td><td>false</td></tr>
<tr><td>KAFKA_BA_TOPIC</td><td>false</td><td>Topic where to send the <code>before-after</code>messages which represent the Single View document before and after a change. This feature is deprecated in favor of <code>ADD_BEFORE_AFTER_CONTENT</code>using the 'sv-update' event and it will be removed soon</td><td>-</td></tr>
<tr><td>SEND_SV_UPDATE_TO_KAFKA</td><td>false</td><td>True if you want to send to Kafka the <code>sv-update</code>message about the update changes of the Single View</td><td>false</td></tr>
<tr><td>ADD_BEFORE_AFTER_CONTENT</td><td>false</td><td>True if you want to add the <code>_before_</code>and <code>_after_</code>content to the <code>sv-update</code>message, works only if <code>SEND_SV_UPDATE_TO_KAFKA</code>is set to true</td><td>false</td></tr>
<tr><td>KAFKA_SV_UPDATE_TOPIC</td><td>false</td><td>Topic where to send the <code>sv-update</code>message</td><td>-</td></tr>
<tr><td>UPSERT_STRATEGY</td><td>false</td><td>(<code>v3.1.0</code>or higher) If it is set to "replace", the whole Single View document will be replaced with the new one. If it is set to "update", the existing one will be updated with the new one, but fields not present in the latter will be kept. Otherwise, a path pointing to a custom function can be specified.</td><td>replace</td></tr>
<tr><td>DELETE_STRATEGY</td><td>false</td><td>(<code>v3.1.0</code>or higher) If it is set to "delete", the whole Single View document will be deleted. Otherwise, a path pointing to a custom function can be specified.</td><td>delete</td></tr>
<tr><td>SINGLE_VIEWS_MAX_PROCESSING_MINUTES</td><td>false</td><td>(<code>v3.4.2</code>or higher) time to wait before processing again a Projection with <code>IN_PROGRESS</code>state</td><td>30</td></tr>
<tr><td>CA_CERT_PATH</td><td>false</td><td>The path to the CA certificate, which should include the file name as well, e.g. <code>/home/my-ca.pem</code></td><td>-</td></tr>
<tr><td>ER_SCHEMA_FOLDER</td><td>false</td><td>The path to the ER Schema folder, e.g. <code>/home/node/app/erSchema</code></td><td>-</td></tr>
<tr><td>AGGREGATION_FOLDER</td><td>false</td><td>The path to the Aggregation folder, e.g. <code>/home/node/app/aggregation</code></td><td>-</td></tr>
<tr><td>USE_AUTOMATIC</td><td>false</td><td>Specifies whether to use the low code architecture for the Single View Creator service or not</td><td>-</td></tr>
<tr><td>PROJECTIONS_CHANGES_SOURCE</td><td>false</td><td>System to use to handle the Projection Changes, supported methods are <code>KAFKA</code>or <code>MONGO</code></td><td>MONGO</td></tr>
<tr><td>KAFKA_PROJECTION_CHANGES_TOPICS</td><td>false</td><td>Comma separated list of Projection changes topics</td><td>-</td></tr>
<tr><td>KAFKA_PROJECTION_UPDATE_TOPICS</td><td>false</td><td>Comma separated list of Projection update topics</td><td>-</td></tr>
<tr><td>SV_TRIGGER_HANDLER_CUSTOM_CONFIG</td><td>false</td><td>Path to the config defining SV-Patch actions</td><td>-</td></tr>
<tr><td>READ_TOPIC_FROM_BEGINNING</td><td>false</td><td>Available from <code>v5.5.0</code>of the Single View Creator Plugin. If set to true, the Single View Creator will start reading from messages in the Projection Changes topic from the beginning, instead of the message with the latest commmitted offset. This will happen only the first time connecting to the topic, and it has effect only if <code>PROJECTIONS_CHANGES_SOURCE</code> is set to <i>KAFKA</i>.</td><td>false</td></tr>
<tr><td>USE_UPDATE_MANY_SV_PATCH</td><td>false</td><td>Use the MongoDB <code>updateMany</code> operation instead of the <code>findOneAndUpdate</code> with cursors in the sv patch operation. This will speed up the Single View creation/update process but it will not fire the kafka events of Single View Creation/Update. As a natural consequence, if enabled, the following environment vairables will be ignored: <code>SEND_BA_TO_KAFKA</code>, <code>KAFKA_BA_TOPIC</code>, <code>SEND_SV_UPDATE_TO_KAFKA</code>, <code>KAFKA_SV_UPDATE_TOPIC</code>, <code>ADD_BEFORE_AFTER_CONTENT</code>, <code>KAFKA_SVC_EVENTS_TOPIC</code></td><td>false</td></tr>
</table>

If you do not want to use Kafka in the Single View Creator, you can just not set the environment variable *KAFKA_CLIENT_ID* or *KAFKA_BROKERS*. If one of them is missing, Kafka will not be configured by the service (requires *single-view-creator-lib* `v9.1.0` or higher)

## Single View Key

The Single View Key is the Single View Creator part which identifies the Single View document that needs to be updated as consequence of the event that the service has consumed. 

If you are using Low Code, please visit [this section](/fast_data/configuration/single_view_creator/low_code.md#single-view-key), otherwise you can check to the [manual documentation](/fast_data/configuration/single_view_creator/manual.md#single-view-key)

## Aggregation

The Aggregation is the Single View Creator part which aggregates Projections data and generates the Single View that is going to be updated. 

If you are using Low Code, please visit [this section](/fast_data/configuration/single_view_creator/low_code.md#aggregation), otherwise you can check to the [manual documentation](/fast_data/configuration/single_view_creator/manual.md#aggregation)

:::note
Since version `v5.0.0` of the Single View Creator service and `v12.0.0` of the `@mia-platform-internal/single-view-creator-lib`, returning a Single View with the `__STATE__` field set from the aggregation will update the Single View to that state (among the other changes).   
This means, for instance, that if you set the `__STATE__` value to `DRAFT` in the `aggregation.json` in Low Code mode (or in the `pipeline.js` in Manual mode), the Single View updated will have the `__STATE__` field set to `DRAFT`.
Previously, the `__STATE__` field you returned was ignored, and the Single View would always have the `__STATE__` value set to `PUBLIC`.
:::

## Validator

The validation of a Single View determines what to do with the current update. If the Single View is determined as "non-valid", the delete function will be called. Otherwise, if the result of the validation is positive, it will be updated or inserted in the Single Views collection, through the upsert function. Delete function and upsert function will be explained in the next paragraph.

For this reason, the validation procedure should not be too strict, since a Single View declared as "invalid" would not be updated or inserted to the database. Rather, the validation is a check operation to determine if the current Single View should be handled with the upsert or delete functions.


By default, the validator always returns true. So we accept all kinds of Single Views, but, if you need it, you can set your own custom validator.

```js
// (logger: BasicLogger, singleView: Document) => Boolean
function singleViewValidator(logger, singleView) {
  ... checks on singleView

  // returns a boolean
  return validationResult
}
```

:::warning
When the update of an existing Single View is triggered and the validation has a negative outcome, the Single View won't be updated, and instead it will be deleted.
:::

### Plugin

Since version `v3.5.0`, it is possible to specify a custom validator function inside the configuration folder (`CONFIGURATION_FOLDER`).

The file must be named `validator.js` and must export a function that will take as arguments the same as the default validator explained above.

```js title="validator.js"
module.exports = function validator(logger, singleView) {
  ... custom validation logic on singleView

  // returns a boolean
  return customValidationResult
}
```

### Template

The `startCustom` function accepts a function in the configuration object called `validator`, which is the validation function.

The input fields of the validation function are the logger and the Single View, while the output is a boolean containing the result of the validation.

## Customize Upsert and Delete functions

If you want, you can replace both upsert and delete functions with your own custom functions.

These functions represent the last step of the creation (or deletion) of a Single View, in which the Single View collection is actually modified.

In case the validation succeeds, the upsert function will be called with the following arguments:

- `logger` is the logger
- `singleViewCollection` is the Mongo collection object
- `singleView` is the result of the mapping operation
- `singleViewKey` is the Single View key

On the other hand, if the validation has a negative outcome, the delete function will be called with the same arguments, except for the `singleView`, which will not be handled by the delete function.

In both cases, some operation should be done on `singleViewCollection` in order to modify the Single View with the current `singleViewKey`, with the idea of "merging" the current result with the one already present in the database.

For example, we have a "Customer" Single View with a list of products the customer bought from different e-commerce websites, and we receive an update for a new object on a specific shop. In that case we don't want to replace the list of bought products with the last one arrived, but we want to push the product in the list in order to have the complete history of purchases.

For both functions, the output is composed of an object containing two fields:

- `old` which contains the old Single View
- `new` which contains the new Single View

These values will respectively be the `before` and the `after` of the message sent to the `KAFKA_BA_TOPIC` topic, which is the topic responsible for tracking any result of the Single View Creator. The naming convention for this topic can be found [here](/fast_data/inputs_and_outputs.md#single-view-before-after).

```js
async function upsertSingleViewFunction(
  logger,
  singleViewCollection,
  singleView,
  singleViewKey)
{
  logger.trace('Upserting Single View...')
  const oldSingleView = await singleViewCollection.findOne(singleViewKey)

  await singleViewCollection.replaceOne(
    singleViewKey,
    singleView,
    { upsert: true }
  )

  logger.trace({ isNew: Boolean(oldSingleView) }, 'Updated Single View')
  return {
    old: oldSingleView,
    new: singleView,
  }
}

async function deleteSingleViewFunction(
  logger,
  singleViewCollection,
  singleViewKey)
{
  logger.trace('Deleting Single View...')
  const oldSingleView = await singleViewCollection.findOne(singleViewKey)

  if (oldSingleView !== null) {
    try {
      await singleViewCollection.deleteOne(singleViewKey)
    } catch (ex) {
      logger.error(`Error during Single View delete: ${ex}`)
    }
  }

  logger.trace('Single view deletion procedure terminated')
  return {
    old: oldSingleView,
    new: null,
  }
}
```

### Plugin

Add a config map to your service and put the Javascript files into it. These files should contain the custom function you want to use as upsert or delete function. 

For instance:



```js title="myDeleteFunction.js"
module.exports = async function myDeleteFunction(
  logger,
  singleViewCollection,
  singleViewKey)
{
  logger.trace('Checking if it can be deleted...')
  const oldSingleView = await singleViewCollection.findOne(singleViewKey)

  // my custom logic
  // do something...

  if (oldSingleView !== null) {
    
    try {
      await singleViewCollection.deleteOne(singleViewKey)
    } catch (ex) {
      logger.error(`Error during Single View delete: ${ex}`)
    }
  }

  logger.trace('Single view deletion procedure terminated')
  return {
    old: oldSingleView,
    new: null,
  }
}
```

Let's suppose that I put this file in a config map mounted on path `/home/node/app/my-functions`. Then, in order to use that, I need to set the `DELETE_STRATEGY` environment variable to `/home/node/app/my-functions/myDeleteFunction.js`. 

The same logic can be applied to upsert function, but setting the file path to the environment variable `UPSERT_STRATEGY`.

### Template

You can choose to apply the same pattern used in plugin (by setting the environment variables) or to pass your custom functions directly to the `startCustom` method.

```js title="index.js" {6-7} showLineNumbers
const resolvedOnStop = singleViewCreator.startCustom({
  strategy: aggregatorBuilder(projectionsDB),
  mapper,
  validator,
  singleViewKeyGetter: singleViewKey,
  upsertSingleView: upsertFnSV,
  deleteSingleView: deleteSV,
})
```

### Error handling

When generating a Single View, every error that occurs is saved in MongoDB, with a format that satisfies the schema requirements of the CRUD service, so that you can handle those errors using the Console. The fields of the error messages when they are first created are:

- `_id`: a unique identifier of the record, automatically generated
- `portfolioOrigin`: a value concerning the origin of the error, defaults to `UNKNOWN_PORTFOLIO_ORIGIN`
- `type`: the Single View type
- `identifier`: the id of the projection changes
- `errorType`: the error details
- `createdAt`: the time of creation
- `creatorId`: set to `single-view-creator`
- `__STATE__`: set to `PUBLIC`
- `updaterId`: set to `single-view-creator`
- `updatedAt`: the time of creation

It is highly recommended to use a TTL index to enable the automatic deletion of older messages, which can be done directly using the Console, as explained [here](/development_suite/api-console/api-design/crud_advanced.md#indexes).

### CA certs

Since service version `3.9.0`, you can set your CA certs by providing a path to the certification file in the environment variable `CA_CERT_PATH`.

### Single View Patch

:::info
This feature is supported from version `5.6.1` of the Single View Creator
:::

To configure a Single View Creator dedicated to [Single View Patch](/fast_data/configuration/single_views.md#single-view-patch) operations, some steps have to be followed:

* Set the env var `KAFKA_PROJECTION_UPDATE_TOPICS` with the comma separated list of the `pr-update` topics corresponding to the SV-Patch Projection.
* Set the env var `SV_TRIGGER_HANDLER_CUSTOM_CONFIG` with the path to the main file defining SV-Patches actions, for example `/home/node/app/svTriggerHandlerCustomConfig/svTriggerHandlerCustomConfig.json`
* Create a new ConfigMap with this Runtime Mount Path: `.../svTriggerHandlerCustomConfig`

This last config map is composed by a main file, `svTriggerHandlerCustomConfig.json`, which defines where to read the Patch Action for each Projection.

It is structured as following: 

```json
{
  "patchRules": [
    {
      "projection": "projection_A",
      "patchAction": "__fromFile__[customPatchForA.js]"
    },
    {
      "projection": "projection_B",
      "patchAction": "__fromFile__[customPatchForB.js]"
    }
    // You can define more than one patch action for each projection too!
  ]
}
```

In the same config map, we have to insert the other files that are defined in the `patchRules` of the `svTriggerHandlerCustomConfig.json` (in the above example `customPatchForA.js` and `customPatchForB.js`).

They are structured as following:

```javascript
'use strict'

module.exports = (logger, projection) => {
  logger.info('Function custom patch for projection A')
  return {
    filter: { 'sv-field': projection['projection-field'] },
    update: { $set: { 'field-0': projection['changed-field'] } },
  }
}
```

Basically we can define any update operation we want. This operation will be performed on all the Single Views matching the filter.

#### Filtering which elements to update inside arrays

If the update must happen inside an array, you'll probably need to filter which elements need to be updated. To do that you can use the `arrayFilters` option inside the `patchAction` Javascript file, which behaves exactly like the [`arrayFilters`](https://www.mongodb.com/docs/manual/reference/operator/update/positional-filtered/#---identifier--) option in a MongoDB operation.

Example of its usage:

```javascript
'use strict'

module.exports = (logger, projection) => {
  logger.info('Function custom patch for projection A')
  return {
    filter: { 'sv-field': 'someValue' }, // This can be an empty object if needed
    update: {
      $set: {
        "array-field.$[item-name].array-item-field": projection['changed-field']
      }
    },
    arrayFilters: [{
      "item-name.array-item-field-id": projection['projection-A-field-id']
    }]
  }
}
```