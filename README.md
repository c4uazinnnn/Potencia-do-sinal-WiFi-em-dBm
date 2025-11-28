# Potencia-do-sinal-WiFi-em-DBM

Link do drive: https://drive.google.com/file/d/1X2qJ-gIeUutHhs_PHoYuxY_ti2o9CAFW/view?usp=sharing

Codigo do projeto:

```
/*
  ESP32 - publica RSSI (dBm) via MQTT no Adafruit IO (feed: wifi-conect)
  Bibliotecas necessárias:
    - WiFi (já vem com o core ESP32)
    - PubSubClient (Library Manager: "PubSubClient" por Nick O'Leary)
*/

#include <WiFi.h>
#include <WiFiClient.h>
#include <PubSubClient.h>

// --------- CONFIGURAÇÕES DO WIFI ----------
const char* WIFI_SSID = "Inteli.Iot";
const char* WIFI_PASS = "%(Yk(sxGMtvFEs.3";

// --------- CONFIGURAÇÕES ADAFRUIT IO (MQTT) ----------
const char* AIO_SERVER   = "io.adafruit.com";
const uint16_t AIO_PORT  = 1883;
const char* AIO_USERNAME = "SEU_USERNAME_AQUI"; 
const char* AIO_KEY      = "aio_zoAY63U57fkuLm2WBC6B0uHe7USG";

const char* AIO_FEEDNAME = "wifi-conect"; 

WiFiClient wifiClient;
PubSubClient mqttClient(wifiClient);

unsigned long lastPublish = 0;
const unsigned long publishInterval = 5000; // ms — publica a cada 5s (ajuste se quiser)

/* helper: monta tópico de publish: <username>/feeds/<feedname> */
String aioFeedTopic() {
  String t = String(AIO_USERNAME) + "/feeds/" + String(AIO_FEEDNAME);
  return t;
}

void connectWiFi() {
  Serial.print("Conectando no WiFi ");
  Serial.print(WIFI_SSID);
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  while (WiFi.status() != WL_CONNECTED) {
    Serial.print('.');
    delay(500);
  }
  Serial.println();
  Serial.print("WiFi conectado. IP: ");
  Serial.println(WiFi.localIP());
}

void connectMQTT() {
  mqttClient.setServer(AIO_SERVER, AIO_PORT);

  while (!mqttClient.connected()) {
    Serial.print("Conectando ao Adafruit IO MQTT...");
    // clientId pode ser qualquer string única
    String clientId = "ESP32-RSSI-";
    clientId += String((uint32_t)ESP.getEfuseMac(), HEX); // ajuda a tornar único

    if (mqttClient.connect(clientId.c_str(), AIO_USERNAME, AIO_KEY)) {
      Serial.println("Conectado ao MQTT!");
      // (Opcional) poderia se inscrever em tópicos se quiser
    } else {
      Serial.print("Erro conectar MQTT, rc=");
      Serial.print(mqttClient.state());
      Serial.println(" tentando novamente em 2s");
      delay(2000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  delay(100);

  connectWiFi();
  connectMQTT();
}

void loop() {
  if (WiFi.status() != WL_CONNECTED) {
    Serial.println("WiFi desconectado. Reconectando...");
    connectWiFi();
  }

  if (!mqttClient.connected()) {
    Serial.println("MQTT desconectado. Reconectando...");
    connectMQTT();
  }

  mqttClient.loop();

  unsigned long now = millis();
  if (now - lastPublish >= publishInterval) {
    lastPublish = now;

    int rssi = WiFi.RSSI(); // valor em dBm (negativo geralmente)
    char payload[16];
    snprintf(payload, sizeof(payload), "%d", rssi);

    String topic = aioFeedTopic();

    Serial.print("Publicando RSSI (dBm) = ");
    Serial.print(rssi);
    Serial.print(" para tópico: ");
    Serial.println(topic);

    boolean ok = mqttClient.publish(topic.c_str(), payload);
    if (!ok) {
      Serial.println("Falha ao publicar!");
    } else {
      Serial.println("Publicado com sucesso.");
    }
  }

  // restante do loop...
}

```
