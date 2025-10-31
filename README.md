# Documentação: Integração NodeMCU (ESP8266), Sinric Pro e Alexa

Este documento explica como funciona a comunicação entre um dispositivo NodeMCU (ESP8266), a plataforma de IoT Sinric Pro e a assistente de voz Alexa, permitindo o controle de um relé (ou qualquer dispositivo) por voz, aplicativo e um botão físico.

## Visão Geral dos Componentes

Para que a mágica aconteça, três sistemas principais trabalham juntos:

1.  **NodeMCU (O Hardware):** É o cérebro físico do projeto. Este microcontrolador com Wi-Fi (ESP8266) está conectado ao seu relé e ao botão físico. Ele executa o código (`.ino`) que você forneceu. Sua única "visão" do mundo exterior é a sua conexão Wi-Fi.

2.  **Sinric Pro (O Intermediário / A Ponte):** É uma plataforma de nuvem (Cloud Service) para IoT. Ele atua como o principal intermediário:

    - Ele "autentica" seu dispositivo (usando `APP_KEY` e `APP_SECRET`).
    - Ele "expõe" seu dispositivo para outros serviços, como a Alexa.
    - Ele mantém uma conexão constante (via WebSocket) com o seu NodeMCU para enviar comandos (Ex: "Ligar") e receber atualizações de estado (Ex: "Fui ligado manualmente").

3.  **Alexa (A Interface de Voz):** É a interface do usuário (voz). Quando você diz "Alexa, ligue a luz", o sistema Alexa não sabe onde está o seu NodeMCU. Em vez disso:
    - O Amazon Echo processa sua voz e a envia para a nuvem da Amazon (AVS).
    - A nuvem da Amazon identifica que o dispositivo "luz" está vinculado à **Skill Sinric Pro**.
    - O serviço da Amazon (Alexa) envia um comando padronizado para a nuvem do **Sinric Pro**.

**Importante:** O NodeMCU e a Alexa **nunca** se comunicam diretamente. Ambos se comunicam apenas com o Sinric Pro, que atua como o tradutor e roteador central.

## Protocolos e Autenticação: O "Como" Funciona

Antes de detalhar o fluxo de dados, é crucial entender _como_ esses três componentes estabelecem confiança e conversam entre si.

### 1. Conexão Alexa <-> Sinric Pro (OAuth 2.0 e API Smart Home)

Esta é a "ligação de conta" (Account Linking) que você faz uma única vez no aplicativo Alexa.

- **Autenticação (OAuth 2.0):** Quando você ativa a Skill "Sinric Pro" no app Alexa, um fluxo de autenticação padrão da indústria chamado **OAuth 2.0** é iniciado.

  1.  O app Alexa redireciona você para a página de login do Sinric Pro.
  2.  Você faz login com sua conta Sinric Pro e "autoriza" a Amazon a acessar seus dispositivos.
  3.  O Sinric Pro envia de volta para a Amazon um "token de acesso" seguro.
  4.  A partir desse momento, a nuvem da Amazon (Alexa) armazena esse token e o utiliza para provar quem você é em todas as futuras comunicações com o Sinric Pro. Ela pode, efetivamente, enviar comandos _em seu nome_.

- **Comunicação (API REST/HTTPs):** Após a autenticação, a comunicação entre as duas nuvens (Amazon e Sinric) não é permanente. Ela é baseada em requisições HTTPs (similares a uma API REST).
  - **Alexa -> Sinric:** Quando você dá um comando de voz, a nuvem Alexa envia uma "diretiva" (um comando em formato JSON) para um _endpoint_ de API específico do Sinric Pro.
  - **Sinric -> Alexa:** Quando o estado muda fisicamente (via botão), o Sinric Pro envia um "evento" (também um JSON) para a API de eventos da Amazon, informando a mudança (`StateReport`).

### 2. Conexão Sinric Pro <-> NodeMCU (WebSockets)

Esta é a conexão mais crítica e em tempo real do projeto.

