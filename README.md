# Contents

- [Introduction](#introduction)
  - [Examples](#examples)
- [API Access](#api-access)
  - [HELLA API Server](#hella-api-server)
  - [Local Access](#local-access)
    - [Address Resolution using mDNS](#address-resolution-using-mdns)
    - [Local TLS Peer Verification](#local-tls-peer-verification)
- [Access Control](#access-control)
  - [Access Control using the HELLA API Server](#access-control-using-the-hella-api-server)
  - [Access Control using Local API Access](#access-control-using-local-api-access)
  - [Revoke API access](#revoke-api-access)
- [API Description](#api-description)
  - [General Concepts](#general-concepts)
    - [API Versions](#api-versions)
    - [Base URL](#base-url)
    - [Device Properties and Actions](#device-properties-and-actions)
      - [Numeric Properties](#numeric-properties)
      - [Enumeration Properties](#enumeration-properties)
      - [Actions](#actions)
    - [Property Animations](#property-animations)
    - [Device Commands and Command Priorities](#device-commands-and-command-priorities)
      - [Command Priority](#command-priority)
      - [Command Attributes](#command-attributes)
  - [API Endpoints](#api-endpoints)
    - [Clock and Timezone Information](#clock-and-timezone-information)
    - [List all devices](#list-all-devices)
    - [List all groups](#list-all-groups)
      - [Device Details](#device-details)
      - [Group Details](#group-details)
    - [Device Command](#device-command)
    - [Cancel Device Command](#cancel-device-command)
    - [Send Group Command](#send-group-command)
    - [Delete Group Command](#delete-group-command)
    - [Event Stream](#event-stream)
- [Device Properties and Actions](#device-properties-and-actions-1)
  - [Motor Controllers](#motor-controllers)
  - [Lights](#lights)
  - [Weather Stations](#weather-stations)

# Introduction 

ONYX is a specialized home automation system developed by HELLA Sonnen- und Wetterschutztechnik GmbH. It controls an array of shading products such as awnings, venetian blinds, raffstores and pergolas.

The system consists of the following main components:

- [ONYX.CENTER](https://www.hella.info/produkte/onyx-center) is the central hub of the system
- [ONYX.NODE](https://www.hella.info/produkte/onyx-node), [ONYX.CONNECTOR](https://www.hella.info/de/produkte/onyx-connector) and ONYX.LED control the motors and lights in the shading products
- [ONYX.MOTOR](https://www.hella.info/produkte/onyx-r-silent-motor) is a specialized motor for rollershutters with ONYX built-in
- [ONYX.WEATHER](https://www.hella.info/de/produkte/onyx-weather) provides environmental data such as wind speed, temperature and brightness
- [ONYX.CLICK](https://www.hella.info/de/produkte/onyx-click) is a tiny handheld remote control for the system

ONYX provides a public API which can be used to query the state of the individual devices and to control them by sending control commands. This document describes the API and how to use it.

## Examples

The examples in this document use the `curl` utility to call the API and the `json_pp` utility to format the returned output for better readability. All examples have been tested using the `bash` shell on Linux and macOS with `curl` version 7.x. If you are using a different shell or operating system you may need to adjust the syntax accordingly.

For simplicity the examples all use the [HELLA API server](#hella-api-server). If you are using [local API access](#local-access), you need to replace the base URL of each request with the [mDNS Hostname](#address-resolution-using-mdns) of your ONYX.CENTER in your local network.

# API Access

The API can be invoked by sending JSON commands via HTTPS to a HELLA operated relay server or directly to an ONYX.CENTER in the local network.

To access the latest version of the API, make sure to install the most recent update on your ONYX.CENTER via the ONYX App which is available for Android and iOS.

## HELLA API Server

If your ONYX.CENTER is connected to the internet, the recommended way to access the API is via the HELLA API server at https://api.hella.link. The server is operated by HELLA, located in the EU and only acts as a relay. All commands are forwarded directly to the individual ONYX.CENTERs and are not modified in transit.

HELLA provides the API service free of charge and without restrictions. We do not provide any uptime or performance guarantees.

## Local Access

Starting with ONYX.CENTER firmware version 2.4 and API version 3 we also support direct, local access to the API. Local API access is available even when ONYX.CENTER can't connect to the internet but has the drawback that TLS peer verification and address resolution is more complex.

### Address Resolution using mDNS

ONYX uses [mDNS/DNS-SD aka Bonjour](https://datatracker.ietf.org/doc/html/rfc6762) to reliably connect to ONYX.CENTER even when the assigned IP address changes over time. Its important that you do not use fixed IP addresses when connecting to the local ONYX API and make sure that the host name is always resolved via mDNS.

In the following example we use [avahi-browse](https://manpages.debian.org/buster/avahi-utils/avahi-browse.1.en.html) on Debian Linux to search for an ONYX.CENTER on the local network and then use the discovered hostname to query its supported API versions with `curl` using [nss-mdns](https://github.com/lathiat/nss-mdns):

```
> avahi-browse -d local _https._tcp. -t --resolve
+   eno1 IPv6 b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61 Secure Web =   eno1 IPv4 b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61 Secure Web Site      local
   hostname = [ONYX-CENTER-C0-00-01-5e.local]
   address = [192.168.32.16]
   port = [443]
   txt = ["hardware_version=1.0.0" "name=ONYX.CENTER"]

> curl -k https://ONYX-CENTER-C0-00-00-03.local./api/versions
{"versions":["v1","v2","v3"]}
```

### Local TLS Peer Verification

For local API access its unfortunately not possible to verify the TLS certificate of ONYX.CENTER normally. For test purposes you can use the `-k` flag of curl to skip verification of the certificate completely.

If you are implementing a commercial product that integrates with ONYX we recommend that you make use of _certificate pinning_ after the first successful connection to protect against man-in-the-middle attacks.

# Access Control

To use the API, client applications need the _fingerprint_ of the ONYX.CENTER and an API _access token_. The steps to obtain the _access token_ differ slightly between the [HELLA API server](#hella-api-server) and [Local API Access](#local-access):

## Access Control using the HELLA API Server

1.  Make sure your ONYX.CENTER is connected to the Internet and can access https://api.hella.link
2.  Open the ONYX App and connect to your ONYX.CENTER
3.  In the app, go to _Settings/Access Control_ and tap on the "+" button to create a _temporary access code_. Enter a name for the API client and tap on the button to generate the code.

    This temporary code is only valid for 15 minutes and can be exchanged for a permanent API access token by sending it to the API endpoint at https://api.hella.link/authorize

    The server responds with status `200 - OK` and returns the _fingerprint_ and _access token_ if the temporary code is valid. If it is not, the server responds with status `401 - Unauthorized`.

    **Note:** You must use the returned API token to make at least one API request to make it permanent. Otherwise the token will be deleted after 15 minutes.

4.  Once you have the _fingerprint_ and API _access token_ you can use them to call the API endpoints of your ONYX.CENTER.

    **Important:** The access token needs to be sent as an HTTP header with every request you make to the API in the following format: `Authorization: Bearer {access token}`

#### Example

```
> curl -X POST  https://api.hella.link/authorize -H "Content-Type: application/json" -d '{"code" : "H99xV2yT"}'  | json_pp

{
    "fingerprint" : "b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61",
    "token" : "6EcYUqFHQulXobR7Cui1Vvplk111LZTn0KcLKieStCfQ6xiDKOFO7ZyV43333UgyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm"
}
```

## Access Control using Local API Access

1.  Open the ONYX App and connect to your ONYX.CENTER
2.  In the app, go to _Settings/Access Control_ and tap on the "+" button to create a _temporary access code_. Enter a name for the API client and tap on the button to generate the code.

    This temporary code is only valid for 15 minutes and can be exchanged for a permanent API access token by sending it to the local API endpoint at https://LOCAL_ONYX_CENTER_HOSTNAME/api/v3/authorize

    The server responds with status `200 - OK` and returns the _fingerprint_ and _access token_ if the temporary code is valid. If it is not, the server responds with status `401 - Unauthorized`.

    **Note:** You must use the returned API token to make at least one API request to make it permanent. Otherwise the token will be deleted after 15 minutes.

3.  Once you have the _fingerprint_ and API _access token_ you can use them to call the local API endpoints of your ONYX.CENTER.

    **Important:** The access token needs to be sent as an HTTP header with every request you make to the API in the following format: `Authorization: Bearer {access token}`

#### Example

```
> curl -k -X POST  https://ONYX-CENTER-C0-00-00-03.local./api/v3/authorize -H "Content-Type: application/json" -d '{"code" : "H99xV2yT"}' | json_pp

{
    "fingerprint" : "b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61",
    "token" : "6EcYUqFHQulXobR7Cui1Vvplk111LZTn0KcLKieStCfQ6xiDKOFO7ZyV43333UgyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm"
}
```

## Revoke API access

You can revoke the API access for an application just like you would revoke access for any other device connected to your ONYX.CENTER. Just go to _Settings/Access Control_ in the ONYX App, select the API access you want to revoke and delete it.

# API Description

## General Concepts

### API Versions

Whenever possible you should use the latest API version supported by your ONYX.CENTER. There are two API versions that are officially supported and documented: v1 and v3

API version 2 was never publicly documented because it has several implementation issues. If your API client application relies on undocumented features available in API v2 you should upgrade your client application to API v3 to ensure compatiblity going forward.

API Version 3 introduces the following new features in addition to everything in v1:

- [Local API Access](#local-access)
- [Property Animations](#property-animations) to simplify client side property interpolation and to correctly track the variable drive speeds of newer devices like [ONYX.MOTOR](https://www.hella.info/de/produkte/onyx-r-silent-motor)
- [Command Attributes](#command-attributes) to control the way in which commands are executed by devices
- A [streaming API](#device-event-source) conforming to the [W3C Server-Sent Events specification](https://www.w3.org/TR/eventsource/)

With the rollout of API v3, ONYX.CENTER now actively restricts API access to [publicly documented device properties and actions](#device-properties-and-actions). This applies to previous API versions as well.

### Base URL

All API endpoints are located at the following base URL when using the [HELLA API server](#hella-api-server):

https://api.hella.link/box/{box_fingerprint}/api/{api_version}/

Your ONYX center may support multiple API versions. You can query the supported versions with a GET request to the API endpoint https://api.hella.link/box/{box_fingerprint}/api/versions

Querying the supported API versions does not require an API token.

When using local access you instead need to use the [local mDNS hostname](#address-resolution-using-mdns) of _your_ ONYX.CENTER to construct the base URL as follows:

[https://{mDNS_hostname}/api/versions]()

[https://{mDNS_hostname}/api/{api_version}/...]()

#### Examples

```
> curl https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/versions
{"versions":["v1", "v2", "v3"]}

> curl https://ONYX-CENTER-C0-00-00-03.local./api/versions
{"versions":["v1", "v2", "v3"]}
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

### Property Animations

ONYX tries its best to limit the amount of radio traffic that is generated to keep system performance high and command latency low and this applies to property updates as well. The last known property values of each device are always cached on ONYX.CENTER and are only updated every few minutes unless the device state changes unexpectedly or a new command is sent to a device. This means that by the time you query the device state via the API, the devices most likely have already moved from their `actual_position`, `actual_angle`, etc.

When you need more precise position information, like for a real time visualization, it's the responsibility of your API client to extrapolate the new property values from the available device state.

To make these calculations possible, the ONYX API includes keyframe `animation` objects for those properties that need to be extrapolated, such as the `actual_position`, `actual_angle` and `actual_brightness`.
Animations describe which values a property will take on at specific times in the future and the interpolation method used between them.

Animations always include at least the following fields:

<table>
<tr>
    <th>Property</th>
    <th>Description</th>
</tr>
<tr>
    <td>start</td>
    <td>The point in time at which this animation starts as a UNIX timestamp.<br/>It is critically important that your API client and ONYX center have synchronized clocks, ideally using NTP, to ensure this timestamp is interpreted correctly.</td>
</tr>
<tr>
    <td>current_value</td>
    <td>The value of the property at the start of the animation.</td>
</tr>
<tr>
    <td>keyframes</td>
    <td>A list of keyframes that specify which values the property will take on in the future, the duration of the change and the interpolation curve used from one value to the next.</td>
</tr>
</table>

#### Example

```JSON
{
    "actual_position": {
        "animation": {
            "start": 1626706957.5299976,
            "current_value": 9,
            "keyframes": [
                {
                    "value": 11,
                    "duration": 2.132,
                    "interpolation": "linear"
                },
                {
                    "value": 82,
                    "duration": 31.416,
                    "interpolation": "linear"
                },
                {
                    "value": 100,
                    "duration": 19.47,
                    "interpolation": "linear"
                }
            ]
        }
    }
}
```

In the example above the animation for `actual_position` starts at 2021-07-19 15:02:37.5299976 UTC with the value `9`. It then progresses to the value 11 over a duration of 2.132 seconds with a `linear` interpolation. After that, it progresses to the value 82 over a duration of 31.416 seconds, also with a linear interpolation and so forth.

**Important:** The available interpolation methods may expand for future devices, even for already published API versions. If the `interpolation` value for a keyframe is not yet supported by your API client implementation you should always fallback to `linear`.

### Device Commands and Command Priorities

You can only affect the state of a device via the API by sending a control command. A command can execute _one_ action _or_ change multiple property values at once.
Each command has a priority, a start date (`valid_from`) and an expiration date (`best_before`). The `valid_from` date determines when a command becomes valid and can be used to schedule commands in the future. The `best_before` time defines when a command expires and should be deactivated for a device.

Commands can also specify optional [`attributes`](#command-attributes) that modify the way in which the commands should be executed by the devices.

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

#### Command Attributes

Command attributes convey additional information to the targeted device to specify **_how_** the command should be executed if possible. Devices can choose to ignore command attributes depending on the current operating conditions, if conflicting attributes are specified or if the device doesn't support them.

The following attributes are currently defined:

<table>
<tr>
    <th>Name</th>
    <th>Description</th>
</tr>
<tr>
    <td>urgent</td>
    <td>This command is urgent and should be executed as fast as possible, e.g. by moving supplying more power to a motor, even at the risk of overheating.<br/><b>Only use this attribute for safety critical functions.</b></td>
</tr>
<tr>
    <td>fast</td>
    <td>If the device can execute a command at several speeds, it should select the fastest one.</td>
</tr>
<tr>
    <td>slow</td>
    <td>If the device can execute a command at several speeds, it should select the slowest one.</td>
</tr>
<tr>
    <td>silent</td>
    <td>If the device can execute a command at several speeds, it should select the speed that causes the least noise.</td>
</tr>
<tr>
    <td>efficient</td>
    <td>If the device can execute a command at several speeds, it should select the most energy efficient one.</td>
</tr>
</table>

Currently only [ONYX.MOTOR](https://www.hella.info/produkte/onyx-r-silent-motor) is able to support command attributes.

## API Endpoints

### Clock and Timezone Information

<table>
    <tr>
        <th align="left">API Version</th><td>v1 and v3</td>
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
        <th align="left">API Version</th><td>v1 and v3</td>
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
        <th align="left">API Version</th><td>v1 and v3</td>
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
        <th align="left">API Version</th><td>v1 and v3</td>
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
   "actions" : [
      "close",
      "open",
      "stop",
      "wink"
   ],
   "properties" : {
      "actual_angle" : {
         "maximum" : 360,
         "minimum" : 0,
         "readonly" : true,
         "type" : "numeric",
         "value" : 0
      },
      "target_angle" : {
         "maximum" : 360,
         "minimum" : 0,
         "readonly" : false,
         "type" : "numeric",
         "value" : 0
      },
      "actual_position" : {
         "maximum" : 100,
         "minimum" : 0,
         "readonly" : true,
         "type" : "numeric",
         "value" : 100
      },
      "target_position" : {
         "maximum" : 100,
         "minimum" : 0,
         "readonly" : false,
         "type" : "numeric",
         "value" : 100
      },
      "system_state" : {
         "readonly" : true,
         "type" : "enumeration",
         "value" : "ok",
         "values" : [
            "collision",
            "collision_not_calibrated",
            "not_calibrated",
            "ok"
         ]
      }
   }
}
```

#### Group Details

<table>
    <tr>
        <th align="left">API Version</th><td>v1 and v3</td>
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

### Device Command

<table>
    <tr>
        <th align="left">API Version</th><td>v1 and v3</td>
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

Read the section on [device commands and priorities](#device-commands-and-command-priorities) for more details. The `valid_from` and `best_before` attributes can be ommitted when creating a command. The default value for `valid_from` is "right now" and for `best_before` it is 15 minutes in the future.

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

Move devices [_quietly_](#command-attributes) to a specified position:

```
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v1/devices/034a3ac8-8d62-4b41-95b5-d0f4e84420fa/command -X POST -d  '{"properties" : {"target_position" : 20}, "attributes": ["silent"]}' | json_pp
{
   "valid_from" : 1527510327,
   "best_before" : 1527510337,
   "attributes" : [
      "silent"
   ],
   "properties" : {
      "target_position" : 20,
      "target_angle" : 45,
   }
}
```

### Cancel Device Command

<table>
    <tr>
        <th align="left">API Version</th><td>v1 and v3</td>
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

With this API endpoint, clients can cancel commands they have previously sent. Note that only commands that have been created with the same _authorization token_ can be cancelled in this way. Cancelling an interactive command allows the ONYX.CENTER to send other active commands from the convenience priority class.

#### Example

```
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v1/devices/2b5e5e60-a9ff-4dab-a910-c3a6d475c37d/command -X DELETE
```

### Send Group Command

<table>
    <tr>
        <th align="left">API Version</th><td>v1 and v3</td>
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

Read the section on the [Device Command Endpoint](#device-command) for more details.

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
        <th align="left">API Version</th><td>v1 and v3</td>
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

Read the section on the [Cancel Device Command endpoint](#cancel-device-command) for more information. This API endpoint behaves in the same way but cancels the commands for all devices in a group.

#### Example

```
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v1/groups/36e956d3-b61d-410e-b248-73fcfc9c6494/command -X DELETE
```

### Event Stream

<table>
    <tr>
        <th align="left">API Version</th><td>v3 only</td>
    </tr>
    <tr>
        <th align="left">Method</th><td>GET</td>
    </tr>
    <tr>
        <th align="left">Path</th><td>/events</td>
    </tr>
    <tr>
        <th align="left">Description</th>
        <td><p>Streams changes to the device states in real time using <a href="https://www.w3.org/TR/eventsource/">Server-Sent Events</a> and <a href="https://datatracker.ietf.org/doc/html/rfc7386">JSON Merge Patch</a></p></td>
    </tr>
</table>

#### Notes

The `/events` endpoint allows API clients to observe the state of ONYX devices and groups in real time. To use this endpoint you keep the HTTPS connection to ONYX.CENTER open and continually read events from the response body as they are generated. There are two types of events that are emitted:

- `snapshot`: The initial snapshot is immediately emitted when you open the connection to the `/events` endpoint and contains the current state of all devices as available through the [Device Details](#device-details) API endpoint.

- `patch`: After the initial snapshot, ONYX.CENTER emits `patch` events whenever the state of a device changes, new devices are added or existing devices are removed. Each `patch` is a [JSON Merge Patch](https://datatracker.ietf.org/doc/html/rfc7386) document that can be applied to the previous `snapshot` to update it to the current state of your ONYX system.

#### Example

```
> curl -k -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v3/events
event: snapshot
data: {"devices":{"53ab8f34-e3ea-449f-851f-c1a821bbd9aa":{"name":"LED 04-ff-00-28","type":"dimmable_light","actions":["light_off","light_on","stop","wink"],"properties":{"actual_brightness":{"type":"numeric","minimum":0,"maximum":65535,"value":0,"readonly":true},"dim_duration":{"type":"numeric","minimum":20,"maximum":3600000,"value":5000,"readonly":false},"target_brightness":{"type":"numeric","minimum":0,"maximum":65535,"value":0,"readonly":false}}},"667e3604-001b-4416-841f-647857c4fdb1":{"name":"NODE 00-00-02-f2","type":"pergola_slat_roof","actions":["close","open","stop","wink"],"properties":{"actual_angle":{"type":"numeric","minimum":0,"maximum":360,"value":0,"readonly":true},"actual_position":{"type":"numeric","minimum":0,"maximum":100,"value":0,"readonly":true},"system_state":{"type":"enumeration","value":"ok","values":["collision","collision_not_calibrated","not_calibrated","ok"],"readonly":true},"target_angle":{"type":"numeric","minimum":0,"maximum":360,"value":0,"readonly":false},"target_position":{"type":"numeric","minimum":0,"maximum":100,"value":0,"readonly":false}}},"ae8a1d92-97c0-4ad0-9810-69190e14585e":{"name":"CONNECTOR 02-00-09-2b","type":"raffstore_90","actions":["close","open","stop","wink"],"properties":{"actual_angle":{"type":"numeric","minimum":0,"maximum":360,"value":89,"readonly":true},"actual_position":{"type":"numeric","minimum":0,"maximum":100,"value":11,"readonly":true},"system_state":{"type":"enumeration","value":"ok","values":["collision","collision_not_calibrated","not_calibrated","ok"],"readonly":true},"target_angle":{"type":"numeric","minimum":0,"maximum":360,"value":89,"readonly":false},"target_position":{"type":"numeric","minimum":0,"maximum":100,"value":11,"readonly":false}}}},"groups":{}}

event: patch
data: {"devices":{"ae8a1d92-97c0-4ad0-9810-69190e14585e":{"properties":{"actual_angle":{"animation":{"current_value":89,"keyframes":[{"duration":-1,"interpolation":"linear","value":0}],"start":1627048004.7762835}},"actual_position":{"animation":{"current_value":11,"keyframes":[{"duration":5,"interpolation":"linear","value":100}],"start":1627048004.7762835}},"target_angle":{"value":0},"target_position":{"value":100}}}}}

event: patch
data: {"devices":{"53ab8f34-e3ea-449f-851f-c1a821bbd9aa":{"properties":{"actual_brightness":{"animation":{"current_value":0,"keyframes":[{"duration":6,"interpolation":"linear","value":65535}],"start":1627048014.1954331}},"dim_duration":{"value":6000},"target_brightness":{"value":65535}}}}}

event: patch
data: {"groups":{"5705b5d9-84dd-4541-b535-bf4391ee7140":{"devices":["ae8a1d92-97c0-4ad0-9810-69190e14585e","667e3604-001b-4416-841f-647857c4fdb1","53ab8f34-e3ea-449f-851f-c1a821bbd9aa"],"name":"Everything"}}}
```

# Device Properties and Actions

ONYX currently supports three basic kinds of devices:

- Motor controllers

  These are the ONYX.NODE, ONYX.CONNECTOR and ONYX.MOTOR which can control raffstores, awnings and rollershutters and other types of shading products.

- Lights

  ONYX.NODE and ONYX.CONNECTOR can be configured to control normal light fixtures on one of their output relays. In addition we offer a specialized ONYX.LED controller that controls the LED lighting in some awnings and pergolas and supports dimming.

- Weather stations

  ONYX.WEATHER provides measurement data for brightness, wind speed, temperature, air pressure and humidity. Depending on the hardware revision, some measurements may be unavailable.

Depending on the device type and its settings, different properties and actions can be queried and controlled via the API. This section describes the supported properties and actions for each device type. API access to properties not in this document may be restricted in the future and should be avoided.

Note that the exposed properties and actions as well as their value ranges may differ from system to system and are also dependent on the firmware versions of the ONYX.CENTER and the individual devices. Your code should always check the value ranges of each property and should not use hardcoded values.

## Motor Controllers

### Properties

<table>
<tr>
    <th>Name</th>
    <th>Type</th>
    <th>Values</th>
    <th>Writeable</th>
    <th>Animated</th>
    <th>Description</th>
</tr>
<tr>
    <td>target_position</td>
    <td>numeric</td>
    <td>0 - 100</td>
    <td>yes</td>
    <td>no</td>
    <td>The position the shading element should move to. 0 means the element is completely retracted and 100 means it is fully extended.</p></td>
</tr>
<tr>
    <td>target_angle</td>
    <td>numeric</td>
    <td>0 - 180, depending on the device_type</td>
    <td>yes</td>
    <td>no</td>
    <td>The angle the individual blades of the shading element should move to. The value range is dependent on the device type. 90 degrees means horizontal.</td>
</tr>
<tr>
    <td>actual_position</td>
    <td>numeric</td>
    <td>0 - 100</td>
    <td>no</td>
    <td>yes</td>
    <td>The last known position of the shading element. 0 means the element is completely retracted and 100 means it is fully extended.</td>
</tr>
<tr>
    <td>actual_angle</td>
    <td>numeric</td>
    <td>0 - 180, depending on the device_type</td>
    <td>no</td>
    <td>yes</td>
    <td>The last known angle of the individual blades of the shading element. The value range is dependent on the device type. 90 degrees means horizontal.</td>
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
<tr>
    <td>wink</td>
    <td>This action causes the device to move a short distance in a direction and then back and can be used to identify devices.</td>
</tr>
</table>

## Lights

ONYX currently supports two types of lights via the same properties and actions:

- `basic_light` devices can only be turned on and off and ignore the `dim_duration` property. The brightness can only be 0% or 100%
- `dimmable_light` devices support variable brightness levels and can smoothly transition between them

### Properties

<table>
<tr>
    <th>Name</th>
    <th>Type</th>
    <th>Values</th>
    <th>Writeable</th>
    <th>Animated</th>
    <th>Description</th>
</tr>
<tr>
    <td>target_brightness</td>
    <td>numeric</td>
    <td>0 - 65535</td>
    <td>yes</td>
    <td>no</td>
    <td>The target brightness the device should dim to.</p></td>
</tr>
<tr>
    <td>actual_brightness</td>
    <td>numeric</td>
    <td>0 - 65535</td>
    <td>no</td>
    <td>yes</td>
    <td>The current brightness of the light.</p></td>
</tr>
<tr>
    <td>dim_duration</td>
    <td>numeric</td>
    <td>0 - 3600000 milliseconds</td>
    <td>yes</td>
    <td>no</td>
    <td>The duration the light takes to transition from 0 - 100% brightness. Set this property together with the target brightness to control the duration of the transition.</p></td>
</tr>
</table>

### Actions

<table>
<tr>
    <th>Name</th>
    <th>Description</th>
</tr>
<tr>
    <td>light_on</td>
    <td>This action turns the light back back on to the previous brightness level. If the previous level was very low it is increased to a minimum brightness.</td>
</tr>
<tr>
    <td>light_off</td>
    <td>This action turns the light off, storing the current brightness.</td>
</tr>
<tr>
    <td>stop</td>
    <td>This action causes dimming to stop at the current brightness level.</td>
</tr>
<tr>
    <td>wink</td>
    <td>This action causes the light to briefly turn on and off and can be used to identify devices.</td>
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

### Actions

<table>
<tr>
    <th>Name</th>
    <th>Description</th>
</tr>
<tr>
    <td>wink</td>
    <td>This action causes the internal LED light of the weather station to turn on for a couple of seconds and can be used to identify devices.</td>
</tr>
</table>

Copyright 2021 HELLA Sonnen- und Wetterschutztechnik GmbH
