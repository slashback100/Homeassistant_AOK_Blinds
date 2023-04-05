#include <ESP8266WiFi.h>
#include <ESP8266mDNS.h>
#include <WiFiUdp.h>
#include <ArduinoOTA.h>
#include "PubSubClient.h" // Allows us to connect to, and publish to the MQTT broker

const char* ssid = "*****";
const char* password = "*****";

const char* mqtt_server = "192.168.0.x";
const char* mqtt_username = "*****";
const char* mqtt_password = "*****";

#define HOSTNAME "BlindsControler"
#define TOPIC "blinds_etage1a"

#define CMD_TOPIC "cmd/" TOPIC "/"
#define CMD_TOPIC_LISTEN "cmd/" TOPIC "/#"
#define READY_TOPIC "init/" TOPIC "/ready"

#define INFO_TOPIC "log/info/" TOPIC
#define DEBUG_TOPIC "log/debug/" TOPIC
#define ERROR_TOPIC "log/error/" TOPIC

const char* clientID = TOPIC;

// do not change
#define UP               11
#define DOWN             67
#define AFTER_UP_DOWN    36
#define STOP             35
#define PROGRAM          83
#define CHANGE_DIRECTION 80 // Pressing STOP button for 5 seconds
#define START            163

// remote uniq id
#define REMOTE_ID  12891921

#define TRANSMIT_PIN             2      // We'll use digital 13 for transmitting
#define REPEAT_COMMAND           8       // How many times to repeat the same command: original remotes repeat 8 (multi) or 10 (single) times by default
#define DEBUG                    false   // Do note that if you add serial output during transmit, it will cause delay and commands may fail

// If you wish to use PORTB commands instead of digitalWrite, these are for Arduino Uno digital 13:
#define D13high | 0x20; 
#define D13low  & 0xDF; 

// Timings in microseconds (us). Get sample count by zooming all the way in to the waveform with Audacity.
// Calculate microseconds with: (samples / sample rate, usually 44100 or 48000) - ~15-20 to compensate for delayMicroseconds overhead.
// Sample counts listed below with a sample rate of 44100 Hz:
#define AOK_AGC1_PULSE                   5300  // 234 samples
#define AOK_AGC2_PULSE                   530   // 24 samples after the actual AGC bit
#define AOK_RADIO_SILENCE                5030  // 222 samples

#define AOK_PULSE_SHORT                  270   // 12 samples
#define AOK_PULSE_LONG                   565   // 25 samples, approx. 2 * AOK_PULSE_SHORT

#define AOK_COMMAND_BIT_ARRAY_SIZE       65    // Command bit count

char buffer[50];

WiFiClient wifiClient;
PubSubClient client(mqtt_server, 1883, wifiClient); // 1883 is the listener port for the Broker

void connect() {
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  wifiCheck();
  WiFi.setAutoReconnect(true);
  WiFi.persistent(true);
  mqtt();
}
void reconnect(){
  WiFi.reconnect();
  wifiCheck();
  mqtt();
}
void wifiCheck(){
  int tryIt = 30;
  while (WiFi.status() != WL_CONNECTED && tryIt > 0) {
    delay(500);
    tryIt--;
  }
  if(tryIt <= 0){
    ESP.restart();
  }
}
void mqtt(){
  if (!client.connect(clientID, mqtt_username, mqtt_password)) {
    ESP.restart();
  }
  
  client.setCallback(subscribeReceive);
  client.subscribe(CMD_TOPIC_LISTEN);
  
  client.loop();
  delay(100);
}

void setup() {
  
  Serial.begin(9600); // Used for error messages even with DEBUG set to false
      
  if (DEBUG) Serial.println("Starting up...");
  connect();
  client.publish(INFO_TOPIC, "connected");
  
  ArduinoOTA.setHostname(HOSTNAME);
  ArduinoOTA.onStart([]() {
    String type;
    if (ArduinoOTA.getCommand() == U_FLASH) {
      type = "sketch";
    } else { // U_FS
      type = "filesystem";
    }
  });

  ArduinoOTA.begin();
  
  client.publish(READY_TOPIC, "ready");
}
//char receivedTopic[50];
void subscribeReceive(char* topic, byte* payload, unsigned int length){
  // topic is CMD_TOPIC followed by 16 (sam1), 32 (sam2), 01 (cuisine1), 02 (cuisine2), 04 (cuisine3), 08 (cuisineentree)
  // message is u (up), d (down) or s (stop)
  // cmd/blinds_etage1a/{id}/{u,d,s,p}
  
  String receivedPayload = String((char) payload[0]);
  
  //strcpy(receivedTopic, topic);
  int idInt = (topic[19] - '0')*10 + (topic[20] - '0');
  client.publish(DEBUG_TOPIC, ("id : "+String(idInt)).c_str());
  
  if(receivedPayload.equals("d")){
     client.publish(DEBUG_TOPIC, "down");
     sendAOKCommand(REMOTE_ID, idInt, DOWN);
     sendAOKCommand(REMOTE_ID, idInt, AFTER_UP_DOWN);
  } else if(receivedPayload.equals("u")){
    client.publish(DEBUG_TOPIC, "up");
     sendAOKCommand(REMOTE_ID, idInt, UP);
     sendAOKCommand(REMOTE_ID, idInt, AFTER_UP_DOWN);
  } else if(receivedPayload.equals("s")){
     client.publish(DEBUG_TOPIC, "stop");
     sendAOKCommand(REMOTE_ID, idInt, STOP);
  } else if(receivedPayload.equals("p")){
     client.publish(DEBUG_TOPIC, "prog");
     sendAOKCommand(REMOTE_ID, idInt, PROGRAM);
  } else if(receivedPayload.equals("c")){
     client.publish(DEBUG_TOPIC, "change_dir");
     sendAOKCommand(REMOTE_ID, idInt, CHANGE_DIRECTION);
  }
  
  
  client.publish(DEBUG_TOPIC, ("Commande received and treated: "+receivedPayload).c_str());
}

