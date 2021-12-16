# Ring Keypad v1 With Home Assistant

It turns out that for full functionality the keypad must be paired with S2 security, the higher-security Z-Wave mode which wasn't supported by any open-source projects.  Until [August 2021](https://github.com/zwave-js/node-zwave-js/releases/tag/v8.1.0), when ZWaveJS added support for S2.  [ZWaveJS2MQTT](https://github.com/zwave-js/zwavejs2mqtt) added support for S2 soon after, which was the key to using the Ring Keypad with Home Assistant successfully.

## Pairing

Now you're ready to pair.  Plug the keypad into a power outlet (it will not pair on batteries).  Start inclusion by clicking "Manage Nodes" on the main ZWaveJS2MQTT page, then selecting Default mode.  Once inclusion mode starts, hold down the 1 key on the keypad until the green indicator starts flashing.

Eventually, a security choice pop-up will appear.  Click next quickly, and the DSK prompt will appear.  Paste in the PIN that you can find on the back of the keypad and click next quickly.  In a few seconds, you should see confirmation that it was paired with S2 Access Control.  If not, you'll need to try again.

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
| 6 | Arm Stay |]
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
      property: 'value' #this is not a placeholder!
      value: 3
      
You can use any `entity_id` associated with the keypad, or find the device id and use that.  `command_class` will always be 135 (the indicator command class), and `endpoint` will always be 0.

The following tables summarize the indicators by `property`s that actually do anything that I can find.

### Modes, alarms and messages (use `property_key` 1)

| `property` | Description |
| ---------- | ----------- |
| 3 | Disarmed. Keypad says "Disarmed," disarmed light lights up on motion. |
| 19 | Disarmed. But silent. |
| 8 | Code not accepted.  A soft error tone plays. |
| 1 | Armed Stay. Keypad says "Home and armed," armed stay light lights up on motion. |
| 17 | Armed Stay. But silent. |
| 2 | Armed Away. Keypad says "Away and armed," armed away light lights up on motion. |
| 18 | Armed Away. But silent. |
| 4 | Burglar alarm. Plays alarm, flashes light until another mode is selected.  Does not respect duration (property_key 7). |
| 5 | Keypad says "Sensors require bypass." Enter button blinks. |
| 22 | Same as 5, but silent. |
| 20 | Medical alarm. Medical button lights, bar flashes.  No alarm sound plays. Does not respect duration (property_key 7). |

### Delays (there's no way to set the duration of the delays manually so we're stuck with 15, 30 or 45 seconds)
| `property` | Description |
| ---------- | ----------- |
| 22 | Entry delay of 15 seconds. Plays sound, speeding up near the end.  Ring shows countdown. |
| 38 | Entry delay of 30 seconds. Plays sound, speeding up near the end.  Ring shows countdown. |
| 54 | Entry delay of 45 seconds. Plays sound, speeding up near the end.  Ring shows countdown. |
| 23 | Exit delay of 15 seconds. Plays sound, speeding up near the end.  Ring shows countup. |
| 39 | Exit delay of 30 seconds. Plays sound, speeding up near the end.  Ring shows countup. |
| 55 | Exit delay of 45 seconds. Plays sound, speeding up near the end.  Ring shows countup. |

### Sounds

| `property` | Description |
| ---------- | ----------- |
| 15 | Synthy double sound |
| 31 | Acoustic guitar |
| 47 | A nice chime |
| 63 | Ding-dong |
| 79 | Double harpischord like sound |

## Automation Blueprint

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FImSorryButWho%2FHomeAssistantNotes%2Fblob%2Fmain%2Fkeypad_blueprint-v1.yaml)

I've created a [blueprint](https://github.com/ImSorryButWho/HomeAssistantNotes/blob/main/keypad_blueprint-v1.yaml) that will create an automation which handles all the basic interactions between an `alarm_control_panel` instance and the Ring Keypad v1.  Pair the keypad following the instructions above, and configure an Alarmo instance in your Home Assistant.  Then, you can install the Blueprint above and create the automation, giving it the device for the keypad, the entity_id for the alarm instance, as well as the Z-Wave Node ID for the keypad (necessary to identify the events of interest), and the entry and exit delay times for the alarm.

Please feel free to send pull requests for improvements.  This is my first Blueprint.
