#include <SPI.h>
#include <RF24.h>

#define RF_CS 9
#define RF_CSN 10
 
RF24 radio(RF_CS, RF_CSN); 
const uint64_t pipes[2] = { 0xe7e7e7e7e7LL, 0xc2c2c2c2c2LL }; // paralel veri yolu adresleri
bool done = false;
 
void setup() {
  Serial.begin(9600); 
  radio.begin();
  radio.openWritingPipe(pipes[1]);
  radio.openReadingPipe(1, pipes[0]);
  radio.startListening();
  radio.printDetails();
}
char RCVmessage[32]; 
char * GonderilenMesaj="batuhan pc";

void loop() {
  if (radio.available()) {
   // gelen verinin dizi sayýsý 1. Çünkü ya 0 gelecek yada 1 gelecek
    radio.read( &RCVmessage, sizeof(RCVmessage) );
    Serial.println("Alinan mesaj : " + String(RCVmessage)); 
  }
  MesajGonder(GonderilenMesaj); 
  delay(100);
}
void MesajGonder(char * message)
{
  radio.stopListening();
  radio.write( (uint8_t*)message,strlen(message));
  Serial.println("Gonderilen mesaj : " + String(message)); 
  radio.startListening();
}
