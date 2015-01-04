TCC-Semaforo.ino
================
#include <SPI.h>                                            // Inclusão da Biblioteca de Comunicação Serial (SPI)
#include <MFRC522.h>                                        // Inclusão da Biblioteca do Modulo RFID (MFRC522)

// Definindo Pinos usados pelo Semaforo
// PORTD
#define SEM_GRN (1 << 0)                                    // Saída Digital para acionamento da placa do Semaforo Verde
#define SEM_YLW (1 << 1)                                    // Saída Digital para acionamento da placa do Semaforo Amarelo
#define SEM_RED (1 << 2)                                    // Saída Digital para acionamento da placa do Semaforo Vermelho
#define SEM_PED_GRN (1 << 3)                                // Saída Digital para acionamento da placa do Semaforo de Pedestre Verde
#define SEM_PED_RED (1 << 4)                                // Saída Digital para acionamento da placa do Semaforo de Pedestre Vermelho
#define LIGHT_VISUALLY_IMPAIRED (1 << 5)                    // Saída Digital para acionamento da placa de Acrílico - Deficiente Visual
 
// Definindo Pinos usados pelos Audios
// PORTC
#define PLAY_SOUND_1 (1 << 0)                               // Saída Digital para reprodução da placa de audio 1 (Aguarde)
#define PLAY_SOUND_2 (1 << 1)                               // Saída Digital para reprodução da placa de audio 2 (Prossiga)
#define PLAY_SOUND_3 (1 << 2)                               // Saída de Referência - Terra \ Ground
#define PLAY_SOUND_4 (1 << 3)                               // Saída de Referência - Terra \ Ground

// Definindo Pinos usados pelo RFID
// PORTB
#define RFID_SS 10                                          // SS - SDA do RFID
#define RFID_RST 9                                          // Reset do RFID

MFRC522 RFID(RFID_SS, RFID_RST);                            // Criando Instância do RFID

// Cabeçalho das Funções
void VerificaRFID();                                        // Cabeçalho da procedure VerificaRFID
void Semaforo(byte Status);                                 // Cabeçalho da procedure Semaforo
void PlaySound(byte NumberTrack);                           // Cabeçalho da procedure PlaySound
void HabilitaAudio(byte Status, byte Timer, byte Repeat);   // Cabeçalho da procedure HabilitaAudio

// Definições das Variaveis Globais
boolean EnableAudio, FirstAudio = false;                    // Variável Booleana para Habilitar \ Desabilitar circuito de audio
byte StateAudio = 0;                                        // Variável do tipo Byte para controle dos ciclos de audios

void setup(){
  DDRB = DDRC = DDRD = 0xFF;                                // Todos os Pinos configurados como Saida (OUTPUT)
  PORTB = PORTC = PORTD = 0x00;                             // Todos os Pinos configurados como Baixo (LOW)
  
  SPI.begin();                                              // Inicia SPI Bus
  RFID.PCD_Init();                                          // Inicia RFID
}

void loop(){
  for (byte i = 0; i < 4; i++)                              // Loop "I" - Estado do Semáforo (0 = Verde; 1 = Amarelo; 2 = Vermelho; 3 = Transição para Verde)
    for (byte j = 0; j < 5; j++)                            // Loop "J" - Tempo do Semáforo e Leitura do Cartão e Controles de Audio (5 Atualizações)
      for (byte k = 0; k < 2; k++){                         // Loop "K" - 2x Loop "J"
        Semaforo(i); VerificaRFID();                        // Atualiza Semáforo \ Verifica se a algum cartão
        delay((i == 1) ? 150 : 250);                        // Se semaforo for amarelo delay = 150ms senão delay = 250ms - 1º Tempo
        Semaforo(i); HabilitaAudio(i,j,k);                  // Atualiza Semáforo \ Habilita Audio do semaforo de acordo com o estado do mesmo
        delay((i == 1) ? 150 : 250);                        // Se semaforo for amarelo delay = 150ms senão delay = 250ms - 2º Tempo
      }
}

void Semaforo(byte Status){
  switch (Status){
    case 0: PORTD = SEM_GRN | SEM_PED_RED; break;           // Se estado do Semáforo for Verde então atualiza
    case 1: PORTD = SEM_YLW | SEM_PED_RED; break;           // Se estado do Semáforo for Amarelo então atualiza
    case 2: PORTD = SEM_RED | SEM_PED_GRN; break;           // Se estado do Semáforo for Vermelho então atualiza
    case 3: PORTD = (PORTD & SEM_PED_RED)?                  // Se estado do semáforo for de Transição para o Verde então pisca Semáforo de Pedestre Vermelho
                    (SEM_RED & ~SEM_PED_RED):               // Semáforo Pedestre Vermelho - Apagado
                    (SEM_RED | SEM_PED_RED); break;         // Semáforo Pedestre Vermelho - Acesso
  }
  PORTD = (EnableAudio)?(PORTD | LIGHT_VISUALLY_IMPAIRED):  // Se audio habilitado então Acende Acrílico de Deficiente Visual
                        (PORTD & ~LIGHT_VISUALLY_IMPAIRED); // Senão então Apaga Acrílico de Deficiente Visual
}

