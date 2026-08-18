//  PROJETO DE MICROCONTROLADORES – Avaliação P2 – 2026
//  Firmware para Máquina de Lavar com ESP32
//  Grupo 2 – Eficiência Energética de PWM

#include <Arduino.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <BluetoothSerial.h>

//  Bluetooth Serial
#define BLE_DEVICE_NAME "MaqLavar_Grp2"
BluetoothSerial SerialBT;
bool bleConectado = false;
volatile char bleTeclaRecebida = 0;

//  ENDEREÇOS I2C
#define LCD_I2C_ADDR 0x20
#define MCP_I2C_ADDR 0x21

//  MCP23017 – Registradores
#define MCP_IODIRA 0x00
#define MCP_IODIRB 0x01
#define MCP_GPPUA 0x0C
#define MCP_GPPUB 0x0D
#define MCP_GPIOA 0x12
#define MCP_GPIOB 0x13
#define MCP_OLATA 0x14
#define MCP_OLATB 0x15

uint8_t mcpLatchA = 0xFF;
uint8_t mcpLatchB = 0x00;

#define TECLADO_LINHA_MASK 0x0F
#define TECLADO_COL_MASK 0xF0

#define BIT_ENA 0
#define BIT_ENB 1
#define BIT_EN1 2
#define BIT_EN2 3
#define BIT_EN3 4
#define BIT_EN4 5
#define BIT_RELE1 6
#define BIT_RELE2 7

//  PINOS ESP32
#define PIN_ADC_POT2 36
#define PIN_VENTILADOR 15
#define PIN_BUZZER 33
#define PIN_F1_VERMELHO 14
#define PIN_F1_AMARELO 13
#define PIN_F1_VERDE 12
#define PIN_F2_VERMELHO 27
#define PIN_F2_AMARELO 17
#define PIN_F2_VERDE 16

//  TEMPOS DE CICLO (segundos)
#define TEMPO_ENCHIMENTO_S 10
#define TEMPO_LAVAGEM_S 15
#define TEMPO_ENXAGUE_S 15
#define TEMPO_CENTRIFUGACAO_S 15
#define TEMPO_ESCOAMENTO_S 10

//  EFICIÊNCIA ENERGÉTICA
#define MOTOR_POT_NOMINAL_W 500.0f
#define TENSAO_NOMINAL_V 220.0f
#define ADC_MAX_F 4095.0f

//  ESTADOS
typedef enum
{
  ESTADO_IDLE = 0,
  ESTADO_ENCHIMENTO,
  ESTADO_LAVAGEM,
  ESTADO_ESCOAMENTO_1,
  ESTADO_ENXAGUE,
  ESTADO_ESCOAMENTO_2,
  ESTADO_CENTRIFUGACAO,
  ESTADO_FIM,
  ESTADO_ERRO
} EstadoMaquina;

//  TIMERS
hw_timer_t *timerCiclo = NULL;
hw_timer_t *timerPWM = NULL;
hw_timer_t *timerMS = NULL;
volatile uint32_t contadorMS = 0;

void IRAM_ATTR onTimerMS()
{
  contadorMS++;
}

// Substituto de millis()
inline uint32_t getMS()
{
  return contadorMS;
}

// Substituto de delay()
void esperarMS(uint32_t ms)
{
  uint32_t inicio = getMS();
  while (getMS() - inicio < ms)
    ;
}

volatile bool timerDisparou = false;
volatile uint32_t contagemSegundos = 0;
volatile uint32_t tempoEtapaTotal = 0;

volatile uint8_t dutyCycleMotor = 0;
volatile uint8_t contadorPWM = 0;
volatile bool motorLigado = false;
volatile bool motorEstado = false;

//  NÍVEL DE ÁGUA
uint16_t adcNivel = 0;
uint16_t nivelCongelado = 0;

// Eficiência
float potenciaAtual_W = 0.0f;
float energiaTotal_Wh = 0.0f;
float corrente_A = 0.0f;
uint8_t eficienciaPercent = 100;

// LCD
LiquidCrystal_I2C lcd(LCD_I2C_ADDR, 16, 2);

