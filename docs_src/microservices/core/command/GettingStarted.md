---
title: Core Command - Getting Started
---

# Core Command - Getting Started

## Commands via Messaging

### Introduction
Previously, communications from a 3rd party system (enterprise application, cloud application, etc.) to EdgeX in order to acuate a device or get the latest information from a sensor was only accomplished via REST.
The 3rd party system makes a REST call of the command service which then relays a request to a device service also using REST.
There was no built-in means to make a message-based request of EdgeX or the devices/sensors it manages.

From Levski release, core command service adds support for an external MQTT connection (in the same manner that app services provide an external MQTT connection),
which will allow it to act as a bridge between the internal message bus (implemented via MQTT) and external MQTT message bus.

#### Core Command as Message Bus Bridge

The Core Command service will serve as the EdgeX entry point for external, commands via message bus requests to the south side.

![image](../../../design/adr/command-msg.png)

3rd party systems should not be granted access to the EdgeX internal message bus. Therefore, in order to implement communications via message bus (specifically MQTT), the command service needs to take messages from the 3rd party or external MQTT topics and pass them internally onto the EdgeX internal message bus where they can eventually be routed to the device services and then on to the devices/sensors (southside).

In reverse, response messages from the southside will also be sent through the internal EdgeX message bus to the command service where they can then be bridged to the external MQTT topics and respond to the 3rd party system requester.
### Message Structure

Since most message bus protocols lack a generic message header mechanism (as in HTTP), providing request/response metadata is accomplished by defining a `MessageEnvelope` object associated with each request/response.
The message topic names act like the HTTP paths and methods in REST requests. That is, the topic names specify the device receiver of any command request as paths do in the HTTP requests.

![image](../../../design/adr/command-msg-structure.png)

#### Message Envelope
Below is an example of the `MessageEnvelope` for command query request:
```json
{
  "apiVersion" : "{{api_version}}",
  "RequestId": "e6e8a2f4-eb14-4649-9e2b-175247911369",
  "CorrelationID": "14a42ea6-c394-41c3-8bcd-a29b9f5e6835",
  "ContentType": "application/json",
  "QueryParams": {
    "offset": "0",
    "limit": "10"
  }
}
```

Below is an example of the `MessageEnvelope` of command query response:
```json
{
  "ApiVersion":"v2",
  "RequestID":"e6e8a2f4-eb14-4649-9e2b-175247911369",
  "CorrelationID":"14a42ea6-c394-41c3-8bcd-a29b9f5e6835",
  "ErrorCode":0,
  "Payload":"...",
  "ContentType":"application/json"
}
```

The messages for formatted requests and responses are sharing a common base structure.
The outermost JSON object represents the message envelope, which is used to convey metadata about request/response including `ApiVersion`, `RequestID`, `CorrelationID`...etc.

The `Payload` field contains the edgex event and reading.  
The `ErrorCode` field provides the indication of error.
The `ErrorCode` will be 0 (no error) or 1 (indicating error) as the two enums for error conditions.
When there is an error (with `ErrorCode` set to 1), then the payload contains a message string indicating more information about the error.
When there is no error (errorCode 0) then there is no message string in the payload.


### Command Query

Core Command service subscribes to the `QueryRequestTopic` and publishes the response to `QueryResponseTopic` defined in the configuration file.
After receiving the request, Core Command service will try to parse the `<device-name>` from request topic level.
The 3rd party system or application must publish command query requests messages and subscribe to responses from the same topics.
Below is the default topic naming used by Core Command:

- Subscribing command query request topic: `edgex/commandquery/request/#`
- Publishing command query response topic: `edgex/commandquery/response`

The last topic level in request topic must be either `all` or the `<device-name>` to query for.

#### Query by Device Name

Example of querying device core commands by device name via messaging:

1. Send query request message to external MQTT broker on topic `edgex/commandquery/request/Random-Boolean-Device`:
```json
{
  "apiVersion" : "{{api_version}}",
  "ContentType": "application/json",
  "CorrelationID": "14a42ea6-c394-41c3-8bcd-a29b9f5e6835",
  "RequestId": "e6e8a2f4-eb14-4649-9e2b-175247911369"
}
```

2. Receive query response message from external MQTT broker on topic `edgex/commandquery/response`:

