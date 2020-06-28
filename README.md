# Introduction

ONYX is a specialized home automation system developed by HELLA Sonnen- und Wetterschutztechnik GmbH. It controls an array of shading products such as awnings, venetian blinds, and raffstores.

The system consists of the following main components:

- ONYX.CENTER is the central hub of the system
- ONYX.NODE and ONYX.CONNECTOR control the motors in the shading products.
- ONYX.WEATHER provides environmental data such as wind speed, temperature and brightness

ONYX provides a public API which can be used to query the state of the individual devices and to control them by sending control commands. This document describes the API and how to use it.

## Examples

The examples in this document use the `curl` utility to call the API and the `json_pp` utility to format the returned output for better readability. All examples have been tested using the `bash` shell on Linux and macOS with `curl` version 7.x. If you are using a different shell or operating system you may need to adjust the syntax accordingly.

# API Server

To use the API your ONYX.CENTER must be connected to the Internet. The API can be invoked by sending JSON commands via HTTPS to the server https://api.hella.link. The API server is operated by HELLA and only acts as a relay. All commands are forwarded directly to the individual ONYX.CENTERs and are not modified by the server.
To access the latest version of the API, make sure to install the most recent update on your ONYX.CENTER via the ONYX App which is available for Android and iOS.

HELLA provides the API service free of charge and without restrictions. We do not provide any uptime or performance guarantees.

# Access control

To use the API, client applications need the *fingerprint* of the ONYX.CENTER and an API *access token*. You can get both by performing these actions:

1.  Make sure your ONYX.CENTER is connected to the Internet and can access https://api.hella.link
2.  Open the ONYX App and connect to your ONYX.CENTER
3.  In the app, go to *Settings/Access Control* and tap on the "+" button to create a *temporary access code*. Enter a name for the API client and tap on the button to generate the code.
   
    This temporary code is only valid for 15 minutes and can be exchanged for a permanent API access token by sending it to the API endpoint at https://api.hella.link/authorize
   
    The server responds with status `200 - OK` and returns the *fingerprint* and *access token* if the temporary code is valid. If it is not, the server responds with status `401 - Unauthorized`.
    
    **Note:** You must use the returned API token to make at least one API request to make it permanent. Otherwise the token will be deleted after 15 minutes.

4. Once you have the *fingerprint* and API *access token* you can use them to call the API endpoints of your ONYX.CENTER.

    **Important:** The access token needs to be sent as a HTTP header with every request you make to the API in the following format: `Authorization: Bearer {access token}`

### Authorization Example

```
> curl -X POST  https://api.hella.link/authorize -H "Content-Type: application/json" -d '{"code" : "H99xV2yT"}'

{
    "fingerprint" : "b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61",
    "token" : "6EcYUqFHQulXobR7Cui1Vvplk111LZTn0KcLKieStCfQ6xiDKOFO7ZyV43333UgyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm"
}
```

## Revoke API access

You can revoke the API access for an application just like you would revoke access for any other device connected to your ONYX.CENTER. Just go to *Settings/Access Control* in the ONYX App, select the API access you want to revoke and delete it.

# API Description

## General Concepts

### Base URL and API Versions

All API endpoints are located at the following base URL:

https://api.hella.link/box/{box_fingerprint}/api/{api_version}/

Your ONYX center may support multiple API versions. You can query the supported versions with a GET request to the API endpoint https://api.hella.link/box/{box_fingerprint}/api/versions

Querying the supported API versions does not require an API token.

#### Example

```
> curl https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/versions
{"versions":["v1"]}
```

### Device Properties and Actions

Devices in ONYX have properties that describe their current state. To modify the state of a device you can send it a command to change the value of a property directly, or you can execute an `action`. Actions typically cause one or more property changes and are executed directly on the device. 

Properties can be readonly or read/write and they can have one of two types: `numeric` and `enumeration`. 

#### Numeric Properties

```
"temperature" : {
    "type" : "numeric",
    "readonly" : true,    
    "minimum" : -400,
    "maximum" : 1500,
    "value" : 261
}
```

