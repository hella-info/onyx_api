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
    - [List all automations](#list-all-automations)
    - [Enable or disable an automation](#enable-or-disable-an-automation)
    - [Device Details](#device-details)
    - [Group Details](#group-details)
    - [Device Command](#device-command)
    - [Cancel Device Command](#cancel-device-command)
    - [Send Group Command](#send-group-command)
    - [Delete Group Command](#delete-group-command)
    - [Relative Device Command](#relative-device-command)
    - [Relative Group Command](#relative-group-command)
    - [Update Status](#update-status)
    - [Check for Updates](#check-for-updates)
    - [Install an Update](#install-an-update)
    - [Log Out](#log-out)
    - [Event Stream](#event-stream)
    - [Device Property Time Series](#device-property-time-series)
    - [Device Online Time Series](#device-online-time-series)
    - [Time Series Statistics](#time-series-statistics)
- [Device Properties and Actions](#device-properties-and-actions-1)
  - [Motor Controllers](#motor-controllers)
  - [Pergolas](#pergolas)
  - [Lights](#lights)
  - [Weather Stations](#weather-stations)
  - [Sun Sensors](#sun-sensors)
  - [Temperature Sensors](#temperature-sensors)
  - [Rain Sensors and Alarm Inputs](#rain-sensors-and-alarm-inputs)

# Introduction

ONYX is a specialized home automation system developed by HELLA Sonnen- und Wetterschutztechnik GmbH. It controls an array of shading products such as awnings, venetian blinds, raffstores and pergolas.

The system consists of the following main components:

- [ONYX.CENTER](https://www.hella.info/produkte/onyx-center) is the central hub of the system
- [ONYX.NODE](https://www.hella.info/produkte/onyx-node), [ONYX.CONNECTOR](https://www.hella.info/de/produkte/onyx-connector) and ONYX.LED control the motors and lights in the shading products
- [ONYX.MOTOR](https://www.hella.info/produkte/onyx-r-silent-motor) is a specialized motor for rollershutters with ONYX built-in
- [ONYX.WEATHER](https://www.hella.info/de/produkte/onyx-weather) provides environmental data such as wind speed, temperature and brightness
- [ONYX.CLICK](https://www.hella.info/de/produkte/onyx-click) is a tiny handheld remote control for the system
- [ONYX.TAG wind](https://www.hella.info/de/produkte/onyx-tag-wind) is a small sensor to measure the vibration of awnings due to wind
- [ONYX.TAG sun](https://www.hella.info/de/produkte/onyx-tag-sun) is a small sensor to measure the sun brightness
- [ONYX.TAG temperature](https://www.hella.info/de/produkte/onyx-tag-temperature) is a small sensor to measure temperature and humidity

ONYX provides a public API which can be used to query the state of the individual devices and to control them by sending control commands. This document describes the API and how to use it.

## Examples

The examples in this document use the `curl` utility to call the API and the `jq` utility to format the returned output for better readability. All examples have been tested using the `bash` shell on Linux and macOS with `curl` version 8.x or newer. If you are using a different shell or operating system you may need to adjust the syntax accordingly.

For simplicity the examples all use the [HELLA API server](#hella-api-server). If you are using [local API access](#local-access), you need to replace the base URL of each request with the [mDNS Hostname](#address-resolution-using-mdns) of your ONYX.CENTER in your local network.

# API Access

The API can be invoked by sending JSON commands via HTTPS to a HELLA operated relay server or directly to an ONYX.CENTER in the local network.

To access the latest version of the API, make sure to install the most recent update on your ONYX.CENTER via the ONYX App which is available for Android and iOS.

## HELLA API Server

If your ONYX.CENTER is connected to the internet, the recommended way to access the API is via the HELLA API server at <https://api.hella.link>. The server is operated by HELLA, located in the EU and only acts as a relay. All commands are forwarded directly to the individual ONYX.CENTERs and are not modified in transit.

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

1. Make sure your ONYX.CENTER is connected to the Internet and can access <https://api.hella.link>
2. Open the ONYX App and connect to your ONYX.CENTER
3. In the app, go to _Settings/Access Control_ and tap on the "+" button to create a _temporary access code_. Enter a name for the API client and tap on the button to generate the code.

    This temporary code is only valid for 15 minutes and can be exchanged for a permanent API access token by sending it to the API endpoint at <https://api.hella.link/authorize>

    The server responds with status `200 - OK` and returns the _fingerprint_ and _access token_ if the temporary code is valid. If it is not, the server responds with status `401 - Unauthorized`.

    **Note:** You must use the returned API token to make at least one API request to make it permanent. Otherwise the token will be deleted after 15 minutes.

4. Once you have the _fingerprint_ and API _access token_ you can use them to call the API endpoints of your ONYX.CENTER.

    **Important:** The access token needs to be sent as an HTTP header with every request you make to the API in the following format: `Authorization: Bearer {access token}`

#### Example

```
> curl -X POST  https://api.hella.link/authorize -H "Content-Type: application/json" -d '{"code" : "H99xV2yT"}'  | jq

{
    "fingerprint" : "b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61",
    "token" : "6EcYUqFHQulXobR7Cui1Vvplk111LZTn0KcLKieStCfQ6xiDKOFO7ZyV43333UgyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm"
}
```

## Access Control using Local API Access

1. Open the ONYX App and connect to your ONYX.CENTER
2. In the app, go to _Settings/Access Control_ and tap on the "+" button to create a _temporary access code_. Enter a name for the API client and tap on the button to generate the code.

    This temporary code is only valid for 15 minutes and can be exchanged for a permanent API access token by sending it to the local API endpoint at https://LOCAL_ONYX_CENTER_HOSTNAME/api/v3/authorize

    The server responds with status `200 - OK` and returns the _fingerprint_ and _access token_ if the temporary code is valid. If it is not, the server responds with status `401 - Unauthorized`.

    **Note:** You must use the returned API token to make at least one API request to make it permanent. Otherwise the token will be deleted after 15 minutes.

3. Once you have the _fingerprint_ and API _access token_ you can use them to call the local API endpoints of your ONYX.CENTER.

    **Important:** The access token needs to be sent as an HTTP header with every request you make to the API in the following format: `Authorization: Bearer {access token}`

#### Example

```
> curl -k -X POST  https://ONYX-CENTER-C0-00-00-03.local./api/v3/authorize -H "Content-Type: application/json" -d '{"code" : "H99xV2yT"}' | jq

{
    "fingerprint" : "b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61",
    "token" : "6EcYUqFHQulXobR7Cui1Vvplk111LZTn0KcLKieStCfQ6xiDKOFO7ZyV43333UgyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm"
}
```

## Revoke API access

You can revoke the API access for an application just like you would revoke access for any other device connected to your ONYX.CENTER. Just go to _Settings/Access Control_ in the ONYX App, select the API access you want to revoke and delete it.

From API version 6 on, an application can also hand its own access back with the [Log Out](#log-out) endpoint. Well-behaved clients call it when they sign out or are uninstalled, so they do not leave a valid access behind.

# API Description

## General Concepts

### API Versions

Whenever possible you should use the latest API version supported by your ONYX.CENTER. There are currently five API versions that are officially supported and documented: v1, v3, v4, v5 and v6.

API version 2 was never publicly documented because it has several implementation issues. If your API client application relies on undocumented features available in API v2 you should upgrade your client application to API v4 to ensure compatiblity going forward.

API Version 3 introduces the following new features in addition to everything in v1:

- [Local API Access](#local-access)
- [Property Animations](#property-animations) to simplify client side property interpolation and to correctly track the variable drive speeds of newer devices like [ONYX.MOTOR](https://www.hella.info/de/produkte/onyx-r-silent-motor)
- [Command Attributes](#command-attributes) to control the way in which commands are executed by devices
- A [streaming API](#event-stream) conforming to the [W3C Server-Sent Events specification](https://www.w3.org/TR/eventsource/)

With the rollout of API v3, ONYX.CENTER now actively restricts API access to [publicly documented device properties and actions](#device-properties-and-actions). This applies to previous API versions as well.

API version 4 includes all features from version 3 but now also enables access to recorded [time series data](#device-property-time-series) for device property values. This is especially useful if you want to visualize or create reports for weather data collected by your ONYX sensors.

API version 5 includes all features from version 4 and introduces [relative device commands](#relative-device-command) and [time series aggregation](#aggregation). Relative commands allow you to adjust numeric property values by a delta rather than setting them to an absolute value. This is useful for incremental adjustments such as dimming a light up or down by a certain amount, without needing to know the current value first. Time series aggregation lets you request server-side summarization of property data in configurable time intervals using functions like average, minimum, maximum, count and sum.

API version 6 includes all features from version 5. It adds device tint colors, automation state and control, each device's currently active command and the [software update status](#update-status) of ONYX.CENTER to the REST API and [event stream](#event-stream), and it lets a client [give its own API access back](#log-out). The goal of this version is feature parity for everyday device control: an API client can now show the state of the system and control it like the ONYX App does, while system configuration intentionally stays exclusive to the app.

### Base URL

All API endpoints are located at the following base URL when using the [HELLA API server](#hella-api-server):

<https://api.hella.link/box/{box_fingerprint}/api/{api_version}/>

Your ONYX center may support multiple API versions. You can query the supported versions with a GET request to the API endpoint <https://api.hella.link/box/{box_fingerprint}/api/versions>

Querying the supported API versions does not require an API token.

When using local access you instead need to use the [local mDNS hostname](#address-resolution-using-mdns) of _your_ ONYX.CENTER to construct the base URL as follows:

[https://{mDNS_hostname}/api/versions]()

[https://{mDNS_hostname}/api/{api_version}/...]()

#### Examples

```
> curl https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/versions
{"versions":["v1", "v2", "v3", "v4", "v5", "v6"]}

> curl https://ONYX-CENTER-C0-00-00-03.local./api/versions
{"versions":["v1", "v2", "v3", "v4", "v5", "v6"]}
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
        <th align="left">API Version</th><td>v1, v3, v4, v5 and v6</td>
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
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v1/clock | jq
{
   "zone" : "Europe/Vienna",
   "time" : 1527456892.94839,
   "zone_offset" : 7200
}
```

### List all devices

<table>
    <tr>
        <th align="left">API Version</th><td>v1, v3, v4, v5 and v6</td>
    </tr>
    <tr>
        <th align="left">Method</th><td>GET</td>
    </tr>
    <tr>
        <th align="left">Path</th><td>/devices</td>
    </tr>
    <tr>
        <th align="left">Description</th>
        <td><p>Returns a list of all devices along with their names and device types. API v6 also includes the optional device tint color and active command.</p></td>
    </tr>
</table>

#### Example

```
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v1/devices | jq
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

Starting with API v6, devices with a configured tint include a `tint` field in `#RRGGBB` format:

```json
{
   "7d4e1ced-5ff3-4616-a1a8-4bb6c3ec00bf" : {
      "name" : "Awesome Weather Station",
      "tint" : "#0E99DA",
      "type" : "weather"
   }
}
```

The `tint` field is omitted when no tint is configured. API versions v1 through v5 do not expose it.

API v6 also includes the optional `active_command` field described by [Device Details](#device-details).

### List all groups

<table>
    <tr>
        <th align="left">API Version</th><td>v1, v3, v4, v5 and v6</td>
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
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v1/groups | jq
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

### List all automations

<table>
    <tr>
        <th align="left">API Version</th><td>v6</td>
    </tr>
    <tr>
        <th align="left">Method</th><td>GET</td>
    </tr>
    <tr>
        <th align="left">Path</th><td>/automations</td>
    </tr>
    <tr>
        <th align="left">Description</th>
        <td><p>Returns the safety and comfort automations configured on ONYX.CENTER.</p></td>
    </tr>
</table>

#### Notes

Each automation exposes only its identity and enabled state:

- `id`: The identifier to use with the [enable and disable endpoints](#enable-or-disable-an-automation)
- `name`: The display name assigned in the ONYX App
- `emoji`: The emoji assigned in the ONYX App
- `type`: A free-form type identifier assigned by the editor that created the automation, for example `wind`
- `enabled`: Whether the automation is currently active

`name`, `emoji` and `type` are omitted when they are not configured. The configuration of an automation is not exposed and cannot be changed through this API.

Treat `id` as an opaque string. Most automations are identified by a UUID, but an automation that belongs to a single device — the wind protection of a weather sensor, for example — is identified by a URI which has to be percent-encoded before it is used in a path. Such an automation frequently has no `name` of its own; the ONYX App shows it with the name of the device it belongs to.

Automations are listed in the same order as in the ONYX App: safety automations such as wind protection come first, followed by the comfort automations, each group in its configured order.

#### Example

```
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v6/automations | jq
[
   {
      "id" : "26f6b03e-86b1-46d9-8cad-42e0648d1a9c",
      "name" : "Wind protection",
      "emoji" : "🌬",
      "type" : "wind",
      "enabled" : true
   },
   {
      "id" : "http://o.hella.info/d/1ff1014/scenarios?type=info.hella.onyx.wind",
      "type" : "info.hella.onyx.wind",
      "enabled" : true
   },
   {
      "id" : "e94d6e40-3c17-4472-a0bb-fdb9dd7e6ed4",
      "name" : "Evening shade",
      "emoji" : "🌆",
      "type" : "sun",
      "enabled" : false
   }
]
```

### Enable or disable an automation

<table>
    <tr>
        <th align="left">API Version</th><td>v6</td>
    </tr>
    <tr>
        <th align="left">Method</th><td>POST</td>
    </tr>
    <tr>
        <th align="left">Paths</th><td>/automations/{automation_id}/enable<br/>/automations/{automation_id}/disable</td>
    </tr>
    <tr>
        <th align="left">Description</th>
        <td><p>Enables or disables an automation</p></td>
    </tr>
</table>

#### Notes

The change takes effect immediately: disabling an automation also cancels the commands it has issued, and enabling an automation allows it to take control of its devices right away. Keep in mind that disabled safety automations leave your shading products unprotected, for example from wind damage.

`{automation_id}` is a single path segment and has to be percent-encoded, which matters for the automations that belong to a device: their identifier is a URI, and every `:`, `/`, `?` and `=` in it has to be escaped — `encodeURIComponent()` in JavaScript, `urllib.parse.quote(id, safe="")` in Python, `--data-urlencode` or a pre-encoded string with `curl`. An identifier that is sent unencoded spreads over several path segments and fails with status `404 - Not Found`.

The request needs no body. A successful response contains the automation in the same format as [List all automations](#list-all-automations); unknown automation IDs fail with status `404 - Not Found`. These endpoints cannot create, edit or delete automations.

Both endpoints are idempotent: enabling an already enabled automation, or disabling an already disabled one, succeeds with status `200 - OK` and returns the automation in its current state. You therefore do not have to read the current state before setting it.

#### Example

```
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v6/automations/26f6b03e-86b1-46d9-8cad-42e0648d1a9c/disable -X POST | jq
{
   "id" : "26f6b03e-86b1-46d9-8cad-42e0648d1a9c",
   "name" : "Wind protection",
   "emoji" : "🌬",
   "type" : "wind",
   "enabled" : false
}
```

The wind protection of a weather sensor, whose identifier `http://o.hella.info/d/1ff1014/scenarios?type=info.hella.onyx.wind` is percent-encoded into a single path segment:

```
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v6/automations/http%3A%2F%2Fo.hella.info%2Fd%2F1ff1014%2Fscenarios%3Ftype%3Dinfo.hella.onyx.wind/enable -X POST | jq
{
   "id" : "http://o.hella.info/d/1ff1014/scenarios?type=info.hella.onyx.wind",
   "type" : "info.hella.onyx.wind",
   "enabled" : true
}
```

### Device Details

<table>
    <tr>
        <th align="left">API Version</th><td>v1, v3, v4, v5 and v6</td>
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

In API v6, the response includes the same optional `tint` field described by [List all devices](#list-all-devices). It also includes `active_command` while the device is executing a command:

```json
{
   "active_command" : {
      "source" : {
         "type" : "automation",
         "id" : "26f6b03e-86b1-46d9-8cad-42e0648d1a9c",
         "name" : "Wind protection",
         "emoji" : "🌬"
      },
      "properties" : {
         "target_position" : 0
      },
      "attributes" : [
         "fast"
      ],
      "valid_from" : 1527510327,
      "best_before" : 1527510900,
      "active_indicators" : [
         "wind"
      ]
   }
}
```

Only the currently active command is exposed. Future, queued and suppressed commands stay private, and the command contains only the actions and properties that are visible to your API client. `active_indicators` is an optional list of free-form tags that describe why the command is active, for example `wind` or `frost`.

The `source` object describes where the command came from, so that your application can display its origin without further lookups:

- `client`: The command was sent by an app, another API client or a local service. `name` and `emoji` are the display name and emoji of that client as shown in _Settings/Access Control_ of the ONYX App.
- `automation`: The command was issued by an automation. `id` refers to an entry in [List all automations](#list-all-automations). Automations without a name of their own inherit the name of the device they belong to, matching the ONYX App.
- `device`: The command was issued by another ONYX device, such as a wall switch, a remote control or a weather station. `id` is the public API ID of that device.

`id`, `name` and `emoji` are omitted when a source does not provide them. The whole `source` field is omitted when the command cannot be attributed, and `active_command` is omitted entirely while there is no active command. API versions v1 through v5 do not expose active commands.

```
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v1/devices/3988d54d-b8c2-4a65-b45a-47725ef4964a | jq
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

### Group Details

<table>
    <tr>
        <th align="left">API Version</th><td>v1, v3, v4, v5 and v6</td>
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
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v1/groups/36e956d3-b61d-410e-b248-73fcfc9c6494 | jq
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
        <th align="left">API Version</th><td>v1, v3, v4, v5 and v6</td>
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
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v1/devices/034a3ac8-8d62-4b41-95b5-d0f4e84420fa/command -X POST -d  '{"properties" : {"target_position" : 20}}' | jq
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
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v1/devices/034a3ac8-8d62-4b41-95b5-d0f4e84420fa/command -X POST -d  '{"action" : "open"}, "valid_from" : 1527510702, "best_before": 1527510900}' | jq
{
   "valid_from" : 1527510727,
   "best_before" : 1527510737,
   "action" : "open"
}
```

Move devices [_quietly_](#command-attributes) to a specified position:

```
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v1/devices/034a3ac8-8d62-4b41-95b5-d0f4e84420fa/command -X POST -d  '{"properties" : {"target_position" : 20}, "attributes": ["silent"]}' | jq
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
        <th align="left">API Version</th><td>v1, v3, v4, v5 and v6</td>
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
        <th align="left">API Version</th><td>v1, v3, v4, v5 and v6</td>
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
curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v1/groups/36e956d3-b61d-410e-b248-73fcfc9c6494/command -X POST -d  '{"action" : "close"}}' | jq
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
        <th align="left">API Version</th><td>v1, v3, v4, v5 and v6</td>
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

### Relative Device Command

<table>
    <tr>
        <th align="left">API Version</th><td>v5 and v6</td>
    </tr>
    <tr>
        <th align="left">Method</th><td>POST</td>
    </tr>
    <tr>
        <th align="left">Path</th><td>/devices/{device_id}/command/relative</td>
    </tr>
    <tr>
        <th align="left">Description</th>
        <td><p>Send a relative control command to a single device to adjust numeric property values by a delta</p></td>
    </tr>
</table>

#### Notes

Unlike the [Device Command](#device-command) endpoint which sets properties to absolute values, this endpoint adjusts numeric properties _relative_ to their current value. This is useful when you want to increment or decrement a property without first querying the current device state.

The request body must contain a `changes` object that maps property names to change descriptors. Each change descriptor has the following fields:

<table>
<tr>
    <th>Field</th>
    <th>Type</th>
    <th>Required</th>
    <th>Description</th>
</tr>
<tr>
    <td>by</td>
    <td>number</td>
    <td>yes</td>
    <td>The delta to apply to the current property value. Positive values increase the property, negative values decrease it. The result is clamped to the effective property range.</td>
</tr>
<tr>
    <td>min</td>
    <td>number</td>
    <td>no</td>
    <td>Optionally narrows the minimum of the effective range for this adjustment. The value is clamped to the property's native minimum.</td>
</tr>
<tr>
    <td>max</td>
    <td>number</td>
    <td>no</td>
    <td>Optionally narrows the maximum of the effective range for this adjustment. The value is clamped to the property's native maximum.</td>
</tr>
<tr>
    <td>on_limit</td>
    <td>object</td>
    <td>no</td>
    <td>A map of property names to change descriptors that should be applied instead when the target property is already at its effective limit in the direction of the change. This allows a single command to define fallback behavior, for example adjusting the angle when the position limit is reached. Only one level of nesting is allowed: entries in <code>on_limit</code> must not themselves contain <code>on_limit</code>.</td>
</tr>
</table>

Only numeric, writeable properties can be used with relative commands.

The optional top-level fields `valid_from`, `best_before` and `attributes` behave the same way as for the [Device Command](#device-command) endpoint.

#### Example

Increase the target position of a device by 10, falling back to increasing the target angle by 20 if the position is already at its limit:

```
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v5/devices/034a3ac8-8d62-4b41-95b5-d0f4e84420fa/command/relative -X POST -d '{"changes": {"target_position": {"by": 10, "on_limit": {"target_angle": {"by": 20}}}}}' | jq
{
   "changes": {
      "target_position": {
         "by": 10,
         "on_limit": {
            "target_angle": {
               "by": 20
            }
         }
      }
   },
   "valid_from": 1527510327,
   "best_before": 1527510337
}
```

Dim a light down by 5000 units, clamping the range to a minimum of 1000:

```
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v5/devices/53ab8f34-e3ea-449f-851f-c1a821bbd9aa/command/relative -X POST -d '{"changes": {"target_brightness": {"by": -5000, "min": 1000}}}' | jq
{
   "changes": {
      "target_brightness": {
         "by": -5000,
         "min": 1000
      }
   },
   "valid_from": 1527510327,
   "best_before": 1527510337
}
```

### Relative Group Command

<table>
    <tr>
        <th align="left">API Version</th><td>v5 and v6</td>
    </tr>
    <tr>
        <th align="left">Method</th><td>POST</td>
    </tr>
    <tr>
        <th align="left">Path</th><td>/groups/{group_id}/command/relative</td>
    </tr>
    <tr>
        <th align="left">Description</th>
        <td><p>Send a relative control command to all devices in a group</p></td>
    </tr>
</table>

#### Notes

Read the section on the [Relative Device Command](#relative-device-command) endpoint for more details on the request payload.

Sending a relative command to a group creates a separate command for each device in the group. The JSON response includes the generated command and the HTTP status codes that would have been returned, had the command been sent to each device individually.

#### Example

```
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v5/groups/36e956d3-b61d-410e-b248-73fcfc9c6494/command/relative -X POST -d '{"changes": {"target_position": {"by": -10}}}' | jq
{
   "command": {
      "changes": {
         "target_position": {
            "by": -10
         }
      },
      "valid_from": 1527541177,
      "best_before": 1527541187
   },
   "results": {
      "2b5e5e60-a9ff-4dab-a910-c3a6d475c37d": {
         "status_code": 200,
         "status_text": "OK"
      },
      "0bbbbbcf-3713-4372-a224-81cd6b3a82ea": {
         "status_code": 200,
         "status_text": "OK"
      },
      "40594f06-7f85-40e3-9877-e5c9f38a0dd2": {
         "status_code": 400,
         "status_text": "Bad Request"
      }
   }
}
```

### Update Status

<table>
    <tr>
        <th align="left">API Version</th><td>v6</td>
    </tr>
    <tr>
        <th align="left">Method</th><td>GET</td>
    </tr>
    <tr>
        <th align="left">Path</th><td>/update</td>
    </tr>
    <tr>
        <th align="left">Description</th>
        <td><p>Returns the installed ONYX.CENTER software version and the available update, if any</p></td>
    </tr>
</table>

#### Notes

The response contains the following fields:

- `current`: The currently installed software version
- `available`: The update that can be installed. This field is omitted while ONYX.CENTER is up to date.
- `available.progress`: The fraction of the update that has been transferred to ONYX.CENTER, between `0` and `1`. The field is omitted unless a transfer is running.

The status only refreshes when ONYX.CENTER checks for updates, either during its periodic background checks or when you trigger [Check for Updates](#check-for-updates).

#### Example

```
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v6/update | jq
{
   "current" : "2.13.0",
   "available" : {
      "version" : "2.14.0"
   }
}
```

### Check for Updates

<table>
    <tr>
        <th align="left">API Version</th><td>v6</td>
    </tr>
    <tr>
        <th align="left">Method</th><td>POST</td>
    </tr>
    <tr>
        <th align="left">Path</th><td>/update/check</td>
    </tr>
    <tr>
        <th align="left">Description</th>
        <td><p>Checks the HELLA update service for a new ONYX.CENTER software version</p></td>
    </tr>
</table>

#### Notes

The request needs no body and blocks until the check is complete. A successful response contains the refreshed [update status](#update-status). The check fails with status `500 - Internal Server Error` when ONYX.CENTER cannot reach the update service, for example without an internet connection.

#### Example

```
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v6/update/check -X POST | jq
{
   "current" : "2.13.0",
   "available" : {
      "version" : "2.14.0"
   }
}
```

### Install an Update

<table>
    <tr>
        <th align="left">API Version</th><td>v6</td>
    </tr>
    <tr>
        <th align="left">Method</th><td>POST</td>
    </tr>
    <tr>
        <th align="left">Path</th><td>/update/install</td>
    </tr>
    <tr>
        <th align="left">Description</th>
        <td><p>Installs the available ONYX.CENTER software update</p></td>
    </tr>
</table>

#### Notes

The request needs no body and starts the installation of the update reported by [Update Status](#update-status). It fails with status `500 - Internal Server Error` when there is no update to install.

Installing an update restarts ONYX.CENTER. Open API connections, including the [event stream](#event-stream), are closed during the restart and your client needs to reconnect once the update is complete.

#### Example

```
> curl -s -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v6/update/install -X POST | jq
{
   "current" : "2.13.0",
   "available" : {
      "version" : "2.14.0"
   }
}
```

### Log Out

<table>
    <tr>
        <th align="left">API Version</th><td>v6</td>
    </tr>
    <tr>
        <th align="left">Method</th><td>POST</td>
    </tr>
    <tr>
        <th align="left">Path</th><td>/logout</td>
    </tr>
    <tr>
        <th align="left">Description</th>
        <td><p>Revokes the API access the request was authorized with</p></td>
    </tr>
</table>

#### Notes

The request needs no body and answers with `204 - No Content` once the access has been revoked. From then on the access token is invalid: further requests answer with `401 - Unauthorized`, open [event streams](#event-stream) using it are closed, and the access no longer appears under _Settings/Access Control_ in the ONYX App.

Use this endpoint when your application signs out or is uninstalled. Deleting your copy of the access token is not enough on its own — the access stays on ONYX.CENTER until it is revoked here, [removed in the app](#revoke-api-access), or, for a temporary access, until it expires.

The endpoint revokes only the access used for the request. It cannot revoke any other access; use the ONYX App for that.

If ONYX.CENTER cannot revoke the access it answers with `503 - Service Unavailable`. Treat that response like an unreachable ONYX.CENTER: your application can still sign out locally, but the access remains valid until it expires or somebody removes it in the app.

#### Example

```
> curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v6/logout -X POST
204

# The token is dead afterwards
> curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v6/devices
401
```

### Event Stream

<table>
    <tr>
        <th align="left">API Version</th><td>v3, v4, v5 and v6</td>
    </tr>
    <tr>
        <th align="left">Method</th><td>GET</td>
    </tr>
    <tr>
        <th align="left">Path</th><td>/events</td>
    </tr>
    <tr>
        <th align="left">Description</th>
        <td><p>Streams changes to the device states in real time using <a href="https://html.spec.whatwg.org/multipage/server-sent-events.html">Server-Sent Events</a> and <a href="https://datatracker.ietf.org/doc/html/rfc7386">JSON Merge Patch</a></p></td>
    </tr>
</table>

#### Notes

The `/events` endpoint allows API clients to observe the state of ONYX devices and groups in real time. To use this endpoint you keep the HTTPS connection to ONYX.CENTER open and continually read events from the response body as they are generated. There are three types of events that are emitted:

- `pulse`: This event is used as a heartbeat. It is send periodically and contains as data the current system time in unix timestamp format.

- `snapshot`: The initial snapshot is immediately emitted when you open the connection to the `/events` endpoint and contains the current state of all devices as available through the [Device Details](#device-details) API endpoint. In API v6 it also contains the `automations` array from [List all automations](#list-all-automations) and, when known, the `update` status from [Update Status](#update-status).

- `patch`: After the initial snapshot, ONYX.CENTER emits `patch` events whenever the state of a device changes, new devices are added or existing devices are removed. Each `patch` is a [JSON Merge Patch](https://datatracker.ietf.org/doc/html/rfc7386) document that can be applied to the previous `snapshot` to update it to the current state of your ONYX system.

In API v6, the initial snapshot includes each configured device `tint` and `active_command`. Changing or clearing either value emits a device patch, as does a change to the resolved `source` of an active command, for example after renaming the device that issued it. Clearing a tint or removing an active command is represented by `null`, which removes the corresponding field when the merge patch is applied.

Automation changes replace the complete `automations` array in a patch, preserving its order. This includes enabling and disabling as well as adding, removing and reordering automations.

The `update` status is patched while an update is being transferred to ONYX.CENTER and when a transfer completes. An update discovered by [Check for Updates](#check-for-updates) does not patch already open event streams; it is reflected in the snapshot of new connections.

#### Example

```
> curl -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v3/events
event: snapshot
data: {"devices":{"53ab8f34-e3ea-449f-851f-c1a821bbd9aa":{"name":"LED 04-ff-00-28","type":"dimmable_light","actions":["light_off","light_on","stop","wink"],"properties":{"actual_brightness":{"type":"numeric","minimum":0,"maximum":65535,"value":0,"readonly":true},"dim_duration":{"type":"numeric","minimum":20,"maximum":3600000,"value":5000,"readonly":false},"target_brightness":{"type":"numeric","minimum":0,"maximum":65535,"value":0,"readonly":false}}},"667e3604-001b-4416-841f-647857c4fdb1":{"name":"NODE 00-00-02-f2","type":"pergola_slat_roof","actions":["close","open","stop","wink"],"properties":{"actual_angle":{"type":"numeric","minimum":0,"maximum":360,"value":0,"readonly":true},"actual_position":{"type":"numeric","minimum":0,"maximum":100,"value":0,"readonly":true},"system_state":{"type":"enumeration","value":"ok","values":["collision","collision_not_calibrated","not_calibrated","ok"],"readonly":true},"target_angle":{"type":"numeric","minimum":0,"maximum":360,"value":0,"readonly":false},"target_position":{"type":"numeric","minimum":0,"maximum":100,"value":0,"readonly":false}}},"ae8a1d92-97c0-4ad0-9810-69190e14585e":{"name":"CONNECTOR 02-00-09-2b","type":"raffstore_90","actions":["close","open","stop","wink"],"properties":{"actual_angle":{"type":"numeric","minimum":0,"maximum":360,"value":89,"readonly":true},"actual_position":{"type":"numeric","minimum":0,"maximum":100,"value":11,"readonly":true},"system_state":{"type":"enumeration","value":"ok","values":["collision","collision_not_calibrated","not_calibrated","ok"],"readonly":true},"target_angle":{"type":"numeric","minimum":0,"maximum":360,"value":89,"readonly":false},"target_position":{"type":"numeric","minimum":0,"maximum":100,"value":11,"readonly":false}}}},"groups":{}}

event: pulse
data: 1730803627

event: patch
data: {"devices":{"ae8a1d92-97c0-4ad0-9810-69190e14585e":{"properties":{"actual_angle":{"animation":{"current_value":89,"keyframes":[{"duration":-1,"interpolation":"linear","value":0}],"start":1627048004.7762835}},"actual_position":{"animation":{"current_value":11,"keyframes":[{"duration":5,"interpolation":"linear","value":100}],"start":1627048004.7762835}},"target_angle":{"value":0},"target_position":{"value":100}}}}}

event: pulse
data: 1730803657

event: patch
data: {"devices":{"53ab8f34-e3ea-449f-851f-c1a821bbd9aa":{"properties":{"actual_brightness":{"animation":{"current_value":0,"keyframes":[{"duration":6,"interpolation":"linear","value":65535}],"start":1627048014.1954331}},"dim_duration":{"value":6000},"target_brightness":{"value":65535}}}}}

event: patch
data: {"groups":{"5705b5d9-84dd-4541-b535-bf4391ee7140":{"devices":["ae8a1d92-97c0-4ad0-9810-69190e14585e","667e3604-001b-4416-841f-647857c4fdb1","53ab8f34-e3ea-449f-851f-c1a821bbd9aa"],"name":"Everything"}}}
```

### Device Property Time Series

<table>
    <tr>
        <th align="left">API Version</th><td>v4, v5 and v6</td>
    </tr>
    <tr>
        <th align="left">Method</th><td>GET</td>
    </tr>
    <tr>
        <th align="left">Path</th><td>/blackbox/{device_id}/properties/{property_name}.{format}?{query}</td>
    </tr>
    <tr>
        <th align="left">Description</th>
        <td><p>Download recorded time series data for device properties.</a></p></td>
    </tr>
</table>

#### Notes

With this API endpoint, clients can download time series data that ONYX records whenever device properties change. The number of available datapoints depends on the number of total devices in your system and how frequently their properties changed over time. Currently ONYX.CENTER stores data locally for up to one year.

Time series data can be downloaded in [CSV](https://en.wikipedia.org/wiki/Comma-separated_values), [CBOR](https://cbor.io) and [JSON](https://www.json.org) formats by using the matching filename extension in the request URL.

Using the optional query parameters, you can limit the returned data stream:

- `limit={n}`: Limits the maximum number of data points to `n` entries
- `offset={n}`: Skips the first `n` number of data points
- `order={asc|desc}`: Order the returned data points by timestamp either ascending or descending
- `from={timestamp}`: Only return data points that lie after the given UNIX `timestamp`
- `to={timestamp}`: Only return data points that lie before the given UNIX `timestamp`

#### Aggregation

> **Availability:** Aggregation query parameters require API **v5** or later.

To reduce the amount of transferred data, you can request server-side aggregation by specifying both of the following query parameters together:

- `interval={duration}`: The time bucket size. Supported units: `s` (seconds), `m` (minutes), `h` (hours), `d` (days), `w` (weeks). Examples: `15m`, `1h`, `1d`, `1w`.
- `aggregate={function}`: The aggregation function to apply per time bucket. Supported values:
  - `avg` – Average value (returned as a floating point number rounded to two decimal places)
  - `min` – Minimum value
  - `max` – Maximum value
  - `count` – Number of data points
  - `sum` – Sum of all values

When aggregation is active, each returned data point represents one time bucket. The timestamp corresponds to the start of the bucket and the value is the result of the aggregation function applied to all data points within that bucket. Empty buckets (time intervals with no data) are omitted.

> **Note:** Aggregation only works with numeric property values. Non-numeric values (e.g. text) are automatically excluded from the aggregation.

#### Example

```
# Download the latest 10 sun brightness values for a weather station in CSV format
> curl -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v4/blackbox/2fbdfc79-f798-4a1b-bd00-1643bcbabf4a/properties/sun_brightness.csv\?limit\=10\&order\=desc
2025-01-23T13:01:05.371904563Z,1339
2025-01-23T13:00:19.447694291Z,1475
2025-01-23T12:59:48.342021494Z,1620
2025-01-23T12:58:54.496925548Z,1917
2025-01-23T12:57:34.542690903Z,1780
2025-01-23T12:56:40.209122959Z,1825
2025-01-23T12:56:19.594088638Z,1764
2025-01-23T12:54:59.634808044Z,1848
2025-01-23T12:53:32.79664961Z,1475
2025-01-23T12:50:25.718903102Z,1696

# Download all available wind speed values for the 18th of January 2025 in CBOR format
>  curl -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v4/blackbox/2fbdfc79-f798-4a1b-bd00-1643bcbabf4a/properties/wind_peak.cbor\?from\=1737158400\&to\=1737244800 -o wind_peak_2025_01_18.cbor
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1160  100  1160    0     0   3836      0 --:--:-- --:--:-- --:--:--  3828

# Download all target position changes for a device in JSON format
> curl -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v4/blackbox/e1c1f5e8-bf80-443a-9edd-e6a40ca471b2/properties/target_position.json
[
    ["2025-01-23T17:24:24.603552174Z",100],
    ["2025-01-23T11:39:40.178658431Z",84],
    ["2025-01-23T10:02:50.300189514Z",84],
    ["2025-01-23T10:02:43.195747098Z",0],
    ["2025-01-23T06:49:33.390272474Z",100],
    ["2025-01-23T06:49:15.827330448Z",88],
    ["2025-01-23T06:49:10.511349074Z",0]
]

# Download hourly average sun brightness for a specific day
> curl -H "Authorization: Bearer ..." https://api.hella.link/box/.../api/v5/blackbox/2fbdfc79-f798-4a1b-bd00-1643bcbabf4a/properties/sun_brightness.json\?from\=1737158400\&to\=1737244800\&interval\=1h\&aggregate\=avg\&order\=asc
[
    ["2025-01-18T06:00:00Z",245.5],
    ["2025-01-18T07:00:00Z",892.33],
    ["2025-01-18T08:00:00Z",1450.75],
    ["2025-01-18T09:00:00Z",1823.17],
    ["2025-01-18T10:00:00Z",2105.0],
    ["2025-01-18T11:00:00Z",1987.5]
]

# Download the daily maximum wind peak for the last 7 days
> curl -H "Authorization: Bearer ..." https://api.hella.link/box/.../api/v5/blackbox/2fbdfc79-f798-4a1b-bd00-1643bcbabf4a/properties/wind_peak.json\?from\=1736553600\&to\=1737158400\&interval\=1d\&aggregate\=max\&order\=asc
[
    ["2025-01-11T00:00:00Z",42],
    ["2025-01-12T00:00:00Z",38],
    ["2025-01-13T00:00:00Z",67],
    ["2025-01-14T00:00:00Z",51],
    ["2025-01-15T00:00:00Z",29],
    ["2025-01-16T00:00:00Z",44],
    ["2025-01-17T00:00:00Z",35]
]
```

### Device Online Time Series

<table>
    <tr>
        <th align="left">API Version</th><td>v4, v5 and v6</td>
    </tr>
    <tr>
        <th align="left">Method</th><td>GET</td>
    </tr>
    <tr>
        <th align="left">Path</th><td>/blackbox/{device_id}/online.{format}?{query}</td>
    </tr>
    <tr>
        <th align="left">Description</th>
        <td><p>Download recorded time series data for devices online/offline state.</a></p></td>
    </tr>
</table>

#### Notes

With this API endpoint, clients can download time series data that ONYX records when the device connection state changes between online and offline. This endpoint supports the same formats and query parameters as [Device Property Time Series](#device-property-time-series).

#### Example

```
> curl -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v4/blackbox/8419d3b6-626d-4717-9d41-49daec7ac53e/online.json
[
    ["2025-01-23T18:24:37.752183809Z",true],
    ["2025-01-23T18:24:37.106369859Z",false],
    ["2025-01-23T18:02:30.236849327Z",true]
]
```

### Time Series Statistics

<table>
    <tr>
        <th align="left">API Version</th><td>v4, v5 and v6</td>
    </tr>
    <tr>
        <th align="left">Method</th><td>GET</td>
    </tr>
    <tr>
        <th align="left">Path</th><td>/blackbox</td>
    </tr>
    <tr>
        <th align="left">Description</th>
        <td><p>Download statistics and configuration information about the time series store.</a></p></td>
    </tr>
</table>

#### Notes

Using this API endpoint, clients can inspect information about the time series store. The information returned is in JSON format and contains the following fields:

```javascript
{
    // The total size of the time series database in bytes
    "size": 4276224,

    // The file size limit of the time series database on disk in bytes
    "limit": 300000000,

  "device_events": {

    // Timestamp of the newest entries in the database - rounded to months for performance reasons
    "newest": "2026-02-28T23:59:59.999999999Z",

    // Timestamp of the oldest entries in the database - rounded to months for performance reasons
    "oldest": "2025-12-01T00:00:00Z",

    // The age cutoff of entries when the size limit is reached
    "max_age": 31536000
  }
}
```

#### Example

```
> curl -H "Authorization: Bearer 6EcYUqFHQulXobR7Cui1Vvplk2111ZTn0KcLKieStCfQ6xiDKOFO7ZyV4o3333gyyTODsQpCFi1NXAK9Wd4zpC1HdWnKKS1FTH0oDzqz1L6zec5Ebqjx1Dfx303WMmm" https://api.hella.link/box/b381b2fc691ebbbce2a681b22d493e4ddcccaef58738f0caca4e9f61/api/v4/blackbox/ | jq
{
  "size": 4276224,
  "limit": 300000000,
  "device_events": {
    "newest": "2026-02-28T23:59:59.999999999Z",
    "oldest": "2025-12-01T00:00:00Z",
    "max_age": 31536000
  }
}
```

# Device Properties and Actions

ONYX currently supports the following basic kinds of devices:

- Motor controllers

  These are the ONYX.NODE, ONYX.CONNECTOR and ONYX.MOTOR which can control raffstores, awnings and rollershutters and other types of shading products.

- Pergolas

  ONYX.NODE, ONYX.CONNECTOR and the ONYX.PERGOLA control unit drive the slat roofs, awning roofs and sides of pergolas. They support all motor controller features, and the ONYX.PERGOLA control unit additionally measures temperature and humidity.

- Lights

  ONYX.NODE and ONYX.CONNECTOR can be configured to control normal light fixtures on one of their output relays. In addition we offer specialized ONYX.LED controllers that control the LED lighting in some awnings and pergolas and support dimming and, depending on the model, colors.

- Weather stations

  ONYX.WEATHER provides measurement data for brightness, wind speed, temperature, air pressure and humidity. Depending on the hardware revision, some measurements may be unavailable.

- Sun sensors

  ONYX.TAG sun supplies brightness measurement data.

- Temperature sensors

  ONYX.TAG temperature is able to measure the temperature and humidity.

- Rain sensors and alarm inputs

  ONYX.NODE can forward the state of wired contacts such as rain sensors and fire, wind or generic alarms.

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

## Pergolas

The pergola device types `pergola_slat_roof`, `pergola_awning_roof` and `pergola_side` support all [Motor Controller](#motor-controllers) properties and actions. Note that the textile side screens of a pergola may also be configured with the `screen` device type instead of `pergola_side`. The two are functionally equivalent.

To tilt the slats of a slat roof step by step without knowing their current angle, use a [relative command](#relative-device-command) on `target_angle`.

In addition, pergolas provide the following properties:

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
    <td>temperature</td>
    <td>numeric</td>
    <td>-400 - 850 in °C * 10</td>
    <td>no</td>
    <td>no</td>
    <td>The temperature measured by the ONYX.PERGOLA control unit. Divide the value by 10 to get °C. Pergolas driven by other controllers do not provide this property.</td>
</tr>
<tr>
    <td>humidity</td>
    <td>numeric</td>
    <td>0 - 100 in %</td>
    <td>no</td>
    <td>no</td>
    <td>The relative humidity measured by the ONYX.PERGOLA control unit. Pergolas driven by other controllers do not provide this property.</td>
</tr>
</table>

## Lights

ONYX currently supports three types of lights via the same properties and actions:

- `basic_light` devices can only be turned on and off and ignore the `dim_duration` property. The brightness can only be 0% or 100%
- `dimmable_light` devices support variable brightness levels and can smoothly transition between them
- `rgbw_light` devices additionally support the color properties listed below. Depending on `color_mode`, the light either shows the configured hue and saturation or a white tone with the configured color temperature.

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
<tr>
    <td>target_hue</td>
    <td>numeric</td>
    <td>0 - 359 degrees</td>
    <td>yes</td>
    <td>no</td>
    <td>The hue the light should transition to. Only available on <code>rgbw_light</code> devices.</td>
</tr>
<tr>
    <td>actual_hue</td>
    <td>numeric</td>
    <td>0 - 359 degrees</td>
    <td>no</td>
    <td>yes</td>
    <td>The current hue of the light. Only available on <code>rgbw_light</code> devices.</td>
</tr>
<tr>
    <td>target_saturation</td>
    <td>numeric</td>
    <td>0 - 100 in %</td>
    <td>yes</td>
    <td>no</td>
    <td>The color saturation the light should transition to. Only available on <code>rgbw_light</code> devices.</td>
</tr>
<tr>
    <td>actual_saturation</td>
    <td>numeric</td>
    <td>0 - 100 in %</td>
    <td>no</td>
    <td>yes</td>
    <td>The current color saturation of the light. Only available on <code>rgbw_light</code> devices.</td>
</tr>
<tr>
    <td>target_color_temperature</td>
    <td>numeric</td>
    <td>2700 - 10500 in Kelvin</td>
    <td>yes</td>
    <td>no</td>
    <td>The white color temperature the light should transition to. Only available on <code>rgbw_light</code> devices.</td>
</tr>
<tr>
    <td>actual_color_temperature</td>
    <td>numeric</td>
    <td>2700 - 10500 in Kelvin</td>
    <td>no</td>
    <td>yes</td>
    <td>The current white color temperature of the light. Only available on <code>rgbw_light</code> devices.</td>
</tr>
<tr>
    <td>color_mode</td>
    <td>enumeration</td>
    <td>hsv, temperature</td>
    <td>no</td>
    <td>no</td>
    <td>Whether the light currently shows the configured hue and saturation (<code>hsv</code>) or a white tone with the configured color temperature (<code>temperature</code>). Only available on <code>rgbw_light</code> devices.</td>
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
    <td>wind_speed</td>
    <td>numeric</td>
    <td>0 - 100000 in mm/s</td>
    <td><p>The current wind speed in mm/s. Divide by 1000 to convert to m/s.</p></td>
</tr>
<tr>
    <td>sun_brightness</td>
    <td>numeric</td>
    <td>0 - 150000 lx</td>
    <td><p>The brightness value of the last measurement</p></td>
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
    <td>temperature</td>
    <td>numeric</td>
    <td>-400 - 1500 in °C * 10</td>
    <td><p>The temperature of the weather station. Divide the value by 10 to get °C.</p></td>
</tr>
<tr>
    <td>humidity</td>
    <td>numeric</td>
    <td>0 - 100 in %</td>
    <td><p>The relative humidity of the air around the weather station.</p></td>
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

## Sun Sensors

### Properties

<table>
<tr>
    <th>Name</th>
    <th>Type</th>
    <th>Values</th>
    <th>Description</th>
</tr>
<tr>
    <td>sun_brightness</td>
    <td>numeric</td>
    <td>0 - 150000 lx</td>
    <td><p>The brightness value of the last measurement.</p></td>
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
</table>

## Temperature Sensors

### Properties

<table>
<tr>
    <th>Name</th>
    <th>Type</th>
    <th>Values</th>
    <th>Description</th>
</tr>
<tr>
    <td>temperature</td>
    <td>numeric</td>
    <td>-400 - 1500 in °C * 10</td>
    <td><p>The temperature of the sensor. Divide the value by 10 to get °C.</p></td>
</tr>
<tr>
    <td>humidity</td>
    <td>numeric</td>
    <td>0 - 100 in %</td>
    <td><p>The relative humidity of the air around the sensor.</p></td>
</tr>
</table>

## Rain Sensors and Alarm Inputs

Wired contacts such as rain sensors, fire alarms or generic alarm outputs of third-party systems can be connected to an ONYX.NODE input. Each configured contact appears as a separate device with the device type `input`. The state of these devices can only be read; how the system reacts to a triggered contact is configured on ONYX.CENTER itself.

Inputs that are configured to forward button presses to other devices are not exposed through the API. Their presses appear as the command source of the devices they control instead.

### Properties

<table>
<tr>
    <th>Name</th>
    <th>Type</th>
    <th>Values</th>
    <th>Description</th>
</tr>
<tr>
    <td>input_type</td>
    <td>enumeration</td>
    <td>rain_sensor, wind_alarm, generic_alarm, fire_alarm</td>
    <td><p>The role that is configured for the contact.</p></td>
</tr>
<tr>
    <td>input_active</td>
    <td>enumeration</td>
    <td>true, false</td>
    <td><p>Whether the contact is currently triggered. For a rain sensor, <code>true</code> means it is raining.</p></td>
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
    <td>This action causes the LED of the ONYX.NODE the contact is connected to to light up and can be used to identify devices.</td>
</tr>
</table>

Copyright 2024 HELLA Sonnen- und Wetterschutztechnik GmbH

```

```