!!! edgey "EdgeX 4.0"
    In EdgeX 4.0, base64 encoding is no longer applied by default to the payload. However, users can still enable base64 encoding by adding the environment variable to overwrite the default setting.

```json
{
   "apiVersion":"{{api_version}}",
   "receivedTopic":"edgex/commandquery/response",
   "correlationID":"14a42ea6-c394-41c3-8bcd-a29b9f5e6835",
   "requestID":"e6e8a2f4-eb14-4649-9e2b-175247911369",
   "errorCode":0,
   "payload":{
      "apiVersion":"{{api_version}}",
      "requestId":"e6e8a2f4-eb14-4649-9e2b-175247911369",
      "statusCode":200,
      "deviceCoreCommand":{
         "deviceName":"Random-Boolean-Device",
         "profileName":"Random-Boolean-Device",
         "coreCommands":[
            {
               "name":"WriteBoolValue",
               "set":true,
               "path":"/api/{{api_version}}/device/name/Random-Boolean-Device/WriteBoolValue",
               "url":"http://edgex-core-command:59882",
               "parameters":[
                  {
                     "resourceName":"Bool",
                     "valueType":"Bool"
                  },
                  {
                     "resourceName":"EnableRandomization_Bool",
                     "valueType":"Bool"
                  }
               ]
            },
            {
               "name":"WriteBoolArrayValue",
               "set":true,
               "path":"/api/{{api_version}}/device/name/Random-Boolean-Device/WriteBoolArrayValue",
               "url":"http://edgex-core-command:59882",
               "parameters":[
                  {
                     "resourceName":"BoolArray",
                     "valueType":"BoolArray"
                  },
                  {
                     "resourceName":"EnableRandomization_BoolArray",
                     "valueType":"Bool"
                  }
               ]
            },
            {
               "name":"Bool",
               "get":true,
               "set":true,
               "path":"/api/{{api_version}}/device/name/Random-Boolean-Device/Bool",
               "url":"http://edgex-core-command:59882",
               "parameters":[
                  {
                     "resourceName":"Bool",
                     "valueType":"Bool"
                  }
               ]
            },
            {
               "name":"BoolArray",
               "get":true,
               "set":true,
               "path":"/api/{{api_version}}/device/name/Random-Boolean-Device/BoolArray",
               "url":"http://edgex-core-command:59882",
               "parameters":[
                  {
                     "resourceName":"BoolArray",
                     "valueType":"BoolArray"
                  }
               ]
            }
         ]
      }
   },
   "contentType":"application/json"
}
```

#### Query All

Example of querying all device core commands via messaging:

1. Send query request message to external MQTT broker on topic `edgex/commandquery/request/all`:
```json
{
  "apiVersion" : "{{api_version}}",
  "ContentType": "application/json",
  "CorrelationID": "14a42ea6-c394-41c3-8bcd-a29b9f5e6835",
  "RequestId": "e6e8a2f4-eb14-4649-9e2b-175247911369",
  "QueryParams": {
    "offset": "0",
    "limit": "5"
  }
}
```

2. Receive query response message from external MQTT broker on topic `edgex/commandquery/response`:
```json
{
   "apiVersion":"v3",
   "receivedTopic":"edgex/commandquery/response",
   "correlationID":"14a42ea6-c394-41c3-8bcd-a29b9f5e6835",
   "requestID":"e6e8a2f4-eb14-4649-9e2b-175247911369",
   "errorCode":0,
   "payload":{
      "apiVersion":"v3",
      "requestId":"e6e8a2f4-eb14-4649-9e2b-175247911369",
      "statusCode":200,
      "totalCount":5,
      "deviceCoreCommands":[
         {
            "deviceName":"Random-Boolean-Device",
            "profileName":"Random-Boolean-Device",
            "coreCommands":[
               {
                  "name":"BoolArray",
                  "get":true,
                  "set":true,
                  "path":"/api/v3/device/name/Random-Boolean-Device/BoolArray",
                  "url":"http://edgex-core-command:59882",
                  "parameters":[
                     {
                        "resourceName":"BoolArray",
                        "valueType":"BoolArray"
                     }
                  ]
               },
               {
                  "name":"WriteBoolValue",
                  "set":true,
                  "path":"/api/v3/device/name/Random-Boolean-Device/WriteBoolValue",
                  "url":"http://edgex-core-command:59882",
                  "parameters":[
                     {
                        "resourceName":"Bool",
                        "valueType":"Bool"
                     },
                     {
                        "resourceName":"EnableRandomization_Bool",
                        "valueType":"Bool"
                     }
                  ]
               }
   ..............
   "contentType":"application/json"
}
```

