# Ring Keypad v2 With Home Assistant

I've been looking for a physical alarm keypad to use with Home Assistant for quite a while, and have't found any good options.  I was excited when I found out that the [Ring Alarm Keypad](https://ring.com/products/alarm-keypad-v2) is Z-Wave based and widely available, but I was never able to get it working with HA.  Until very recently.

It turns out that for full functionality the keypad must be paired with S2 security, the higher-security Z-Wave mode which wasn't supported by any open-source projects.  Until [August 2021](https://github.com/zwave-js/node-zwave-js/releases/tag/v8.1.0), when ZWaveJS added support for S2.  [ZWaveJS2MQTT](https://github.com/zwave-js/zwavejs2mqtt) added support for S2 soon after, which was the key to using the Ring Keypad with Home Assistant successfully.

In this document, I'll talk about how to use the Ring keypad with Home Assistant, taking full advantage of all the functionality it offers.  In my opinion, this is by far the most polished option for an alarm keypad that works with HA available currently

## Pairing

## Entry Control (receiving button events)

## Notifications (lights and sounds)

