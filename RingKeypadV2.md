# Ring Keypad v2 With Home Assistant

I've been looking for a physical alarm keypad to use with Home Assistant for quite a while, and have't found any good options.  I was excited when I found out that the [Ring Alarm Keypad](https://ring.com/products/alarm-keypad-v2) is [Z-Wave based](https://support.ring.com/hc/en-us/articles/360042696611-Ring-Alarm-Keypad-2nd-gen-Z-Wave-Technical-Manual) and widely available, but I was never able to get it working with HA.  Until very recently.

It turns out that for full functionality the keypad must be paired with S2 security, the higher-security Z-Wave mode which wasn't supported by any open-source projects.  Until [August 2021](https://github.com/zwave-js/node-zwave-js/releases/tag/v8.1.0), when ZWaveJS added support for S2.  [ZWaveJS2MQTT](https://github.com/zwave-js/zwavejs2mqtt) added support for S2 soon after, which was the key to using the Ring Keypad with Home Assistant successfully.

In this document, I'll talk about how to use the Ring keypad with Home Assistant, taking full advantage of all the functionality it offers.  In my opinion, this is by far the most polished option for an alarm keypad that works with HA available currently

## Pairing

As of this writing, ZWaveJS2MQTT is the only method I can find to successfully pair the keypad with Home Assistant with full functionality.  I assume the standard ZWaveJS implementation will gain support for S2 security eventually, but currently (October 2021) the UI doesn't support it.

So, to make this work, be sure you're using ZWaveJS2MQTT.  The easiest way to do that is via the [community add-on](https://github.com/hassio-addons/addon-zwavejs2mqtt), which is part of the default add-on store.  Migrating from the standard ZWaveJS add-on to ZWaveJS2MQTT is out of the scope of this document, but should be straightforward.

If you haven't set up any devices with S2 security before, you likely don't have S2 keys defined.  In the ZWaveJS2MQTT settings page ZWave area, click the two arrows button next to the three S2 keys to generate new ones.  (Of course, don't do this if you already have keys defined, or you'll lose the ability to control whatever devices you already have paired!)

There's one issue I've had with this keypad: the security negotiation seems to time out quickly.  I found I need to have the DSK PIN (from the QR code on the back of the keypad) entered within a couple of seconds, or security negotiation fails.  I'd strongly suggest you type the PIN into a text document and copy it to your clipboard, ready to paste in as soon as ZWaveJS2MQTT asks for it.

Now you're ready to pair.  Plug the keypad into a power outlet (it will not pair on batteries).  Start inclusion by clicking "Manage Nodes" on the main ZWaveJS2MQTT page, then selecting Default mode.  Once inclusion mode starts, hold down the 1 key on the keypad until the green indicator starts flashing.

Eventually, a security choice pop-up will appear.  Click next quickly, and the DSK prompt will appear.  Paste in the PIN you copied earlier and click next quickly.  In a few seconds, you should see confirmation that it was paired with S2 Access Control.  If not, you'll need to try again.

If it failed, go to Manage Nodes -> Exclusion, and while you're in exclusion mode insert the reset pin that was included in the box in the hole on the back of the keypad.  Try the inclusion process again.  It took me a few tries to get it paired, but I eventually succeeded.

## Entry Control (receiving button events)

Now that you've got the keypad paired, you'll want to start using it in Home Assistant.  The keypad sends key events using the Entry Control Command Class.  To see what they look like, go to the Developer Tools section of Home Assistant and listen for `zwave_js_notification` events on the Events tab.  When you interact with the keypad, you'll see notifications from the node which look approximately like:

    {
        "event_type": "zwave_js_notification",
        "data": {
            "domain": "zwave_js",
            "node_id": REDACTED,
            "home_id": REDACTED,
            "device_id": "d3b1a89bf6937f9ed9c385839e792025",
            "command_class": 111,
            "command_class_name": "Entry Control",
            "event_type": 2,
            "data_type": 2,
            "event_data": "1234"
        },
        "origin": "LOCAL",
        "time_fired": "2021-10-02T12:56:13.417333+00:00",
        "context": {
            "id": "8b2cdc10d0cf1e90be769ce8e9d89e59",
            "parent_id": null,
            "user_id": null
        }
    }
    
This event was generated from entering the code "1234", and pressing the Enter button.  The `event_data` field tells you what code was entered, and the `event_type` tells you which button was pressed.  The following table explains the meanings of the different `event_type` values the keypad can produce:

| `event_type` | Meaning |
| ------------ | ------- |
| 0 | User started entering a code |
| 1 | User entered a code, but didn't press enter (or another button) before the timeout. |
| 2 | Enter |
| 3 | Disarm |
| 5 | Arm Away |
| 6 | Arm Stay |
| 16 | Fire (not sent unless button is held down until all 3 lights go out) |
| 17 | Police (not sent unless button is held down until all 3 lights go out) |
| 19 | Medical (not sent unless button is held down until all 3 lights go out) |
| 25 | Cancel |

To use this, you'll want to create some Home Assistant automations like:

     automation:
       - alias: "Disarm alarm from keypad"
         trigger: 
           platform: event
           event_type: "zwave_js_notification"
           event_data:
               command_class: 111
               node_id: *NODE_ID*
               event_type: 2
         action:
           - service: alarm_control_panel.alarm_disarm
             entity_id: alarm_control_panel.home_alarm
             data:
               code: "{{ trigger.event.data.event_data }}"
                 

## Indicators (lights and sounds)

The lights and sounds on the keypad are controlled with the Z-Wave Indicator Command class.  Unfortunately, Home Assistant does not have a nice API for this command class.  Fortunately, it does provide the `zwave_js.set_value` action for performing arbitrary Z-Wave commands, which we can use for this purpose.

For example, to set the keypad to disarmed mode, we can run the following command:

    service: zwave_js.set_value
    target:
       entity_id: sensor.ring_keypad_v2_battery_level
    data:
      command_class: '135'
      endpoint: '0'
      property: '2'
      property_key: '1'
      value: 1
      
You can use any `entity_id` associated with the keypad, or find the device id and use that.  `command_class` will always be 135 (the indicator command class), and `endpoint` will always be 0.  For most messages, I find it makes sense to use a `property_key` of 1 and `value` of 1, just indicating that the indicator should be turned on (meaning that we're playing a message or changing the mode of the keypad).  For indicators where a time make sense (e.g. entry delay, exit delay or alarms), use `property_key` 7 and a value of the number of seconds you want it to last.

The following tables summarize the indicators by `property`s that actually do anything that I can find.

### Modes, alarms and messages (use `property_key` 1)

| `property` | Description |
| ---------- | ----------- |
| 2 | Disarmed.  Keypad says "Disarmed," disarmed light lights up on motion. |
| 9 | Code not accepted.  A soft error tone plays. |
| 10 | Armed Stay.  Keypad says "Home and armed," armed stay light lights up on motion. |
| 11 | Armed Away.  Keypad says "Away and armed," armed away light lights up on motion. |
| 12 | Generic alarm.  Plays alarm, flashes light until another mode is selected.  Does not respect duration (property_key 7). |
| 13 | Burglar alarm.  Identical to 12. |
| 14 | Smoke alarm.  Plays smoke alarm, flashes light until another mode is selected.  Does not respect duration (property_key 7). |
| 15 | Carbon monoxide alarm.  Plays intermittent beeping alarm, flashes light until another mode is selected.  Does not respect duration (property_key 7). |
| 16 | Keypad says "Sensors require bypass."  Enter button blinks. |
| 19 | Medical alarm.  Medical button lights, bar flashes.  No alarm sound plays.  Does not respect duration (property_key 7). |

### Delays (use `property_key` 7 to specify length in seconds)

| `property` | Description |
| ---------- | ----------- |
| 17 | Entry delay.  Keypad says "Entry delay started." Plays sound, speeding up near end of specified duration.  Bar shows countdown. |
| 18 | Exit delay.  Keypad says "Exit delay started." Plays sound, speeding up near end of specified duration.  Bar shows countup. |

## Automation Blueprint

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FImSorryButWho%2FHomeAssistantNotes%2Fblob%2Fmain%2Fkeypad_blueprint.yaml)

I've created a [blueprint](https://github.com/ImSorryButWho/HomeAssistantNotes/blob/main/keypad_blueprint.yaml) that will create an automation which handles all the basic interactions between an `alarm_control_panel` instance and the Ring Keypad v2.  Pair the keypad following the instructions above, and configure an Alarmo instance in your Home Assistant.  Then, you can install the Blueprint above and create the automation, giving it the device for the keypad, the entity_id for the alarm instance, as well as the Z-Wave Node ID for the keypad (necessary to identify the events of interest), and the entry and exit delay times for the alarm.

Please feel free to send pull requests for improvements.  This is my first Blueprint.
