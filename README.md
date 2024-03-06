## `Pulsebridge Gateway App`

[![pulsebridge-app build and test](https://github.com/aamitn/pulsebridge-app/actions/workflows/build.yml/badge.svg)](https://github.com/aamitn/pulsebridge-app/actions/workflows/build.yml)

Download APKs from: https://github.com/medic/pulsebridge-gateway-app/releases

---

An SMS gateway for Android. Send and receive SMS from your webapp via an Android phone.

```plaintext
+--------+                 +-----------+
|PBgatway|                 |pulsebridge| <-------- SMS
| server | <---- HTTP ---- |gateway-app|
|        |                 | (android) | --------> SMS
+--------+                 +-----------+
```

## Use

## Getting Started

Follow these three simple steps to get started with PulseBridge Gateway:

1.   Run PulseBridge Gateway Server by \`cloning\` the [**Repo**](https://github.com/aamitn/pulsebridge-gateway), following the [**Docs**](https://github.com/aamitn/pulsebridge-gateway/blob/master/README.md) and click `setup credentials` button. (\* Server requires valid SSL and https enabled for the mobile app to work). 
2.  [**Download**](https://github.com/aamitn/pulsebridge-app/releases/download/v1.0.9/pulsebridge-app-v1.0.9-release.apk) **PulseBridge Gateway Mobile App from release section of this** [**Repo**](https://github.com/aamitn/pulsebridge-app/) **and set the URL in app provided by the server.**
3.  **Send SMS from the Server Frontend or API**

    With the PulseBridge Gateway Server running, access the user-friendly interface at [https://domain.tld](http://localhost/) to send SMS messages. Alternatively, integrate the SMS functionality into your applications using the provided API.


## Installation

Install the latest APK from https://github.com/medic/pulsebridge-gateway-app/releases

## Configuration

### PulseBridge App

If you're configuring `pulsebridge-gateway-app` for use with hosted [`pulsebridge-gateway webserver`](https://github.com/medic/cht-core), with a URL of e.g. `https://example.com` and a username of `user`and a password of `password`, fill in the settings as follows:

```plaintext
WebappUrl: https://user:passowrd@example.com/api/sms
```

### Other

If you're configuring `pulsebridge-gateway-app` for use with other services, you will need to use the _generic_ build of `pulsebridge-gateway-app`, and find out the value for _CHT URL_ from your tech support.

### CDMA Compatibility Mode

Some CDMA networks have limited support for multipart SMS messages. This can occur within the same network, or only when sending SMS from a GSM network to a CDMA network. Check this box if `pulsebridge-gateway-app` is running on a GSM network and:

*   multipart messages sent to CDMA phones never arrive; or
*   multipart messages sent to CDMA phones are truncated

## Passwords

When using HTTP Basic Auth with gateway, all characters in the password must be chosen from the [ISO-8859-1](https://en.wikipedia.org/wiki/ISO/IEC_8859-1) characterset, excluding `#`, `/`, `?`, `@`.

## API

This is the API specification for communications between `pulsebridge-gateway-app` and a web server. Messages in both directions are `application/json`.

Where a list of values is expected but there are no values provided, it is acceptable to:

*   provide a `null` value; or
*   provide an empty array (`[]`); or
*   omit the field completely

Bar array behaviour specified above, `pulsebridge-gateway-app` _must_ include fields specified in this document, and the web server _must_ include all expected fields in its responses. Either party _may_ include extra fields as they see fit.

## Idempotence

N.B. messages are considered duplicate by `pulsebridge-gateway-app` if they have identical values for `id`. The webapp is expected to do the same.

`pulsebridge-gateway-app` will not re-process duplicate webapp-originating messages.

`pulsebridge-gateway-app` may forward a webapp-terminating message to the webapp multiple times.

`pulsebridge-gateway-app` may forward a delivery status report to the webapp multiple times for the same message. This should indicate a change of state, but duplicate delivery reports may be delivered in some circumstances, including:

*   the phone receives multiple delivery status reports from the mobile network for the same message
*   `pulsebridge-gateway-app` failed to process the webapp's response when the delivery report was last forwarded from `pulsebridge-gateway-app` to webapp

## Authorisation

`pulsebridge-gateway-app` supports [HTTP Basic Auth](https://en.wikipedia.org/wiki/Basic_access_authentication). Just include the username and password for your web endpoint when configuring `pulsebridge-gateway-app`, e.g.:

```plaintext
https://username:password@example.com/pulsebridge-gateway-api-endpoint
```

## Messages

The entire API should be implemented by a web application server at a single endpoint, e.g. https://exmaple.com/pulsebridge-gateway-app-api-endpoint

### GET

Expected response:

```plaintext
{
    "pulsebridge-gateway": true
}
```

### POST

`pulsebridge-gateway-app` will accept and process any relevant data received in a response. However, it may choose to only send certain types of information in a particular request (e.g. only provide a webapp-terminating SMS), and will also poll the web service periodically for webapp-originating messages, even if it has no new data to pass to the web service.

### Request

#### Headers

The following headers will be set by requests:

| header | value |
| --- | --- |
| `Accept` | `application/json` |
| `Accept-Charset` | `utf-8` |
| `Accept-Encoding` | `gzip` |
| `Cache-Control` | `no-cache` |
| `Content-Type` | `application/json` |

Requests and responses may be sent with `Content-Encoding` set to `gzip`.

#### Content

```plaintext
{
    "messages": [
        {
            "id": <String: uuid, generated by `pulsebridge-gateway-app`>,
            "from": <String: international phone number>,
            "content": <String: message content>,
            "sms_sent": <long: ms since unix epoch that message was sent>,
            "sms_received": <long: ms since unix epoch that message was received>
        },
        ...
    ],
    "updates": [
        {
            "id": <String: uuid, generated by webapp>,
            "status": <String: PENDING|SENT|DELIVERED|FAILED>,
            "reason": <String: failure reason (optional - only present for status:FAILED)>
        },
        ...
    ],
}
```

The status field is defined as follows.

| Status | Description |
| --- | --- |
| PENDING | The message has been sent to the gateway's network |
| SENT | The message has been sent to the recipient's network |
| DELIVERED | The message has been received by the recipient's phone |
| FAILED | The delivery has failed and will not be retried |

### Response

#### Success

##### HTTP Status: `2xx`

Clients may respond with any status code in the `200`\-`299` range, as they feel is  
appropriate. `pulsebridge-gateway-app` will treat all of these statuses the same.

##### Content

```plaintext
{
    "messages": [
        {
            "id": <String: uuid, generated by webapp>,
            "to": <String: local or international phone number>,
            "content": <String: message content>
        },
        ...
    ],
}
```

#### HTTP Status `400`+

Response codes of `400` and above will be treated as errors.

If the response's `Content-Type` header is set to `application/json`, `pulsebridge-gateway-app` will attempt to parse the body as JSON. The following structure is expected:

```plaintext
{
    "error": true,
    "message": <String: error message>
}
```

The `message` property may be logged and/or displayed to users in the `pulsebridge-gateway-app` UI.

#### Other response codes

Treatment of response codes below `200` and between `300` and `399` will _probably_ be handled sensibly by Android.

## SMS Retry Mechanism

Gateway will retry to send the SMS when any of these errors occurs: `RESULT_ERROR_NO_SERVICE`, `RESULT_ERROR_NULL_PDU` and `RESULT_ERROR_RADIO_OFF`.

1.  A possible temporary error occurs and Gateway retries sending the SMS:  
    1.1 SMS status will be updated to `UNSENT`, so Gateway will find it and add it into the `send queue` automatically.  
    1.2 SMS' `retry counter` increases by 1.  
    1.3 The retry attempt is scheduled based on this formula: `SMS' last activity time + ( 1 minute * (retry counter ^ 1.5) )`. This means the time between retries is incremental.  
    1.4 Gateway logs: the error, the retry counter and the retry scheduled time. Sample: `Sending SMS to +1123123123 failed (cause: radio off) Retry #5 in 15 min`
2.  Gateway has a maximum limit of attempts to retry sending SMS (currently 20), If this is reached then:  
    2.1 Gateway will hard fail the SMS by updating its status to `FAILED` and won't retry again.  
    2.2 Gateway logs error. Sample: `Sending message to +1123123123 failed (cause: radio off) Not retrying`
3.  At this point the user has the option of manually select the SMS and press `Retry` button.  
    3.1 If they do and SMS fails again, then the process will restart from step # 1.

## Development

## Requirements

Development guides are available in the "Android" section of the [Community Health Toolkit Docs Site](https://docs.communityhealthtoolkit.org/core/guides/android/). You will find instructions of how to setup your development environment, build and test new features, how to work with "flavor" apps, release, publish... and so on.

### `pulsebridge-gateway` ([Repo](hith))

*   composer

## Building locally

More details of how to setup and build the app [here](https://docs.communityhealthtoolkit.org/core/guides/android/development-setup/). The following are the most common tasks:

### Build and install

To build locally and install to an attached android device:

```plaintext
make 
```

OR, generate release APK using 

```plaintext
make assemble-release
```

OR, build using gradle

```plaintext
gradlew build -x test
```

### Tests

To run unit tests and static analysis tools locally:

```plaintext
make test
```

To run end to end tests, first either connect a physical device, or start an emulated android device, and then:

```plaintext
make test-ui
```

End to end tests only run in devices with Android _4.4 - 9.0_. Also it's possible that at the end of the tests when the SDK tries to uninstall the app from the device the following error is shown:

```plaintext
com.android.build.gradle.internal.testing.ConnectedDevice > runTests[4034G - 6.0] FAILED 
        com.android.builder.testing.api.DeviceException: com.android.ddmlib.InstallException: INSTALL_FAILED_VERSION_DOWNGRADE
```

Don't worry about that, it means the tests ran OK, but the SDK failed to remove the app for compatibility issues with your device, but this error only happens with the tests.

## Android Version Support

## "Default SMS/Messaging app"

Some changes were made to the Android SMS APIs in 4.4 (Kitkat®). The significant change was this:

> from android 4.4 onwards, apps cannot delete messages from the device inbox _unless they are set, by the user, as the default messaging app for the device_

Some reading on this can be found at:

*   http://android-developers.blogspot.com.es/2013/10/getting-your-sms-apps-ready-for-kitkat.html
*   https://www.addhen.org/blog/2014/02/15/android-4-4-api-changes/

Adding support for kitkat® means that there is some extra code in `pulsebridge-gateway-app` whose purpose is not obvious:

### Non-existent activities in `AndroidManifest.xml`

Activities `HeadlessSmsSendService` and `ComposeSmsActivity` are declared in `AndroidManifest.xml`, but are not implemented in the code.

### Unwanted permissions

The `BROADCAST_WAP_PUSH` permission is requested in `AndroidManifest.xml`, and an extra `BroadcastReceiver`, `MmsIntentProcessor` is declared. When `pulsebridge-gateway-app` is the default messaging app on a device, incoming MMS messages will be ignored. Actual WAP Push messages are probably ignored too.

### Extra intents

To support being the default messaging app, `pulsebridge-gateway-app` listens for `SMS_DELIVER` as well as `SMS_RECEIVED`. If the default SMS app, we need to ignore `SMS_RECEIVED`.

## Runtime Permissions

Since Android 6.0 (marshmallow), permissions for sending and receiving SMS must be requested both in `AndroidManifest.xml` and also at runtime. Read more at https://developer.android.com/intl/ru/about/versions/marshmallow/android-6.0-changes.html#behavior-runtime-permissions

## Copyright

Copyright 2018-2024 Bitmutex Technologies \<support@bitmutex.com>

## License

The software is provided under Apache 2.0 License. Contributions to this project are accepted under the same license.