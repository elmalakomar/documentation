---
id: overview
title: Appointment Manager
sidebar_label: Overview
---

<!--
WARNING: this file was automatically generated by Mia-Platform Doc Aggregator.
DO NOT MODIFY IT BY HAND.
Instead, modify the source file and run the aggregator to regenerate this file.
-->

The **Appointment Manager** (also referred for brevity as *AM*) is a microservice to manage appointments, availabilities, slots and exceptions.

Leveraging the following services:

- [CRUD Service][crud-service-doc]
- [Messaging Service][messaging-service-doc]
- [Timer Service][timer-service-doc]
- [Teleconsultation Service][teleconsultation-service-be-doc]

the AM can be used to perform the basic *CRUD* operations - create, retrieve, update and delete - on availabilities, exceptions and appointments 
and automatically send messages and reminders to appointment participants.

The service can be seen as an enriched proxy to the CRUD, implementing the same interfaces of the CRUD Service and allowing you
to perform a set of operations on a CRUD collection as you normally would with the CRUD Service itself. On top of that it can be
configured to send messages and set reminders to notify users about appointment events.

In the documentation, when you will encounter any of the terms listed below, you should assume they have the described meaning, unless stated otherwise.

| Term            | Meaning |
|-----------------|--------------|
| *AM*            | Abbreviation for *Appointment Manager*. |
| *Appointment*   | A scheduled appointment. |
| *Availability*  | A (recurrent) period of time when a resource is available to deliver some services (e.g. every Monday from 8 to 12 until the end of the year). They are classified as *single*, if they occur just once, or *recurring*, if they occur on multiple days within a given period. |
| *Custom field*  | A custom field is a property of an availability, appointment or exception stored in the CRUD alongside the fields required and recognized by the AM. |  
| *Exception*     | A period of time when a resource is not available and unable to provide some service (e.g. a doctor on vacation for two weeks). |
| *Fixed slot*    | A slot from an availability with a fixed slot duration. |
| *Flexible slot* | A slot from an availability without a slot duration. |ù
| *NM*            | Abbreviation for *Notification Manager*. |
| *Participant*   | A doctor, patient, etc. involved in an appointment. |
| *Resource*      | A person, room, equipment, etc. required to provide some services. |
| *Service*       | A service delivered by a resource. |
| *Slot*          | A time-slot when you can book one or more appointment. |

:::danger
**v2.0.0.**
Unlike version 1.x.x, we refer to *slots* as equivalent to time slots, meaning the relationship between a slot and an appointment is no longer one-to-one but one-to-many (you can book multiple appointments on the same slot). This key semantic difference propagates at every level, from the CRUD collections to the AM REST API. For this reason, AM no longer needs a CRUD collection for the slots and each appointment. Instead of using the deprecated `slotId`, now has a new field, called `availabilityId`, which links the appointment directly to the availability. The slot is then uniquely identified by the combination of an appointment `availabilityId`, `startDate` and `endDate`. Keep this difference in mind while reading the rest of the documentation.
:::

## Resource

Each availability, exception and appointment is linked to a resource, like a doctor, a room or an equipment required to provide a service. Each resource (e.g. doctor John Watson) must have a unique ID (e.g. `john_watson_md`) stored under a field having the same name across all CRUD collections (e.g. `resourceId`). We recommend using a unique ID related to the identity of the resource in your authentication system (e.g. Auth0).

Thus, when you create a new availability, you send in the request body a JSON containing the resource ID:

```json
{
  "startDate": "...",
  "endDate": "...",
  "resourceId": "john_watson_md"
}
```

If the doctor goes on sick leave, the equipment is not available due to maintenance, etc. you create an exception and again pass in the request body the resource ID:

```json
{
  "startDate": "...",
  "endDate": "...",
  "reason": "Sick leave",
  "resourceId": "john_watson_md"
}
```

Finally, when you create an appointment, you pass in the request body the resource ID:

```json
{
  "startDate": "...",
  "endDate": "...",
  "resourceId": "john_watson_md"
}
```

