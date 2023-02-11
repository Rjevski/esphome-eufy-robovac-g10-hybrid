# ESPHome on Eufy Robovac G10 Hybrid

This repo provides an ESPHome configuration to replace the Tuya module in an Eufy Robovac G10 Hybrid. It is likely to work with their other vacuums and provides a good starting point for future development.

## Installation

Tweak the config in `esphome.yaml` - make sure to set secrets, etc and of course modify it for your target hardware platform (this one was done on ESP8266, but should run on anything compatile with ESPHome).

Compile the resulting config and program your microcontroller. Once done and the micro is visible over the network (very important to check first, so you don't have to open your vacuum again in case you need to tweak the firmware), disassemble your vacuum (~10 Philips screws, very easy), you'll see a little Tuya module with 4 twisted wires going to it from the main board - remove that and put your module in its place - use a bit of adhesive to secure it if necessary. Wire colors are as follows:

* Red: 3.3V
* Green: ground

Not sure about those ones, if it doesn't work flip them around (you may be able to do that in your ESPHome config with no soldering required)

* Blue: RX (so connect to TX pin of your micro)
* Black: TX (so connect to RX pin of your micro)

Once done, close it back up and connect it to Home Assistant.

## Home Assistant configuration

ESPHome [doesn't (yet?) support](https://github.com/esphome/feature-requests/issues/2096) the `vacuum` platform which means it's unable to create a `vacuum` entity in HASS like it can for lights/etc - therefore this config only exposes ad-hoc sensors, selects and switches (marked as `entity_category: config`) so they don't pollute your dashboard and you need to manually create a vacuum entity in Hass using the provided `homeassistant.yaml` configuration - this uses the [template vacuum](https://www.home-assistant.io/integrations/vacuum.template/) integration. Adjust it to match your vacuum's entity IDs when your ESPHome device was added to Home Assistant.


## Future reverse engineering


### Decompiling the Android app

Android APK can be downloaded from [APKMirror](https://www.apkmirror.com/apk/anker/eufyhome/) and is a good reference when it comes to understanding the meaning of each Tuya DP.

Once downloaded, you have 2 options:

#### Apktool

Using [Apktool](https://ibotpeaches.github.io/Apktool/) you can decompile the APK into a set of `smali` files:

    apktool d com.eufylife.smarthome_2.14.0-442_minAPI21\(arm64-v8a,armeabi-v7a\)\(nodpi\)_apkmirror.com.apk -o apktool-decompiled

#### Dex2Jar & JD-GUI

Using [dex2jar](https://github.com/pxb1988/dex2jar) you can convert the APK to a JAR which can then be opened with [JD-GUI](https://github.com/java-decompiler/jd-gui).

Note that the original dex2jar fails with a `java.lang.StringIndexOutOfBoundsException` error on this APK, but a [fork](https://github.com/ThexXTURBOXx/dex2jar) fixes the issue.

    d2j-dex2jar APKs/com.eufylife.smarthome_2.14.0-442_minAPI21\(arm64-v8a,armeabi-v7a\)\(nodpi\)_apkmirror.com.apk -o d2j-decompiled

Open the resulting `d2j-decompiled.jar` in JD-GUI and browse around. Among all kinds of Facebook and similar spyware, you'll find some references to Tuya as well as DPs - good things to search for are `122`, `111`, `103` and go from there.


### Proxying the real Tuya module

This config maps most Tuya data point numbers and values but there's a lot more - in some cases it might be useful to reconnect the official Tuya module. We can do so with software without needing to open the vacuum again.

Connect the original Tuya module to your computer via a serial to UART converter - you can use an Arduino, ESP/etc microcontroller with serial converter built-in, just hold the microcontroller in reset so it doesn't interfere with the communication. This would also provide you with a source of 3.3V which is required by the Tuya module. Connect the appropriate pins to the Tuya module.

When connected to the Tuya module you can test by opening `minicom` (115200/8N1) and if you see some garbage (`???UU`) this means you are receiving the data from the Tuya module. The data is in raw bytes so it's normal for it to look like garbage.

Now you can use an ESPHome serial to TCP component, of which there are multiple:

* https://github.com/oxan/esphome-stream-server
* https://github.com/tube0013/esphome-stream-server-v2

(and probably a lot more if you search around for it - I'm surprised there isn't a proper [RFC 2217](https://www.rfc-editor.org/rfc/rfc2217)-compliant equivalent in ESPHome itself)

Comment out the `tuya` platform and all the entities in the ESPHome config, and add the following instead:


```
external_components:
  - source: github://<username>/<repo>  # whatever repo you chose above

stream_server:
   uart_id: vacuum_uart
```

Recompile and reflash. This will now expose the serial port over raw TCP on port 6638.

At this point I recommend power-cycling the vacuum (using the switch under it) and your Tuya module, so both start from a known state.

You can now use `socat` to proxy between the two:

    socat -x TCP:<ESPHome IP address or hostname>:6638,nodelay,retry,interval=5 <serial port device node, like /dev/ttyUSB0>,echo=0,ispeed=115200,ospeed=115200,raw

Note: on macOS, serial ports are exposed as 2 devices, one like `/dev/tty.*` and one `/dev/cu.` for a [historical reason](https://stackoverflow.com/questions/8632586/whats-the-difference-between-dev-tty-and-dev-cu-on-macos). **Make sure to use the CU one and not the TTY one** - the latter will block forever and `socat` will just hang trying to open the port (in fact even Ctrl+C doesn't kill it, had to physically disconnect the serial to USB).

Now both sides can communicate and you can observe the traffic. Serial protocol is described in the [Tuya documentation](https://developer.tuya.com/en/docs/iot/tuya-cloud-universal-serial-port-access-protocol?id=K9hhi0xxtn9cb).

You can now issue commands using the official app and observe the communication with the vacuum's application processor.


### Hardware

The vacuum itself is powered by an [Amicro](http://en.amicro.com.cn/?platform/open/) ARM CPU - there's pretty much no details out there about it.

There are 2 UARTs on the board - 1 is unpopulated, the second one is connected to the Tuya module (more about it below).

Connecting to the unpoulated UART does provide some sort of boot log but no shell of any kind nor obvious hint as to what OS it's running on - I suspect we're dealing more with a microcontroller than an actual computer with OS, so it's likely running some raw image on bare-metal.

Searching some strings from the UART output led to some [leaks](https://github.com/abc158/shsh01) (if it gets taken down just email me and I'll send you a copy) of what looks like 2017-level era source code - couldn't find anything regarding a shell or any way to interact with it over the unpopulated UART, and it doesn't contain the actual toolchain that would be necessary to create & flash a firmware. Maybe capturing a firmware update sequence over the proxied Tuya module would provide enough info to run our own code on the main CPU?


### Using the existing Tuya module

If you want to interact with the vacuum through the existing Tuya module, you'll need the "local key". In the past I've [reverse-engineered](https://github.com/Rjevski/eufy-clean-local-key-grabber) the Android app to emulate it (including some shitty request signature behavior designed to thwart this, because how dare users want to be able to control the hardware they paid money for) so you can give it Eufy credentials and it'll spit out the keys for every device on that account.

Note that if you look through that code you'll notice there is (was?) a security vulnerability with how their client talks to Tuya - the credentials you provide only authenticate you to Eufy from where a (sequential) account ID is retrieved - the actual auth to Tuya uses that sequential account ID along with an deterministic password.

Once you have the local key you can use one of the many Tuya-related HASS integrations:

* https://github.com/mitchellrj/eufy_robovac (looks unmaintained)
* https://github.com/bmccluskey/robovac (fork of the above which integrates my local key grabber)
* https://github.com/Rjevski/ha-eufy-robovac-g10-hybrid (my attempt at a HASS integration, includes the key grabber, unmaintained and for reference only)
* https://github.com/rospogrigio/localtuya (looks the most popular and supports a wide range of devices, so I'd suggest trying this one and manually configuring the DPs)

Alternatively if you want to go lower level there are many Tuya local protocol implementations (I think there's a [Node.JS one](https://github.com/codetheweb/tuyapi) which is quite active) you can use.