Numeric properties, such as the `temperature` of an ONYX.WEATHER weather station, always have a `minimum` and `maximum` attribute to define their value range. The precision of the accepted values is defined by the device.
The `temperature` property in the example is marked as readonly and can not be changed by sending the device a command.

#### Enumeration Properties

```
"device_type" : {
    "type" : "enumeration",
    "value" : "rollershutter",
    "values" : [
        "awning",
        "raffstore_180",
        "raffstore_90",
        "rollershutter"
    ]
}
```

Enumeration properties, such as the `device_type` property in the example above, only accept a certain set of values. The possible values are listed in the `values` attribute.

#### Actions

Actions are simpler to handle than properties but they don't offer the precise control of changing properties values directly. Most acions affect multiple property values at once and cause the device to enter a defined state. The `stop` action for example, changes the `target_position` and `target_angle` properties of a device to stop it from moving.
The available actions for a device are defined as a simple list in the device details:

```
"actions" : [
    "stop"
    "open",
    "close"
]
```

### <a name="commands_and_priorities"></a>Device Commands and Command Priorities

You can only affect the state of a device via the API by sending a control command. A command can execute *one* action *or* change multiple property values at once. 
Each command has a priority, a start date (`valid_from`) and an expiration date (`best_before`). The `valid_from` date determines when a command becomes valid and can be used to schedule commands in the future. The `best_before` time defines when a command expires and should be deactivated for a device.

If a command is valid for a longer period or if packets are lost when communicating with a device, the commands may be sent multiple times by your ONYX.CENTER.

#### Command Priority

The priority of a command determines if it is sent to a device and if it suppresses other commands that may be present for a device. Currently, ONYX defines 3 command priority classes:

  - Safety
    
    This caetgory includes the wind, rain and hale sensors and is the most important. It suppresses any commands from the other categories

  - Interactive

    This category includes all user commands and commands sent via the API. This includes commands from wall switches, remote controls, Alexa and the ONYX Apps. Interactive commands suppress any commands from the convenience category.

    The interactive category is different from the other categories in two important ways. First, there can only be one interactive command per device. If a new command arrives from another source it cancels any other interactive command for that device.
    Second, if an interactive command can't be sent out immediately because it is suppressed by a safety command, the interactive command is discarded. This happens even if the interactive command has a later `best_before` date than the safety command.

  - Convenience
    
    This category includes all other automatic programms that are configured on your ONYX.CENTER, such as sun sensors and timers. Commands from the convenience category are only active when there are no interactive or safety commands for a device.

As mentioned above, all API commands have the priority "interactive" and are handled like any other user input.


## API Endpoints

### Clock and Timezone Information

<table>
    <tr>
        <th align="left">API Version</th><td>1.0</td>
    </tr>
    <tr>
        <th align="left">Method</th><td>GET</td>
    </tr>
    <tr>
        <th align="left">Path</th><td>/clock</td>
    </tr>
    <tr>
        <th align="left">Description</th>
        <td><p>Returns information on the system clock and timezone</p></td>
    </tr>
</table>

#### Notes

Your ONYX.CENTER might be located in a different timezone than your API client device or, if no NTP server can be contacted, the clock of your ONYX.CENTER might be out of sync. This endpoint gives you access to the system clock data to correct the start and end times for any commands you send and to offset the property timestamps of devices if needed.

#### Example

```
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v1/clock | json_pp
{
   "zone" : "Europe/Vienna",
   "time" : 1527456892.94839,
   "zone_offset" : 7200
}
```

### List all devices

<table>
    <tr>
        <th align="left">API Version</th><td>1.0</td>
    </tr>
    <tr>
        <th align="left">Method</th><td>GET</td>
    </tr>
    <tr>
        <th align="left">Path</th><td>/devices</td>
    </tr>
    <tr>
        <th align="left">Description</th>
        <td><p>Returns a list of all devices along with their names and device types</p></td>
    </tr>
</table>

#### Example