The `resourceId` field enables the Appointment Manager to:

- track the status of each availability slot, in particular if a slot overlaps, even partially, with an exception linked to the same resource as the availability (in that case the slot is marked as unavailable and the corresponding appointments are flagged);
- return calendar events filtered according to a specific resource;
- track the ownership of appointments not based on slots;
- etc. 

To correctly configure the Appointment Manager, you need to:

- set the `RESOURCE_ID_FIELD_NAME` environment variable with the name you choose for the CRUD field storing the resource ID (e.g. `resourceId`);
- add a required field with the chosen name and type `string` to each CRUD collection (for availabilities, exceptions and appointments).

More details about the AM configuration can be found in the [*Configuration*][configuration] section.

## Availabilities and slots

Availabilities allow you to define when a resource is available to deliver a service and contain one or more time-slots that are available to book an appointment.

### Overview

An availability has these basic properties:

- `startDate`: start date/time;
- `endDate`: end date/time;
- `slotDuration`: duration of each slot (in minutes);
- `simultaneousSlotsNumber`: number of appointments you can book on each time slot.

:::info
**v2.2.0**
Since version 2.2.0, the `slotDuration` field is optional. If not set, the available slots are computed, based on appointments reserved or booked, as the longest intervals where an appointment can be booked.
:::

:::tip
Remember that `startDate` and `endDate` refer to the start and end date/time of the first occurrence of an availability and must be within the same day. For recurring availabilities, having multiple occurrences, the last occurrence will end before the date/time in the `untilDate` field, if specified, otherwise an endless number of occurrences and consequently slots can be expected.
:::

### Single and recurring availabilities

You can create several types of availabilities, based on the scheduling pattern:

- a **single availability**, that just occurs once (e.g. on October 20th, 2022 from 9 to 13)

```json
{
    "startDate": "2022-10-20T09:00:00.000Z",
    "endDate": "2022-10-20T13:00:00.000Z",
    "slotDuration": 60,
    "simultaneousSlotsNumber": 2
}
```

- a **daily availability**, that occurs every day on the same time window (e.g. every day from 9 to 13 since October 20th, 2022 until December 31th, 2022)

```json
{
    "startDate": "2022-10-20T09:00:00.000Z",
    "endDate": "2022-10-20T13:00:00.000Z",
    "slotDuration": 60,
    "simultaneousSlotsNumber": 2,
    "each": "day",
    "untilDate": "2022-12-31T23:59:59.999Z"
}
```

- a **weekly availability**, that occurs every week on the same weekday(s) and time window (e.g. every Tuesday and Thursday since October 20th, 2022 until December 31th, 2022)

```json
{
    "startDate": "2022-10-20T09:00:00.000Z",
    "endDate": "2022-10-20T13:00:00.000Z",
    "slotDuration": 60,
    "simultaneousSlotsNumber": 2,
    "each": "week",
    "on": [2, 4],
    "untilDate": "2022-12-31T23:59:59.999Z"
}
```

- a **monthly availability**, that occurs every month on the same day and time window (e.g. every 1st day of the month since November 2022 until June 2023)

```json
{
    "startDate": "2022-11-01T09:00:00.000Z",
    "endDate": "2022-11-01T13:00:00.000Z",
    "slotDuration": 60,
    "simultaneousSlotsNumber": 2,
    "each": "month",
    "untilDate": "2023-06-30T23:59:59.999Z"
}
```

The last three are generally referred as **recurring availabilities**, having multiple occurrences spanning over a given period.

:::tip
**v2.0.0**
From v2.0.0 you can define recurring availabilities with no expiration date, by simply omitting the `untilDate` field.
:::

### Fixed slots

For example, an availability from 9:00 (`startDate`) to 11:00 (`endDate`) with a 30 minutes slot (`slotDuration`) and 2 simultaneous slots (`simultaneousSlotsNumber`) will result in 4 time-slots:

```json
{
    "startDate": "2022-10-20T09:00:00.000Z",
    "endDate": "2022-10-20T11:00:00.000Z",
    "slotDuration": 30,
    "simultaneousSlotsNumber": 2
}
```

