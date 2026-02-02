// Inclui biblioteca WiFi
#include "ESP8266WiFi.h"
// Inclui bibliotecas do DHT
#include "Adafruit_Sensor.h"
#include "DHT.h"
// Inclui bibliotecas de comunicação
#include "AsyncMqttClient.h"
#include "Ticker.h"

// ===== Wi-Fi (Hotspot do celular) =====
const char* WIFI_SSID = "Juju";
const char* WIFI_PASSWORD = "jujulinda";

// ===== Configurações DHT =====
#define DHTTYPE DHT11
#define DHTPIN 2              // D4 = GPIO2
DHT dht(DHTPIN, DHTTYPE);

// ===== ThingSpeak MQTT =====
#define MQTT_HOST "mqtt3.thingspeak.com"
#define MQTT_PORT 1883

#define MQTT_USERNAME  "mwa0000040165771"
#define MQTT_CLIENT_ID "ESP8266_DHT11"
#define MQTT_PASSWORD  "H5TEJ4Q1OZXKYONY"

// 👉 COLOQUE AQUI O ID DO SEU CANAL
#define CHANNEL_ID "XXXXXXX"

#define MQTT_PUB_TOPIC "channels/" CHANNEL_ID "/publish"

// ===== Variáveis =====
float temp;
float hum;

// Cliente MQTT
AsyncMqttClient mqttClient;
Ticker mqttReconnectTimer;

WiFiEventHandler wifiConnectHandler;
WiFiEventHandler wifiDisconnectHandler;
Ticker wifiReconnectTimer;

// Intervalo de leitura
unsigned long previousMillis = 0;
const long interval = 5000;

// ===== Wi-Fi =====
void connectToWifi() {
  Serial.println("Connecting to Wi-Fi...");
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
}

void onWifiConnect(const WiFiEventStationModeGotIP& event) {
  Serial.println("Connected to Wi-Fi.");
  Serial.print("IP do ESP: ");
  Serial.println(WiFi.localIP());
  connectToMqtt();
}

void onWifiDisconnect(const WiFiEventStationModeDisconnected& event) {
  Serial.println("Disconnected from Wi-Fi.");
  mqttReconnectTimer.detach();
  wifiReconnectTimer.once(2, connectToWifi);
}

// ===== MQTT =====
void connectToMqtt() {
  Serial.println("Connecting to ThingSpeak MQTT...");
  mqttClient.connect();
}

void onMqttConnect(bool sessionPresent) {
  Serial.println("Connected to MQTT (ThingSpeak).");
}

void onMqttDisconnect(AsyncMqttClientDisconnectReason reason) {
  Serial.println("Disconnected from MQTT.");
  if (WiFi.isConnected()) {
    mqttReconnectTimer.once(2, connectToMqtt);
  }
}

void onMqttPublish(uint16_t packetId) {
  Serial.print("Publish OK | PacketId: ");
  Serial.println(packetId);
}

// ===== Setup =====
void setup() {
  Serial.begin(115200);
  Serial.println();

  dht.begin();

  wifiConnectHandler = WiFi.onStationModeGotIP(onWifiConnect);
  wifiDisconnectHandler = WiFi.onStationModeDisconnected(onWifiDisconnect);

  mqttClient.onConnect(onMqttConnect);
  mqttClient.onDisconnect(onMqttDisconnect);
  mqttClient.onPublish(onMqttPublish);

  mqttClient.setServer(MQTT_HOST, MQTT_PORT);
  mqttClient.setCredentials(MQTT_USERNAME, MQTT_PASSWORD);
  mqttClient.setClientId(MQTT_CLIENT_ID);

  connectToWifi();
}

// ===== Loop =====
void loop() {
  unsigned long currentMillis = millis();

  if (currentMillis - previousMillis >= interval) {
    previousMillis = currentMillis;

    hum = dht.readHumidity();
    temp = dht.readTemperature();

    if (isnan(temp) || isnan(hum)) {
      Serial.println("Erro na leitura do DHT!");
      return;
    }

    // Payload exigido pelo ThingSpeak
    String payload = "field1=" + String(temp) + "&field2=" + String(hum);

    mqttClient.publish(MQTT_PUB_TOPIC, 0, false, payload.c_str());

    Serial.println("Dados enviados ao ThingSpeak:");
    Serial.print("Temperatura: ");
    Serial.println(temp);
    Serial.print("Umidade: ");
    Serial.println(hum);
  }
}
