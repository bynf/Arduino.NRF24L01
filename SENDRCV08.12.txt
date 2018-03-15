#include <LCD5110_Basic.h> // LCD kütüphanesi eklendi
extern uint8_t SmallFont[];
LCD5110 myGLCD(4,5,6,7,8); // CD pinlerinin hangi Arduino pinlerine baðlý olduðunu belirtiyoruz


#include <SPI.h>
#include <RF24.h>
#define RF_CS 9
#define RF_CSN 10 
RF24 radio(RF_CS, RF_CSN); 
const uint64_t pipes[2] = { 0xe7e7e7e7e7LL, 0xc2c2c2c2c2LL }; // paralel veri yolu adresleri 
#include <Keypad.h>
const byte satirSayisi= 4;
const byte sutunSayisi= 4;
char tuslar[satirSayisi][sutunSayisi]=
{
{'1', '2', '3', 'A'},
{'4', '5', '6', 'B'},
{'7', '8', '9', 'C'},
{'*', '0', '#', 'D'}
};
byte satirPinleri[satirSayisi] = {A0,A1,A2,A3};
byte sutunPinleri[sutunSayisi]= {A4,A5,2,1};
Keypad tusTakimi= Keypad(makeKeymap(tuslar), satirPinleri, sutunPinleri, satirSayisi, sutunSayisi);

char basilanTus;
String Message;
bool MesajYaziliyor=false;
bool BildirimEkrani = false;
bool EkranBildirimYazisi=false;
int enkarakter=0;
int sira=23;

char RCVmessage[32]; 
char* GonderilenMesaj="ilk mesaj testi";
String sonGelenMesaj;
String sonYazilanMesaj;
String sonGonderilenMesaj;


bool SerialState=true;

void setup()
{
  Serial.begin(9600);
  radio.begin();
  radio.openWritingPipe(pipes[1]);
  radio.openReadingPipe(1, pipes[0]);
  radio.startListening();
  radio.printDetails();
  pinMode(3, OUTPUT); // LCD ekranýn arkaplan aydýnlatmasýna giden pini çýkýþ olarak ayarlýyoruz
  digitalWrite(3,HIGH); // Ekrana aydýnlatýldý
  myGLCD.InitLCD(); // Ekraný baþlatýldý
  myGLCD.setContrast(70); //0 ila 127 arasý bir deðer verilebilir
  myGLCD.setFont(SmallFont);
  EkranTemizle(); 
}
void EkranTemizle(){
  myGLCD.clrScr();
  String ek="";
  if(sonYazilanMesaj != sonGelenMesaj)
  {
    ek=" var"; 
  }else
  {
    ek=" yok";
  }
  myGLCD.print("*: msj "+ ek,CENTER,5);
  myGLCD.print("#: msj gonder",CENTER,15);
  Serial.println("Anasayfa açýldý");
}
void EkranUyari(String Message)
{
  myGLCD.clrScr();
  myGLCD.print("HATA",CENTER,5);
  myGLCD.print(Message,CENTER,20);
  delay(2000);
  EkranTemizle();
}
void EkranGonderildi()
{
  myGLCD.clrScr();
  myGLCD.print("BASARILI",CENTER,5);
  myGLCD.print("Mesaj",CENTER,20);
  myGLCD.print("Gonderildi",CENTER,25);
  delay(2000);
  EkranTemizle();
}


void loop()
{   
    basilanTus = tusTakimi.getKey(); 
    if (radio.available()) {
      // gelen verinin dizi sayýsý 1. Çünkü ya 0 gelecek yada 1 gelecek
      radio.read(&RCVmessage, sizeof(RCVmessage));
      sonGelenMesaj=String(RCVmessage);
      if(SerialState) {Serial.println("Alinan mesaj : " + String(RCVmessage)); }
      if(!EkranBildirimYazisi && sonYazilanMesaj != sonGelenMesaj && !BildirimEkrani && !MesajYaziliyor)
      {
        EkranBildirimYazisi=true;
        EkranTemizle();
        return;
      }
      delay(10);
    }
  
    if(MesajYaziliyor)
    {
      myGLCD.clrScr();
      myGLCD.print("* : sil",0,5);
      myGLCD.print("# : gonder",0,15);
      while(basilanTus != '#')
      {
        if(basilanTus != NO_KEY && Message.length() < 42){ 
          if(basilanTus=='*')
          {
            if(enkarakter == 0 && sira >23) { sira-=5; enkarakter=84;}
            if(enkarakter > 0) {enkarakter-=6;}
            myGLCD.print("_",enkarakter,sira);
            Message=Message.substring(0,Message.length()-1);
          }else {
            myGLCD.print(String(basilanTus),enkarakter,sira); 
            Message = Message + basilanTus; 
            enkarakter +=6;
            if(enkarakter > 78)
            {
             sira+=5;
             enkarakter=0;
            }
          }
        }
        basilanTus = tusTakimi.getKey();
      }
      MesajYaziliyor=false;
      
      if(Message.length()>0)
      {
        //MESAJ GONDERME BÖLÜMÜ KODLARI BURADA OLACAK
        Message.toCharArray(GonderilenMesaj,32); 
        Serial.println("Gonderilen Mesaj : " + Message); 
        EkranGonderildi();
      }else
      {
        EkranUyari("Bos mesaj!"); 
      }

      return;// gönderdikten sonra tekrar aþaðýdaki iflere giriyor yoksa
    }
    
    
    if(basilanTus=='#' && !MesajYaziliyor)
    { 
      Serial.println("Mesaj yazma ekraný açýldý.");
      Message="";
      enkarakter=0;
      sira=23;
      MesajYaziliyor=true; 
      return;
    }
    if(basilanTus=='*' && !MesajYaziliyor && !BildirimEkrani && sonYazilanMesaj != sonGelenMesaj)
    {
      Serial.println("Bildirim görüntüleniyor..");
      BildirimEkrani=true;
      EkranBildirimYazisi=false;
      myGLCD.clrScr();
      myGLCD.print("*:(X)",0,5);
      myGLCD.print("Gelen mesaj :",0,15);
      
      myGLCD.print(sonGelenMesaj.substring(0,14),0,20);
      if(sonGelenMesaj.length()>14)
      {
        myGLCD.print(sonGelenMesaj.substring(14,28),0,25);
      }

      sonYazilanMesaj=sonGelenMesaj;
      return;
    }
    if(BildirimEkrani==true && basilanTus=='*')
    {
      BildirimEkrani=false;
      EkranTemizle();
      return;
    }
    
    if (basilanTus != NO_KEY){
      //Serial.println(basilanTus);
    }

    MesajGonder(GonderilenMesaj); 
    delay(100);
}
void MesajGonder(char * message)
{
  radio.stopListening();
  radio.write( (uint8_t*)message,strlen(message));
  if(SerialState==true) {Serial.println("Gonderilen mesaj : " + String(message)); }
  radio.startListening();
}