| Slot | Start date | End date |
| ---- | ---------- | -------- |
| 1    |  9:00      | 09:30    |
| 2    |  9:30      | 10:00    |
| 3    | 10:00      | 10:30    |
| 4    | 10:30      | 11:00    |

Since you can book two appointments in each slot (`simultaneousSlotsNumber`), this availability can handle 8 appointments.

Each slot can have of the following status:

- `AVAILABLE`: if you can book an appointment on the slot;
- `UNAVAILABLE`: if an exception associated to the availability resource overlaps the slot and therefore it is not possible to book an appointment;
- `BOOKED`: the maximum number of appointments you can book on the slot has been reached and therefore it is not possible to book more appointments.

:::info
**v2.0.0**
The `UNAVAILABLE` slot status has been introduced with v2.0.0 to handle exceptions (also a new feature). See [the section below][overview-exceptions] for more details about exceptions.
:::

### Flexible slots

:::info
**v2.2.0**
Flexible duration slots are available since version 2.2.0.
:::

If you do not set the `slotDuration` field, an availability from 9:00 (`startDate`) to 11:00 (`endDate`) will result in a single time slot and the `simultaneousSlotsNumber` is interpreted as the maximum number of appointments at any given time:

```json
{
    "startDate": "2022-10-20T09:00:00.000Z",
    "endDate": "2022-10-20T11:00:00.000Z",
    "simultaneousSlotsNumber": 2
}
```

| Available time slots | Start date | End date |
|----------------------|------------|----------|
| 1                    | 9:00       | 11:00    |

Within this single time slot, you can book an appointment lasting at most two hours, as long as it starts and ends between 9:00 and 11:00.

As new appointments are booked or exceptions are created, the AM automatically recompute the available time slots; let's see how this works with an example.

If two appointments are booked, one from 9:00 to 9:30 and another from 9:30 to 10:30, you still get the same available time slot, since the two appointments do not overlap and the availability is below its maximum capacity of two simultaneous appointments (`simultaneousSlotsNumber`). It is possible, then, to book another appointment anytime between 9:00 and 11:00.

| Appointment | Start date | End date |
|-------------|------------|----------|
| 1           | 9:00       | 9:30     |
| 2           | 9:30       | 10:30    |

If another appointment is booked from 9:15 to 9:45, then, for that period, we have two appointments overlapping, so the availability has reached its maximum capacity (`simultaneousSlotsNumber`).

| Appointment | Start date | End date |
|-------------|------------|----------|
| 1           | 9:00       | 9:30     |
| 2           | 9:30       | 10:30    |
| 3           | 9:15       | 9:45     |

As a result, the remaining available time slots are the following:

| Available time slots | Start date | End date |
|----------------------|------------|----------|
| 1                    | 9:00       | 9:15     |
| 2                    | 9:45       | 11:00    |

Finally, if an exception is created from 10:15 to 10:45, in that period no time slot is available, therefore the second time slot is no longer entirely available and we have three shorter available time slots:

| Available time slots | Start date | End date |
|----------------------|------------|----------|
| 1                    | 9:00       | 9:15     |
| 2                    | 9:45       | 10:15    |
| 3                    | 10:45      | 11:00    |

### Custom behaviors

:::info
**v2.2.0**
Custom behaviors are supported only since version 2.2.0, using the `onCreate`, `onUpdate`, `onDelete` and `onCompute` fields.
:::

You can customize certain behaviors of an availability when it's created, updated, deleted or its occurrences and slots are computed using one of the availability fields described in the following subsections.

#### On create

The `onCreate` field contains an object whose properties determine how the availability behaves when it is created. The following table provides a list of the currently supported fields.

| Name                       | Type    | Default value | Default behavior                                                       | Custom behavior                                                                                                                                   |
|----------------------------|---------|---------------|------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------|
| `ignoreResourceAppointments` | boolean | `true`        | When a new availability is created, existing appointments are ignored. | When set to `false`, the creation of a new availability returns an error if the same resource already has booked appointments in the same period. |

