# ESPHome Project: Sonos Controller

My daughter loves listening to music in her bathroom, and often wants to choose a specific song from her playlist.  That's traditionally involved yelling down the stairs for someone to skip to a different track... until now.

I ran across the [M5Stack Dial](https://docs.m5stack.com/en/core/M5Dial), saw it's [supported in ESPHome](https://devices.esphome.io/devices/M5Stack-Dial), and figured I had a basis for a nice solution.

![PXL_20250309_142529758 MP](https://github.com/user-attachments/assets/cc3b965e-50b0-411c-8d6d-d7d3f82504a0)

Currently, it's awfully hacky: I have a bunch of helpers in Home Assistant that are updated when the playlist or queue position changes, and get reflected directly into the UI.  A nicer solution would be to use [LVGL](https://esphome.io/components/lvgl/index.html) to display the whole queue locally, but I haven't taken the time to figure that out.  An ehancement for the future.