### Command Request

Core Command service subscribes to the `CommandRequestTopic` defined in the configuration file.
After receiving the request, Core Command service will try to parse `<device-name>` `<command-name>` and `<method>` from request topic level,
and send the response back with `<device-name>`, `<command-name>` and `<method>` appended to `CommandResponseTopicPrefix` defined in the configuration file.
The 3rd party system or application must publish command requests messages and subscribe to responses from the same topics.
Below is the default topic naming used by Core Command:

- Subscribing command request topic: `edgex/command/request/#`
- Publishing command response topic: `edgex/command/response/<device-name>/<command-name>/<method>`

The last topic level (`<method>`) in request topic must be either `get` or `set`.

#### Get Command

Example of making get command request via messaging:

1. Send command request message to external MQTT broker on topic `edgex/command/request/Random-Boolean-Device/Bool/get`:
```json
{
  "apiVersion" : "{{api_version}}",
  "ContentType": "application/json",
  "CorrelationID": "14a42ea6-c394-41c3-8bcd-a29b9f5e6835",
  "RequestId": "e6e8a2f4-eb14-4649-9e2b-175247911369",
  "QueryParams": {
    "ds-pushevent": "false",
    "ds-returnevent": "true"
  }
}
```
2. Receive command response message from external MQTT broker on topic `edgex/command/response/#`:
```json
{
  "ReceivedTopic":"edgex/command/response/Random-Boolean-Device/Bool/get",
  "CorrelationID":"14a42ea6-c394-41c3-8bcd-a29b9f5e6835",
  "ApiVersion":"v2",
  "RequestID":"e6e8a2f4-eb14-4649-9e2b-175247911369",
  "ErrorCode":0,
  "Payload":"eyJhcGlWZXJzaW9uIjoidjIiLCJyZXF1ZXN0SWQiOiJlNmU4YTJmNC1lYjE0LTQ2NDktOWUyYi0xNzUyNDc5MTEzNjkiLCJzdGF0dXNDb2RlIjoyMDAsImV2ZW50Ijp7ImFwaVZlcnNpb24iOiJ2MiIsImlkIjoiM2JiMDBlODYtMTZkZi00NTk1LWIwMWEtMWFhNTM2ZTVjMTM5IiwiZGV2aWNlTmFtZSI6IlJhbmRvbS1Cb29sZWFuLURldmljZSIsInByb2ZpbGVOYW1lIjoiUmFuZG9tLUJvb2xlYW4tRGV2aWNlIiwic291cmNlTmFtZSI6IkJvb2wiLCJvcmlnaW4iOjE2NjY1OTE2OTk4NjEwNzcwNzYsInJlYWRpbmdzIjpbeyJpZCI6IjFhMmM5NTNkLWJmODctNDhkZi05M2U3LTVhOGUwOWRlNDIwYiIsIm9yaWdpbiI6MTY2NjU5MTY5OTg2MTA3NzA3NiwiZGV2aWNlTmFtZSI6IlJhbmRvbS1Cb29sZWFuLURldmljZSIsInJlc291cmNlTmFtZSI6IkJvb2wiLCJwcm9maWxlTmFtZSI6IlJhbmRvbS1Cb29sZWFuLURldmljZSIsInZhbHVlVHlwZSI6IkJvb2wiLCJ2YWx1ZSI6ImZhbHNlIn1dfX0=",
  "ContentType":"application/json",
  "QueryParams":{}
}
```

The Payload will be as below:
```json
{
  "apiVersion":"v2",
  "requestId":"e6e8a2f4-eb14-4649-9e2b-175247911369",
  "statusCode":200,
  "event":{
    "apiVersion":"v2",
    "id":"3bb00e86-16df-4595-b01a-1aa536e5c139",
    "deviceName":"Random-Boolean-Device",
    "profileName":"Random-Boolean-Device",
    "sourceName":"Bool",
    "origin":1666591699861077076,
    "readings":[
      {
        "id":"1a2c953d-bf87-48df-93e7-5a8e09de420b",
        "origin":1666591699861077076,
        "deviceName":"Random-Boolean-Device",
        "resourceName":"Bool",
        "profileName":"Random-Boolean-Device",
        "valueType":"Bool",
        "value":"false"
      }
    ]
  }
}
```