- **Por que WebSockets?** O protocolo HTTP padrão é "request-response" (o cliente pede, o servidor responde). O servidor (Sinric Pro) não pode _iniciar_ uma conversa com o cliente (NodeMCU).
- **A Solução:** O **WebSocket** estabelece uma conexão persistente e bidirecional (full-duplex) entre o NodeMCU e o servidor Sinric Pro.
  1.  **Início:** Quando seu NodeMCU executa `SinricPro.begin()`, ele se conecta ao servidor WebSocket do Sinric Pro e se autentica usando sua `APP_KEY` e `APP_SECRET`.
  2.  **Manutenção:** A função `SinricPro.handle()` no seu `loop()` faz o trabalho pesado de manter essa conexão viva. Ela envia "pings" de _keep-alive_ e, o mais importante, fica "ouvindo" por quaisquer mensagens que o servidor envie.
- **O Resultado:** Como a conexão está sempre aberta, no instante em que o Sinric Pro recebe um comando da Alexa (via HTTP), ele pode _imediatamente_ "empurrar" (push) esse comando pela conexão WebSocket diretamente para o NodeMCU. Isso garante a resposta quase instantânea.

### 3. O Papel das Funções de Callback (Programação Assíncrona)

Os _callbacks_ são o coração da programação reativa e eficiente no seu NodeMCU.

- **O que é?** Um callback é uma função que você "passa" para outra função, para que ela seja _chamada de volta_ (executada) no futuro, quando um evento específico ocorrer.
- **A Analogia:** A linha `mySwitch.onPowerState(onPowerState);` não executa a função `onPowerState` agora. Em vez disso, ela diz ao objeto `SinricPro`:
  > "Ei, aqui está o 'endereço' da minha função chamada `onPowerState`. Por favor, guarde-o. QUANDO (e se) você receber uma mensagem de `PowerState` pela conexão WebSocket, EXECUTE esta função para mim e me passe os detalhes (o deviceId e o novo estado)."
- **Por que usar?** Isso é programação **assíncrona**. O seu `loop()` principal não fica "parado" esperando um comando. Ele continua executando livremente (verificando o botão, etc.). Quando um comando chega, a biblioteca `SinricPro.handle()` detecta, pausa o loop momentaneamente, executa o callback `onPowerState`, e então devolve o controle ao loop. É um modelo muito eficiente que não desperdiça ciclos de processamento.

## Fluxo de Comunicação: Passo a Passo

Existem dois fluxos principais neste projeto: o comando vindo da Alexa (remoto) e o comando vindo do botão (local).

### Fluxo 1: Comando de Voz (Alexa -> NodeMCU)

Este é o fluxo quando você dá um comando de voz.

1.  **Usuário:** "Alexa, ligar o relé."
2.  **Amazon Echo:** Captura o áudio e o envia para a nuvem da Amazon (AVS).
3.  **Nuvem Alexa:** Processa o áudio, entende a intenção ("ligar") e o dispositivo ("relé").
4.  **Skill Sinric Pro:** A nuvem Alexa vê que o dispositivo "relé" (que você configurou no app Alexa) pertence à Skill do Sinric Pro.
5.  **Nuvem Alexa -> Nuvem Sinric Pro:** A Amazon envia uma diretiva (um comando JSON via API HTTPs, autenticado com o token OAuth 2.0) para os servidores do Sinric Pro, dizendo: "O usuário X quer LIGAR o dispositivo com ID `69014270359ccc32ce17e5e3`".
6.  **Nuvem Sinric Pro -> NodeMCU:**
    - O Sinric Pro recebe o comando da API.
    - Ele localiza a conexão **WebSocket** ativa associada a esse dispositivo.
    - O Sinric Pro envia uma mensagem (via WebSocket) pela internet diretamente para o seu NodeMCU.
7.  **NodeMCU (Seu Código):**
    - A função `SinricPro.handle()` dentro do `loop()` está constantemente ouvindo essa conexão WebSocket.
    - Ela recebe a mensagem de "LIGAR".
    - A biblioteca SinricPro identifica que é um comando `onPowerState` e chama a função de **callback** que você registrou: `onPowerState()`.
8.  **Execução do Callback `onPowerState()`:**
    - A função `onPowerState` é executada com o parâmetro `state` sendo `true` (ligado).
    - `myPowerState = state;` (a variável global agora é `true`).
    - `digitalWrite(RELE_PIN, myPowerState?LOW:HIGH);` -> Isso é avaliado como `digitalWrite(RELE_PIN, LOW);`. O relé é ativado.
    - A função imprime no Serial: `Device ... turned on (via SinricPro)`.
    - A função retorna `true`, sinalizando ao Sinric Pro que o comando foi executado com sucesso.