// Estado
EstadoMaquina estadoAtual = ESTADO_IDLE;
EstadoMaquina estadoAnterior = ESTADO_IDLE;

// Teclado físico
const char mapasTeclado[4][4] = {
    {'1', '2', '3', 'A'},
    {'4', '5', '6', 'B'},
    {'7', '8', '9', 'C'},
    {'*', '0', '#', 'D'}};
char teclaPresionada = 0;

bool portaAberta = false;

// Bluetooth – status periódico
static uint32_t tBLEStatus = 0;
#define BLE_STATUS_INTERVAL_MS 2000

//  MCP23017
void mcpEscrever(uint8_t reg, uint8_t val)
{
  Wire.beginTransmission(MCP_I2C_ADDR);
  Wire.write(reg);
  Wire.write(val);
  Wire.endTransmission();
}
uint8_t mcpLer(uint8_t reg)
{
  Wire.beginTransmission(MCP_I2C_ADDR);
  Wire.write(reg);
  Wire.endTransmission();
  Wire.requestFrom((uint8_t)MCP_I2C_ADDR, (uint8_t)1);
  return Wire.available() ? Wire.read() : 0;
}
void mcpSetBitB(uint8_t bit, bool val)
{
  if (val)
    mcpLatchB |= (1 << bit);
  else
    mcpLatchB &= ~(1 << bit);
  mcpEscrever(MCP_OLATB, mcpLatchB);
}
void mcpSetLinhasTeclado(uint8_t mascara)
{
  mcpLatchA = (mcpLatchA & TECLADO_COL_MASK) | (mascara & TECLADO_LINHA_MASK);
  mcpEscrever(MCP_OLATA, mcpLatchA);
}
uint8_t mcpLerColunas()
{
  return (mcpLer(MCP_GPIOA) & TECLADO_COL_MASK) >> 4;
}
void mcpInit()
{
  mcpEscrever(MCP_IODIRA, 0xF0);
  mcpEscrever(MCP_GPPUA, 0xF0);
  mcpEscrever(MCP_OLATA, 0xFF);
  mcpLatchA = 0xFF;
  mcpEscrever(MCP_IODIRB, 0x00);
  mcpEscrever(MCP_OLATB, 0x00);
  mcpLatchB = 0x00;
}

//  ISRs
void IRAM_ATTR onTimerCiclo()
{
  contagemSegundos++;
  if (contagemSegundos >= tempoEtapaTotal)
    timerDisparou = true;
}

void IRAM_ATTR onTimerPWM()
{
  if (!motorLigado)
  {
    motorEstado = false;
    return;
  }
  contadorPWM++;
  if (contadorPWM >= 100)
    contadorPWM = 0;
  motorEstado = (contadorPWM < dutyCycleMotor);
}

//  PROTÓTIPOS
void iniciarEtapa(EstadoMaquina novo);
void processarMaquinaEstados();
void etapaLavagem();
void etapaEnxague();
void etapaCentrifugacao();
void etapaFim();
void etapaErro(const char *motivo);
void atualizarLCD();
char lerTeclado();
char lerBLE();
void bleSend(const char *msg);
void enviarStatusBLE();
void calcularEficiencia();
void setMotorDutyCycle(uint8_t duty);
void motorParar();
void setFarol(bool farol2, uint8_t cor);
void buzzerBip(uint16_t freq, uint16_t ms);
void lerSensores();
void logarEstado(const char *label);
void congelarNivel();