#### Set Command

Example of making put command request via messaging:

1. Send command request message to external MQTT broker on topic `edgex/command/request/Random-Boolean-Device/WriteBoolValue/set`:
```json
{
  "apiVersion" : "{{api_version}}",
  "ContentType": "application/json",
  "CorrelationID": "14a42ea6-c394-41c3-8bcd-a29b9f5e6835",
  "RequestId": "e6e8a2f4-eb14-4649-9e2b-175247911369",
  "Payload": "eyJCb29sIjogImZhbHNlIn0="
}
```

2. Receive command response message from external MQTT broker on topic `edgex/command/response/#`
```json
{
   "apiVersion":"{{api_version}}",
   "receivedTopic":"edgex/command/response/Random-Boolean-Device/Bool/get",
   "correlationID":"14a42ea6-c394-41c3-8bcd-a29b9f5e6835",
   "requestID":"e6e8a2f4-eb14-4649-9e2b-175247911369",
   "errorCode":0,
   "payload":{
      "apiVersion":"{{api_version}}",
      "event":{
         "apiVersion":"{{api_version}}",
         "deviceName":"Random-Boolean-Device",
         "id":"55eccd1b-d54b-4927-915e-67a787bf630f",
         "origin":1739791220978547200,
         "profileName":"Random-Boolean-Device",
         "readings":[
            {
               "deviceName":"Random-Boolean-Device",
               "id":"28e1b2b9-977d-4093-944c-d2b4df8c8040",
               "origin":1739791220978547200,
               "profileName":"Random-Boolean-Device",
               "resourceName":"Bool",
               "value":"true",
               "valueType":"Bool"
            }
         ],
         "sourceName":"Bool"
      },
      "requestId":"e6e8a2f4-eb14-4649-9e2b-175247911369",
      "statusCode":200
   },
   "contentType":"application/json"
}
```

!!! note "Note"
    There are some cases that Core Command service will be unable to publish the response correctly, for example:  
    - Response topic is not specified in configuration file  
    - Failed to JSON-decoding the request `MessageEnvelope`   
    - Failed to parse either `<device-name>`, `<command-name>` or `<method>`

### Configuring for secure MQTT connection

In real word, users usually need to provide credentials or certificates to connect to external MQTT broker.
To seed such secrets to Secret Store for Command service, you can follow the instructions from the [Seeding Service Secrets](../../../security/SeedingServiceSecrets.md) document.

The following example shows how to set up Command service to connect to external MQTT broker with `usernamepassword` authentication.

!!! example "Example - Setting SecretsFile and ExternalMQTT via environment override"
    ```yaml
    environment:
        EXTERNALMQTT_ENABLED: "true"
        EXTERNALLMQTT_URL: "<url>" # e.g. tcps://broker.hivemq.com:8883
        EXTERNALMQTT_AUTHMODE: usernamepassword
        SECRETSTORE_SECRETSFILE: "/tmp/core-command/secrets.json"
    ...
    volumes:
        - /tmp/core-command/secrets.json:/tmp/core-command/secrets.json
    ```

!!! example "Example - secrets.json"
    ```json
    {
        "secrets": [
            {
                "secretName": "mqtt",
                "imported": false,
                "secretData": [
                    {
                        "key": "username",
                        "value": "edgexuser"
                    },
                    {
                        "key": "password",
                        "value": "p@55w0rd"
                    }
                ]
            }
        ]
    }
    ```

!!! Note
    Since EdgeX 3.0, the `SecretPath` configuration property of `ExternalMQTT` section is renamed to `SecretName`.
    However, in [source code](https://github.com/edgexfoundry/go-mod-bootstrap/blob/3568057c2bc587f06c498046610b571516c920c3/config/types.go#L302-L303) it is still referred as `SecretPath` and will break down the Command service if ExternalMQTT is enabled.
    This is a known issue and will be fixed in EdgeX 3.1.
    Before EdgeX 3.1, to get rid of this issue you need to manually add `SecretPath` to configuration via [Consul UI](../../../api/core/Ch-APICoreConfigurationAndRegistry.md#consul-ui) and restart Command service to take effect.
    