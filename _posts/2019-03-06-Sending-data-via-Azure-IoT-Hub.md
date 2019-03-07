---
title: Sending data via Azure IoT Hub
layout: post
author: Stefan Hudelmaier
---

Sending message to CENTERSIGHT via Azure IoT Hub is straight-forward. We are using
the MQTT command line client from the [Eclipse Mosquitto project](https://mosquitto.org/).
If you are using Ubuntu, they can be installed using

```
apt-get install mosquitto-clients
```

In this example, we are using Shared Access Signatures from IoT Hub. Using client
certificates is also possible and we will show how to retrieve and use them in a future 
blog post. 

We will send a temperature datapoint:

```
mosquitto_pub -q 1 --capath /etc/ssl/certs -p 8883 -V mqttv311 \
    -h ng-prod-iot-hub001.azure-devices.net \
    -m '{"temperature": 38.1}' \
    -i "urn:di:assets:example:1" \
    -u 'ng-prod-iot-hub001.azure-devices.net/urn:di:assets:example:1/api-version=2018-06-30' \
    -t "devices/urn:di:assets:example:1/messages/events/type=simple" \
    -P "SharedAccessSignature sr=xxx" 
```

In this example, the gateway URN - or Device ID in IoT Hub's terms - is `urn:di:assets:example:1` which
is both the client id (option `-i`), part of the username (option `-u`) and part of the topic (option `-t`).
The value of the password (option `-P`) must of course be replaced by the actual SAS. The message (option `-m`)
is a simple CENTERSIGHT JSON format, mapping datapoint keys to their values. 

After you have sent the message using the command above, the value will be visible in CENTERSIGHT. Keep
in mind, that the gateway must have been created beforehand using the UI or REST API.