//  SETUP
void setup()
{
  Serial.begin(115200);
  Serial.println(F("=== MaqLavar Grp2 – Teclado + Bluetooth Serial ==="));
  pinMode(PIN_VENTILADOR, OUTPUT);
  digitalWrite(PIN_VENTILADOR, LOW);
  pinMode(PIN_BUZZER, OUTPUT);
  digitalWrite(PIN_BUZZER, LOW);
  pinMode(PIN_F1_VERMELHO, OUTPUT);
  pinMode(PIN_F1_AMARELO, OUTPUT);
  pinMode(PIN_F1_VERDE, OUTPUT);
  pinMode(PIN_F2_VERMELHO, OUTPUT);
  pinMode(PIN_F2_AMARELO, OUTPUT);
  pinMode(PIN_F2_VERDE, OUTPUT);
  setFarol(false, 0);
  setFarol(true, 0);

  analogReadResolution(12);
  analogSetAttenuation(ADC_11db);

  ledcAttach(PIN_BUZZER, 2000, 8);

  Wire.begin();

  lcd.init();
  lcd.backlight();
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print(F("MaqLavar Grp2  "));
  lcd.setCursor(0, 1);
  lcd.print(F("Iniciando...   "));

  mcpInit();
  Serial.println(F("MCP23017 OK"));

  // Bluetooth Serial
  SerialBT.begin(BLE_DEVICE_NAME);
  bleConectado = false;
  Serial.println(F("Bluetooth Serial iniciado: MaqLavar_Grp2"));

  // Timers
  timerMS = timerBegin(1000000);
  timerAttachInterrupt(timerMS, &onTimerMS);
  timerAlarm(timerMS, 1000UL, true, 0);
  timerStart(timerMS);

  timerCiclo = timerBegin(1000000);
  timerAttachInterrupt(timerCiclo, &onTimerCiclo);
  timerAlarm(timerCiclo, 1000000UL, true, 0);
  timerStop(timerCiclo);

  timerPWM = timerBegin(100000);
  timerAttachInterrupt(timerPWM, &onTimerPWM);
  timerAlarm(timerPWM, 10UL, true, 0);
  timerStart(timerPWM);

  timerStart(timerCiclo);
  timerStop(timerCiclo);

  lcd.clear();
  estadoAtual = ESTADO_IDLE;
  atualizarLCD();
  setFarol(false, 2);
  logarEstado("INICIO");
  buzzerBip(1000, 150);

  Serial.println(F("[1]=Iniciar  [2]=Parar  [3]=Porta  [*]=Reset FIM  [#]=Reset ERRO"));
}

//  LOOP
void loop()
{
  // Detecta conexão/desconexão Bluetooth
  bool conectadoAgora = SerialBT.connected();
  if (conectadoAgora != bleConectado)
  {
    bleConectado = conectadoAgora;
    Serial.println(bleConectado ? F("[BT] Cliente conectado") : F("[BT] Cliente desconectado"));
  }

  // Lê tecla recebida via Bluetooth
  if (SerialBT.available())
  {
    char c = SerialBT.read();
    if (c != '\r' && c != '\n')
    {
      bleTeclaRecebida = c;
      Serial.print(F("[BT RX] "));
      Serial.println(c);
    }
  }

  lerSensores();

  bool estado = motorEstado && motorLigado;
  mcpSetBitB(BIT_ENA, estado);
  mcpSetBitB(BIT_ENB, estado);

  calcularEficiencia();

  // Teclado físico tem prioridade; Bluetooth entra se teclado retornar 0
  teclaPresionada = lerTeclado();
  if (teclaPresionada == 0)
    teclaPresionada = lerBLE();

  if (teclaPresionada == '3')
  {
    portaAberta = !portaAberta;
    Serial.print(F("Porta: "));
    Serial.println(portaAberta ? F("ABERTA") : F("fechada"));
    bleSend(portaAberta ? "Porta: ABERTA" : "Porta: fechada");
    buzzerBip(portaAberta ? 600 : 1200, 80);
  }

  processarMaquinaEstados();

  static uint32_t tLCD = 0;
  if (getMS() - tLCD > 500)
  {
    tLCD = getMS();
    atualizarLCD();
  }

  if (getMS() - tBLEStatus > BLE_STATUS_INTERVAL_MS)
  {
    tBLEStatus = getMS();
    enviarStatusBLE();
  }
}

//  LEITURA BLUETOOTH
char lerBLE()
{
  if (bleTeclaRecebida == 0)
    return 0;
  char c = bleTeclaRecebida;
  bleTeclaRecebida = 0;
  char eco[16];
  snprintf(eco, sizeof(eco), "Tecla BT: %c", c);
  bleSend(eco);
  return c;
}