void VerificaRFID(){
  if (! RFID.PICC_IsNewCardPresent()) return;               // Se não tem Cartão - sai da procedure
  if (! RFID.PICC_ReadCardSerial()) return;                 // Se não leu a ID do Cartão - sai da procedure
  
  String CardID = "";                                       // Variável Local para organização da ID do Cartão.

  for (byte g = 0; g < RFID.uid.size; g++){                 // Enquanto não ler a ID do Cartão
    CardID.concat(String(RFID.uid.uidByte[g] < 0x10 ?       // Se Byte lido da ID do Cartão for menor "0x10"
                                               " 0" :       // Então retorna " 0" na Variável Local "CardID"
                                               " "));       // Senão retorna " " na Variável Local "CardID"
    CardID.concat(String(RFID.uid.uidByte[g], HEX));        // Retorna Byte da ID do Cartão na Variável Local "CardID"
  }
  CardID.toUpperCase();                                     // Converte toda a String em maiuscula
  
  if (CardID.substring(1) == "B4 46 59 46"){                // ID do Cartão
    EnableAudio = FirstAudio = true;                        // Habilita Audio
  }
  if (CardID.substring(1) == "83 83 F7 A4")                 // ID do Chaveiro
    EnableAudio = false;                                    // Desabilita Audio
}

void HabilitaAudio(byte Status, byte Timer, byte Repeat){
  if (EnableAudio == false) return;                         // Se não tiver habilitado o audio - sai da procedure
  
  switch (Status){
    case 2:                                                 // Semáforo Vermelho
      if (StateAudio == 0){                                 // 
        if (FirstAudio) PlaySound(1);                       // 
        if ((Timer == 0) && (Repeat == 0))                  // 
          PlaySound(1);                                     // 
      } else {
        if (FirstAudio) PlaySound(2);                       // 
        if ((Timer == 0) && (Repeat == 0))                  // 
          PlaySound(2);                                     // 
      }
    break;
    case 3:                                                 // Semáforo Piscante
      if ((StateAudio == 0) || (StateAudio == 1)){
        if (FirstAudio) PlaySound(1);                       // 
        if ((Timer == 0) && (Repeat == 0))                  // 
           PlaySound(1);                                    // 
      } else if ((StateAudio == 2) || (StateAudio == 3)){
        if (FirstAudio) PlaySound(2);                       // 
        if ((Timer == 0) && (Repeat == 0))                  // 
          PlaySound(2);                                     // 
        if ((Timer == 4) && (Repeat == 1))                  // 
          EnableAudio = false;                              // 
      }
    break;
    default:                                                // Se Semáforo for Verde ou Amarelo
      if (FirstAudio) PlaySound(1);                         // 
      if ((Timer == 0) && (Repeat == 0))                    // Se tiver no tempo 0 do 1º periodo
        PlaySound(1);                                       // 
    break;
  }
  
  if ((Timer == 4) && (Repeat == 1))                        // Se for no ultimo tempo e na ultima repetição de cada estado do Semáforo
    StateAudio++;                                           // Então incrementa Variável Global de controle do ciclo do audio
    
  if ((Status == 3) && (Timer == 4) && (Repeat == 1))       // 
    StateAudio = 0;                                         // 

  FirstAudio = false;                                       // 
}

void PlaySound(byte NumberTrack){
  switch (NumberTrack){                                     // 
    case 1:                                                 // 
      PORTC = PLAY_SOUND_1;                                 // Habilita comando de reprodução do audio 1 - Aguarde
      delay(10);                                            // Espera 10ms
      PORTC &= ~PLAY_SOUND_1;                               // Desabilita comando de reprodução do audio 1 - Aguarde
    break;
    case 2:                                                 // 
      PORTC = PLAY_SOUND_2;                                 // Habilita comando de reprodução do audio 2 - Prossiga
      delay(10);                                            // Espera 10ms
      PORTC &= ~PLAY_SOUND_2;                               // Desabilita comando de reprodução do audio 2 - Prossiga  
    break;
  }
}
