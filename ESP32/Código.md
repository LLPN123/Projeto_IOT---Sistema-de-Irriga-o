#include <Arduino.h>
#include <WiFi.h>
#include <PubSubClient.h>

#define PINO_SENSOR 34
#define NUMERO_AMOSTRAS 5   // agora são 5 amostras

// Wi-Fi
const char* ssid = "WIFI";
const char* password = "SENHA";

// MQTT
const char* mqtt_server = "broker.hivemq.com";
const char* mqtt_topic_normal = "umidade/lorenzo";
const char* mqtt_topic_critico = "umidade/lorenzocritico";

WiFiClient espClient;
PubSubClient client(espClient);

// calibração
int valor_seco = 4095;
int valor_molhado = 0;

// valor armazenado da última leitura
int ultimo_valor_umidade = -1;


int ler_umidade_solo() {

    int somatoria = 0;

    for (int i = 1; i <= NUMERO_AMOSTRAS; i++) {
        int leitura_sensor = analogRead(PINO_SENSOR);
        somatoria += leitura_sensor;
        delay(200);
    }

    float media = somatoria / (float)NUMERO_AMOSTRAS;

    int umidade_percentual = map(media, valor_seco, valor_molhado, 0, 100);
    return constrain(umidade_percentual, 0, 100);
}

void conectaWiFi() {
    Serial.print("Conectando ao Wi-Fi ");
    Serial.println(ssid);

    WiFi.begin(ssid, password);

    while (WiFi.status() != WL_CONNECTED) {
        delay(500);
        Serial.print(".");
    }

    Serial.println("\nWi-Fi conectado!");
    Serial.print("IP: ");
    Serial.println(WiFi.localIP());
}


void conectaMQTT() {
    while (!client.connected()) {
        Serial.println("Conectando ao broker MQTT...");

        if (client.connect("ESP32_TADEO")) {
            Serial.println("Conectado ao MQTT!");
        } else {
            Serial.print("Falhou, erro: ");
            Serial.println(client.state());
            delay(2000);
        }
    }
}

void setup() {
    Serial.begin(9600);
    delay(1000);

    conectaWiFi();
    client.setServer(mqtt_server, 1883);

    Serial.println("Realizando primeira leitura...");

    // primeira leitura armazenada
    ultimo_valor_umidade = ler_umidade_solo();

    Serial.printf("Umidade inicial: %d%%\n", ultimo_valor_umidade);

    // ENVIA A PRIMEIRA LEITURA AO MQTT
    conectaMQTT();
    char payload[10];
    sprintf(payload, "%d", ultimo_valor_umidade);
    client.publish(mqtt_topic_normal, payload);
    Serial.printf("Primeira leitura enviada ao tópico normal: %s\n", payload);

    // SE FOR CRÍTICO (< 40) também envia
    if (ultimo_valor_umidade < 40) {
        client.publish(mqtt_topic_critico, payload);
        Serial.printf("Primeira leitura enviada ao tópico crítico: %s\n\n", payload);
    }
}

void loop() {

    if (!client.connected()) {
        conectaMQTT();
    }

    client.loop();

    int umidade_atual = ler_umidade_solo();

    // Só envia se o valor mudou
    if (umidade_atual != ultimo_valor_umidade) {

        Serial.printf("Nova umidade detectada: %d%% (antes: %d%%)\n",
                      umidade_atual, ultimo_valor_umidade);

        // atualiza o valor
        ultimo_valor_umidade = umidade_atual;

        // Prepara payload
        char payload[10];
        sprintf(payload, "%d", umidade_atual);

        // Envio normal
        client.publish(mqtt_topic_normal, payload);
        Serial.printf("Enviado ao tópico normal: %s\n", payload);

        // Envio crítico (somente se < 40)
        if (umidade_atual < 40) {
            client.publish(mqtt_topic_critico, payload);
            Serial.printf("Enviado ao tópico CRÍTICO: %s\n", payload);
        }

        Serial.println();
    }

    delay(2000);
}