//  ENVIO BLUETOOTH
void bleSend(const char *msg)
{
  if (!bleConectado)
    return;
  SerialBT.println(msg);
}

//  STATUS PERIÓDICO VIA BLUETOOTH
void enviarStatusBLE()
{
  if (!bleConectado)
    return;
  const char *nomes[] = {"IDLE", "ENCHIMENTO", "LAVAGEM", "ESCOA1", "ENXAGUE", "ESCOA2", "CENTRIFUG", "FIM", "ERRO"};
  char buf[64];
  uint32_t restante = (tempoEtapaTotal > contagemSegundos)
                          ? tempoEtapaTotal - contagemSegundos
                          : 0;
  snprintf(buf, sizeof(buf), "%s DC:%d%% P:%dW Ef:%d%% E:%.2fWh N:%d%% T:%lus",
           nomes[estadoAtual],
           dutyCycleMotor,
           (int)potenciaAtual_W,
           eficienciaPercent,
           energiaTotal_Wh,
           nivelCongelado,
           restante);
  bleSend(buf);
}

//  CONGELAR NÍVEL ao fim do enchimento
void congelarNivel()
{
  nivelCongelado = (uint16_t)map(adcNivel, 0, 4095, 0, 100);
  char buf[32];
  snprintf(buf, sizeof(buf), "Nivel congelado: %d%%", nivelCongelado);
  Serial.println(buf);
  bleSend(buf);
}

//  MÁQUINA DE ESTADOS
void processarMaquinaEstados()
{
  switch (estadoAtual)
  {

  case ESTADO_IDLE:
    if (teclaPresionada == '1')
      iniciarEtapa(ESTADO_ENCHIMENTO);
    break;

  case ESTADO_ENCHIMENTO:
    mcpSetBitB(BIT_RELE1, true);
    if (timerDisparou)
    {
      timerDisparou = false;
      mcpSetBitB(BIT_RELE1, false);
      congelarNivel();
      iniciarEtapa(ESTADO_LAVAGEM);
    }
    if (teclaPresionada == '2')
      etapaErro("Parada manual");
    break;

  case ESTADO_LAVAGEM:
    etapaLavagem();
    if (portaAberta)
    {
      etapaErro("Porta aberta!");
      break;
    }
    if (timerDisparou)
    {
      timerDisparou = false;
      motorParar();
      iniciarEtapa(ESTADO_ESCOAMENTO_1);
    }
    if (teclaPresionada == '2')
      etapaErro("Parada manual");
    break;

  case ESTADO_ESCOAMENTO_1:
    mcpSetBitB(BIT_RELE2, true);
    if (timerDisparou)
    {
      timerDisparou = false;
      mcpSetBitB(BIT_RELE2, false);
      iniciarEtapa(ESTADO_ENXAGUE);
    }
    if (teclaPresionada == '2')
      etapaErro("Parada manual");
    break;

  case ESTADO_ENXAGUE:
    etapaEnxague();
    if (timerDisparou)
    {
      timerDisparou = false;
      motorParar();
      mcpSetBitB(BIT_RELE1, false);
      iniciarEtapa(ESTADO_ESCOAMENTO_2);
    }
    if (teclaPresionada == '2')
      etapaErro("Parada manual");
    break;

  case ESTADO_ESCOAMENTO_2:
    mcpSetBitB(BIT_RELE2, true);
    if (timerDisparou)
    {
      timerDisparou = false;
      mcpSetBitB(BIT_RELE2, false);
      iniciarEtapa(ESTADO_CENTRIFUGACAO);
    }
    if (teclaPresionada == '2')
      etapaErro("Parada manual");
    break;

  case ESTADO_CENTRIFUGACAO:
    etapaCentrifugacao();
    if (portaAberta)
    {
      etapaErro("Porta!Centrifug");
      break;
    }
    if (timerDisparou)
    {
      timerDisparou = false;
      motorParar();
      iniciarEtapa(ESTADO_FIM);
    }
    if (teclaPresionada == '2')
      etapaErro("Parada manual");
    break;

  case ESTADO_FIM:
    etapaFim();
    if (teclaPresionada == '*')
    {
      timerStop(timerCiclo);
      contagemSegundos = 0;
      energiaTotal_Wh = 0.0f;
      nivelCongelado = 0;
      estadoAtual = ESTADO_IDLE;
      setFarol(false, 2);
      setFarol(true, 0);
      atualizarLCD();
    }
    break;

  case ESTADO_ERRO:
    if (teclaPresionada == '#')
    {
      motorParar();
      mcpSetBitB(BIT_RELE1, false);
      mcpSetBitB(BIT_RELE2, false);
      timerStop(timerCiclo);
      contagemSegundos = 0;
      energiaTotal_Wh = 0.0f;
      nivelCongelado = 0;
      portaAberta = false;
      estadoAtual = ESTADO_IDLE;
      setFarol(false, 2);
      setFarol(true, 0);
      atualizarLCD();
    }
    break;
  }
}