```
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v1/devices | json_pp
{
   "7d4e1ced-5ff3-4616-a1a8-4bb6c3ec00bf" : {
      "name" : "Awesome Weather Station",
      "type" : "weather"
   },
   "c0cabe90-764a-4f72-a820-cb3bf8e5c55e" : {
      "name" : "Bedroom Rollershutter 1",
      "type" : "rollershutter"
   },
   "32c4eea9-909a-49cb-a1f3-bf9141252f85" : {
      "name" : "Bedroom Rollershutter 2",
      "type" : "rollershutter"
   }
}
```

### List all groups

<table>
    <tr>
        <th align="left">API Version</th><td>1.0</td>
    </tr>
    <tr>
        <th align="left">Method</th><td>GET</td>
    </tr>
    <tr>
        <th align="left">Path</th><td>/groups</td>
    </tr>
    <tr>
        <th align="left">Description</th>
        <td><p>Returns a list of all groups along with their names and the devices in each group</p></td>
    </tr>
</table>

#### Example

```
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v1/groups | json_pp
{
   "36e956d3-b61d-410e-b248-73fcfc9c6494" : {
      "name" : "Secret Lair",
      "devices" : [
         "034a3ac8-8d62-4b41-95b5-d0f4e84420fa",
         "abdf224f-84e9-48a3-94d6-c2fda22f68aa",
         "3988d54d-b8c2-4a65-b45a-47725ef4964a"
      ]
   },
   "b964881b-02a2-44cd-bb56-ee6a6d74e9a8" : {
      "name" : "Bedroom",
      "devices" : [
         "034a3ac8-8d62-4b41-95b5-d0f4e84420fa",
         "3988d54d-b8c2-4a65-b45a-47725ef4964a"
      ]
   }
}
```

#### Device Details

<table>
    <tr>
        <th align="left">API Version</th><td>1.0</td>
    </tr>
    <tr>
        <th align="left">Method</th><td>GET</td>
    </tr>
    <tr>
        <th align="left">Path</th><td>/devices/{device_id}</td>
    </tr>
    <tr>
        <th align="left">Description</th>
        <td><p>Returns detail information on a device</p></td>
    </tr>
</table>

#### Example

```
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v1/devices/3988d54d-b8c2-4a65-b45a-47725ef4964a | json_pp
{
   "name" : "Bedroom Rollershutter 1",
   "type" : "rollershutter",
   "properties" : {
      "device_type" : {
         "type" : "enumeration",
         "value" : "rollershutter",
         "values" : [
            "awning",
            "raffstore_180",
            "raffstore_90",
            "rollershutter"
         ]
      },
      "target_position" : {
         "maximum" : 100,
         "value" : 100,
         "type" : "numeric"
      },
      "target_angle" : {
         "maximum" : 360,
         "type" : "numeric",
         "value" : 0
      }
   },
   "actions" : [
      "stop",
      "close",
      "open"
   ]
}
```

#### Group Details

<table>
    <tr>
        <th align="left">API Version</th><td>1.0</td>
    </tr>
    <tr>
        <th align="left">Method</th><td>GET</td>
    </tr>
    <tr>
        <th align="left">Path</th><td>/groups/{group_id}</td>
    </tr>
    <tr>
        <th align="left">Description</th>
        <td><p>Returns information on a single group</p></td>
    </tr>
</table>

#### Example

```
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v1/groups/36e956d3-b61d-410e-b248-73fcfc9c6494 | json_pp
{
   "name" : "Secret Lair",
   "devices" : [
      "abdf224f-84e9-48a3-94d6-c2fda22f68aa",
      "3988d54d-b8c2-4a65-b45a-47725ef4964a",
      "034a3ac8-8d62-4b41-95b5-d0f4e84420fa"
   ]
}
```

### <a name="device_command"></a>Device Command

<table>
    <tr>
        <th align="left">API Version</th><td>1.0</td>
    </tr>
    <tr>
        <th align="left">Method</th><td>POST</td>
    </tr>
    <tr>
        <th align="left">Path</th><td>/devices/{device_id}/command</td>
    </tr>
    <tr>
        <th align="left">Description</th>
        <td><p>Send a control command to a single device</p></td>
    </tr>
</table>

#### Notes

