# Zenly Protobuf files

This repo contains reverse-engineered and re-created from
scratch `.proto` files for interacting with Zenly.

They are pretty incomplete (particularly, there are almost no enums), 
but basic functions do work and are described below.

## Table of Contents

* [Usage](#usage)
   * [Headers](#headers)
* [How?](#how)
   * [Intercepting](#intercepting)
   * [Analyzing](#analyzing)
* [Contributing](#contributing)
* [Authorization](#authorization)
   * [Creating session](#creating-session)
      * [SessionCreate](#sessioncreate)
      * [SessionVerify](#sessionverify)
      * [SessionRequestCall](#sessionrequestcall)
   * [Extracting session](#extracting-session)
* [Avatars](#avatars)
* [Users](#users)
   * [Me](#me)
   * [Friends](#friends)
* [Getting location](#getting-location)
   * [PinContextSubscribeStream](#pincontextsubscribestream)
   * [TrackingContextSubscribeStream](#trackingcontextsubscribestream)
* [Sending location](#sending-location)
  * [TrackingContextPublishStream](#trackingcontextpublishstream)
* [License](#license)

# Usage

Zenly uses gRPC to interact with the server. I'm not going
to teach you here how to use it, there are plenty of resources
available on the Internet, as well as GUI clients (I use Kreya).

- RPC endpoint: `https://rpc.znly.co`  
- gRPC mode: Normal 
  - server **does not** support grpc-web

## Headers

Zenly doesn't really enforce any headers to use the API 
(apart from `Authrozation` and gRPC related), but the app sends the following ones:

```yaml
content-type: application/grpc
user-agent: grpc-go/1.29.1
te: trailers
grpc-encoding: gzip
grpc-accept-encoding: gzip
authorization: Token $TOKEN
cycleuuid: c-${randomBytes(16).toString('base64')}
appversion: 4.63.14
string_cache_len: !
string_cache_hash: ENGYQ@"
appbundle: app.zenly.locator
apptype: android
coreversion: 1.96.7
```

> Obviously, versions *will* vary between installations

> When no session is available, `$TOKEN = ''` and the app sends simply `Authorization: Token`

> `cycleuuid` seems to be some client-generated session ID,
> not persisted across restarts. 


# How?

Zenly's client apps have the so called "Zenly Core" bundled
within them. Being a Golang library, it's compiled in native
code and bundled in `lib/<arch>/libgojni.so` file.

There are 2 main ways to work with it: either intercepting 
requests at runtime, or analyzing the binary. I use both
to ensure at least *some* consistency.

## Intercepting

Golang has some weird SSL pinning stuff, I couldn't figure
out how to hook it with Frida. However, the binary includes
a dev-env self-signed certificate which is included into
trust chain at startup. I simply swapped it with my certificate
used for MITM, and replaced the rest of that string with `0x0A` 
(aka `\n`).

I tried using Charles and Proxyman, but for some reason they
both don't work: Proxyman doesn't support HTTP/2, and Charles
for some reason didn't display the requests (though they did
come through).

I ended up using mitmproxy, which worked *kinda* fine with some 
adjustments (i.e. insecure mode and other config options), but still
it wasn't able to capture gRPC streams at all. 

## Analyzing

Another way is to analyze the binary statically by throwing it
into IDA 7.7 (or other tool) and looking at structs. Golang doesn't
strip any metadata, so field names and class names are recovered.

Also Zenly has a pretty consistent naming which helps a lot.

# Contributing

This document and repo are by no means complete nor full. 
If you want to, you are more than welcome to contribute to
either these docs, or to `.proto` files.

# Authorization

Authorization in Zenly is session-based. Once authorized, you 
get a session token, which is valid as long as session is not revoked.

Particularly, the session is revoked once another session 
for the same user is created, meaning that 
**there can only be one active session per user**.

So basically there are two ways to get a session token - either by creating 
a session through RPC, or by snatching it from application files. 

## Creating session

To create a session, client makes 2 to 3 requests.

### `SessionCreate`

First of all, it does `SessionCreate` with some device 
and carrier info:

> `POST /co.znly.users.services.sessions.SessionsService/SessionCreate`
> 
> See `sessions_service.proto` and `sessions.proto` for more info

```js
{
  "phoneNumber": "+78005553535",
  "device": {
    "appVersion": "4.63.14",
    "type": "ANDROID", // value = 2
    "osVersion": "12",
    "model": "xiaomi m2007j20cg", // why yes i use poco x3
    "acceptLanguages": "en-US;q=1.0",
    "coreVersion": "1.96.7",
    "appBundle": "app.zenly.locator"
  },
  "deviceOsUuid": randomBytes(16).toString("base64"),
  "carrierInformations": {
    "networkOperatorCode": "25001",
    "networkOperatorName": "MTS",
    "networkCountryIso": "ru",
    "simOperatorCode": "25001",
    "simOperatorName": "MTS RUS",
    "simCountryIso": "ru"
  }
}
```

> I haven't tested whether it validates this info in any way, so yeah.

The server then responds with:

```js
{
  "session": {
    "uuid": "s-...some...session...id...",
    "status": 2,
    "userUuid": "u-...some...user...id...",
    "phoneNumber": "+78005553535",
    ...
  },
  "deprecatedChallengeType": 2,
  "deprecatedChallengeSize": 4,
  "verifyInfo": {...}
}
```

> Some fields were omitted both for brevity and because I haven't
> looked into them too deep (especially `verifyInfo` field)

### `SessionVerify`

The client then waits for SMS from Zenly (or requests a call, see below) and sends `SessionVerify`:

> `POST /co.znly.users.services.sessions.SessionsService/SessionVerify`
>
> See `sessions_service.proto` and `sessions.proto` for more info

```js
{
  "sessionUuid": result.session.uuid,
  "phoneNumber": "+78005553535",
  "userName": "Pelageya",
  "birthdate": {
    // afaik, pre-5.x used 31 december of <birth year> in local tz
    // 5.x and later ask for a more specific birth date
    // here we have 2004 birth year transformed to "31 Dec 2004 00:00:00 UTC+3" 
    "seconds": 1104440400
  },
  "code": "1234", // code from sms/call
  // these should be the same as before (?)
  "deviceOsUuid": ..., 
  "device": {...},
  "errorIfSuspended": true
}
```

To which the client responds with a large object that also includes
configuration and user profile and preferences:

```js
{
  "session": {
    "uuid": "s-...some...session...id...",
    "userUuid": "u-...some...user...id...",
    ...
  },
  "userPreferences": {...},
  "configuration": {...},
  "me": {
    "uuid": "u-...some...user...id...",
    "name": "Pelageya",
    "avatarUrlPrefix": "https://cdn.zenly-app.com/avatars/img/u--...some...user...id.../1625731610/a",
    "avatarVersion": 1625731610,
    "friendsCount": 10,
    "username": "pela_gay",
    ...
  }
}
```

### `SessionRequestCall`

Client may also choose to request a phone call to receive the code,
in which case it would send `SessionRequestCall`:


> `POST /co.znly.users.services.sessions.SessionsService/SessionRequestCall`
>
> See `sessions_service.proto` and `sessions.proto` for more info

```js
{
  "sessionUuid": result.session.uuid,
  "device": {...}, // same as before (?)
  "type": result.verifyInfo.possibleEventTypes[0]
}
```

Client then uses the session UUID as the token value,
passing it as-is in the `Authorization` header:

```
Authorization: Token s-...some...session...id...
```

## Extracting session

Since I don't have an iOS device, I will only cover Android app here.

Android app stores its session in `session.bin` file located in:

```
/data/data/app.zenly.locator/files/appgroups/group.zenly.locator/co.znly.core/session.bin
```

You can extract that file using some root-based file manager
(I prefer MT Manager). 
> If you don't have root then well my condolences

The file itself contains Protobuf-serialized session object,
from which you can extract the token using the following 
CyberChef recipe: 

[Extract from session.bin](https://gchq.github.io/CyberChef/#recipe=Protobuf_Decode('syntax%20%3D%20%22proto3%22;%5Cn%5Cnmessage%20Session%20%7B%5Cn%20%20string%20session%20%3D%201;%5Cn%7D',false,false))

# Avatars

User object contains `avatarUrlPrefix` field, which is used by the
app to construct a URL to the avatar as follows:

```js
const url = user.avatarUrlPrefix + '.' + size + '.jpg?' + user.avatarVersion
```

> Query parameter is used to avoid caching, but currently the prefix
> itself contains the version, so that's pretty redundant.

`size` can be one of: `64, 128, 256, 512`.

# Users

## `Me`

It is often useful to get an object containing info about the current user.
To do so, clients do `Me` request:

> `POST /co.znly.users.services.users.UserService/Me`
>
> See `users_service.proto` and `users.proto` for more info

Request body *may* contain device info, but it can be omitted.

Server then responds with the current user object:

```js
{
  "me": {
    "uuid": "u-...some...user...id...",
    "createdAt": {
      "seconds": 1626861087,
      "nanoseconds": 624000000
    },
    "name": "...",
    "avatarUrlPrefix": "...",
    "avatarVersion": 1626861231,
    "phoneNumber": "...",
    "roles": [
      "standard"
    ],
    "friendsCount": 5,
    "updatedAt": {
      "seconds": 1635278940,
      "nanoseconds": 341000000
    },
    "birthdate": {
      "seconds": 1059091200
    },
    "username": "...",
    "optOutSuggest": true,
    "events": {
      "inviterCount": 1
    },
    "limitUsernameFriending": true,
    "hidePhoneNumber": false
  }
}
```

## `Friends`

To get the list of friends, clients do `Friends` request:

> `POST /co.znly.users.services.friends.FriendsService/Friends`
>
> See `friends_service.proto` and `friends.proto` for more info

The request doesn't have a body, and the server responds with
an object containing `User` objects array:

```js
{
  "friends": [
    {
      "uuid": "u-...some...friend...user...id...",
      ...
    },
    {
      "uuid": "u-...some...other...friend...user...id...",
      ...
    },
  ]
}
```

# Getting location

I suppose, the most important thing for 3rd-party apps from
Zenly APIs is the ability to get friends' location.

To do that, clients subscribe to `PinContextSubscribeStream` stream
and provide the current viewport dimensions. But since we want
to get all locations at once, we can provide the entire world
as the dimensions. 

## `PinContextSubscribeStream`

> `POST /zenly.protobuf.services.ZenlyService/PinContextSubscribeStream`
>
> See `zenly_service.proto` and `pin.proto` for more info

```js
{
  "viewport": {
    "topLeft": {
      "latitude": -90,
      "longitude": -90,
      "altitude": 0
    },
    "bottomRight": {
      "latitude": 90,
      "longitude": 90,
      "altitude": 0
    }
  },
  "mode": 1, // idk
  "selectedUserUuids": []
}
```

> Client also provides UUID(s) of the selected user(s). It seems
> as if it's requesting the server to update the location and
> to subscribe to their location updates, but I'm not exactly sure

Server then immediately sends an event containing latest friends'
locations, then (if `selectedUserUuids` are provided) - their
updated current locations, and then updates to their locations.

They are all available as `PinContext` object which is pretty
large and I won't explain it in-depth here, but basic info
like lat-lon is available.

Server-sent events look like:

```js
{
  "pinContexts": [
    {
      "userUuid": "u-...some...friend...id...",
      "createdAt": {
        "seconds": 1650972121,
        "nanoseconds": 873788642
      },
      "latRaw": ...,
      "lngRaw": ...,
      "hpRaw": 7,
      "ghostType": 0,
      ...
    },
    ...
  ]
}
```

> **Note:** That array also contains your own location, 
> be sure to correctly handle it as well.

## `TrackingContextSubscribeStream`

There's also another way to get locations, called "tracking context".
Although this one seems to be legacy and deprecated - it currently
sends much less information about friends: only lat/lon, precision, speed
and battery level are present.

> `POST /zenly.protobuf.services.ZenlyService/TrackingContextSubscribeStream`
>
> See `zenly_service.proto` and `tracking.proto` for more info

```js
{
  "viewport": {
    "topLeft": {
      "latitude": -90,
      "longitude": -90,
      "altitude": 0
    },
    "bottomRight": {
      "latitude": 90,
      "longitude": 90,
      "altitude": 0
    }
  },
  "mode": 1, // idk
  "selected"?: "u-...some...user...id...",
  "group"?: {
    "friends": [
      "u-...some...user...id..."
    ]
  }
}
```

> In this request, there seem to be multiple fields for "selected" 
> friends. I'm not sure what's the diference. Anyways, providing 
> some of them seems to also request location update and subscribe
> to newer locations, just like in the previous request.

Unlike the previous one, in this stream one message = one location,
and at the start it sends all the available locations at once.

They are sent as `TrackingContext` objects which, as I said before,
doesn't provide as much info as `PinContext`, despite its structure being
really similar.

Server-sent events look like:

```js
{
  "trackingContext": {
    "createdAt": {
      "seconds": 1651061606,
      "nanoseconds": 390995467
    },
    "userUuid": "u-...some...user...id...",
    "latitude": ...,
    "longitude": ...,
    "batteryLevel": 90,
    "isForeground": false,
    "horizontalPrecision": 5,
    "verticalPrecision": 0,
    "isCharging": false,
    "isGhost": false,
    "ghostType": "REALTIME",
  }
}
```

> **Note:** This one doesn't seem to send your own location,
> but be sure to handle that just in case.

# Sending location

It might also be useful to mock your own location by sending fake info.

## `TrackingContextPublishStream`

Clients send their location using `TrackingContextPublishStream`, which is a
bi-directional stream where the client sends its own location, and the server
sends acks and info about watchers.

> `POST /zenly.protobuf.services.ZenlyService/TrackingContextPublishStream`
>
> See `zenly_service.proto` and `tracking.proto` for more info

Each client-sent event looks basically like this:

```js
{
  "seq": "1",
  "trackingContext": {
    "userUuid": me.uuid,
    "latitude": 55.7520,
    "longitude": 37.6175,
    "batteryLevel": 69,
    "isForeground": false,
    "horizontalPrecision": 5,
    "isCharging": false,
    "isGhost": false,
    "speed": 0,
    "bearing": 0,
    "ghostType": "REALTIME",
  }
}
```

Here we set our own location to Moscow Kremlin. There are also other 
fields described in the proto object, which aren't sent by the 
`TrackingContextSubscribeStream`, but seem to be used in the 
`PinContextSubscribeStream` results. 

I'm not going to cover them because a) there are a lot of them and I'm lazy,
b) I don't exactly understand half of them

`seq` is a sequential number used to map sent tracking context with the 
server-sent acks.

For each sent event, server responds with this object, serving as an acknowledgement:

```js
{
  "seq": "1",
  "watchersUids": []
}
```

It also sends events when `watchersUids` field is changed, in which case `seq` is 
set to the last received tracking context. 

# License

This repo (including this readme) is licensed under MIT license.