9.  **Confirmação (Feedback):** O Sinric Pro envia uma resposta de sucesso para a nuvem Alexa (via HTTPs), que então faz o seu Amazon Echo dizer "OK" ou emitir um som de confirmação.

### Fluxo 2: Acionamento Físico (NodeMCU -> Alexa)

Este fluxo é crucial para manter o estado sincronizado. É o que acontece quando você aperta o botão físico.

1.  **Usuário:** Pressiona o botão físico conectado ao `BUTTON_PIN`.
2.  **NodeMCU (Seu Código):**
    - A função `loop()` chama `handleButtonPress()`.
    - `digitalRead(BUTTON_PIN)` retorna `LOW` (pressionado).
    - O _debounce_ (verificação de `millis()`) é satisfeito.
3.  **Execução de `handleButtonPress()`:**
    - `myPowerState = !myPowerState;` (O estado é invertido. Se estava `false`, vira `true`).
    - `digitalWrite(RELE_PIN, myPowerState?LOW:HIGH);` -> O relé é fisicamente ativado (ou desativado).
4.  **Sincronização com a Nuvem (A Parte Importante):**
    - `SinricProSwitch& mySwitch = SinricPro[SWITCH_ID];`
    - `mySwitch.sendPowerStateEvent(myPowerState);`
    - Esta linha **envia um evento** do NodeMCU **para** a nuvem Sinric Pro (através da conexão WebSocket), dizendo: "Ei, meu novo estado é `true` (ligado). Fui alterado localmente."
5.  **Nuvem Sinric Pro:** Recebe esta atualização de estado via WebSocket. Agora ele sabe que o dispositivo está "ligado".
6.  **Sinric Pro -> Alexa:** O Sinric Pro **automaticamente** envia um "Relatório de Mudança de Estado" (State Report, um JSON via API HTTPs) para a nuvem da Amazon (Alexa).
7.  **Nuvem Alexa:** Recebe essa atualização e armazena o novo estado.

**Resultado:** Se você agora abrir o aplicativo Alexa no seu celular, o dispositivo "relé" aparecerá como "Ligado". Se você perguntar "Alexa, o relé está ligado?", ela saberá a resposta correta, mesmo que a mudança tenha sido feita pelo botão físico.

## Análise do Código

Aqui está um detalhamento de como o código implementa esses fluxos.

