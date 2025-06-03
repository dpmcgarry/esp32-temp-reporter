# esp32-marine-sensorhub

Reads data from sensors and reports the data via MQTT.
Part of a larger architecture to use ESP32 microcontrollers to record, store, report sensor data in a marine environment using commercial-grade open source software.

## Background

I have a boat, I'm a Software Engineer, and I am ~~cheap~~ frugal.
NMEA2000 is pretty nice from a raw capabilities perspective but it has a number of limitations:

1. Everything is isolated to the NMEA2000 ecosystem. This is both a strength from a simplicity / reslience perspective but a weakness in that your visualization, alarming, logging, access, etc. is also limited.
1. The sensors used in NMEA2000 networks aren't anything really "new". Things like temperature sensors, tank sensors, flow sensors, etc. have existed in industrial automation for years.
1. The componenets / bridges to connect to NMEA2000 are more expensive than an ESP32 microcontroller (Hundreds to Thousands of USD vs. ~$10-15USD)

There are a lot of other frameworks / projects that could tackle this problem but they also have their drawbacks (some of them are just personal to me):

1. SignalK aimed to address some of the above constraints but there is fairly tight coupling between the open data standard and the reference implementation.
1. The SignalK standard itself (at least from what I could decipher) didn't lend itself to extensibility - e.g. I wanted to define arbitrary temperature sensors in locations of my choosing and the spec only seemed to allow for pre-defined temperature locations without any ability to extend it.
1. The reference implementation is implemented using NodeJS / Javascript which is not my personal preference for programming language. More concerning is that as with most JS based projects dependencies are managed using npm that pulls in a LOT of unvetted 3P dependencies. I didn't want to put something on my boat that took on that level of risk.
1. ESPHome / HomeAssistant is viable as well and there is an awesome ecosystem there. This is the road I initially went down, but when I started to try to get a ESP32-C6 to work I ran into a lot of issues. HA/ESPHome are highly complex OSS projects and are working on having the most impact to the smarthome community. As such, there are a number of layers of complexity to add new microcontroller support / tweak the behavior of the sytem to work well in a marine environment / etc. tldr - for me, this complexity didn't provide the value-add I was looking for in my case. I would imagine that many others would decide the opposite.

## My High Level Requirements

So I thought about what my requirements were and have at least these in mind for now:

1. Use commercially proven OSS solutions / technologies
1. Target linux since I am running a mini-pc 'server' on my boat running linux
1. Support 'soft real-time' visualization of data
1. Support archiving of all data received with export to something off-boat when there is connectivity for analytics/backup
1. Follow the KISS principal - have components do a small set of well-defined things and do it in a reliable / resilient manner
1. Minimize the attack / complexity surface where possible
1. Support getting all data from my existing NMEA2000 network into this platform (I am thinking readonly at this point)

## Capabilities

Here are some things I know that I want to 'solve' with this system. This may grow / shrink over time.
(This is somewhat of an addendum of specifics onto the high level requirements)

1. Monitor the temperature of different rooms on the boat
1. Monitor the temperature of fridges / freezers
1. Monitor the dry and wet exhaust temperatures of all engines
1. Monitor tank levels (Fuel, Water, Waste)
1. Monitor vacuum levels (Fuel Filtration)
1. Monitor voltages (Different Battery Banks)
1. Monitor other engine basics where feasible (Oil Pressure, Coolant Temp, Transmission Pressure, Transmission Temperature)
1. Monitor flow of water for seawater cooling pumps (for the A/C and Freezer)
1. Support multiple engines with multiple devices (I have a main engine, generator, and wing engine)

## Approach / HLD

I wanted to start with monitoring room and fridge/freezer temperature since that was straightforward. I initially was going to reuse some old 433MHz sensors I had lying around but most of those limited me to a max of ~3 different locations.
I then discovered that BLE "Beacon" type sensors were extremely cheap and did exactly what I was looking for - you could place one of these sensors wherever and as long as you could receive their BLE beacon messages you were good to go. No need to worry about wires going through fridge / freezer doors and just as cheap as the 433MHz sensors with no limit on the number of sensors you can have in a single installation.  I ended up selecting these two sensors:

* <https://www.amazon.com/dp/B08S3CGZ3Q> << Does temp only which is fine for fridge/freezer
* <https://www.amazon.com/dp/B08S34C5X9> << Does temp and humidity for an extra $5

Since these sensors are buried away in different spots that BLE can't reach the server we need an intermediary. That's where the esp32-marine-sensorhub comes in. It reads data from these sensors and then reports it upstream to the server. I'll continue to apply this pattern for other future sensor types.

I *could* have used a few (maybe 2 or 3) RaspberryPi devices could do this as well and considered that. The downside of that is that while only slightly more expensive you then have an entire linux OS to contend with. That sure does make a lot of things easier, but it also adds a bunch of complexity such as patching OS's / dealing with computer hardware failures / debugging / etc. In my case, the ESP32 was a better choice. If one dies I can just drop in a replacement. I don't need to patch an OS and there is little to no debugging you need to do for the device itself.