Read the section on [device commands and priorities](#commands_and_priorities) for more details. The `valid_from` and `best_before` attributes can be ommitted when creating a command. The default value for `valid_from` is "right now" and for `best_before` it is 15 minutes in the future. 

#### Example

Send a command with two properties and use the default start and end times:

```
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v1/devices/034a3ac8-8d62-4b41-95b5-d0f4e84420fa/command -X POST -d  '{"properties" : {"target_position" : 20}}' | json_pp
{
   "valid_from" : 1527510327,
   "best_before" : 1527510337,
   "properties" : {
      "target_position" : 20,
      "target_angle" : 45,
   }
}
```

Send a command with an action to execute with a defined start and end time:

```
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v1/devices/034a3ac8-8d62-4b41-95b5-d0f4e84420fa/command -X POST -d  '{"action" : "open"}, "valid_from" : 1527510702, "best_before": 1527510900}' | json_pp
{
   "valid_from" : 1527510727,
   "best_before" : 1527510737,
   "action" : "open"
}
```

### <a name="cancel_command"></a>Cancel Device Command

<table>
    <tr>
        <th align="left">API Version</th><td>1.0</td>
    </tr>
    <tr>
        <th align="left">Method</th><td>DELETE</td>
    </tr>
    <tr>
        <th align="left">Path</th><td>/devices/{device_id}/command</td>
    </tr>
    <tr>
        <th align="left">Description</th>
        <td><p>Cancels an active or scheduled command</p></td>
    </tr>
</table>

#### Notes

With this API endpoint, clients can cancel commands they have previously sent. Note that only commands that have been created with the same *authorization token* can be cancelled in this way. Cancelling an interactive command allows the ONYX.CENTER to send other active commands from the convenience priority class.

#### Example

```
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v1/devices/2b5e5e60-a9ff-4dab-a910-c3a6d475c37d/command -X DELETE
```

### Send Group Command

<table>
    <tr>
        <th align="left">API Version</th><td>1.0</td>
    </tr>
    <tr>
        <th align="left">Method</th><td>POST</td>
    </tr>
    <tr>
        <th align="left">Path</th><td>/groups/{group_id}/command</td>
    </tr>
    <tr>
        <th align="left">Description</th>
        <td><p>Send a control command to all devices in a group</p></td>
    </tr>
</table>

#### Notes

Read the section on the [Device Command Endpoint](#device_command) for more details.

Sending a command to a group creates a separate command for each device in the group. Thus, this function is only a performance optimization so only one request is needed to control multiple devices.

The JSON response for this API endpoint includes the generated command and the HTTP status codes that would have been returned, had the command been sent to each device individually.

#### Example

```
curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v1/groups/36e956d3-b61d-410e-b248-73fcfc9c6494/command -X POST -d  '{"action" : "close"}}' | json_pp
{
   "command" : {
      "best_before" : 1527541187,
      "action" : "close",
      "valid_from" : 1527541177
   },
   "results" : {
      "2b5e5e60-a9ff-4dab-a910-c3a6d475c37d" : {
         "status_code" : 200,
         "status_text" : "OK"
      },
      "0bbbbbcf-3713-4372-a224-81cd6b3a82ea" : {
         "status_code" : 200,
         "status_text" : "OK"
      },
      "40594f06-7f85-40e3-9877-e5c9f38a0dd2" : {
         "status_code" : 200,
         "status_text" : "OK"
      }
   }
}
```

### Delete Group Command

<table>
    <tr>
        <th align="left">API Version</th><td>1.0</td>
    </tr>
    <tr>
        <th align="left">Method</th><td>DELETE</td>
    </tr>
    <tr>
        <th align="left">Path</th><td>/groups/{group_id}/command</td>
    </tr>
    <tr>
        <th align="left">Description</th>
        <td><p>Cancels an active or scheduled command for all devices in a group</p></td>
    </tr>
</table>

#### Notes

Read the section on the [Cancel Device Command endpoint](#cancel_command) for more information. This API endpoint behaves in the same way but cancels the commands for all devices in a group.  


#### Example

```
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v1/groups/36e956d3-b61d-410e-b248-73fcfc9c6494/command -X DELETE
```


# Device Properties and Actions

ONYX currently supports two basic kinds of devices:

  - Motor controllers

    These are the ONYX.NODE and ONYX.CONNECTOR which can control raffstores, awnings and rollershutters.

  - Weather stations

    ONYX.WEATHER provides measurement data for brightness, wind speed, temperature, air pressure and humidity. Depending on the hardware revision, some measurements may be unavailable.

Depending on the device type and its settings, different properties and actions can be queried and controlled via the API. This section describes the most important properties and actions for the supported device types. API access to properties not in this document may be restricted in the future and should be avoided.

Note that the exposed properties and actions as well as their value ranges may differ from system to system and are also dependent on the firmware versions of the ONYX.CENTER and the individual devices. Your code should always check the value ranges of each property and should not use hardcoded values.

## Motor Controllers

### Properties

<table>
<tr>
    <th>Name</th>
    <th>Type</th>
    <th>Values</th>
    <th>Description</th>
</tr>
<tr>
    <td>device_type</td>
    <td>enumeration</td>
    <td>rollershutter, awning, raffstore_90, raffstore_180</td>
    <td><p>This property determines the operating mode of the motor controller and the way button presses are interpretet.</p></td>
</tr>
<tr>
    <td>target_position</td>
    <td>numeric</td>
    <td>0 - 100</td>
    <td><p>The position the shading element should move to. 0 means the element is completely retracted and 100 means it is fully extended.</p></td>
</tr>
<tr>
    <td>target_angle</td>
    <td>numeric</td>
    <td>0 - 180, depending on the device_type</td>
    <td><p>The angle the individual blades of the shading element should move to. The value range is dependent on the device type. 90 degrees means horizontal.</p></td>
</tr>
</table>

### Actions

<table>
<tr>
    <th>Name</th>
    <th>Description</th>
</tr>
<tr>
    <td>open</td>
    <td><p>This action sets the target_position to its minimum value and the target_angle to its maximum value, causing the device to fully retract.</p></td>
</tr>
<tr>
    <td>close</td>
    <td><p>This action sets the target_position to its maximum value and the target_angle to its minimum value, causing the device to fully extend.</p></td>
</tr>
<tr>
    <td>stop</td>
    <td><p>This action sets the target_position and target_angle to the values the device is currently at, causing the device to stop.</p></td>
</tr>
</table>

## Weather Stations

### Properties

<table>
<tr>
    <th>Name</th>
    <th>Type</th>
    <th>Values</th>
    <th>Description</th>
</tr>
<tr>
    <td>device_type</td>
    <td>enumeration</td>
    <td>weather</td>
    <td><p>Weather stations always have the device type set to "weather"</p></td>
</tr>
<tr>
    <td>wind_peak</td>
    <td>numeric</td>
    <td>0 - 500000 in mm/s</td>
    <td><p>The maximum wind speed measured in the last 15 minutes in mm/s. Divide by 1000 to convert to m/s.</p></td>
</tr>
<tr>
    <td>sun_brightness_peak</td>
    <td>numeric</td>
    <td>0 - 150000 lx</td>
    <td><p>The maximum brightness measured in the last 15 minutes in Lux.</p></td>
</tr>
<tr>
    <td>sun_brightness_sink</td>
    <td>numeric</td>
    <td>0 - 150000 lx</td>
    <td><p>The minimum brightness measured in the last 15 minutes in Lux.</p></td>
</tr>
<tr>
    <td>air_pressure</td>
    <td>numeric</td>
    <td>98582 - 150000, divide the value by 100 to get hPa</td>
    <td><p>The local absolute air pressure in hPa * 100. Note that this is absolute air pressure and varies depending on the altitude the weather station is located at.</p></td>
</tr>
<tr>
    <td>humidity</td>
    <td>numeric</td>
    <td>0 - 100 in %</td>
    <td><p>The relative humidity of the air around the weather station.</p></td>
</tr>
<tr>
    <td>temperature</td>
    <td>numeric</td>
    <td>-400 - 1500 in °C * 10</td>
    <td><p>The temperature of the weather station. Divide the value by 10 to get °C.</p></td>
</tr>
</table>


Copyright 2018 HELLA Sonnen- und Wetterschutztechnik GmbH