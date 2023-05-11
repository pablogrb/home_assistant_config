# Whole flat environmental monitoring

Using Home Assistant to collect data from a met station, thermo-hygrometers, particulate matter sensors, and smart energy meter.

# Part 0: Home Assistant setup

Due to the overwhelming shortage of Raspberry Pi boards worldwide, the existing functions of my 3B+ board should be mantained where possible.

Currently the Pi is running 2 main pieces of software:
- EmulationStation through RetroPie
- Kodi as a RetroPie port

Now that we have a wii at home, the need for retro-games emulation has been superseeded by it. The Pi now has to run two pieces of software
- Kodi
- Home Assistant

### Bill of materials
- Raspberri Pi 3B+
- Raspberri Pi 3B+ power supply. Upgraded to a Raspberry Pi 4 power supply with a USB-C to micro B converter as the original 3B+ PSU couldn't handle the power draw when playing 1080p video.
- Micro SD to SD card adapter

### Task list
- Install [Raspberry Pi OS Lite](https://www.raspberrypi.com/software/operating-systems/) using the official 64 bit installer. In addition the fan control config was added and the Withings2Garmin script was installed. The base code is currently broken, but the branch for [pull request #11](https://github.com/sodelalbert/Withings2Garmin/tree/fix/api-update) is currently working.
- Install and setup Kodi following these [instructions](https://forums.raspberrypi.com/viewtopic.php?t=251645). Set-up suggestions require way too much video memory allocation wich starves the pi 3B+. Video memory was set to 128. Additionally, Kodi was crashing when playing some videos due to audio issues. To fix this Audio Passthrough was enabled. My TV supports Dolby Digital, Dolby Digital Plus and TrueHD, so those options were enabled.
- Install Docker following these instructions [here](https://jfrog.com/connect/post/the-best-way-to-install-docker-on-a-raspberry-pi/).
- Install the Home Assistant Docker container following these [instructions](https://www.home-assistant.io/installation/raspberrypi#install-home-assistant-container)