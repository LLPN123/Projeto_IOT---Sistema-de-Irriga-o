# Projeto_IOT---Sistema-de-Irriga-o

Beleza, Tadeo — vou te explicar **passo a passo como montar e entender esse código**, como se fosse um tutorial “faça você mesmo”. Vou cobrir **hardware**, **bibliotecas**, **ligação**, **cada bloco do código**, **calibração**, **testes** e **problemas comuns**. Vamos lá!

---
**Objetivo do projeto**

Ler a umidade do solo com um ESP32, calcular uma média com 5 amostras, armazenar a última leitura e publicar via MQTT em **dois tópicos**:

* `umidade/lorenzo` (sempre que houver mudança)
* `umidade/lorenzocritico` (quando mudar **e** for < 40%)

---

**Materiais necessários**

* Placa ESP32 DevKit
* Sensor de umidade do solo (capacitativo recomendado)
* Cabos jumper, protoboard
* Acesso a uma rede Wi-Fi (SSID + senha)
* Broker MQTT (ex.: `broker.hivemq.com` público ou seu Mosquitto local)

---

**Bibliotecas e configurações antes de compilar**

* No Arduino IDE:instalar **PubSubClient** (Nick O’Leary).
* Se usar WiFi e ESP32, instale o pacote ESP32 Boards via Boards Manager e selecione **ESP32 Dev Module** em Tools > Board.
* Configure baud (9600 no código) no Serial Monitor.

# 6) Linha a linha


```cpp
#include <Arduino.h>
#include <WiFi.h>
#include <PubSubClient.h>

#define PINO_SENSOR 34
#define NUMERO_AMOSTRAS 5
```

* `WiFi.h` e `PubSubClient.h` são para conectar no Wi-Fi e no broker MQTT.
* `PINO_SENSOR` define onde está conectado o sensor (GPIO34).
* `NUMERO_AMOSTRAS` = quantas leituras serão feitas para tirar média (5).

---


### Função de leitura (média das amostras)

```cpp
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
```

* Lê `NUMERO_AMOSTRAS` vezes (com `delay(200)` entre elas), soma e calcula a média.
* `map(media, valor_seco, valor_molhado, 0, 100)` inverte a escala (úmido -> 100, seco -> 0) se `valor_seco` for maior que `valor_molhado`.
* `constrain` garante que fique entre 0 e 100.

---

### Conectar Wi-Fi e MQTT

```cpp
void conectaWiFi() { ... }
void conectaMQTT() { ... }
```

* `conectaWiFi()` tenta conectar até obter `WL_CONNECTED`.
* `conectaMQTT()` tenta conectar até `client.connected()` ser true; usa `client.connect("ESP32_TADEO")`.

Dica: substitua `"ESP32_TADEO"` por um client id único (ex.: `ESP32_TADEO_1234`) para evitar colisão no broker.

---

### setup()

```cpp
void setup() {
    Serial.begin(9600);
    delay(1000);
    conectaWiFi();
    client.setServer(mqtt_server, 1883);
    ultimo_valor_umidade = ler_umidade_solo();
    Serial.printf("Umidade inicial: %d%%\n", ultimo_valor_umidade);
    conectaMQTT();
    char payload[10]; sprintf(payload, "%d", ultimo_valor_umidade);
    client.publish(mqtt_topic_normal, payload);
    if (ultimo_valor_umidade < 40) client.publish(mqtt_topic_critico, payload);
}
```

* Inicializa Serial, Wi-Fi e define o servidor MQTT.
* Faz a **primeira leitura** (com 5 amostras), imprime e **envia** ao tópico normal. Se <40 envia também ao tópico crítico.

---

### loop()

```cpp
void loop() {
    if (!client.connected()) conectaMQTT();
    client.loop();
    int umidade_atual = ler_umidade_solo();
    if (umidade_atual != ultimo_valor_umidade) {
        // print, atualizar ultimo_valor_umidade, publicar normal e possivelmente crítico
    }
    delay(2000);
}
```

* Mantém a conexão MQTT com `client.loop()` (necessário para o PubSubClient).
* Faz nova média com 5 amostras.
* **Somente** quando o valor mudou (`!=`) ele:

  * imprime o novo/antigo,
  * atualiza `ultimo_valor_umidade`,
  * publica em `umidade/lorenzo`,
  * e se < 40 publica também em `umidade/lorenzocritico`.

---


# Testes práticos (antes e depois)

* **Testar MQTT**: num PC, rode:

  ```bash
  mosquitto_sub -h broker.hivemq.com -t umidade/lorenzo
  mosquitto_sub -h broker.hivemq.com -t umidade/lorenzocritico
  ```

  Você verá as mensagens publicadas pelo ESP32 quando ocorrerem.

* **Testar reconnect**: desligue Wi-Fi do roteador e volte; código tenta reconectar no loop.

