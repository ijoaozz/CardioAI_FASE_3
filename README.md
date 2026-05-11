# 🌾 Projeto Cap 1 - CardioIA Conectada: IoT e Visualização de Dados para a Saúde Digital

---

## 👨‍🎓 Integrantes

| Nome Completo                   | RM       |
| ------------------------------- | -------- |
| Daniele Antonieta Garisto Dias  | RM565106 |
| Leandro Augusto Jardim da Cunha | RM561395 |
| Luiz Eduardo da Silva           | RM561701 |
| João Victor Viana de Sousa      | RM565136 |

---

## PARTE 1 – Armazenamento e Processamento Local (Edge Computing)

### 1.1 Visão Geral

A Parte 1 do projeto consiste em uma aplicação desenvolvida no simulador Wokwi utilizando o microcontrolador ESP32. O objetivo é demonstrar os princípios de Edge Computing aplicados à saúde digital, onde o dispositivo realiza coleta, processamento local e gerenciamento de resiliência offline de dados de sinais vitais.

### 1.2 Sensores Utilizados

Foram empregados dois sensores distintos no projeto:

**Sensor 1 – DHT22 (Temperatura e Umidade)**
Conectado ao pino GPIO 15, o DHT22 é responsável pela leitura periódica de temperatura (°C) e umidade relativa do ar (%). Por se tratar de um sensor dual, é contabilizado como um único sensor conforme especificação do enunciado. As leituras são validadas com a função `isnan()` para garantir a integridade dos dados antes de qualquer processamento.

**Sensor 2 – PIR (Sensor de Movimento)**
Conectado ao pino GPIO 4, o sensor PIR detecta presença no ambiente. No contexto do CardioIA, representa a detecção de movimento do paciente monitorado, sendo um parâmetro relevante em cenários de queda ou imobilidade prolongada.

### 1.3 Fluxo de Funcionamento

O sistema opera em ciclos de leitura a cada 5 segundos, controlados por temporização não bloqueante (`millis()`). A cada ciclo:

1. O ESP32 lê temperatura, umidade e estado do sensor PIR.
2. Os dados são serializados em formato JSON utilizando a biblioteca ArduinoJson.
3. O sistema verifica o estado de conectividade (variável booleana `isOnline`).
4. Dependendo do estado de conexão, os dados são enviados ou armazenados localmente.

O payload gerado segue o seguinte formato JSON:
```json
{"temperatura": 36.5, "umidade": 62.0, "movimento": false}
```

### 1.4 Lógica de Resiliência Offline (Edge Computing)

A resiliência offline é implementada por meio de um buffer local em memória RAM do ESP32, com capacidade para armazenar até **20 leituras** (estratégia definida conforme o modelo de negócio do projeto).

**Funcionamento do Buffer:**
- Quando o sistema está **offline**, cada nova leitura é salva no array `buffer[]` por meio da função `saveLocal()`.
- Quando o buffer atinge sua capacidade máxima, é aplicada uma política **FIFO** (First In, First Out): a leitura mais antiga é descartada e a mais recente ocupa o último slot. Isso garante que o ESP32 sempre mantenha os dados mais atualizados mesmo com espaço limitado.
- Quando a conexão é **restabelecida**, a função `syncData()` percorre todos os registros armazenados, exibe-os no Monitor Serial (simulando o envio para a nuvem) e limpa o buffer.

**Estratégia de capacidade:**
20 amostras a cada 5 segundos representam 100 segundos (~1,6 minutos) de operação offline sem perda de dados. Para o contexto de um dispositivo vestível cardíaco hospitalar, essa janela é adequada para cobrir perdas de sinal momentâneas em ambientes com interferência.

**Alternativa para SPIFFS:**
Como o simulador Wokwi não suporta persistência SPIFFS (o sistema de arquivos é volátil ao reiniciar a simulação), o Monitor Serial foi adotado como alternativa de saída, conforme orientação do enunciado. Em um chip ESP32 físico, os dados poderiam ser gravados em arquivo CSV via SPIFFS ou em cartão microSD.