:::info

The `ignoreResourceAppointments` field can be set to `false` only when creating single availabilities.
An error is returned if you try to set it to `false` on recurrent availabilities.

:::

#### On update

The `onUpdate` field contains an object whose properties determine how the availability behaves when it is updated. The AM does not currently support any custom behavior associated to update operations.

#### On delete

The `onDelete` field contains an object whose properties determine how the availability behaves when it is deleted. The AM does not currently support any custom behavior associated to delete operations.

#### On compute

The `onCompute` field contains an object whose properties determine how the availability occurrences and slots computation is performed. The following table provides a list of the currently supported fields.

| Name             | Type    | Default value | Default behavior                                                                                                         | Custom behavior                                                                                                                                                      |
|------------------|---------|---------------|--------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `ignoreExceptions` | boolean | `false`       | When we compute the status of the availability slots, if an exception overlaps a slot, its status will be `UNAVAILABLE`. | When set to `true`, when we compute the status of the availability slots, we ignore any exception overlapping a slot and its status will be `AVAILABLE` or `BOOKED`. |

## Exceptions

:::info
**v2.0.0**
This feature has been introduced in v2.0.0 and is not available for previous versions.
:::

Exceptions allow you to define when a resource is not available and provide a reason (a doctor on vacation or sick leave, an equipment out of use, ...).

An exception has the following basic properties:

- `startDate`: start date/time;
- `endDate`: end date/time;
- `reason`: reason for the absence or unavailability of the resource.

Due to their exceptional nature, they span over an arbitrary period of time and have higher priority than availabilities, meaning that if an exception overlaps one or more availabilities slots, the slots will report their status as `UNAVAILABLE` and any existing appointment will be flagged (see next section below for more details about appointments).

## Appointments

:::info
**v2.2.0**
From v2.2.0 a new `participants` field is used to track additional information about the participants when the [`isParticipantStatusAvailable` configuration option][is-participant-status-available] is set to `true`.
This field is fully managed by the AM and is accessible from the API in read only.
You must keep using the [configured custom fields][users] and the AM will ensure they always match the `participants` field.
:::

:::danger
**v2.0.0**
From v2.0.0 the `slotId` field is no longer used and has been replaced by `availabilityId`. **This change of the schema is breaking**, so if you are using AM v1.x you need to migrate your existing data to the new schema before upgrading to AM v2.
:::

An appointment represents a planned meeting to deliver some services and has the following basic properties:

- `availabilityId`: the availability the appointment belongs to;
- `startDate`: the start date/time;
- `endDate`: the end date/time;
- the delivery mode (with teleconsultation or not);
- if the appointment may need to be rescheduled because of conflicts (see section *Flagged appointments* below);
- two or more participants (the field names used are mapped in the service configuration file);
- a list with additional details about the participants, like if they are required or they accepted or declined the event.

The combination of the `availabilityId`, `startDate` and `endDate` uniquely identifies the slot the appointment belongs to. If the appointment does not belong to a slot and therefore to an availability, the `availabilityId` field should be omitted.

### Flagged appointments

:::info
**v2.0.0**
From v2.0.0 you have more freedom in term of updating or deleting availabilities and exceptions, which can affect appointments as illustrated in this section. See section [Usage][usage] for more details.
:::

When you perform one of the following operations:

- create an exception
- update an exception
- delete an availability
- update an availability

you can affect existing slots, making them no longer available and bringing existing appointment, booked in those slots, into an inconsistent state. AM tries to automatically track those conflicts and flag the appointments.

:::note
Due to technical constraints, under unexpected circumstances (networking issues, CRUD service not responding, ...), it may happen that appointments in an inconsistent state are not correctly flagged (false negatives). However, the system is designed to prevent having perfectly valid appointments being incorrectly flagged (false positives).
:::

### User participants

The service assumes that the users involved in an appointment (also referred as participants) are contained in one or more attributes of the appointment itself. For example, an appointment related to a visit may have the following structure:

```json
{
  "other_props": "...",
  "doctor": "doctor_id",
  "patients": ["patient_id_1"]
}
```

In this example, the participants are categorized in _doctor_ and _patients_ and contained in two different attributes.

The service gives you complete freedom in choosing which category of users you want to add to a teleconsultation, or to send messages for each appointment lifecycle phase (creation, update or deletion).

To do so, you can leverage the `users` property of the [service configuration][users]. Here you can specify the category of users you want to use, along with their properties for the messaging service.

Using the previous example, the configuration file for the service involving one doctor and a list of patients will be:

```json
{
  "users": {
    "doctor": {},
    "patients": {}
  }
}
```

:::note
These categories can contain additional properties that are useful only when using the messaging service, and can be left blank otherwise
:::

#### Participant status

If you set the [`isParticipantStatusAvailable` configuration field][is-participant-status-available] to `true`, for each participant, you can track some additional information:

- `status`: if the user (tentatively) accepted or declined the appointment;
- `required`: if the user is required to attend the appointment;
- `acceptanceRequired`: if the user is expected to explicitly accept or decline the appointment.

In addition, by setting the [`isUserAvailable` configuration field][is-user-available] to `true`, you can also add to each participant additional user properties, that you may need or want when displaying the appointment, like the first and last name, from a [users CRUD][crud-users].

This field is fully managed by the AM and updated automatically every time a participant is added or removed from the appointment through the [`users` custom fields][users].

This field is available in read only thorugh the API, therefore you can only set its value by configuring the default value of each field in the [`users` configuration field][participant-status].

Each user can update its own participation status by sending a request to the [`PATCH /appointments/:id/status`][patch-appointment-participant-status].

For example, if we had configured the AM with two custom user fields:

- `doctor`: the doctor accepts the appointment by default (`status`) and is required (`required`);
- `patients`: a list of patients, each one is required (`required`), acceptance status is unknown (`status`) and an explicit acceptance is required (`acceptanceRequired`);

and two user properties to add to each participant:

- first name (`firstName`);
- last name (`lastName`);

an appointment looking like this:

```json
{
  "doctor": "dr.juliana.dunam",
  "patients": ["walter.white", "mark.greene"]
}
```

would result in a `participants` field looking like this:

```json
{
  "doctor": "dr.juliana.dunam",
  "patients": ["walter.white", "mark.greene"],
  "participants": [
    {
      "id": "dr.juliana.dunam",
      "type": "doctor",
      "status": "accepted",
      "required": true,
      "userProperties": {
        "firstName": "Juliana",
        "lastName": "Dunam"
      }
    },
    {
      "id": "walter.white",
      "type": "patients",
      "status": "needs-action",
      "required": true,
      "acceptanceRequired": true,
      "userProperties": {
        "firstName": "Walter",
        "lastName": "White"
      }
    },
    {
      "id": "mark.greene",
      "type": "patients",
      "status": "needs-action",
      "required": true,
      "acceptanceRequired": true,
      "userProperties": {
        "firstName": "Mark",
        "lastName": "Greene"
      }
    }
  ]
}
```

## Sending messages

Whenever you create, update or delete an appointment, you may want to send a specific message to the participants.

Since version 2.3.0, the AM supports both the Messaging Service and the Notification Manager to send messages. You must choose which one you want to use when configuring the AM.

### Messaging Service

:::caution
In order to send messages you need deploy an instance of the [Messaging Service][messaging-service-doc] 
configure the [environment variables][environment-variables] and enable the integration in the [service configuration][service-configuration].
:::

Users are defined in the `users` property of the [service configuration][service-configuration].
This property is a map having as key the category of users you want to send messages to and as value another map linking
a message template id (see [Messaging Service][messaging-service-doc] documentation for more information) to
each appointment lifecycle phase.

In response of each change in the appointment lifecycle the service will send a message to each user category that has a
template id specified for that phase in the configuration. If a category is not in the configuration, or it does not have
a template id for that phase, it will simply not receive the message.

