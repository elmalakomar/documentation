---
id: external
title: External Integration
sidebar_label: External
---
In this page you will find the required information to perform REST calls for external payments and custom integrations.

## Endpoints

Every endpoint has this prefix path `/v3/{external}`, where `{external}` is the name of the external service where the payment will be executed.

The external integrated service must expose the same endpoints, described below, exposed by the Payment Gateway Manager (or at least a reasonable subset of them). 

### Pay

`POST - /pay`
This endpoint allows to execute payments via the external service.

The `providerData` object will be sent to the external service.

### Refund

`POST - /refund`
This endpoint allows to execute a refund via the external service.

The `providerData` object will be sent to the external service.

### Status

`GET - /status?paymentId={paymentId}`
This endpoint allows to get the current status of the payment identified by the **required** query parameter `paymentId`.

### Check

`GET - /check?paymentId={paymentId}`
This endpoint allows to get the current status of the payment identified by the **required** query parameter `paymentId` and also to send a notification to the external service as specified by `PAYMENT_CALLBACK_URL` environment variable.

### Callback

`GET - /callback?paymentId={paymentId}`
This endpoint allows to send updates about the payment identified by the **required** query parameter `paymentId`. 
It should only be called by the external service.

### Schedule Subscription

`POST - /subscription/schedule`
This endpoint allows to schedule an automatic subscription (if supported) on the external service.

### Update Subscription

`POST - /subscription/update/{subscriptionToken}`
This endpoint allows to update the automatic subscription (if supported) referenced by the **required** path parameter `subscriptionToken`, on the external service.

### Start (manual) Subscription

`POST - /subscription/start`
This endpoint allows to start a manual subscription on the external service.

### Pay (manual) Subscription

`POST - /subscription/pay`
This endpoint allows to execute a manual payment related to a subscription on the external service.

### Expire Subscription

`DELETE - /subscription/expire/{subscriptionToken}`
This endpoint allows to expire the subscription referenced by the **required** path parameter `subscriptionToken`, on the external service.