//  INICIAR ETAPA
void iniciarEtapa(EstadoMaquina novo)
{
  estadoAnterior = estadoAtual;
  estadoAtual = novo;
  timerDisparou = false;
  contagemSegundos = 0;

  switch (novo)
  {
  case ESTADO_ENCHIMENTO:
    tempoEtapaTotal = TEMPO_ENCHIMENTO_S;
    break;
  case ESTADO_LAVAGEM:
    tempoEtapaTotal = TEMPO_LAVAGEM_S;
    break;
  case ESTADO_ESCOAMENTO_1:
  case ESTADO_ESCOAMENTO_2:
    tempoEtapaTotal = TEMPO_ESCOAMENTO_S;
    break;
  case ESTADO_ENXAGUE:
    tempoEtapaTotal = TEMPO_ENXAGUE_S;
    break;
  case ESTADO_CENTRIFUGACAO:
    tempoEtapaTotal = TEMPO_CENTRIFUGACAO_S;
    break;
  default:
    tempoEtapaTotal = 0;
    break;
  }

  if (tempoEtapaTotal > 0)
    timerStart(timerCiclo);
  else
    timerStop(timerCiclo);

  logarEstado("INICIO");

  const char *nomes[] = {"IDLE", "ENCHIMENTO", "LAVAGEM", "ESCOA1", "ENXAGUE", "ESCOA2", "CENTRIFUG", "FIM", "ERRO"};
  char buf[24];
  snprintf(buf, sizeof(buf), ">> ETAPA: %s", nomes[novo]);
  bleSend(buf);

  buzzerBip(1000, 150);
}

//  ETAPAS
void etapaLavagem()
{
  uint8_t duty = (uint8_t)map(nivelCongelado, 0, 100, 30, 100);
  setMotorDutyCycle(duty);
  mcpSetBitB(BIT_EN1, true);
  mcpSetBitB(BIT_EN2, false);
  digitalWrite(PIN_VENTILADOR, HIGH);
  setFarol(false, 1);
  setFarol(true, 2);
}

void etapaEnxague()
{
  uint8_t duty = (uint8_t)map(nivelCongelado, 0, 100, 20, 60);
  setMotorDutyCycle(duty);
  mcpSetBitB(BIT_EN1, true);
  mcpSetBitB(BIT_EN2, false);
  mcpSetBitB(BIT_RELE1, true);
  setFarol(false, 1);
  setFarol(true, 1);
}

void etapaCentrifugacao()
{
  setMotorDutyCycle(95);
  mcpSetBitB(BIT_EN1, true);
  mcpSetBitB(BIT_EN2, false);
  digitalWrite(PIN_VENTILADOR, HIGH);
  setFarol(false, 3);
  setFarol(true, 3);
}

void etapaFim()
{
  static bool bipFeito = false;
  if (!bipFeito)
  {
    bipFeito = true;
    buzzerBip(1500, 250);
    esperarMS(80);
    buzzerBip(1500, 250);
    esperarMS(80);
    buzzerBip(1500, 250);
    char buf[48];
    snprintf(buf, sizeof(buf), "** CONCLUIDO ** E:%.3f Wh", energiaTotal_Wh);
    bleSend(buf);
  }
  setFarol(false, 1);
  setFarol(true, 1);
}