For example, consider the appointment of the last example and the following configuration.

```json
{
  "users": {
    "doctor": {
      "create": "doctor_create_message_template_id",
      "delete": "doctor_delete_message_template_id"
    },
    "patients": {
      "create": "patient_create_message_template_id",
      "update": "patient_update_message_template_id",
      "delete": "patients_delete_message_template_id"
    }
  }
}
```

When the appointment is *created*, both the doctor and the patients involved will receive a message using the dedicated template
id.

When the appointment is *updated*, only the patients involved will receive a message using the dedicated template id.

When the appointment is *deleted*, both the doctor and the patients involved will receive a message using the dedicated template
id.

For more details on how messages are sent in each phase of the lifecycle, see the [usage section][usage].

### Notification Manager

:::info

The Notification Manager is supported since version 2.3.0 of the AM.

:::

The [Notification Manager][notification-manager-doc] adopts an event-driven architecture and provides an high-level API to send messages and set reminders.

Unlike the Messaging Service, the AM sends to the NM an event looking like this:

```json
{
  "name": "AM/AppointmentCreated/v1",
  "key": "appointment-12345",
  "metadata": {
    "userFields": ["resourceId", "participantIds"]
  },
  "payload": {
    "startDate": "2023-08-01T09:30:00Z",
    "endDate": "2023-08-01T10:15:00Z",
    "status": "BOOKED",
    "lockExpiration": null,
    "resourceId": "auth0|dr.mario.rossi",
    "participantIds": [
      "auth0|jenny.king",
      "auth0|dr.mark.greene"
    ]
  }
}

```

and all the notification settings must be configured in the NM, including:

- which users receive messages for a given event (identified by user ID, group, role and/or cluster);
- which template is used to send the messages;
- on which channels the messages are sent;
- …

Therefore, if you use the Notification Manager, you must not configure the templates for each user category in the service configuration, but directly in the NM notification settings.

Please take a look at the [Notification Manager documentation][notification-manager-doc] for further configuration and usage details. 

## Setting reminders

When you create a new appointment, the service may set multiple reminders to send a message to the participants. Each reminder is characterized by a message template and the amount of time before the appointment that the reminder has to be sent.

Since version 2.3.0, the AM supports both the Messaging Service and the Notification Manager to set reminders. You must choose which one you want to use when configuring the AM.

### Messaging Service

:::caution
In order to send reminders you need to deploy an instance of the [Messaging Service][messaging-service-doc] and [Timer Service][timer-service-doc], 
configure the  [environment variables][environment-variables] and 
enable the integration in the [service configuration][service-configuration].
:::

To trigger the creation of the reminders at least one user category must have the `reminders` property defined in the configuration file
(see the [configuration page][configuration] for more information).

:::tip
You can avoid sending reminders for appointments created/updated below a given threshold by setting the `reminderThresholdMs` 
field in the configuration file (see the [service configuration section][reminders-threshold] for more information).
:::

As for messages, the service uses its [configuration][service-configuration] to know which users categories should receive reminders.

Going back to the example above, we can consider this appointment

```json
{
  "other_props": "...",
  "doctor": "doctor_id",
  "patients": ["patient_id_1"]
}
```

and this enriched configuration

```json
{
  "users": {
    "doctor": {
      "create": "doctor_create_message_template_id",
      "delete": "doctor_delete_message_template_id"
    },
    "patients": {
      "create": "patient_create_message_template_id",
      "update": "patient_update_message_template_id",
      "delete": "patients_delete_message_template_id",
      "reminders":[
         {
          "template": "patients_reminder_message_template_id_1",
          "reminderMilliseconds": 86400000
        },
        {
          "template": "patients_reminder_message_template_id_2",
          "reminderMilliseconds": 3600000
        },
      ]
    }
  }
}
```

When the appointment is *created*, two different reminders will be scheduled for the patients involved. The first reminder will be sent a day before the appointment using the template `patients_reminder_message_template_id_1`; the second one will be sent an hour before the appointment using the template `patients_reminder_message_template_id_2`.

