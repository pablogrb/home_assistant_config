# Whole flat environmental monitoring

Using Home Assistant to collect data from a met station, thermo-hygrometers, particulate matter sensors, and smart energy meter.

# Part 0: Home Assistant setup

## 0.1. Starting Point

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

## 0.2 Docker Run

Launching and re-launching the Home Assistant Docker container can be made easier with a run script that contains the `docker run` command:
`docker_run.sh`
```bash
docker run -d --name homeassistant --privileged --restart=unless-stopped -e TZ=Europe/London -v /home/pi/homeassistant:/config --network=host --device /dev/ttyUSB0:/dev/ttyUSB0 ghcr.io/home-assistant/home-assistant:stable
```

# Part 1: Air Quality

## 1.1. Indoor PM with the IKEA Vindriktning

The base sensor will be an [IKEA Vindriktning](https://www.ikea.com/gb/en/p/vindriktning-air-quality-sensor-80515910/) (VDK) air quality monitor. These are currently sold for £10 and due to their low price, availability and open architecture, have developed a significant IoT community around them.

Github user [@Hypfer](https://github.com/Hypfer) developed an [ESP8266](https://en.wikipedia.org/wiki/ESP8266) firmware to add [MQTT](https://en.wikipedia.org/wiki/MQTT) connectivity to the Vindriktning sensor as detailed [here](https://github.com/Hypfer/esp8266-vindriktning-particle-sensor). The firmware supports Home Assistant Autodiscovery to integrate the sensor data stream into the Home Assistant database.

### Bill of materials

- 3x IKEA Vindriktning Air Quality sensors
- 3x WeMos D1 mini ESP8266 boards (Thanks [@olivercrask](https://github.com/olivercrask))
- Soldering iron [ifixit](https://store.ifixit.co.uk/products/soldering-station)
- Helping hands [ifixit](https://store.ifixit.co.uk/products/helping-hands?variantid=31652614832177)
- Flush cutters [ifixit](https://store.ifixit.co.uk/products/flush-cutter-c-h-p-chp-170)
- Solder [ifixit](https://store.ifixit.co.uk/products/stannol-solder-hs10fair)
- Long PH0 Screwdriver
- Data-enabled USB A to USB micro B cable

### Task list

- Solder contacts to the sensor board and the Wemos using the test pads of the sensor and the through holes of the wemos following [these](https://style.oversubstance.net/2021/08/diy-use-an-ikea-vindriktning-air-quality-sensor-in-home-assistant-with-esphome/) instructions.
- Flash the firmware to the wemos. Resolved: Get the Arduino IDE from the Microsoft Store, install ESP support _then_ select the wemos board as the board type _then_ install the libraries. When setting up the mqtt server, only the ip of the mosquitto container is needed (no `mqtt:` prefix).
- Set up an MQTT broker as a [docker instance](https://hometechhacker.com/mqtt-home-assistant-using-docker-eclipse-mosquitto/)
- Set up the MQTT integration on home assistant following the [official documentation](https://www.home-assistant.io/integrations/mqtt/)

### Home Assistant Setup

During the configuration of the Wemos firmware the WiFi and MQTT info is configured for each sensor. If the MQTT broker and integration were successful the sensors and associated data should be autodiscovered by Home Assistant.

## 1.2 Outdoor air quality with an EarthSense Zephyr(R)

The Zephyr(R) is a low cost air quality monitor from EarthSense that measures atmospheric pollutants and sends the data to the Earthsense servers to be viewed online on the MyAir website or accessed through the Zephyr API.

### Bill of materials

- Earthsense Zephyr
- Power supply. An extension to the power supply cable was needed to deploy the sensor outdoors and keep the powersupply indoors. All outdoor connections were protected with electrical tape.
- Mounting hardware

### Home Assistant setup

Access to the Zephyr API data is achieved using a `platform: rest` sensor in the `configuration.yml` file with a value template set to "OK". The actual values are parsed as json attributes.

```yaml
  - platform: rest
    name: "zephyr_311"
    resource_template: https://data.earthsense.co.uk/measurementdata/v1/311/{{ (utcnow()-timedelta(minutes=17)).strftime('%Y%m%d%H%M') }}/{{ (utcnow()-timedelta(minutes=2)).strftime('%Y%m%d%H%M') }}/A/3
    unique_id: "zephyr_311"
    # check every 15 minutes
    scan_interval: 900
    force_update: true
    headers:
      accept: application/json
      username: <username>
      userkey: <userkey>
    json_attributes_path: "$['data']['15 min average on the quarter hours']['slotA']"
    json_attributes:
      - "O3"
      - "NO"
      - "NO2"
      - "particulatePM1"
      - "particulatePM25"
      - "particulatePM10"
      - "tempC"
      - "humidity"
      - "ambTempC"
      - "ambHumidity"
      - "ambPressure"
    value_template: "OK"
 ```
 
The sensor values are then parsed out of the attributes and assigned names, units and ids using template sensors. Note that $NO_x$ is calculated on the fly from the $NO$ and $NO_2$ with $NO_x = NO_2 + 1.5335 NO$
 
 ```yaml
   - platform: template
    sensors:
      zephyr_311_o3:
        value_template: "{{ state_attr('sensor.zephyr_311', 'O3')['data'][0] }}"
        device_class: ozone
        unit_of_measurement: "μg/m³"
        unique_id: zephyr_311_ozone
      zephyr_311_no:
        value_template: "{{ state_attr('sensor.zephyr_311', 'NO')['data'][0] }}"
        device_class: nitrogen_monoxide
        unit_of_measurement: "μg/m³"
        unique_id: zephyr_311_no
      zephyr_311_no2:
        value_template: "{{ state_attr('sensor.zephyr_311', 'NO2')['data'][0] }}"
        device_class: nitrogen_dioxide
        unit_of_measurement: "μg/m³"
        unique_id: zephyr_311_no2
      zephyr_311_nox:
        value_template: "{{ state_attr('sensor.zephyr_311', 'NO2')['data'][0] + state_attr('sensor.zephyr_311', 'NO')['data'][0]*1.5335 }}"
        # device_class: nitrogen_oxides
        unit_of_measurement: "μg/m³"
        unique_id: zephyr_311_nox
      zephyr_311_pm1:
        value_template: "{{ state_attr('sensor.zephyr_311', 'particulatePM1')['data'][0] }}"
        device_class: pm1
        unit_of_measurement: "μg/m³"
        unique_id: zephyr_311_pm1
      zephyr_311_pm25:
        value_template: "{{ state_attr('sensor.zephyr_311', 'particulatePM25')['data'][0] }}"
        device_class: pm25
        unit_of_measurement: "μg/m³"
        unique_id: zephyr_311_pm25
      zephyr_311_pm10:
        value_template: "{{ state_attr('sensor.zephyr_311', 'particulatePM10')['data'][0] }}"
        device_class: pm10
        unit_of_measurement: "μg/m³"
        unique_id: zephyr_311_pm10
      zephyr_311_tempc:
        value_template: "{{ state_attr('sensor.zephyr_311', 'tempC')['data'][0] }}"
        device_class: temperature
        unit_of_measurement: "°C"
        unique_id: zephyr_311_tempc
      zephyr_311_humidity:
        value_template: "{{ state_attr('sensor.zephyr_311', 'humidity')['data'][0] }}"
        device_class: humidity
        unit_of_measurement: "%"
        unique_id: zephyr_311_humidity
      zephyr_311_ambtempc:
        value_template: "{{ state_attr('sensor.zephyr_311', 'ambTempC')['data'][0] }}"
        device_class: temperature
        unit_of_measurement: "°C"
        unique_id: zephyr_311_ambtempc
      zephyr_311_ambhumidity:
        value_template: "{{ state_attr('sensor.zephyr_311', 'ambHumidity')['data'][0] }}"
        device_class: humidity
        unit_of_measurement: "%"
        unique_id: zephyr_311_ambhumidity
      zephyr_311_ambpressure:
        value_template: "{{ state_attr('sensor.zephyr_311', 'ambPressure')['data'][0] / 100 }}"
        device_class: pressure
        unit_of_measurement: "hPa"
        unique_id: zephyr_311_ambpressure
 ```

 # Part 2: Met data

## Meteorology from the Bresser 5 in 1 met station.

The [Bresser 5 in 1](https://www.bresseruk.com/Weatherstations/Weather-Center/BRESSER-WIFI-color-weather-center-with-5in1-profi-sensor.html) met station sends data to [Weather Underground](https://www.wunderground.com/dashboard/pws/INOTTI173) and [Weather Cloud](https://app.weathercloud.net/d5768241226#profile) but the Home Assistant integrations with these platforms is incomplete at best and limited hourly data.

However the station broadcasts the data in the 868mHz band so it can be sniffed using a radio receiver.

Github user [@jcallaghan](https://github.com/jcallaghan) is tracking the develpment of a Home Assistant stack to integrate data from the Bresser station into Home Assistant through an [issue](https://github.com/jcallaghan/home-assistant-config/issues/13) in his Home Assistant Config repository.

This requires significant research still, but the _gist_ of it is that an 868mHz software-defined radio receiver is used to sniff the data, the data is then passed to an MQTT sink, and then the MQTT sink is passed to Home Assistant.

The key ingredient is the [rtl_433 generic data receiver](https://github.com/merbanan/rtl_433) by github user [@merbanan](https://github.com/merbanan). This receiver has support for the Bresser 5 in 1 station, and additionally for the Bresser Thermo-/Hygro-Sensor 3CH, which hopefully will be compatible to the [Thermo-/Hygro-Sensor 7CH](https://www.bresseruk.com/BRESSER-Thermo-Hygro-Sensor-7-Channel-868-MHz.html) that's co-located with the living room Vindriktning

It's not clear how the thermo-hygrometer in the base station transmits its data. WeatherCloud receives and displays its data and according to someone who's link I lost, it's also transmitted to Weather Underground, although it's not displayed. While not having access to that data is not critical, it would be useful if a T-RH-PM calibration system is implemented.

### Bill of materials
- Bresser 5-in-1 weather station from [Amazon](https://www.amazon.co.uk/Bresser-Wetter-Center-WiFi-Profi-Sensor/dp/B07JMVG1WC)
- 868mHz software-defined radio receiver from [Amazon](https://www.amazon.co.uk/gp/product/B009U7WZCA)
- Raspberry Pi for hosting the radio receiver

### Task list
- Look for a cheap 868mHz SDR receiver that's compatible with rtl_433. The Nooelec NESDR Mini has the RTL2832U and R820T tunners reported as needed by the SDR software stack
- Install the rtl_433. Build according to the [build instructions](https://github.com/merbanan/rtl_433/blob/master/docs/BUILDING.md). Tested and received data with this command:
`rtl_433 -f 868M -s 1024k`.
The output contains data from the weather station and the thermohygrometer.
After some finegling I have the rtl_433 sending data to the MQTT broker and successfully receiving it in CLI and in the Home Assistant MQTT integration with the command
`rtl_433 -f 868M -s 1024k -F "mqtt:<raspi ip>,user=<username>,pass=<password>,retain=0,devices=rtl_433[/id]"`
- Integrate the MQTT rtl_433 command as a service in RaspberryPi OS following [this instructions](https://www.dexterindustries.com/howto/run-a-program-on-your-raspberry-pi-at-startup/). The `rtl_433.service` file integrates the `launch_rtl_433.sh` script that waits for the MQTT container to boot and then lauches the rtl_433 receiver with logging.

`launch_rtl_433.sh`
```bash
#!/bin/bash
# wait for the MQTT container to load
until [ "`docker inspect -f {{.State.Running}} mqtt`"=="true" ] 
do
    sleep 2
    done
# launch rtl_433
rtl_433 -f 868M -s 1024k -F "mqtt:<raspi ip>,user=<username>,pass=<password>,retain=0,devices=rtl_433[/id]" > /home/pi/rtl_433.log 2>&1 
```

`rtl_433.service`
```service
[Unit]
Description=rtl_433 to MQTT bridge
After=multi-user.target

[Service]
Type=idle
ExecStart=/bin/bash /home/pi/launch_rtl_433.sh

[Install]
WantedBy=multi-user.target
```

### Home Assistant setup

The MQTT sensors must be configured manually in the `configuration.yml`. Note the state topics, for example `rtl_433/406136645/temperature_C`. Here `406136645` is a unique identifier for the met station. You can find which id's your rtl_433 connected devices are using by listenting to the `rtl_433/#` topic in the MQTT integration settings page.

`configuration.yaml`
```yaml
mqtt:
  sensor:
  # MQTT config for met and env data
    # Bresser 5-in-1 met station sensor
    - name: outside_temp_C
      state_topic: "rtl_433/406136645/temperature_C"
      device_class: temperature
      unique_id: bresser_temp_0
      unit_of_measurement: '°C'

    - name: outside_relhum
      state_topic: "rtl_433/406136645/humidity"
      device_class: humidity
      unique_id: bresser_rh_0
      unit_of_measurement: '%'

    - name: outside_rain
      state_topic: "rtl_433/406136645/rain_mm"
#      device_class: None
      unique_id: bresser_rain_0
      unit_of_measurement: "mm"

    - name: outside_wind_speed
      state_topic: "rtl_433/406136645/wind_avg_m_s"
#      device_class: None
      unique_id: bresser_wind_speed_0
      unit_of_measurement: "m/s"

    - name: outside_wind_gust
      state_topic: "rtl_433/406136645/wind_max_m_s"
#      device_class: None
      unique_id: bresser_wind_gust_0
      unit_of_measurement: "m/s"

    - name: outside_wind_dir
      state_topic: "rtl_433/406136645/wind_dir_deg"
#      device_class: None
      unique_id: bresser_wind_dir_0
      unit_of_measurement: "°"
```

A template sensor can be used to derive the *feels like* temperature from the recorded met data using the [Australian apparent temperature](https://en.wikipedia.org/wiki/Wind_chill).

`configuration.yaml`
```yaml
template:
  sensor:
    # Feels-like temperature
    - name: outside_feels_like
      unit_of_measurement: '°C'
      device_class: temperature
      unique_id: bresser_feel_temp
      state: >
        {% set e = 2.71828 %}
        {% set t_a = states('sensor.outside_temp_c') | float %}
        {% set RH  = states('sensor.outside_relhum') | float %}
        {% set ws  = states('sensor.outside_wind_speed') | float %}
        {% set wvp = RH/100*6.105*e**((12.27*t_a)/(237.7+t_a)) %}
        {% set AT = t_a + 0.33*wvp - 0.7*ws - 4 %}
        {{AT}}
```

# 3: Power data

Power data is collected using the [SMETS2 Display and CAD](https://shop.glowmarkt.com/products/display-and-cad-combined-for-smart-meter-customers) from Hildebrand Glowmarkt. This display collects data from the E-ON smart meter and exposes it to Home Assistant via MQTT.

### Bill of materials
- [SMETS2 Display and CAD](https://shop.glowmarkt.com/products/display-and-cad-combined-for-smart-meter-customers)

### Home Assistant setup

User [Robert A](https://community.home-assistant.io/u/robertalexa) has documented how to setup the Glowmarkt CAD with home assistant [here](https://community.home-assistant.io/t/glow-hildebrand-display-local-mqtt-access-template-help/428638). These instructions have been sumarized and extended by [Techbits](https://techbits.io/hildebrand-glow-display-mqtt-home-assistant-setup/). I added a couple of entries to get all the data from the CAD json.

`configuration.yaml`
```yaml
mqtt:
  sensor:
  # MQTT config for power data
    - name: "Smart Meter Electricity: Import"
      unique_id: "smart_meter_electricity_import"
      state_topic: "glow/441793526434/SENSOR/electricitymeter"
      device_class: "energy"
      unit_of_measurement: "kWh"
      state_class: "total_increasing"
      value_template: >
        {% if value_json['electricitymeter']['energy']['import']['cumulative'] == 0 %}
          {{ states('sensor.smart_meter_electricity_import') }}
        {% else %}
          {{ value_json['electricitymeter']['energy']['import']['cumulative'] }}
        {% endif %}
      icon: "mdi:flash"
    - name: "Smart Meter Electricity: Import (Today)"
      unique_id: "smart_meter_electricity_import_today"
      state_topic: "glow/441793526434/SENSOR/electricitymeter"
      device_class: "energy"
      unit_of_measurement: "kWh"
      state_class: "measurement"
      value_template: >
        {% if value_json['electricitymeter']['energy']['import']['day'] == 0 
          and now() > now().replace(hour=0).replace(minute=1).replace(second=0).replace(microsecond=0) %}
          {{ states('sensor.smart_meter_electricity_import_today') }}
        {% else %}
          {{ value_json['electricitymeter']['energy']['import']['day'] }}
        {% endif %}
      icon: "mdi:flash"
    - name: "Smart Meter Electricity: Import (This week)"
      unique_id: "smart_meter_electricity_import_week"
      state_topic: "glow/441793526434/SENSOR/electricitymeter"
      device_class: "energy"
      unit_of_measurement: "kWh"
      state_class: "measurement"
      value_template: "{{ value_json['electricitymeter']['energy']['import']['week'] }}"
      icon: "mdi:flash"
    - name: "Smart Meter Electricity: Import (This month)"
      unique_id: "smart_meter_electricity_import_month"
      state_topic: "glow/441793526434/SENSOR/electricitymeter"
      device_class: "energy"
      unit_of_measurement: "kWh"
      state_class: "measurement"
      value_template: "{{ value_json['electricitymeter']['energy']['import']['month'] }}"
      icon: "mdi:flash"
    - name: "Smart Meter Electricity: Import Unit Rate"
      unique_id: "smart_meter_electricity_import_unit_rate"
      state_topic: "glow/441793526434/SENSOR/electricitymeter"
      device_class: "monetary"
      unit_of_measurement: "GBP/kWh"
      state_class: "measurement"
      value_template: "{{ value_json['electricitymeter']['energy']['import']['price']['unitrate'] }}"
      icon: "mdi:cash"
    - name: "Smart Meter Electricity: Import Standing Charge"
      unique_id: "smart_meter_electricity_import_standing_charge"
      state_topic: "glow/441793526434/SENSOR/electricitymeter"
      device_class: "monetary"
      unit_of_measurement: "GBP"
      state_class: "measurement"
      value_template: "{{ value_json['electricitymeter']['energy']['import']['price']['standingcharge'] }}"
      icon: "mdi:cash"
    - name: "Smart Meter Electricity: Power"
      unique_id: "smart_meter_electricity_power"
      state_topic: "glow/441793526434/SENSOR/electricitymeter"
      device_class: "power"
      unit_of_measurement: "kW"
      state_class: "measurement"
      value_template: >
        {% if value_json['electricitymeter']['power']['value'] < 0 %}
          {{ states('sensor.smart_meter_electricity_power') }}
        {% else %}
          {{ value_json['electricitymeter']['power']['value'] }}
        {% endif %}
      icon: "mdi:flash""
```

Additionally, a template sensor is used to calculate the electricity cost for the day.
`configuration.yaml`
```yaml
template:
  sensor:
    # Energy Costs
    - name: "Smart Meter Electricity: Cost (Today)"
      unique_id: smart_meter_electricity_cost_today
      device_class: monetary
      unit_of_measurement: "GBP"
      state_class: "total_increasing"
      icon: mdi:cash
      state: "{{ (
          states('sensor.smart_meter_electricity_import_today') | float 
          * states('sensor.smart_meter_electricity_import_unit_rate') | float 
          + states('sensor.smart_meter_electricity_import_standing_charge') | float
        ) | round(2) }}"
```