### 1.5 Simulação de Conectividade

A conectividade é simulada de forma automática pela própria lógica do `loop()`: a variável `isOnline` alterna entre `true` e `false` a cada 15 segundos (dentro de um ciclo de 30 segundos usando `millis() % 30000`). Isso permite observar no Monitor Serial o comportamento completo do sistema tanto em modo online quanto offline, sem necessidade de intervenção manual.

### 1.6 Alertas Locais

Além do armazenamento, o sistema implementa regras de alerta processadas localmente:

- **Temperatura > 38°C** → exibe `"⚠️ ALERTA: FEBRE DETECTADA"` no Monitor Serial.
- **Movimento detectado pelo PIR** → exibe `"⚠️ ALERTA: PRESENÇA DETECTADA"`.

Esses alertas demonstram que o ESP32, como dispositivo de borda, é capaz de executar lógica de negócio sem depender de conexão com a nuvem.

---

## PARTE 2 – Transmissão para Nuvem e Visualização (Fog/Cloud Computing)

### 2.1 Visão Geral

A Parte 2 integra o ESP32 a uma infraestrutura completa de Fog e Cloud Computing, composta por broker MQTT (Mosquitto local), Node-RED (camada de fog), InfluxDB (banco de dados de séries temporais) e Grafana (visualização). O objetivo é demonstrar o fluxo completo de dados desde a captura no dispositivo até a exibição em dashboards com alertas automáticos.

### 2.2 Sensores Utilizados

**Sensor 1 – DHT22 (Temperatura)**
Conectado ao GPIO 15, realiza leitura de temperatura a cada 10 segundos. Os dados são publicados no tópico MQTT `sensor/temperatura` no formato JSON: `{"temperatura": 36.5}`.

**Sensor 2 – Botão (Simulação de Batimentos Cardíacos)**
Conectado ao GPIO 22 com `INPUT_PULLUP`, o botão simula os batimentos cardíacos do paciente. O usuário pressiona o botão repetidamente para representar batimentos, e o sistema conta os acionamentos ao longo de uma janela de 10 segundos. O cálculo de BPM é feito pela fórmula: `BPM = contagem × 6`. Um LED no GPIO 17 fornece feedback visual a cada batimento detectado. Os dados são publicados no tópico `sensor/batimentos` no formato: `{"bpm": 72}`.

### 2.3 Protocolo MQTT e Configuração do Broker

**MQTT (Message Queuing Telemetry Transport)** é um protocolo de mensageria leve baseado no padrão publish/subscribe, ideal para dispositivos IoT com recursos limitados. Suas principais características para este projeto são:

- **Baixo overhead**: headers pequenos, adequado para ESP32 com memória limitada.
- **Desacoplamento**: o ESP32 publica dados sem precisar saber quem os consome.
- **QoS configurável**: garante entrega mesmo em redes instáveis.

**Configuração utilizada:**
- Broker: Mosquitto rodando localmente (`192.168.0.120:1883`)
- Client ID: `esp32-fiap-01`
- Tópicos: `sensor/temperatura` e `sensor/batimentos`
- Biblioteca: PubSubClient

A função `reconnect_mqtt()` implementa reconexão automática em caso de queda de conexão com o broker, garantindo resiliência na camada de transmissão.

### 2.4 Fluxo Completo de Comunicação

O fluxo de dados segue a arquitetura em três camadas:

```
ESP32 → [MQTT] → Mosquitto Broker → Node-RED → InfluxDB → Grafana
```

**Detalhamento de cada etapa:**

**1. Edge (ESP32):**
- Coleta temperatura via DHT22 a cada 10 segundos.
- Conta batimentos via botão e calcula BPM a cada 10 segundos.
- Serializa os dados em JSON e publica via MQTT.