When the appointment is *updated*, if the date of the appointment has been updated, the reminders scheduled for the patients will be rescheduled.

When the appointment is *deleted*, the reminders scheduled for the patients will be aborted.

### Notification Manager

As illustrated in the [messages section][nm-messages], the Notification Manager computes the reminders to schedule when processing an event based on the notification settings configured for the users.

Therefore, if you use the Notification Manager, you must not configure the reminders for each user category in the service configuration, but directly in the NM notification settings.

The Notification Manager still relies on the [Timer Service][timer-service-doc] to send reminders, but you must configure it directly in the NM, while you should left it disabled in the AM, by setting `isTimerAvailable` to `false`.

Please take a look at the [Notification Manager documentation][notification-manager-doc] for further configuration and usage details. 

## Functional test

This section provides several integration test suites written using Postman, that you can download and run against your environment.

### Appointment Manager basic interactions

This integration test suite covers the most common interactions from a client perspective, in particular:

- create a recurring availability (`POST /availabilities/`);
- create an exception (`POST /exceptions/`);
- view the calendar (`GET /calendar/`);
- book an appointment (`POST /appointments/`);
- cancel an appointment (`DELETE /appointments/:id`).

The test suite - a Postman collection and its environment - can be downloaded from the following links:

- <a download target="_blank" href="/docs_files_to_download/appointment-manager/integration_tests_basic.postman_collection.json">Postman collection</a>
- <a download target="_blank" href="/docs_files_to_download/appointment-manager/integration_tests_basic.postman_environment.json">Postman environment</a>.

### Notification Manager integration

This integration test suite covers the events sent to the [Notification Manager][notification-manager-doc] during an appointment lifecycle.

The test suite - a Postman collection and its environment - can be downloaded from the following links:

- <a download target="_blank" href="/docs_files_to_download/appointment-manager/integration_tests_notification_manager.postman_collection.json">Postman collection</a>
- <a download target="_blank" href="/docs_files_to_download/appointment-manager/integration_tests_notification_manager.postman_environment.json">Postman environment</a>.

The test suite covers the following operations:

- create an appointment (`POST /appointments/`);
- update an appointment (`PATCH /appointments/:id`);
- cancel an appointment (`DELETE /appointments/:id`);
- cancel multiple appointments (`POST /appointments/state`);


[crud-service-doc]: ../../runtime_suite/crud-service/overview_and_usage "CRUD Service"
[messaging-service-doc]: ../../runtime_suite/messaging-service/overview "Messaging Service"
[notification-manager-doc]: ../../runtime_suite/messaging-service/overview "Notification Manager"
[timer-service-doc]: ../../runtime_suite/timer-service/overview "Timer Service"
[teleconsultation-service-be-doc]: ../../runtime_suite/teleconsultation-service-backend/overview "Teleconsultation Service BE"

[overview-exceptions]: #exceptions "Exceptions | Overview"
[nm-messages]: #notification-manager

[configuration]: ./20_configuration.md "Configuration page"
[service-configuration]: ./20_configuration.md#service-configuration "Service configuration | Configuration"
[users]: ./20_configuration.md#users "`users` | Service configuration | Configuration"
[participant-status]: ./20_configuration.md#participant-status "Participant status | `users` | Service configuration | Configuration"
[is-participant-status-available]: ./20_configuration.md#isparticipantstatusavailable "`isParticipantStatusAvailable` | Service configuration | Configuration"
[is-user-available]: ./20_configuration.md#isuseravailable
[environment-variables]: ./20_configuration.md#environment-variables "Environment variables | Configuration"
[reminders-threshold]: ./20_configuration.md#reminderthresholdms "reminderThresholdMs | Service configuration | Configuration"
[crud-appointments]: ./20_configuration.md#appointments-crud-collection "Appointments CRUD collection | CRUD collections | Configuration"
[crud-users]: ./20_configuration.md#users-crud-collection

[usage]: ./30_usage.md "Usage page"
[patch-appointment-participant-status]: ./30_usage.md#patch-appointmentsidstatus
