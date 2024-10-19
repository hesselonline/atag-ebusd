# ATAG-EBUSD

This repository contains configuration files for using [ebusd](https://github.com/john30/ebusd) with an ATAG Energion heat pump setup. The data from the heat pump is extracted via an eBus adapter, published to an MQTT broker, and displayed in Home Assistant for real-time monitoring and control.
The config and template files in this repository are based on the [Ariston config of wrongisthenewright](https://github.com/wrongisthenewright/ebusd-configuration-ariston-bridgenet). There is currently 1 config-file in this repository for an Energion Light Full Electric setup, the goal is to have other variants of the ATAG Energion heatpump in a seperate folder configuration file. Please create a pull request if you have a config or changes to share.

## Contents of this repository

[atag_energion_lightb.csv](atag_energion_lightb/atag_energion_lightb.csv) This is the config for a ATAG Energion heatpump in combination with a Lightbox unit on the inside; quite a minimal setup, missing some hybrid codes / information.
[\_templates.csv](atag_energion_lightb/_templates.csv) This is the template file for the ATAG Energion heatpump config, needed to define specific datatypes.
[mqtt-hassio.cfg](mqtt-hassio.cfg) This is the config file for the MQTT broker, needed to define the MQTT topics directly in Home Assistant.
[hass_ebusd_mqtt_poll_all.yaml](homeassistant_scripts/hass_ebusd_mqtt_poll_all.yaml) This is a script to use in combination with Home Assistant to force a read of all the sensors in the ebusd config.
[hass_ebusd_mqtt_poll_energymgr.yaml](homeassistant_scripts/hass_ebusd_mqtt_poll_energymgr.yaml) This is a script to use in combination with Home Assistant to force a read of the energy manager sensors in the ebusd config.
[hass_ebusd_mqtt_poll_heatpump.yaml](homeassistant_scripts/hass_ebusd_mqtt_poll_heatpump.yaml) This is a script to use in combination with Home Assistant to force a read of the heatpump sensors in the ebusd config.
[hass_ebusd_mqtt_poll_specific.yaml](homeassistant_scripts/hass_ebusd_mqtt_poll_specific.yaml) This is a script to use in combination with Home Assistant to force a read of a specific sensor in the ebusd config.

## Prerequisites

- **Docker**: Ensure Docker is installed and running on your system. [Docker installation guide](https://docs.docker.com/get-docker/)
- **eBus Adapter**: An eBus adapter is required to interface with the heat pump. I personally use this adapter: [Ebus Wifi Adapter](https://www.elecrow.com/ebus-to-wifi-adapter-module-v5-2.html)
- **MQTT Broker**: You will need a running MQTT broker to publish the data (e.g., Mosquitto).
- **Home Assistant**: A running instance of Home Assistant to visualize and interact with the heat pump data.

## Installation

1. Set up the Docker ebusd and MQTT containers. To do this you can either run two seperate containers or use a docker-compose file. My docker compose is the following (I run this on a Synology Diskstation, the config also includes Home Assistant):

```
services:
  homeassistant:
    image: homeassistant/home-assistant:latest
    restart: always
    volumes:
      - /volume1/docker/homeassistant/config:/config
      - /volume1/docker/certificate:/le-ssl
    devices:
      - /dev/ttyUSB0
      - /dev/ttyUSB1
    environment:
      - TZ=Europe/Amsterdam
    network_mode: host
  mosquitto:
    image: eclipse-mosquitto:latest
    restart: always
    volumes:
      - /volume1/docker/mosquitto/mosquitto.conf:/mosquitto/config/mosquitto.conf
    network_mode: bridge
    ports:
      - 1883:1883
  ebusd:
    image: john30/ebusd
    network_mode: host
    volumes:
      - /volume1/docker/ebusd/:/ebus/cfg
    environment:
      - EBUSD_MQTTHOST=192.168.X.XXX
      - EBUSD_MQTTPORT=1883
      - EBUSD_MQTTINT=/ebus/cfg/mqtt/mqtt-hassio.cfg
      - EBUSD_MQTTJSON=
      - EBUSD_MQTTTOPIC=ebusd/%circuit/%name
      - EBUSD_DEVICE=enh:192.168.X.YYY:3335
      - EBUSD_PORT=8128
      - EBUSD_HTTPPORT=8129
      - EBUSD_CONFIGPATH=/ebus/cfg/ebus
      - EBUSD_LOG=all:error
      - EBUSD_POLLINTERVAL=30
      - EBUSD_ACQUIRERETRIES=5
      - EBUSD_SENDRETRIES=10
      - EBUSD_SCANRETRIES=5
      - EBUSD_ENABLEHEX=
      - EBUSD_RECEIVETIMEOUT=10000
      - EBUSD_LATENCY=20000
      - EBUSD_ACQUIRETIMEOUT=5000
```

Please note here that the docker config refers to a specific config file for the MQTT broker, this file is included in this repository. The config file is needed to define the MQTT topics directly in Home Assistant. Also, the EBUSD_CONFIGPATH mentioned must contain the config files for the ATAG Energion heatpump. The config files are also included in this repository.

2. Configure your eBus adapter to work with your ATAG Energion heat pump. For the Wi-Fi adapter I’m using, the connection was simple. I 3D-printed a case (model can be found via the EBUS Adapter’s website).
   I connected it with a cable to the back of the Neoz (behind the screws there are now 2 wires instead of 1). The positive wire of the EBUS adapter is connected to the B terminal of the Neoz. I used a multimeter to check the polarity (+/-) of the connection. The controller used must connect to my Wi-Fi, which is explained on the website of the controller.

3. Ensure that Home Assistant has a working integration with you MQTT broker, the steps in doing so are beyond the scope of this guide, but please check the [Home Assistant documentation](https://www.home-assistant.io/integrations/mqtt/) on how to do this.

4. If the steps above are followed correctly you should be able to see the MQTT topics being created both your MQTT broker and in Home Assistant. The topics are created based on the config files in the ebusd config folder. The topics are created based on the following format: `ebusd/%circuit/%name`. The %circuit and %name are defined in the config files in the ebusd config folder. The Home Assistant specific topics should be in the format `homeassistant/sensor/%circuit_%name/state`, these topics refer to the topics mentioned above for there values.

5. Configure a Home Assistant script to refresh the data:
   Not all sensors are refreshed automatically, to force a refresh of the data you can use the following scripts in Home Assistant:
   [hass_ebusd_mqtt_poll_all.yaml](homeassistant_scripts/hass_ebusd_mqtt_poll_all.yaml) This is a script to use in combination with Home Assistant to force a read of all the sensors in the ebusd config.
   [hass_ebusd_mqtt_poll_energymgr.yaml](homeassistant_scripts/hass_ebusd_mqtt_poll_energymgr.yaml) This is a script to use in combination with Home Assistant to force a read of the energy manager sensors in the ebusd config.
   [hass_ebusd_mqtt_poll_heatpump.yaml](homeassistant_scripts/hass_ebusd_mqtt_poll_heatpump.yaml) This is a script to use in combination with Home Assistant to force a read of the heatpump sensors in the ebusd config.
   [hass_ebusd_mqtt_poll_specific.yaml](homeassistant_scripts/hass_ebusd_mqtt_poll_specific.yaml) This is a script to use in combination with Home Assistant to force a read of a specific sensor in the ebusd config.
   Use these scripts by copy-pasting there contents in a new script defined in Home Assistant. You can create these scripts in the Home Assistant UI by going to Settings -> Automations & Scenes -> Scripts -> Add script.

## Troubleshooting

If things are not working as expected, you can check the logs of the ebusd container by running the following command:
`docker logs ebusd` where ebusd is the name of the container used in the docker-compose file. You can find the name of the container by running `docker ps`.
The log should show you streaming data across the ebus, not all messages are decoded, but messages should be visible. If you see no data, check the connection of the eBus adapter and the configuration of the ebusd container.
If you see data, but no MQTT topics are created, check the configuration of the MQTT broker in the docker compose script. If the MQTT topics are created, but not visible in Home Assistant, check the configuration of the MQTT broker in Home Assistant.
You can use a tool like MQTT Explorer to check the MQTT topics and messages.

## Contributing

Feel free to submit issues or pull requests if you encounter any problems or have suggestions for improvements.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.