**2. Fog (Node-RED):**
- Subscreve os tópicos MQTT `sensor/temperatura` e `sensor/batimentos`.
- Faz parse do JSON recebido.
- Aplica regras de alerta:
  - Temperatura > 38°C → dispara alerta de febre.
  - BPM > 120 → dispara alerta de taquicardia.
- Exibe os dados em tempo real nos dashboards (gráficos de linha, medidores gauge e indicadores de alerta).
- Encaminha os dados para o InfluxDB via node HTTP/influxdb.

**3. Cloud (InfluxDB + Grafana):**
- O InfluxDB armazena os dados como séries temporais, com timestamps automáticos.
- O Grafana consome os dados do InfluxDB e exibe dashboards interativos com:
  - Gráfico histórico de temperatura.
  - Gráfico histórico de BPM.
  - Indicadores gauge de valores atuais.
  - Painéis de alerta visual.

### 2.5 Dashboard Node-RED

O dashboard foi configurado com os seguintes componentes:

- **Gráfico de linha (Chart):** exibe a variação dos batimentos cardíacos (BPM) ao longo do tempo em tempo real.
- **Medidor (Gauge):** exibe a temperatura atual com escala visual de 0°C a 50°C.
- **Indicador de alerta:** muda de cor e exibe mensagem quando temperatura > 38°C ou BPM > 120.

As evidências do funcionamento do dashboard estão registradas nas capturas de tela disponíveis na pasta `assets/` do projeto:
- `dashboard_batimento_alert.png` – alerta de taquicardia ativo.
- `dashboard_batimento_sucess.png` – batimentos dentro do limite normal.
- `dashboard_temperatura_alert.png` – alerta de febre ativo.
- `dashboard_temperatura_sucess.png` – temperatura dentro do limite normal.

### 2.6 Integração com Grafana e InfluxDB

Além do Node-RED, o projeto integra o **Grafana Cloud** com o **InfluxDB** para visualização avançada dos dados históricos. Essa integração permite:

- Consultas com linguagem Flux para análise de tendências.
- Criação de painéis customizados com múltiplos gráficos e alertas persistentes.
- Armazenamento de longo prazo dos sinais vitais coletados.

As capturas de tela das interfaces Grafana e InfluxDB estão registradas em `assets/grafana.png`, `assets/influxdb.png` e `assets/imagem_node_red.png`.

---

## 3. Arquitetura Geral do Sistema

O CardioIA Fase 3 implementa uma arquitetura IoT distribuída em três camadas bem definidas:

| Camada | Tecnologia | Função |
|--------|-----------|--------|
| Edge | ESP32 + DHT22 + PIR/Botão | Coleta, processamento local, resiliência offline |
| Fog | Node-RED + Mosquitto | Roteamento, regras de negócio, dashboard em tempo real |
| Cloud | InfluxDB + Grafana | Armazenamento histórico, análise e visualização avançada |

Essa divisão garante:
- **Baixa latência**: alertas críticos são processados localmente no ESP32.
- **Escalabilidade**: novos sensores podem ser adicionados sem alterar a camada de nuvem.
- **Resiliência**: o sistema continua operando mesmo sem conexão.
- **Rastreabilidade**: todos os dados ficam persistidos com timestamp no InfluxDB.

---

## 4. Considerações Finais

O projeto demonstrou de forma prática o ciclo completo de um sistema IoT aplicado à saúde digital: **captura → processamento → transmissão → visualização → alerta**. A simulação no Wokwi permitiu validar a lógica do firmware sem necessidade de hardware físico, enquanto o Node-RED, InfluxDB e Grafana formaram uma infraestrutura realista de monitoramento.

As tecnologias utilizadas — MQTT, Edge Computing com buffer FIFO, séries temporais e dashboards interativos — são amplamente aplicadas em soluções reais de saúde digital, evidenciando a aderência do projeto ao estado da arte do mercado de IoT médico.

---

*Relatório elaborado para a Fase 3 do projeto CardioIA – FIAP*
