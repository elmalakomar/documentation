---
id: no_code_overview
title: Fast Data Low Code/ No Code Overview
sidebar_label: Fast Data Low Code/ No Code Overview
---

:::tip
If you want to try the Fast Data Low Code with a simple example, here's a step by step [tutorial](/tutorial/fast_data/fast_data_tutorial.mdx)
:::

The "Low Code" and "No Code" versions of Fast Data are the new standards that leverage the power of configurations to overcome coding complexity in Fast Data setup.
This experience is based on two main fundamentals: No Code Fast Data and Low Code Fast Data.

Thanks to No Code, you will have the possibility to configure your project with a few clicks. And as of version 10.6.0, if you enable the No Code feature (you may have to contact your System Administrator in order to do so), you will have the ability to define relationships between the projections in the ER Schema.

Thanks to Low Code, you will only need to define for your Fast Data projects:

1. The relationships between the projections, in the ER Schema. *(Can also be defined with No Code now)*
2. A flow of the projection changes that should trigger specific Single Views to update. *(Coming soon in No Code)*
3. A declarative definition of the Single Views, based on the fields of the projections and other, more flexible solutions. *(Coming soon in No Code)*

On this page, you can find an overview of which are the steps involved in a No Code workflow or Low Code one, with links to the technicalities for delving into greater detail. Before that, we want to clarify what we mean by "No Code" and "Low Code".

## What are Low Code and No Code?

With No Code we refer to the console capability to reduce the development effort using the graphic interface and automations, allowing both experienced and inexperienced users to manage a Fast Data project. Thanks to No Code you can focus on creating your Fast Data architecture while the console will support you in all the development phases, creating for your microservices and configurations.

With Low code, we mean that instead of writing JavaScript code to generate the strategies, projection changes and single views, you will only need to write `json` files that declaratively specify how you want your Fast Data System to be.
If you have used Fast Data before, you know that manually configuring everything can be a time-consuming end error-prone, repetitive activity. Thanks to our experience in the field, we have developed the functionality to interpret simple and human-readable `json` configurations that express most of the configurations (if not all) you will ever need.
In case the automation is not expressive enough for your use case, you can always write custom functions to close the gap, and this is why it is a "Low Code" instead of "No Code" configuration, but most of the time you will be cruising in "No Code" heaven.

## Step by step Low Code journey

The Fast Data Low Code experience is basically composed by some steps with No Code approach, and some other steps with a Low Code approach. In order to understand the proper functioning of the system, it is important to go in deep with the [Fast Data overview](/fast_data//what_is_fast_data.md) documentation.

### Creating a System of Records (No Code)

The creation of the System of Records is one of the No Code steps of the Fast Data configuration. With few clicks, it is possible to create a System of Records that, after saving the configuration inside the console, it is linked to a [Real Time Updater Low Code](/fast_data//configuration/realtime_updater/low_code.md) microservice.

### ER Schema definition and other Configmaps (Low Code/ No Code)

#### Low Code

The Real Time Updater Low Code needs some configurations:

- [erSchema.json](/fast_data/configuration/config_maps/erSchema.md) configuration: useful to define the interconnection between projections
- The [projectionChangeSchema.json](/fast_data//configuration/realtime_updater/common.md#projection-changes): useful to the system to know which single view needs to be updated

In both cases, it is possible to write your file inside the console and if needed, you can share them with other microservices.

#### No Code

If this feature is enabled, you will be able to build an ER Schema using an interactive canvas without the use of code. You will still have the Low Code JSON editor if needed, and you can seamlessly switch editing between it and the No Code canvas. (Available on versions after 10.6.0)

### Adding a Low Code Single View Creator (Low Code)

The other fundamental component of your Fast Data Low Code project is the [Single View Creator Low Code](/fast_data//configuration/single_view_creator/low_code.md).
You can create it from our Marketplace.
Also, in this case, it is needed to configure some Config Maps:

- singleViewKey.json: it is fundamental to generate the query that will be applied on the single view
- erSchema.json: it needs to be the same Real Time Updater Low Code's erSchema.json of the System of Records
- aggregation.json: useful to define the aggregation and generate the single view's fields

![Singleviewlowcode](./img/singleviewlowcode.png)

### Linking Strategies (No Code)

The Fast Data No Code experience ends with the possibility to link a [strategy](/fast_data/the_basics.md#strategies) to your single view. To do that, you need to go in the `strategies` section of your single view and choose Low Code Strategy source in this way you allow the console to automatically manage the strategy for you.