I'm using MQTT over TLS to send messages to the server. Right now I am using username/password based authentication because I didn't want to setup a private CA to issue certs to devices and I don't *think* that you can get device certs issued from Let's Encrypt. I am using Let's Encrypt for all the certs on my boat. I have a PrivateCA at home where I do development.
The server runs the Mosquitto MQTT Broker from Eclipse (<https://mosquitto.org/>) which is extremely proven technology.

The ESP32 code should be simple, predictable, and auditable. It needs to support WiFi connectivity, MQTT, and the needed code to read from the sensors. No more, no less. I want to focus more of my energy on reliability so handling flaky network/sensors/etc. is of higher importance than supporting a lot of different 'features' on the ESP32 itself.

In order to pull in my existing N2K data (as well as some bonus data) I am using the CerboGX on my boat. I connected the Cerbo to the N2K network and enabled MQTT functionality in the Cerbo. Voila I can get my N2K data over MQTT in my system. (Note that the MQTT integration into the platform is still pending and I will add a repo here once I have that in a state where I can share it).

On the 'server' I am running:

* Ubuntu as the base OS
* Mosquitto as mentioned for MQTT
* InfluxDB for timeseries data storage(<https://www.influxdata.com/products/influxdb/>)
* Grafana for visualization(<https://grafana.com/>)
* MariaDB for non-time-series data storage (<https://mariadb.org/>)
* HAProxy as a forward proxy to support SNI and hostname-based routing (<https://www.haproxy.org/>)
* Certbot/Let's Encrypt for Certificates (<https://certbot.eff.org/>)
* Technitium for local DNS (<https://technitium.com/dns/>)
* PiHole for Ad Blocking (<https://pi-hole.net/>)
* AWS for backup/off-boat analytics (<https://aws.amazon.com/>)
* Marine Sensorhub MQTT Router (<https://github.com/dpmcgarry/marine-sensorhub-mqtt>)

All of these technologies are commercially proven and would likely be considered by most to be 'boring' technology (which is a good thing) <https://boringtechnology.club/>.

The entire server is setup with a single ansible playbook. If I can cleanup the don-specific/sensitive content in that I can share it.

This project is tackling the ESP side of things. As I do more and have things of value to share with whatever code I need to write on the server and/or other configuration stuff I'll create additional repos and share them here.

I previously approached this by trying to go lower-level and not leveraging the Arduino framework. That turned out to be a mistake IMO because I didn't understand that using Arduino didn't actually *limit* you at all. You can still access FreeRTOS and raw ESP IDF goodness under the hood if you like. The Arduino framework gives you a LOT of great libraries and is C++ oriented vs. pure C oriented. I was investing way too many innovation tokens getting things working with just the IDF and pure C. Being able to leverage some of the awesome code from AdaFruit (buy stuff from them!) is way too valuable not to pass up.

## Getting Started

I tried to make forking / reusing this pretty straightforward.

You should be able to use/fork this by doing the following:

1. Pull down the code
1. Install Arduino IDE / CLI on your machine
1. Install the ESP-IDF on your machine (installing the VSCode extension makes this semi-trivial)
1. Create your own settings CSV file based on the example. You can create multiple environments by creating multiple settings CSV files.
1. Right now I am working with ESP32-C6 and ESP32-S3 devkits so that *should* work out of the box. However adding support for new hardware shouldn't be too hard as long as the ESP implementation of Arduino supports it.
1. Running the BLE poller, the RTD, and the heartbeat on a C6 starts to bog it down a bit. Since I am thinking of having a few different instances of this running for various tasks, the S3 may be a better fit for wired sensors. You give up AX WiFi but you gain a second core at bump from 160MHz to 240MHz. Will do some testing as I roll this out.
1. That said, right now I am focused on supporting ESP32 chips using the Arduino Framework. I don't see value in supporting hardware outside of the ESP32 ecosystem. This could of course change with time.
1. Build and flash the sketch
1. Generate and flash the settings partition using the commands below
1. Restart your ESP and enjoy

Commands To Generate and Write NVS Partition - assumed the IDF is installed in your home directory (this is what the IDF installer in VSCode does) - Change mshprefs.csv to match your CSV name:

```bash
. ~/esp/esp-idf/export.sh
python ~/esp/esp-idf/components/nvs_flash/nvs_partition_generator/nvs_partition_gen.py generate ./devprefs.csv ./devprefs.bin 20480
python ~/esp/esp-idf/components/partition_table/parttool.py --port /dev/ttyUSB0 write_partition --partition-name=nvs --input ./devprefs.bin 

```

Get Intellisense working by adding this to your include path:

```json
"~/.arduino15/packages/esp32/tools/esp32-arduino-libs/idf-release_v5.1-b6b4727c58/esp32c6/include/**",
"~/.arduino15/packages/esp32/tools/esp-rv32/2302/risarduino-cli compile --fqbn esp32:esp32:esp32c6:FlashSize=8M,PartitionScheme=default_8MB esp32-marine-sensorhub.ino -v -e
ESP32C6:

```bash
arduino-cli compile --fqbn esp32:esp32:esp32c6:FlashSize=8M,PartitionScheme=default_8MB esp32-marine-sensorhub.ino -v -e
```

ESP32S3:

```bash
arduino-cli compile --fqbn esp32:esp32:esp32s3:FlashSize=8M,PartitionScheme=default_8MB esp32-marine-sensorhub.ino -v -e
```

Manual Build Command with Version Defined (Need to merge cpp.extra_flags from platform.txt):

ESP32C6:

```bash
arduino-cli compile --fqbn esp32:esp32:esp32c6:FlashSize=8M,PartitionScheme=default_8MB --build-property "compiler.cpp.extra_flags=-DMSH_VERSION=\"42.0.0\" -MMD -c" esp32-marine-sensorhub.ino -v -e
```

ESP32S3:

```bash
arduino-cli compile --fqbn esp32:esp32:esp32s3:FlashSize=8M,PartitionScheme=default_8MB --build-property "compiler.cpp.extra_flags=-DMSH_VERSION=\"42.0.0\" -MMD -c" esp32-marine-sensorhub.ino -v -e
```

Manual Flash Command:

ESP32C6:

```bash
. ~/esp/esp-idf/export.sh
esptool.py --chip esp32c6 --port "/dev/ttyUSB0" --baud 921600  --before default_reset --after hard_reset write_flash  -z --flash_mode keep --flash_freq keep --flash_size keep 0x0 "esp32-marine-sensorhub.ino.bootloader.bin" 0x8000 "esp32-marine-sensorhub.ino.partitions.bin" 0xe000 ~/.arduino15/packages/esp32/hardware/esp32/3.1.0-RC1/tools/partitions/boot_app0.bin 0x10000 "esp32-marine-sensorhub.ino.bin" 
```

ESP32S3:

```bash
. ~/esp/esp-idf/export.sh
esptool.py --chip esp32s3 --port "/dev/ttyUSB0" --baud 921600  --before default_reset --after hard_reset write_flash  -z --flash_mode keep --flash_freq keep --flash_size keep 0x0 "esp32-marine-sensorhub.ino.bootloader.bin" 0x8000 "esp32-marine-sensorhub.ino.partitions.bin" 0xe000 ~/.arduino15/packages/esp32/hardware/esp32/3.1.0-RC1/tools/partitions/boot_app0.bin 0x10000 "esp32-marine-sensorhub.ino.bin" 
```

Manual Monitor Command:

ESP32C6:

```bash
arduino-cli monitor --port /dev/ttyUSB0 --config baudrate=115200
```

### ESP Notes

#### ESP32S3

SPI Pins:
MOSI: 11
MISO: 13
SCK: 12
CS: 10

FQBN: esp32:esp32:esp32s3:FlashSize=8M,PartitionScheme=default_8MB

#### ESP32C6

SPI Pins:
MOSI: 19
MISO: 20
SCK: 21
CS: 18

FQBN: esp32:esp32:esp32c6:FlashSize=8M,PartitionScheme=default_8MB

#### XiaoS3

SPI Pins:
MOSI: 18
MISO: 20
SCK: 19
SS: 21

FQBN: esp32:esp32:XIAO_ESP32S3

## TODO

* ~~Cleanup the WiFi Utils code~~
* ~~Support Static IP/DNS~~
* ~~Write data as JSON~~
* ~~Add SNTP Support (Needed for TLS among other things)~~
* ~~Change BLE Implementation to be Sync not async since it is task based which will be cleaner~~
* ~~Move BLE Listener to own task that enqueues messages for main task~~
* ~~Add reconnect logic for WiFi and MQTT~~
* ~~Add topic structure / config to MQTT~~
* ~~MQTT Auth~~
* ~~Support private CA signed certs~~
* ~~Confirm that multiple BLE sensors works~~
* ~~Add BLE Humidity Sensor Support~~
* Add thermocouple support
* ~~Set Device Status using Pixel LED~~
* ~~Add device heartbeat / status~~
* ~~Add PT RTD support~~
* Add configuration support for multiple RTD / Thermocouple Sensors by CS Pin
* Add message structure for multiple RTD / Theromocouples
* ~~NTP Time is flaky - appears to be a race condition of some kind~~
* ~~MQTT does not gracefully reconnect~~
* ~~Test what happens when WiFi is lost / regained~~
* Figure out how to make a single flash file
* Add INA219 Support for Oil Pressure, Engine Temp
* ~~Test integration with Telegraf~~
* Network logging (Syslog)
* Add external ADC support for 4-20ma sensors
* Store some log buffer locally to send over network on reconnect
* OTA Support
* Add an actual architecture diagram in here
* Figure out how to use / add the default root CA trust store
* ~~Setup a CI build for this so I can share releases~~
* Add XiaoS3 Support