void etapaErro(const char *motivo)
{
  motorParar();
  digitalWrite(PIN_VENTILADOR, LOW);
  mcpSetBitB(BIT_RELE1, false);
  mcpSetBitB(BIT_RELE2, false);
  timerStop(timerCiclo);
  estadoAtual = ESTADO_ERRO;
  setFarol(false, 3);
  setFarol(true, 3);
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print(F("** ERRO **     "));
  lcd.setCursor(0, 1);
  lcd.print(motivo);
  buzzerBip(400, 600);
  Serial.print(F("ERRO: "));
  Serial.println(motivo);
  char buf[32];
  snprintf(buf, sizeof(buf), "ERRO: %s [#]=Reset", motivo);
  bleSend(buf);
}

//  MOTOR
void setMotorDutyCycle(uint8_t duty)
{
  if (duty > 100)
    duty = 100;
  dutyCycleMotor = duty;
  motorLigado = (duty > 0);
}

void motorParar()
{
  motorLigado = false;
  dutyCycleMotor = 0;
  mcpSetBitB(BIT_ENA, false);
  mcpSetBitB(BIT_ENB, false);
  mcpSetBitB(BIT_EN1, false);
  mcpSetBitB(BIT_EN2, false);
  mcpSetBitB(BIT_EN3, false);
  mcpSetBitB(BIT_EN4, false);
  digitalWrite(PIN_VENTILADOR, LOW);
}

//  SENSORES
void lerSensores()
{
  adcNivel = analogRead(PIN_ADC_POT2);
}

//  EFICIÊNCIA ENERGÉTICA
void calcularEficiencia()
{
  float dutyF = dutyCycleMotor / 100.0f;
  float nivelF = nivelCongelado / 100.0f;
  potenciaAtual_W = MOTOR_POT_NOMINAL_W * dutyF * dutyF;
  corrente_A = potenciaAtual_W / TENSAO_NOMINAL_V;
  eficienciaPercent = (uint8_t)(100.0f - (dutyF * 100.0f * (1.0f - nivelF)));
  if (eficienciaPercent > 100)
    eficienciaPercent = 100;
  energiaTotal_Wh += potenciaAtual_W * (10e-3f / 3600.0f);
}

//  Tela LCD
void atualizarLCD()
{
  lcd.clear();
  switch (estadoAtual)
  {
  case ESTADO_IDLE:
    lcd.setCursor(0, 0);
    lcd.print(F("Aguardando...  "));
    lcd.setCursor(0, 1);
    lcd.print(portaAberta ? F("[1]Ini [3]Porta!") : F("[1]=Iniciar    "));
    break;
  case ESTADO_ENCHIMENTO:
    lcd.setCursor(0, 0);
    lcd.print(F("Enchimento     "));
    lcd.setCursor(0, 1);
    {
      uint16_t nv = (uint16_t)map(adcNivel, 0, 4095, 0, 100);
      lcd.print(F("Nivel:"));
      lcd.print(nv);
      lcd.print(F("%  "));
    }
    break;
  case ESTADO_LAVAGEM:
    lcd.setCursor(0, 0);
    lcd.print(F("Lav "));
    lcd.print(dutyCycleMotor);
    lcd.print(F("% "));
    lcd.print((int)potenciaAtual_W);
    lcd.print(F("W"));
    lcd.setCursor(0, 1);
    lcd.print(F("Ef:"));
    lcd.print(eficienciaPercent);
    lcd.print(F("% "));
    lcd.print(tempoEtapaTotal - contagemSegundos);
    lcd.print(F("s"));
    break;
  case ESTADO_ESCOAMENTO_1:
  case ESTADO_ESCOAMENTO_2:
    lcd.setCursor(0, 0);
    lcd.print(F("Escoando...    "));
    lcd.setCursor(0, 1);
    lcd.print(F("Resta:"));
    lcd.print(tempoEtapaTotal - contagemSegundos);
    lcd.print(F("s  "));
    break;
  case ESTADO_ENXAGUE:
    lcd.setCursor(0, 0);
    lcd.print(F("Enxaguando     "));
    lcd.setCursor(0, 1);
    lcd.print(F("Motor:"));
    lcd.print(dutyCycleMotor);
    lcd.print(F("% "));
    lcd.print(tempoEtapaTotal - contagemSegundos);
    lcd.print(F("s"));
    break;
  case ESTADO_CENTRIFUGACAO:
    lcd.setCursor(0, 0);
    lcd.print(F("Centrifugando  "));
    lcd.setCursor(0, 1);
    lcd.print(F("95% "));
    lcd.print(tempoEtapaTotal - contagemSegundos);
    lcd.print(F("s  "));
    break;
  case ESTADO_FIM:
    lcd.setCursor(0, 0);
    lcd.print(F("** CONCLUIDO **"));
    lcd.setCursor(0, 1);
    lcd.print(F("E:"));
    lcd.print(energiaTotal_Wh, 2);
    lcd.print(F("Wh *rst"));
    break;
  case ESTADO_ERRO:
    lcd.setCursor(0, 0);
    lcd.print(F("** ERRO **     "));
    lcd.setCursor(0, 1);
    lcd.print(F("#=Reset        "));
    break;
  }
}