```cpp
// Inclui as bibliotecas necessárias
#include <Arduino.h>
#include <ESP8266WiFi.h>
#include <SinricPro.h>        // Biblioteca principal do Sinric Pro
#include <SinricProSwitch.h>  // Definição específica para um dispositivo "Switch"

// Define constantes para conexão Wi-Fi e SinricPro
#define WIFI_SSID "nome da rede"
#define WIFI_PASS "senha"
#define APP_KEY "3d6f9d29-...-6d3e879be3cc" // Chave de API do Sinric Pro (autentica o dispositivo)
#define APP_SECRET "..." // Chave secreta (autentica o dispositivo)
#define SWITCH_ID "69014270359ccc32ce17e5e3" // O ID único do dispositivo no Sinric Pro
#define BAUD_RATE 9600

// Define os pinos GPIO
#define BUTTON_PIN 0 // GPIO para o BOTÃO (D3 no NodeMCU)
#define RELE_PIN 5   // GPIO para o RELE (D1 no NodeMCU)

bool myPowerState = false; // Variável global para guardar o estado atual (ligado/desligado)
unsigned long lastBtnPress = 0; // Usado para evitar "debounce" do botão

// -------------------------------------------------------------------------
// Esta função é o CALLBACK. Ela é chamada QUANDO o Sinric Pro envia um comando.
// -------------------------------------------------------------------------
bool onPowerState(const String &deviceId, bool &state) {
  // `deviceId` é o ID do dispositivo que recebeu o comando
  // `state` é o estado desejado (true para 'on', false para 'off')

  Serial.printf("Device %s turned %s (via SinricPro) \r\n", deviceId.c_str(), state?"on":"off");

  myPowerState = state; // Atualiza a variável de estado global

  // Controla o pino físico do relé.
  // A lógica ternária `myPowerState?LOW:HIGH` é usada.
  // Se myPowerState for true (ligar), define o pino como LOW.
  // Se myPowerState for false (desligar), define o pino como HIGH.
  // (Isso sugere que seu relé é ativo em nível baixo, o que é comum)
  digitalWrite(RELE_PIN, myPowerState?LOW:HIGH);

  return true; // Retorna 'true' para dizer ao Sinric Pro que o comando foi bem-sucedido.
}

// -------------------------------------------------------------------------
// Esta função trata o acionamento do botão físico.
// -------------------------------------------------------------------------
void handleButtonPress() {
  unsigned long actualMillis = millis();

  // Verifica se o botão está pressionado (LOW) e se já passou 1 segundo (1000ms)
  // desde o último toque, para evitar leituras múltiplas (debounce).
  if (digitalRead(BUTTON_PIN) == LOW && actualMillis - lastBtnPress > 1000) {

    myPowerState = !myPowerState; // Inverte o estado (true -> false, false -> true)
    digitalWrite(RELE_PIN, myPowerState?LOW:HIGH); // Aciona o relé fisicamente

    // !! ESTA É A PARTE CRÍTICA DA SINCRONIZAÇÃO !!
    // Pega a instância do dispositivo "Switch" no Sinric Pro
    SinricProSwitch& mySwitch = SinricPro[SWITCH_ID];

    // Envia o NOVO estado para a nuvem Sinric Pro via WebSocket.
    // Isso atualiza o app Sinric Pro e também dispara o evento para a Alexa.
    mySwitch.sendPowerStateEvent(myPowerState);

    Serial.printf("Device %s turned %s (manually via flashbutton)\r\n",
                  mySwitch.getDeviceId().c_str(), myPowerState?"on":"off");

    lastBtnPress = actualMillis; // Registra o tempo do último toque
  }
}

// Função para conectar o ESP8266 ao Wi-Fi
void setupWiFi() {
  Serial.printf("\r\n[Wifi]: Connecting");
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  while (WiFi.status() != WL_CONNECTED) {
    Serial.printf(".");
    delay(250);
  }
  Serial.printf("connected!\r\n[WiFi]: IP-Address is %s\r\n", WiFi.localIP().toString().c_str());
}

// Função para configurar a conexão com SinricPro
void setupSinricPro() {
  // Pega a instância do dispositivo no Sinric Pro
  SinricProSwitch& mySwitch = SinricPro[SWITCH_ID];

  // !! REGISTRA O CALLBACK !!
  // "Ei, Sinric Pro, QUANDO você receber um comando 'onPowerState' para este
  // dispositivo, por favor, EXECUTE a minha função 'onPowerState'."
  mySwitch.onPowerState(onPowerState);

  // Funções de callback (anônimas/lambda) para monitorar o status da conexão
  SinricPro.onConnected([](){ Serial.printf("Connected to SinricPro\r\n"); });
  SinricPro.onDisconnected([](){ Serial.printf("Disconnected from SinricPro\r\n"); });

  // Inicia a conexão WebSocket com o Sinric Pro usando suas credenciais
  SinricPro.begin(APP_KEY, APP_SECRET);
}

// -------------------------------------------------------------------------
// Função principal de configuração (executada uma vez no boot)
// -------------------------------------------------------------------------
void setup() {
  // Configura o pino do botão como entrada com resistor de pull-up interno
  // Isso significa que o pino lê HIGH quando solto e LOW quando pressionado (conectado ao GND)
  pinMode(BUTTON_PIN, INPUT_PULLUP);

  // Configura o pino do relé como saída
  pinMode(RELE_PIN, OUTPUT);

  // Garante que o relé comece desligado (HIGH, neste caso)
  digitalWrite(RELE_PIN, HIGH);

  Serial.begin(BAUD_RATE);
  setupWiFi();
  setupSinricPro();
}

// -------------------------------------------------------------------------
// Loop principal (executado continuamente)
// -------------------------------------------------------------------------
void loop() {
  // Verifica constantemente se o botão físico foi pressionado
  handleButtonPress();

  // !! FUNÇÃO MAIS IMPORTANTE !!
  // `SinricPro.handle()` faz todo o trabalho pesado:
  // 1. Mantém a conexão WebSocket com a nuvem Sinric Pro.
  // 2. Recebe comandos da nuvem (e chama os callbacks, como `onPowerState`).
  // 3. Envia eventos (como `sendPowerStateEvent`) para a nuvem.
  // 4. Gerencia pings de "keep-alive" para não ser desconectado.
  SinricPro.handle();
}
```
