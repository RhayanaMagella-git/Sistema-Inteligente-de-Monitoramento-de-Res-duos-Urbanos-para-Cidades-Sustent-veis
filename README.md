# Sistema Inteligente de Monitoramento de Resíduos Urbanos para Cidades Sustentáveis

**Protocolo MQTT com ESP32 e Arduino UNO** · **ODS 11 – Cidades Sustentáveis**

Faculdade de Computação e Informática – Universidade Presbiteriana Mackenzie (UPM) · 2025

---

## i) Funcionamento e Como Reproduzir o Projeto

### Visão Geral do Sistema

O sistema monitora o nível de resíduos em lixeiras urbanas utilizando um sensor ultrassônico HC-SR04 instalado na parte superior da lixeira. Quando o nível de resíduos atinge 90% da capacidade, alertas visuais (LED) e sonoros (buzzer) são acionados localmente e os dados são transmitidos via protocolo MQTT para um broker na nuvem (HiveMQ), podendo ser monitorados remotamente em tempo real.

```
HC-SR04 → Arduino UNO → [Serial UART] → ESP32 → [WiFi TCP/IP] → HiveMQ → MQTT Explorer
```

### Fluxo de Funcionamento

1. O sensor HC-SR04 mede a distância entre o topo da lixeira e o nível dos resíduos a cada 2 segundos
2. O Arduino UNO converte a distância em percentual de preenchimento (0% a 100%)
3. Se o nível >= 90%, o LED vermelho e o buzzer são acionados automaticamente
4. Os dados são serializados em JSON e enviados ao ESP32 via comunicação Serial (UART)
5. O ESP32 recebe o JSON, valida a estrutura e publica no broker HiveMQ via WiFi e protocolo MQTT
6. O monitoramento remoto é feito pelo MQTT Explorer conectado ao broker

### Requisitos de Software

