# Homeassistant AOK Blinds

This repo contains 2 parts:
- The communication between an ESP8266 and a/several a AOK blinds, using RTS technology (RF 433MHz)
- The pairing of the homemade remote and the blind(s)
- The integration of those blinds to Homeassistant using MQTT. This integration allow positioning of the blinds in a specific position.


## ESP8266

### Hardware
You will need an ESP8266 and a RF emitter. 
- I'm using a NodeMCU for the ESP 
- I'm using a simple rf emitter like [the FS1000A](https://search.brave.com/images?q=rf+433+emitter+fs1000A&source=web)
- To make the emitter more powerfull, I used [this tutorial](https://www.instructables.com/433-MHz-Coil-loaded-antenna/) to create a home made antenna. Works like a charm

Just connect the Data pin of the RF emitter to the pin A4 (pin id 2), the ground to the ground and the Vcc to the 3V of the ESP.

### Software
The arduino code principle is the following : it receives an mqtt message with the channel corresponding to one/several blinds and with an action, and it send the corresponding signal to the RF emitter.

See the code of ****.ino.
- Adapt the information concerning the wifi connection
- Choose your own adress for your remote. It can be whatever between 1 and 2^16.
- You can adapt the topic the code lisens to, but if you do, it will have impact on homeassistant side. 


https://www.aokfrance.com/ressources/common/Notices/NOTICE_AM25_35_45-ES-E.pdf
##