//  TECLADO FÍSICO
char lerTeclado()
{
  static char ultimaTecla = 0;
  static bool debounce = false;
  static uint32_t tDebounce = 0;

  if (debounce)
  {
    if (getMS() - tDebounce > 150)
      debounce = false;
    else
      return 0;
  }
  for (uint8_t l = 0; l < 4; l++)
  {
    mcpSetLinhasTeclado(~(1 << l) & TECLADO_LINHA_MASK);
    delayMicroseconds(100);
    uint8_t cols = mcpLerColunas();
    for (uint8_t c = 0; c < 4; c++)
    {
      if (!(cols & (1 << c)))
      {
        char tecla = mapasTeclado[l][c];
        if (tecla != ultimaTecla)
        {
          ultimaTecla = tecla;
          debounce = true;
          tDebounce = getMS();
          mcpSetLinhasTeclado(TECLADO_LINHA_MASK);
          return tecla;
        }
      }
    }
  }
  mcpSetLinhasTeclado(TECLADO_LINHA_MASK);
  ultimaTecla = 0;
  return 0;
}

//  FARÓIS
void setFarol(bool farol2, uint8_t cor)
{
  uint8_t pV, pA, pR;
  if (!farol2)
  {
    pV = PIN_F1_VERDE;
    pA = PIN_F1_AMARELO;
    pR = PIN_F1_VERMELHO;
  }
  else
  {
    pV = PIN_F2_VERDE;
    pA = PIN_F2_AMARELO;
    pR = PIN_F2_VERMELHO;
  }
  digitalWrite(pV, LOW);
  digitalWrite(pA, LOW);
  digitalWrite(pR, LOW);
  if (cor == 1)
    digitalWrite(pV, HIGH);
  else if (cor == 2)
    digitalWrite(pA, HIGH);
  else if (cor == 3)
    digitalWrite(pR, HIGH);
}

//  BUZZER
void buzzerBip(uint16_t freq, uint16_t ms)
{
  ledcWriteTone(PIN_BUZZER, freq);
  esperarMS(ms);
  ledcWriteTone(PIN_BUZZER, 0);
}

//  LOG
void logarEstado(const char *label)
{
  const char *nomes[] = {"IDLE", "ENCHIMENTO", "LAVAGEM", "ESCOA1", "ENXAGUE", "ESCOA2", "CENTRIFUG", "FIM", "ERRO"};
  Serial.print(F("["));
  Serial.print(label);
  Serial.print(F("] "));
  Serial.print(nomes[estadoAtual]);
  Serial.print(F(" "));
  Serial.print(tempoEtapaTotal);
  Serial.println(F("s"));
}