| Ferramenta | Uso |
|---|---|
| Arduino IDE 2.x | Gravação dos firmwares |
| NewPing (by Tim Eckel) | Biblioteca Arduino para HC-SR04 |
| PubSubClient (by Nick O'Leary) | Biblioteca ESP32 para MQTT |
| ArduinoJson v6.x (by Benoit Blanchon) | Biblioteca ESP32 para JSON |
| Suporte ESP32 na Arduino IDE | Via Boards Manager |
| Wokwi (https://wokwi.com) | Simulação sem hardware físico |
| MQTT Explorer (https://mqtt-explorer.com) | Visualização das mensagens |

### Passo a Passo para Reproduzir

1. Instalar a Arduino IDE e adicionar suporte ao ESP32
2. Instalar as três bibliotecas via **Tools → Manage Libraries**
3. Montar o circuito conforme esquema de ligação (seção iii)
4. Gravar `arduino_lixeira.ino` no Arduino UNO (Board: Arduino Uno, 9600 baud)
5. Editar o SSID e senha WiFi no `esp32_mqtt.ino`
6. Gravar `esp32_mqtt.ino` no ESP32 (Board: ESP32 Dev Module, 115200 baud)
7. Conectar Arduino e ESP32 via jumpers conforme esquema
8. Abrir MQTT Explorer, conectar em `broker.hivemq.com:1883` e assinar `cidade/lixeira/01/nivel`

---

## ii) Software Desenvolvido e Documentação de Código

### Estrutura de Arquivos

```
projeto_residuos/
├── arduino_lixeira/
│   └── arduino_lixeira.ino    ← Firmware do Arduino UNO
├── esp32_mqtt/
│   └── esp32_mqtt.ino         ← Firmware do ESP32
├── esp32_wokwi/
│   └── esp32_wokwi.ino        ← Versão para simulação no Wokwi
└── README.md
```

### Arquivo 1: `arduino_lixeira.ino`

Responsável pela leitura do sensor HC-SR04, cálculo do nível de preenchimento, controle dos atuadores e envio dos dados via Serial para o ESP32.

| Função | Descrição |
|---|---|
| `setup()` | Inicializa Serial (9600 baud), configura pinos de saída |
| `loop()` | Executa leitura a cada 2 segundos via `millis()` |
| `calcularNivel(dist)` | Converte distância (cm) em percentual de preenchimento |
| `enviarJSON(...)` | Serializa dados em JSON e envia via Serial para o ESP32 |

**Trecho principal:**

```cpp
#include <NewPing.h>

#define PINO_TRIG          9   // Trigger do HC-SR04
#define PINO_ECHO         10   // Echo do HC-SR04
#define PINO_LED           7   // LED de alerta
#define PINO_BUZZER        8   // Buzzer de alerta
#define ALTURA_LIXEIRA_CM 30   // Altura interna da lixeira (cm)
#define LIMIAR_ALERTA     90   // Percentual para acionar alertas

// Cálculo do nível de preenchimento:
int nivel = (ALTURA_LIXEIRA_CM - distCM) / ALTURA_LIXEIRA_CM * 100;

// Acionamento dos alertas:
if (nivelPercent >= LIMIAR_ALERTA) {
  digitalWrite(PINO_LED,    HIGH);
  digitalWrite(PINO_BUZZER, HIGH);
}
```

### Arquivo 2: `esp32_mqtt.ino`

Responsável pela conexão WiFi, recepção dos dados do Arduino via Serial2 e publicação no broker HiveMQ via protocolo MQTT.

| Função | Descrição |
|---|---|
| `setup()` | Inicializa Serial2 (RX=GPIO16), conecta WiFi e broker MQTT |
| `loop()` | Mantém conexão MQTT ativa e lê dados da Serial2 |
| `processarEPublicar()` | Valida JSON com ArduinoJson e publica no tópico MQTT |
| `conectarWiFi()` | Conecta ao WiFi configurado, bloqueia até conseguir |
| `conectarMQTT()` | Conecta ao broker HiveMQ com reconexão automática |
| `piscarLED()` | Pisca LED interno ao publicar mensagem com sucesso |

**Trecho principal:**

```cpp
const char* MQTT_SERVER = "broker.hivemq.com";
const int   MQTT_PORT   = 1883;
const char* MQTT_TOPICO = "cidade/lixeira/01/nivel";

// Recepção Serial do Arduino:
while (Serial2.available()) {
  char c = Serial2.read();
  if (c == '\n') { processarEPublicar(bufferSerial); }
  else { bufferSerial += c; }
}

// Publicação MQTT:
mqttClient.publish(MQTT_TOPICO, jsonStr.c_str(), false);
```

---

## iii) Descrição do Hardware Utilizado

### Plataformas de Desenvolvimento

| Componente | Especificação | Função no Projeto |
|---|---|---|
| Arduino UNO R3 | ATmega328P, 5V, 16MHz | Leitura do sensor, controle dos atuadores, envio Serial |
| ESP32 Dev Module | Xtensa LX6, 3.3V, WiFi 802.11 b/g/n | Conectividade WiFi, cliente MQTT, gateway IoT |

### Sensores e Atuadores

| Componente | Qtd. | Conexão | Função |
|---|---|---|---|
| Sensor HC-SR04 | 1 | Trig: D9 / Echo: D10 | Mede distância para calcular nível de resíduos |
| LED Vermelho 5mm | 1 | D7 (com resistor 220Ω) | Alerta visual quando nível >= 90% |
| Buzzer Ativo 5V | 1 | D8 | Alerta sonoro quando nível >= 90% |
| Resistor 220Ω | 1 | Série com LED | Proteção do LED contra sobrecorrente |
| Protoboard | 1 | — | Montagem dos circuitos |
| Jumpers | ~15 | — | Conexões entre componentes |

### Especificações do Sensor HC-SR04

- Tensão de operação: 5V DC
- Frequência de operação: 40 kHz
- Alcance de medição: 2 cm a 400 cm
- Ângulo de abertura: 15 graus
- Precisão: ±3 mm
- Pinos: VCC, Trig, Echo, GND

### Esquema de Ligação

**Arduino UNO → Sensores e Atuadores:**

```
HC-SR04  VCC  → 5V (Arduino)
HC-SR04  GND  → GND (Arduino)
HC-SR04  Trig → Pino Digital 9
HC-SR04  Echo → Pino Digital 10
LED Anodo     → Resistor 220Ω → Pino Digital 7
LED Catodo    → GND
Buzzer (+)    → Pino Digital 8
Buzzer (-)    → GND
```

**Arduino UNO ↔ ESP32 (comunicação Serial):**

```
Arduino TX (Pino 1) → [Divisor 5V→3.3V] → ESP32 GPIO16 (RX2)
Arduino GND         → ESP32 GND
```

**ATENÇÃO:** O ESP32 opera em 3.3V. O TX do Arduino (5V) requer divisor de tensão resistivo (10kΩ/20kΩ) para proteger o pino RX do ESP32.

### Simulação (sem hardware físico)

Os testes foram realizados no simulador **Wokwi** (https://wokwi.com), plataforma online de simulação compatível com Arduino e ESP32, amplamente utilizada em projetos acadêmicos. O simulador permite validar o funcionamento completo do código, sensores, atuadores e comunicação MQTT sem necessidade de hardware físico.

---

## iv) Documentação das Interfaces, Protocolos e Módulos de Comunicação

### Interface 1: Serial UART (Arduino ↔ ESP32)

| Parâmetro | Valor |
|---|---|
| Protocolo | UART (Universal Asynchronous Receiver-Transmitter) |
| Baud Rate | 9600 bps |
| Formato | 8N1 (8 bits dados, sem paridade, 1 stop bit) |
| Interface física | TX Arduino (Pino 1) → RX2 ESP32 (GPIO 16) |
| Nível lógico | 5V (Arduino) convertido para 3.3V (ESP32) |
| Terminador | Caractere newline (`\n`) indica fim de mensagem |
| Intervalo | A cada 2000ms (`INTERVALO_LEITURA_MS`) |

### Interface 2: MQTT sobre TCP/IP (ESP32 ↔ HiveMQ)

| Parâmetro | Valor |
|---|---|
| Protocolo | MQTT 3.1.1 (Message Queuing Telemetry Transport) |
| Transporte | TCP/IP sobre WiFi 802.11 b/g/n |
| Broker | broker.hivemq.com (público, sem autenticação) |
| Porta | 1883 (TCP padrão MQTT) |
| Tópico de publicação | `cidade/lixeira/01/nivel` |
| QoS | 0 (At most once) |
| Retain | false |
| Keep Alive | 60 segundos |
| Client ID | `esp32_residuos_01` (único por dispositivo) |
| Reconexão | Automática a cada 5 segundos em caso de queda |

### Formato da Mensagem MQTT

```json
{
  "id": "lixeira_01",
  "nivel": 90,
  "distancia_cm": 3,
  "alerta_visual": true,
  "alerta_sonoro": true
}
```

| Campo | Tipo | Descrição |
|---|---|---|
| `id` | string | Identificador único da lixeira |
| `nivel` | int (0–100) | Percentual de preenchimento da lixeira |
| `distancia_cm` | int | Distância bruta medida pelo HC-SR04 em cm |
| `alerta_visual` | boolean | `true` quando nível >= 90% (LED acionado) |
| `alerta_sonoro` | boolean | `true` quando nível >= 90% (Buzzer acionado) |

### Estrutura de Tópicos MQTT

```
cidade/
└── lixeira/
    ├── 01/
    │   └── nivel    ← Publica nível da lixeira 01
    ├── 02/
    │   └── nivel    ← Publica nível da lixeira 02
    └── +/nivel      ← Wildcard: assina TODAS as lixeiras
```

### Bibliotecas de Comunicação Utilizadas

| Biblioteca | Plataforma | Função |
|---|---|---|
| NewPing v1.9.x | Arduino | Interface com sensor HC-SR04, filtro de medições |
| PubSubClient v2.8 | ESP32 | Cliente MQTT para publicação e assinatura de tópicos |
| ArduinoJson v6.x | ESP32 | Serialização e validação de mensagens JSON |
| WiFi.h (built-in) | ESP32 | Gerenciamento de conexão WiFi 802.11 |

---

## v) Comunicação via Internet (TCP/IP) e Protocolo MQTT

### Comunicação TCP/IP

O ESP32 estabelece comunicação com a internet via protocolo WiFi 802.11, obtendo endereço IP dinâmico via DHCP da rede local. Toda a comunicação com o broker MQTT utiliza o protocolo TCP/IP, garantindo entrega confiável das mensagens.

### Modelo Publish/Subscribe MQTT

```
┌─────────┐   publica    ┌──────────┐   entrega    ┌───────────────┐
│  ESP32  │ ──────────► │  HiveMQ  │ ──────────► │ MQTT Explorer │
│(Publisher)│            │ (Broker) │            │ (Subscriber)  │
└─────────┘             └──────────┘             └───────────────┘
```

- **ESP32** atua como **PUBLISHER**: publica dados do sensor no broker
- **HiveMQ** atua como **BROKER**: intermedia a comunicação
- **MQTT Explorer** atua como **SUBSCRIBER**: recebe e exibe os dados em tempo real

### Resultado dos Testes

| N. | Sensor/Atuador | Tempo de Resposta | Resultado |
|---|---|---|---|
| 1 | HC-SR04 (nível 20%, dist. 24cm) | 2 segundos | JSON publicado com sucesso no HiveMQ |
| 2 | HC-SR04 (nível 30%, dist. 21cm) | 2 segundos | JSON publicado com sucesso no HiveMQ |
| 3 | HC-SR04 (nível 40%, dist. 18cm) | 2 segundos | JSON publicado com sucesso no HiveMQ |
| 4 | HC-SR04 + LED + Buzzer (nível 90%, dist. 3cm) | 2 segundos | Alertas acionados + `alerta_visual: true` |
| 5 | HC-SR04 + LED + Buzzer (nível 100%, dist. 0cm) | 2 segundos | Alertas acionados + `alerta_sonoro: true` |

### Evidências do Funcionamento

**Serial Monitor do ESP32 (Wokwi):**

```
[BOOT] ESP32 – Monitoramento de Residuos (Wokwi)
[WiFi] Conectando a 'Wokwi-GUEST'......
[WiFi] Conectado! IP: 10.10.0.2
[MQTT] Conectando a broker.hivemq.com:1883 ... OK!
[SIMULADO] {"id":"lixeira_01","nivel":20,"distancia_cm":24,"alerta_visual":false,"alerta_sonoro":false}
[MQTT] Publicado em 'cidade/lixeira/01/nivel' ✓
[SIMULADO] {"id":"lixeira_01","nivel":90,"distancia_cm":3,"alerta_visual":true,"alerta_sonoro":true}
[MQTT] Publicado em 'cidade/lixeira/01/nivel' ✓
```

**MQTT Explorer (broker.hivemq.com):**

```
broker.hivemq.com
└── cidade
    └── lixeira
        └── 01
            └── nivel = {"id":"lixeira_01","nivel":90,"distancia_cm":3,"alerta_visual":true,"alerta_sonoro":true}
```

