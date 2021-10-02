# Ring Keypad v2 With Home Assistant

I've been looking for a physical alarm keypad to use with Home Assistant for quite a while, and have't found any good options.  I was excited when I found out that the [Ring Alarm Keypad](https://ring.com/products/alarm-keypad-v2) is Z-Wave based and widely available, but I was never able to get it working with HA.  Until very recently.

It turns out that for full functionality the keypad must be paired with S2 security, the higher-security Z-Wave mode which wasn't supported by any open-source projects.  Until [August 2021](https://github.com/zwave-js/node-zwave-js/releases/tag/v8.1.0), when ZWaveJS added support for S2.  [ZWaveJS2MQTT](https://github.com/zwave-js/zwavejs2mqtt) added support for S2 soon after, which was the key to using the Ring Keypad with Home Assistant successfully.

In this document, I'll talk about how to use the Ring keypad with Home Assistant, taking full advantage of all the functionality it offers.  In my opinion, this is by far the most polished option for an alarm keypad that works with HA available currently

## Pairing

As of this writing, ZWaveJS2MQTT is the only method I can find to successfully pair the keypad with Home Assistant with full functionality.  I assume the standard ZWaveJS implementation will gain support for S2 security eventually, but currently (October 2021) the UI doesn't support it.

So, to make this work, be sure you're using ZWaveJS2MQTT.  The easiest way to do that is via the [community add-on](https://github.com/hassio-addons/addon-zwavejs2mqtt), which is part of the default add-on store.  Migrating from the standard ZWaveJS add-on to ZWaveJS2MQTT is out of the scope of this document, but should be straightforward.

If you haven't set up any devices with S2 security before, you likely don't have S2 keys defined.  In the ZWaveJS2MQTT settings page ZWave area, click the two arrows button next to the three S2 keys to generate new ones.  (Of course, don't do this if you already have keys defined, or you'll lose the ability to control whatever devices you already have paired!)

There's one issue I've had with this keypad: the security negotiation seems to time out quickly.  I found I need to have the DSK PIN (from the QR code on the back of the keypad) entered within a couple of seconds, or security negotiation fails.  I'd strongly suggest you type the PIN into a text document and copy it to your clipboard, ready to paste in as soon as ZWaveJS2MQTT asks for it.

Now you're ready to pair.  Plug the keypad into a power outlet (it will not pair on batteries).  Start inclusion by clicking "Manage Nodes" on the main ZWaveJS2MQTT page, then selecting Default mode.  Once inclusion mode starts, hold down the 1 key on the keypad until the green indicator starts flashing.

Eventually, a security choice pop-up will appear.  Click next quickly, and the DSK prompt will appear.  Paste in the PIN you copied earlier and click next quickly.  In a few seconds, you should see confirmation that it was paired with S2 Access Control.  If not, you'll need to try again.

If it failed, go to Manage Nodes -> Exclusion, and while you're in exclusion mode insert the reset pin that was included in the box in the hole on the back of the keypad.  Try the inclusion process again.

## Entry Control (receiving button events)

## Indicators (lights and sounds)

