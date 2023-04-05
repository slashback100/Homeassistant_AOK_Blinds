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
- Adapt the information concerning your mqtt broker
- Choose your own remote id for your remote. It can be whatever between 1 and 2^16.
- You can adapt the topic the code lisens to, but if you do, it will have impact on homeassistant side. 

The DIY remote we just created can handle up to 16 blinds. You can adapt the code to handle as many blinds as you want but then you will have to deal with several remote id.

## Pairing your new remote
We use the tutorial of the blinds to do that : [user manual](https://www.aokfrance.com/ressources/common/Notices/NOTICE_AM25_35_45-ES-E.pdf)

- Push STOP during 5 seconds on the remote alread working with your first blind. The blind will make a little movement
- Send an mqtt message: 
-- Topic: `cmd/blinds_etage1a/01` (`01` represents the channel)
-- Payload: `u` (for `up`)
- The blind will make a little movement
And thats is, your DIY remote can now control your blind on the channel 1.  You can send `u` on the same topic to open your blind, `d` to close it or `s` to stop it.

Do the same with your second blind, but with `02` instead of `01`.
Do the same with your third blind, but with `04`.
Do the same with your fourth blind, but with `08`.
Do the same with your fifth blind, but with `16`.
Do the same with your sixth blind, but with `32`.

Notice that you have to double the channel each time. Why? Each channel correspond, in a binary reprentation, to a series of `0` with a single `1`:
```
01 == 00000000 00000001
02 == 00000000 00000010
04 == 00000000 00000100
08 == 00000000 00001000
16 == 00000000 00010000
32 == 00000000 00100000
```
That way, if you want to open blinds with channels 01, 04 and 16, you can send 21 :
```
01 ==     00000000 00000001
04 ==     00000000 00000100
16 ==     00000000 00010000
-----------------------
1+4+16 -> 00000000 00010101 == 21  
```





##