void loop() {
  if(WiFi.status() != WL_CONNECTED){
    reconnect();
    client.publish(ERROR_TOPIC, "Mqtt reconnected.");
  }
  if(!client.connected()){
    mqtt();
  }
  doTheJob();
 
  ArduinoOTA.handle();
  client.loop();
}
void doTheJob(){
}

void sendAOKCommand(int remote, int address, int cmd) {
  //compute check sum
  int checkum = (((remote%256)%256) + ((remote/256)%256) + (remote/(256*256)) + (address%256) + (address/256) + cmd) % 256;
  String commandStr = intToBytes(START, 1) + intToBytes(remote, 3) + intToBytes(address, 2) + intToBytes(cmd, 1) + intToBytes(checkum, 1) + "1";
  
  char command [AOK_COMMAND_BIT_ARRAY_SIZE];
  commandStr.toCharArray(command, AOK_COMMAND_BIT_ARRAY_SIZE + 1);
  
  // Prepare for transmitting and check for validity
  pinMode(TRANSMIT_PIN, OUTPUT); // Prepare the digital pin for output
  /*if (strlen(command) < AOK_COMMAND_BIT_ARRAY_SIZE) {
    errorLog("sendAOKCommand(): Invalid command (too short), cannot continue.");
    return;
  }
  */
  if (strlen(command) > AOK_COMMAND_BIT_ARRAY_SIZE) {
    errorLog("sendAOKCommand(): Invalid command (too long), cannot continue.");
    return;
  }
  
  // Repeat the command:
  for (int i = 0; i < REPEAT_COMMAND; i++) {
    doAOKTribitSend(command);
  }

  // Disable output to transmitter to prevent interference with
  // other devices. Otherwise the transmitter will keep on transmitting,
  // disrupting most appliances operating on the 433.92MHz band:
  digitalWrite(TRANSMIT_PIN, LOW);
}
// ----------------------------------------------------------------------------------------------------------------------------------------------------------------

// ----------------------------------------------------------------------------------------------------------------------------------------------------------------
void doAOKTribitSend(char* command) {

  // Starting (AGC) bits:
  transmitHigh(AOK_AGC1_PULSE);
  transmitLow(AOK_AGC2_PULSE);

  // Transmit command:
  for (int i = 0; i < AOK_COMMAND_BIT_ARRAY_SIZE; i++) {
      // If current bit is 0, transmit HIGH-LOW-LOW (100):
      if (command[i] == '0') {
        transmitHigh(AOK_PULSE_SHORT);
        transmitLow(AOK_PULSE_LONG);
      }

      // If current bit is 1, transmit HIGH-HIGH-LOW (110):
      if (command[i] == '1') {
        transmitHigh(AOK_PULSE_LONG);
        transmitLow(AOK_PULSE_SHORT);
      }   
   }

  // Radio silence at the end.
  // It's better to go a bit over than under minimum required length:
  transmitLow(AOK_RADIO_SILENCE);
  
  if (DEBUG) {
    Serial.println();
    Serial.print("Transmitted ");
    Serial.print(AOK_COMMAND_BIT_ARRAY_SIZE);
    Serial.println(" bits.");
    Serial.println();
  }
}
// ----------------------------------------------------------------------------------------------------------------------------------------------------------------

// ----------------------------------------------------------------------------------------------------------------------------------------------------------------
void transmitHigh(int delay_microseconds) {
  digitalWrite(TRANSMIT_PIN, HIGH);
  //PORTB = PORTB D13high; // If you wish to use faster PORTB calls instead
  delayMicroseconds(delay_microseconds);
}
// ----------------------------------------------------------------------------------------------------------------------------------------------------------------

// ----------------------------------------------------------------------------------------------------------------------------------------------------------------
void transmitLow(int delay_microseconds) {
  digitalWrite(TRANSMIT_PIN, LOW);
  //PORTB = PORTB D13low; // If you wish to use faster PORTB calls instead
  delayMicroseconds(delay_microseconds);
}
// ----------------------------------------------------------------------------------------------------------------------------------------------------------------

// ----------------------------------------------------------------------------------------------------------------------------------------------------------------
void errorLog(String message) {
  Serial.println(message);
}
// ----------------------------------------------------------------------------------------------------------------------------------------------------------------
String intToBytes(int intVal, int nbOfBytes){
  String val = "" ;
  while(intVal > 0){
    if (intVal % 2 == 0){
       val = "0" + val;
    } else {
       val = "1" + val;
    }
    intVal = intVal / 2;
  }
  while (val.length() < nbOfBytes*8){
    val = "0" + val;
  }
  return val;
}
