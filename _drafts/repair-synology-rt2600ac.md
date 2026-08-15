---
layout: post
title: Trying to repair a blocked Synology RT2600ac
---

This article is about trying to fix a blocked Synology RT2600ac.

# Situation
The LED for 2.4G does not light up after the boot process.
* STATUS is solid orange
* 2.4G is off
* 5G is solid green
When a computer is wired to an Ethernet port, the corresponding LED (WAN1, WAN2[^wan2], 1, 2, 3, 4) is not blinking either.

According to Synology Knowledge Center[^synology-led-indicators], the solid orange status indicates the router is booting up, but this state does not change even after hours of patience.

There is no clear information if this is due to a hardware or software problem with the Wi-Fi interface for 2.4G, or something else.

Even though the LEDs for wired are not blinking, the computer is assigned an IP address on the network 192.168.1.0/24.
The first time install loads, but blocks when setting the password, despite meeting complexity requirements.

Also, no wireless network is broadcasted on the 5G band.

The router offers 2x USB-A ports and 1x SD card slot. However, no solution is documented to perform an update update via them without requiring the web interface.

[^wan2]: WAN2 can be configured on the RJ45 port 1.
[^synology-led-indicators]: [What do the LED indicators on my Synology Router mean?](https://kb.synology.com/en-global/SRM/tutorial/What_do_LED_indicators_mean_MR2200ac)

# Action plan
As the status indicates the boot sequence is not complete, but the router already seems to have features started, like wired networking and first time installation.
Trying to force completion of the first time installation is a good first path to explore, with a goal to update the router with a newer software version.

Ultimately, opening the router to find serial and debug ports to tap.

## Force first time install
A web browser is opened to the URL `http://192.168.1.1:8000/webman/index.cgi`.
Opening the development tool at the console shows two exceptions.

### 1st error
When landing on the first page of the first time installation.

> Uncaught TypeError: can't access property "filter", s is undefined
>     loadInitialData FirstTimeInstall.js:1291

Code of `loadInitialData` where error was triggered (`FirstTimeInstall.js:3:21707`):
```js
    loadInitialData: function (g, a) {
      if (!g) {
        SYNO.Debug('Unknown error!', a);
        return
      } else {
        if (a.has_fail) {
          var k = a.result.filter(
            function (x) {
              if (
                'SYNO.Core.Network.MACClone' === x.api ||
                'SYNO.Core.Network.Router.ConnectionList' === x.api
              ) {
                return false
              }
              return (false === x.success)
            }
          );
          if (0 < k.length) {
            SYNO.Debug('Initial data failed!', a)
          }
        }
      }
      var s = SYNO.API.Util.GetValByAPI(a, 'SYNO.Wifi.Network.Setting', 'get', 'profiles');
      var e = s.filter(function (x) { // Line 1291
        return (0 === x.id)
      }) [0];
      var p = SYNO.API.Util.GetValByAPI(a, 'SYNO.Wifi.CountryCode.Capability', 'get');
      var n = p.country_codes;
      var o = p.immutable;
      var b = SYNO.API.Util.GetValByAPI(a, 'SYNO.Wifi.CountryCode.Setting', 'get', 'country_code');
      var u = SYNO.API.Util.GetValByAPI(a, 'SYNO.Core.Region.NTP', 'listzone', 'zonedata');
      var h = SYNO.API.Util.GetValByAPI(a, 'SYNO.Core.Group.Member', 'list', 'users');
      var d = SYNO.API.Util.GetValByAPI(a, 'SYNO.Core.User', 'list', 'users');
      var w = SYNO.API.Util.GetValByAPI(a, 'SYNO.Core.Network.MACClone', 'getRemoteMACAddress');
      var m = SYNO.API.Util.GetValByAPI(a, 'SYNO.Core.Network.Router.ConnectionList', 'get', 'stations') ||
      [];
      var q = false;
      if (Ext.isDefined(w.code) && 4307 === w.code) {
        q = true
      }
      var i = false;
      if (!Ext.isEmpty(m) && Ext.isDefined(w.mac)) {
        var l = m.find(function (x) {
          return (w.mac === x.mac && 'wifi' === x.connection)
        });
        if (Ext.isDefined(l)) {
          i = true
        }
      }
      var t = _S('lang');
      var r = SYNO.SDS.Utils.getSupportedLanguage(false);
      r = r.map(function (x) {
        return x[0]
      });
      var j = (r.includes(t)) ? t : 'enu';
      var v = SYNO.SDS.Utils.getSupportedLanguageCodepage(false);
      v = v.map(function (x) {
        return x[0]
      });
      var c = (v.includes(t)) ? t : 'enu';
      var f = {};
      d.forEach(function (x) {
        f[x.name] = x.description
      });
      this.defaultParams = {
        countryCodeList: n,
        immutable: o,
        currentCountryCode: b,
        networkProfile: e,
        region: this.getTimeZone(u),
        mailLang: j,
        codePageLang: c,
        isWifi: i,
        isWan: q,
        adminList: h.map(function (x) {
          return x.name
        }),
        userList: d.map(function (x) {
          return x.name
        }),
        userDescMap: f
      }
    },
```

A call to the API looks to have returned a null value, which is confirmed by an unsuccessful request visible in the Network tab.

Request:
```sh
curl 'http://192.168.1.1:8000/webapi/_______________________________________________________entry.cgi' \
  -X POST \
  -H 'Content-Type: application/x-www-form-urlencoded; charset=UTF-8' \
  --data-raw 'stop_when_error=false&compound=%5B%7B%22api%22%3A%22SYNO.Wifi.CountryCode.Capability%22%2C%22method%22%3A%22get%22%2C%22version%22%3A1%7D%2C%7B%22api%22%3A%22SYNO.Wifi.CountryCode.Setting%22%2C%22method%22%3A%22get%22%2C%22version%22%3A1%7D%2C%7B%22api%22%3A%22SYNO.Wifi.Network.Setting%22%2C%22method%22%3A%22get%22%2C%22version%22%3A1%7D%2C%7B%22api%22%3A%22SYNO.Core.Region.NTP%22%2C%22method%22%3A%22listzone%22%2C%22version%22%3A1%7D%2C%7B%22api%22%3A%22SYNO.Core.Group.Member%22%2C%22method%22%3A%22list%22%2C%22version%22%3A1%2C%22group%22%3A%22administrators%22%7D%2C%7B%22api%22%3A%22SYNO.Core.User%22%2C%22method%22%3A%22list%22%2C%22version%22%3A1%2C%22additional%22%3A%5B%22description%22%5D%7D%2C%7B%22api%22%3A%22SYNO.Core.Network.MACClone%22%2C%22method%22%3A%22getRemoteMACAddress%22%2C%22version%22%3A1%7D%2C%7B%22api%22%3A%22SYNO.Core.Network.Router.ConnectionList%22%2C%22method%22%3A%22get%22%2C%22version%22%3A1%2C%22scannertype%22%3A%22all%22%7D%5D&api=SYNO.Entry.Request&method=request&version=1'
```

Response:
```json
{
    "data": {
        "has_fail": true,
        "result": [
            {
                "api": "SYNO.Wifi.CountryCode.Capability",
                "error": {
                    "code": 117
                },
                "method": "get",
                "success": false,
                "version": 1
            },
            {
                "api": "SYNO.Wifi.CountryCode.Setting",
                "error": {
                    "code": 117
                },
                "method": "get",
                "success": false,
                "version": 1
            },
            {
                "api": "SYNO.Wifi.Network.Setting",
                "error": {
                    "code": 117
                },
                "method": "get",
                "success": false,
                "version": 1
            },
            {
                "api": "SYNO.Core.Region.NTP",
                "data": {
                    "zonedata": [
                        {
                            "display": "...",
                            "value": "..."
                        },
                        {
                            "display": "(GMT+08:00) Taipei",
                            "value": "Taipei"
                        },
                        {
                            "display": "...",
                            "value": "..."
                        }
                    ]
                },
                "method": "listzone",
                "success": true,
                "version": 1
            },
            {
                "api": "SYNO.Core.Group.Member",
                "data": {
                    "offset": 0,
                    "total": 1,
                    "users": [
                        {
                            "description": "System default user",
                            "name": "admin",
                            "uid": 1024
                        }
                    ]
                },
                "method": "list",
                "success": true,
                "version": 1
            },
            {
                "api": "SYNO.Core.User",
                "data": {
                    "offset": 0,
                    "total": 2,
                    "users": [
                        {
                            "description": "System default user",
                            "name": "admin",
                            "uid": 1024
                        },
                        {
                            "description": "Guest",
                            "name": "guest",
                            "uid": 1025
                        }
                    ]
                },
                "method": "list",
                "success": true,
                "version": 1
            },
            {
                "api": "SYNO.Core.Network.MACClone",
                "data": {
                    "mac": "xx:re:da:ct:ed:xx"
                },
                "method": "getRemoteMACAddress",
                "success": true,
                "version": 1
            },
            {
                "api": "SYNO.Core.Network.Router.ConnectionList",
                "data": {
                    "stations": [
                        {
                            "asso_time": "16:04:05 2022/12/31",
                            "brand": "",
                            "connection": "ethernet",
                            "hostname": "COMPUTER",
                            "ip": "192.168.1.179",
                            "ip6": "",
                            "mac": "xx:re:da:ct:ed:xx",
                            "model": "",
                            "netif": "lbr0",
                            "status": true,
                            "type": "default"
                        }
                    ]
                },
                "method": "get",
                "success": true,
                "version": 1
            }
        ]
    },
    "success": true
}
```

Response `"has_fail": true` with error code 117 for `SYNO.Wifi.CountryCode.Capability`, `SYNO.Wifi.CountryCode.Setting` and `SYNO.Wifi.Network.Setting`.

According to DSM Login Web API Guide[^synology-web-api-guide-2] the error code 117 means `The network connection is unstable or the system is busy`.

This explains this first exception.

[^synology-web-api-guide-2]: https://kb.synology.com/fr-fr/DG/DSM_Login_Web_API_Guide/2

### 2nd error
When reaching the account creation page, after completion of the password fields.

> Uncaught TypeError: can't access property "userDescMap", this.owner.owner.defaultParams is null
>     validatePassword FirstTimeInstall.js:344

Code of `loadInitialData` where error was triggered (`FirstTimeInstall.js:344:17`):
```js
    validatePassword: function (h) {
      if ('' === h) {
        return true
      }
      this.strength.checkStrength(h);
      var k = [];
      if (this.passwdRules.exclude_username) {
        var c = this.getParams();
        var f = c.username;
        var a = this.owner.owner.defaultParams.userDescMap; // Line 344
        var j = (f in a) ? a[f] : '';
        if ((f && h.include(f)) || (j && h.include(j))) {
          k.push(_T('passwd', 'exclude_username'))
        }
      }
      if (this.passwdRules.included_numeric_char) {
        var d = /\d/;
        if (!h.match(d)) {
          k.push(_T('passwd', 'included_numeric_char'))
        }
      }
      if (this.passwdRules.included_special_char) {
        var g = /\W/;
        if (!h.match(g)) {
          k.push(_T('passwd', 'included_special_char'))
        }
      }
      if (this.passwdRules.min_length_enable) {
        var b = parseInt(this.passwdRules.min_length, 10);
        if (h.length < b) {
          k.push(String.format('{0}: {1}', _T('passwd', 'min_length_enable'), b))
        }
      }
      if (this.passwdRules.mixed_case) {
        var e = /[a-z]/;
        var i = /[A-Z]/;
        if (!(h.match(e) && h.match(i))) {
          k.push(_T('passwd', 'mixed_case'))
        }
      }
      if (k.length > 0) {
        return String.format('{0}{1}', _T('passwd', 'passwd_hint_rule'), k.join(', '))
      }
      return true
    },
```

The property `userDescMap` is undefined, in fact it is defined in `loadInitialData` where the first exception occurred.

So, overriding the API response may allow to continue the first time installation.

### Bypass first time installation
During the process of reconstructing the missing data, a bypass was found.

Using the feature to override a response from `http://192.168.1.1:8000/webapi/_______________________________________________________entry.cgi`.
Responding the data with failures and refreshing the page leads straight to SRM.
Before clicking anywhere, the override feature must be disabled.

Notifications show service `synowifidaemon` failed to start.

Access to SRM is as `admin`, so its password is reset.

SSH service is enabled, but connection are rejected.

Upgrading SRM is failing.

Wi-Fi Connect does not allow to manage parameters.

## Open the router
6 screws must be removed, of which 2 are under the dome-shaped rubber feet and one under the label (this may void the warranty). Once done, 10 plastic clips (4 on a long edge, 1 on a short one) must be released to open the case.

The PCB is protected by two large heat sinks, fortunately a 6-pin header is accessible!

### UART connection

Pinout of the 6-pin header J4:
| Pin | Description |
| --- | --- |
|  2  | GND |
|  4  | RX |
|  6  | TX |
|  1  | VCC 3.3V |
|  3  | Maybe RST |
|  5  | Maybe 12V power input |
Pin numbers 1, 2, 5 and 6 visible on the silkscreen.

Connecting a serial interface using 3.3V on GND, RX and TX, allows to see the boot process and some messages from the system.
Boot can be interrupted `Ctlr+C to interrupt`.

User is not logged automatically, but admin account can be used with the previously defined password.
```
SynologyRouter login: admin
Password:

admin@SynologyRouter:~$
```

Upgrading the router was possible from the console, using command `sudo synoupgrade --auto`.

```sh
Copyright (c) 2003-2014 Synology Inc. All rights reserved.

        --auto
        --check
        --download
        --start
        --patch ABSOULATE_UPGRADE_FILE_PATH
        --cksum
```

Two upgrades were done to reach the latest release, all are successful. However, the problem is still present.

These messages does not sound good:
```
[    6.135676] 1b700000.pci supply vdda not found, using dummy regulator
[    6.135791] 1b700000.pci supply vdda_phy not found, using dummy regulator
[    6.135885] 1b700000.pci supply vdda_refclk not found, using dummy regulator
[    6.136810] PCI host bridge /soc/pci@1b700000 ranges:
[    6.136853]    IO 0x31e00000..0x31efffff -> 0x31e00000
[    6.136875]   MEM 0x2e000000..0x31dfffff -> 0x2e000000
[    7.259191] qcom-pcie 1b700000.pci: phy link never came up
[    7.261207] qcom-pcie 1b700000.pci: hostinit failed
[    7.261225] qcom-pcie 1b700000.pci: cannot initialize host
[    7.261748] qcom-pcie: probe of 1b700000.pci failed with error -110
```

# Extras
## Reconstruct missing data

The function `SYNO.API.Util.GetValByAPI` is located in `sds.js`:
```js
SYNO.API.Util.GetValByAPI = function (response, api, method, property) {
  if (Ext.isObject(response)) {
    if (Ext.isArray(response.result)) {
      var result = response.result;
      for (var index = 0; index < result.length; index++) {
        if (api === result[index].api && method === result[index].method) {
          var properties = result[index].data || result[index].error;
          if (!properties) {
            return
          }
          if (property) {
            return properties[property]
          }
          return properties
        }
      }
      return
    } else {
      if (property) {
        return response[property]
      } else {
        return response
      }
    }
  }
  return
};
```

```js
// FirstTimeInstall.js:1290
var s = SYNO.API.Util.GetValByAPI(a, 'SYNO.Wifi.Network.Setting', 'get', 'profiles');
var e = s.filter(function (x) { return (0 === x.id) }) [0];
var p = SYNO.API.Util.GetValByAPI(a, 'SYNO.Wifi.CountryCode.Capability', 'get');
var n = p.country_codes;
var o = p.immutable;
var b = SYNO.API.Util.GetValByAPI(a, 'SYNO.Wifi.CountryCode.Setting', 'get', 'country_code');
// FirstTimeInstall.js:1333
this.defaultParams = {
  countryCodeList: n,
  immutable: o,
  currentCountryCode: b,
  networkProfile: e,
// FirstTimeInstall.js:1144
loadData: function (a) {
  this.countryCodeStore.loadData(a.countryCodeList);
  if (true === a.immutable) {
    this.getForm().findField('location').setValue(a.currentCountryCode);
    Ext.getCmp(this.countryCodeFieldId).hide();
    Ext.getCmp(this.noteFieldId).hide()
  }
},
// FirstTimeInstall.js:1088
this.countryCodeStore = new Ext.data.ArrayStore({
    fields: ['value', 'display']
}),
// FirstTimeInstall.js:1686
    getWifiProfileSetting: function (ssid, password) {
      var b = this.defaultParams.networkProfile;
      b.radio_list.forEach(
        function (d) {
          d.security.password = a;
          switch (d.radio_type) {
            case 'SmartConnect':
              d.ssid = c;
              break;
            case '2.4G':
              d.ssid = c + '_2.4G';
              break;
            case '5G':
              d.ssid = c + '_5G';
              break;
            case '5G-1':
              d.ssid = c + '_5G-1';
              break;
            case '5G-2':
              d.ssid = c + '_5G-2';
              break
          }
        }
      );
      return [b]
    },

```

```json
{
    "data": {
        "has_fail": true,
        "result": [
            {
                "api": "SYNO.Wifi.CountryCode.Capability",
                "data": {
                    "country_codes": [
                        { "value": "TW", "display": "Taiwwan" }
                    ],
                    "immutable": true
                },
                "method": "get",
                "success": true,
                "version": 1
            },
            {
                "api": "SYNO.Wifi.CountryCode.Setting",
                "data": {
                    "country_code": "TW"
                },
                "method": "get",
                "success": true,
                "version": 1
            },
            {
                "api": "SYNO.Wifi.Network.Setting",
                "data": {
                    "profiles": [
                        {
                            "id": 0,
                            "radio_list": [
                                {
                                    "radio_type": "SmartConnect",
                                    "security": {
                                        // "password": "synology"
                                    },
                                    // "ssid": "Synology_SERIAL"
                                }
                            ]
                        }
                    ]
                },
                "method": "get",
                "success": true,
                "version": 1
            },
            {
                "api": "SYNO.Core.Region.NTP",
                "data": {
                    "zonedata": [
                        {
                            "display": "...",
                            "value": "..."
                        },
                        {
                            "display": "(GMT+08:00) Taipei",
                            "value": "Taipei"
                        },
                        {
                            "display": "...",
                            "value": "..."
                        }
                    ]
                },
                "method": "listzone",
                "success": true,
                "version": 1
            },
            {
                "api": "SYNO.Core.Group.Member",
                "data": {
                    "offset": 0,
                    "total": 1,
                    "users": [
                        {
                            "description": "System default user",
                            "name": "admin",
                            "uid": 1024
                        }
                    ]
                },
                "method": "list",
                "success": true,
                "version": 1
            },
            {
                "api": "SYNO.Core.User",
                "data": {
                    "offset": 0,
                    "total": 2,
                    "users": [
                        {
                            "description": "System default user",
                            "name": "admin",
                            "uid": 1024
                        },
                        {
                            "description": "Guest",
                            "name": "guest",
                            "uid": 1025
                        }
                    ]
                },
                "method": "list",
                "success": true,
                "version": 1
            },
            {
                "api": "SYNO.Core.Network.MACClone",
                "data": {
                    "mac": "xx:re:da:ct:ed:xx"
                },
                "method": "getRemoteMACAddress",
                "success": true,
                "version": 1
            },
            {
                "api": "SYNO.Core.Network.Router.ConnectionList",
                "data": {
                    "stations": [
                        {
                            "asso_time": "16:04:05 2022/12/31",
                            "brand": "",
                            "connection": "ethernet",
                            "hostname": "COMPUTER",
                            "ip": "192.168.1.179",
                            "ip6": "",
                            "mac": "xx:re:da:ct:ed:xx",
                            "model": "",
                            "netif": "lbr0",
                            "status": true,
                            "type": "default"
                        }
                    ]
                },
                "method": "get",
                "success": true,
                "version": 1
            }
        ]
    },
    "success": true
}
```
