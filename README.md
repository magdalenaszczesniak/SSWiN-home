# SSWiN-home
Krok po kroku kod działania Systemu Sygnalizacji Włamania i Napadu
## Krok 1 
wczytanie bibliotek pomocniczych do działania programu 

#include <Keypad.h>                  //biblioteka od klawiatury
#include <LiquidCrystal_I2C.h>       // biblioteka wyświetlacza
#include "DHT.h"                      //biblioteka moduły DHT
#include <virtuabotixRTC.h> 
#include <SoftwareSerial.h>
#include <SPI.h>                                         
#include <MFRC522.h>                    // biblioteka czytnika RFID 
#include <Servo.h>                      // Servo



#define SS_PIN 53              // pin internetowy modułu czytnika RFID                        
#define RST_PIN 46             // pin komunikacyjny modułu czytnika RFID                        
#define smokeA0 29             // czujka dym
#define DHTTYPE DHT11         // typ czujnika  (DHT11)
#define BUZZER 36
#define KONTAKTRON_1 32
#define KONTAKTRON_2 34
#define CZUJNIK_KUCHNIA 13
#define CZUJNIK_SYPIALNIA 12
#define CZUJNIK_LAZIENKA 11
#define CZUJNIK_KORYTARZ 10
#define CZUJKA_TEMPERATURY 38
#define CZUJKA_ZALANIA 40
#define CZUJKA_WIBRACJI 35
#define LED_UZBROJENIE 24
#define LED_ALARM 22

SoftwareSerial mySerial(39,37);      //Tx & Rx
LiquidCrystal_I2C lcd1(0x27, 20, 4);
LiquidCrystal_I2C lcd2(0x26, 16,2);
MFRC522 mfrc522(SS_PIN, RST_PIN);   
Servo myServo;                          //  servo nazwa
DHT dht(CZUJKA_TEMPERATURY, DHTTYPE);     // definicja czujnika

///////////////moduł GSM ////////////
void wyslijsms(){

   mySerial.begin(9600);

   mySerial.println("AT");
  updateSerial();

  mySerial.println("AT+CMGF=1"); 
  updateSerial();
  mySerial.println("AT+CMGS=\"+48504077433\"");     //numer telefonu z przedrostkiem kierunkowym kraju
  updateSerial();
  mySerial.print("WLAMANIE");                     // wiadomość która ma zostać wysłana
  updateSerial();
  mySerial.write(26);
}

void updateSerial()
{
  delay(500);
  while (Serial.available()) 
  {
    mySerial.write(Serial.read());              //Prześlij otrzymany numer seryjny do portu szeregowego oprogramowania
  }
  while(mySerial.available()) 
  {
    Serial.write(mySerial.read());          

  }
}


////////////moduł_czasu_rzeczywistego//// 

// CLK -> 43 , DAT -> 45, Reset -> 47
virtuabotixRTC myRTC(43, 45, 47); 
char timeChar[8];
void dateTimeWrite() {
  lcd1.init();                      // inicjalizowanie LCD
  lcd1.backlight(); 
}
//////////////////////////////////////////

const byte ROWS = 4;                  // ile wierszy
const byte COLS = 4;                  //ile kolumn
 
byte rowPins[ROWS] = {5, 4, 3, 2};     //piny wierszy
byte colPins[COLS] = {6, 7, 8, 9};    //piny kolumn
 
char keys[ROWS][COLS] = {             //mapowanie klawiatury
  {'1','2','3','A'},
  {'4','5','6','B'},
  {'7','8','9','C'},
  {'*','0','#','D'}
};
 
Keypad klawiatura = Keypad( makeKeymap(keys), rowPins, colPins, ROWS, COLS );     //inicjalizacja klawiatury
 
volatile int stanAlarmu = 1;
int pinAlarmuPozycja = 1;
char pinCyfra1 = '1';
char pinCyfra2 = '2';
char pinCyfra3 = '3';
char pinCyfra4 = '4';
 
int ileCzasuMinelo = 0;

 
void setup() {
  Wire.begin();
  Serial.begin(9600);
  
  // myRTC.setDS1302Time(00, 29, 22, 7, 30, 01, 2023);      // sekundy/minuty/godzina/dzien/dzien/miesiac/rok
 
  pinMode(BUZZER, OUTPUT);  
  pinMode(KONTAKTRON_1, INPUT_PULLUP);
  pinMode(KONTAKTRON_2, INPUT_PULLUP);
  pinMode(CZUJNIK_KUCHNIA, INPUT_PULLUP);
  pinMode(CZUJNIK_SYPIALNIA, INPUT_PULLUP);
  pinMode(CZUJNIK_LAZIENKA, INPUT_PULLUP);
  pinMode(CZUJNIK_KORYTARZ, INPUT_PULLUP);
  pinMode(CZUJKA_TEMPERATURY, INPUT_PULLUP);
  pinMode(CZUJKA_ZALANIA, INPUT_PULLUP);
  pinMode(CZUJKA_WIBRACJI, INPUT);
  pinMode(LED_UZBROJENIE, OUTPUT);
  pinMode(LED_ALARM, OUTPUT);
  pinMode(smokeA0, INPUT);
  
  digitalWrite(LED_UZBROJENIE, LOW);
  digitalWrite(LED_ALARM, LOW);
  
  lcd1.begin(20, 4);
  lcd1.backlight();
  lcd1.setCursor(0, 0);
  lcd1.print("Alarm: ");
  lcd1.setCursor(0, 1);
  lcd1.print("Stan: ");
 
  SPI.begin();                                            // magistarala SPI
  mfrc522.PCD_Init();                               // MFRC522
  myServo.attach(23);                                // servo pin
  myServo.write(0);                                  // pozycja startowa servo
  
  lcd2.init();
  lcd2.begin(16, 2);
  lcd2.backlight();
  lcd2.setCursor(0, 0);
  lcd2.print("PROSZE ZBLIZYC");
  lcd2.setCursor(0, 1);
  lcd2.print("KARTE");
  delay(500);
}

// czytnik rfid////

 void RFID(){
   if ( ! mfrc522.PICC_IsNewCardPresent()){        // Szukaj nowych kart
      return;
  }
    if ( ! mfrc522.PICC_ReadCardSerial()) {        // Wybierz jedną z kart
      return;
  }
 String content= "";                                         
 byte letter;                                                     
 for (byte i = 0; i < mfrc522.uid.size; i++){                                     
     content.concat(String(mfrc522.uid.uidByte[i] < 0x10 ? " 0" : " "));  
     content.concat(String(mfrc522.uid.uidByte[i], HEX));                    
  }
   content.toUpperCase();                              
 if (content.substring(1) == "20 65 62 22") {           // UID karty która ma dostęp
    lcd2.clear();
    lcd2.setCursor(0,0);                                    // Ustawienie kursora w pozycji 0,0 (pierwszy wiersz, pierwsza kolumna)
    lcd2.print("PROSZE WEJSC");
    stanAlarmu = 1;                                         //Wszystkie 4 cyfry kodu sa poprawne
      
    digitalWrite(BUZZER, LOW); 
    digitalWrite(LED_ALARM, LOW); 

  myServo.write(180);                               
  delay(5000);                                           //  50 ms
  myServo.write(0);  
  lcd2.setCursor(0, 0);
  lcd2.print("PROSZE ZBLIZYC");
  lcd2.setCursor(0, 1);
  lcd2.print("KARTE");
  } else {
  lcd2.clear();
  lcd2.setCursor(0,0);                            // Ustawienie kursora w pozycji 0,0 (pierwszy wiersz, pierwsza kolumna)
  lcd2.print("NIEZGODNY UID");
  delay(5000);
  lcd2.setCursor(0, 0);
  lcd2.print("PROSZE ZBLIZYC");
  lcd2.setCursor(0, 1);
  lcd2.print("KARTE");                                   
  }
}



/// czujka wibracji////

long TP_init(){
  delay(10);
  long measurement=pulseIn (CZUJKA_WIBRACJI, HIGH);         //poczekaj, aż pin stanie się WYSOKI i zwróci pomiar
  return measurement;
}
//////////// moduł czasu rzeczywistego////////////

void displayTime() {
   myRTC.updateTime();
 lcd1.setCursor(0,3);
 lcd1.print("Data: ");
 lcd1.print(myRTC.dayofmonth);                      //Możliwość przełączania między dniem a miesiącem
 lcd1.print("/");
 lcd1.print(myRTC.month);
 lcd1.print("/");
 lcd1.print(myRTC.year);

 lcd1.setCursor(0,2);
 sprintf(timeChar, "Czas: %02d:%02d:%02d",myRTC.hours, myRTC.minutes, myRTC.seconds);
 lcd1.print(timeChar);                           
}

 
void loop() {
  
   RFID();
// moduł czasu rzeczywsitego 
  displayTime();                                  // wyświetlanie czasu rzeczywistego
  char klawisz = 0;                               //zmienna do przetrzymywania znakow z klawiatury
 
   switch(stanAlarmu) {                            //Wykonywania akcji odpowiedniej dla danego stanu
    case 1:
      //Uzbrajanie
     klawisz = klawiatura.getKey();

     if (klawisz == 'A') {          //STREFA 1
             // alaram uzbrojony 
        digitalWrite(LED_UZBROJENIE, HIGH);         // dioda świeci na zielono
        lcd1.setCursor(7, 0);
        lcd1.print("uzbrojony    ");                // napis "uzbrojony" na wyświetlaczu LCD 
        lcd1.setCursor(7, 1);
        lcd1.print("czuwanie");                     // napis "czuwanie" na wyświetlaczu LCD 
        stanAlarmu = 2; 
        delay(10000);                               // opóznienie 10 sekund
     } else if (klawisz == 'B') {     //STREFA 2
             // alaram uzbrojony 
        digitalWrite(LED_UZBROJENIE, HIGH);         
        lcd1.setCursor(7, 0);
        lcd1.print("uzbrojony    ");                
        lcd1.setCursor(7, 1);
        lcd1.print("czuwanie     ");                
        stanAlarmu = 5; 
        delay(10000);                               
      } else if (klawisz == 'C') {        //STREFA 3
             // alaram uzbrojony 
        digitalWrite(LED_UZBROJENIE, HIGH);         
        lcd1.setCursor(7, 0);
        lcd1.print("uzbrojony    ");               
        lcd1.setCursor(7, 1);
        lcd1.print("czuwanie    ");
        stanAlarmu = 6;
        delay(10000); /// opóznienie 10 sekund
      } else if (klawisz == 'D') {        //STREFA 1,2,3
             // alaram uzbrojony 
        digitalWrite(LED_UZBROJENIE, HIGH);  
        lcd1.setCursor(7, 0);
        lcd1.print("uzbrojony    ");              
        lcd1.setCursor(7, 1);
        lcd1.print("czuwanie    ");
        stanAlarmu = 2,5,6; 
        delay(10000); /// opóznienie 10 sekund
      }else{
        digitalWrite(LED_UZBROJENIE, LOW);
        lcd1.setCursor(7, 0);
        lcd1.print("nie uzbrojony");              // napis "nie uzbrojony" na wyświetlaczu LCD 
        lcd1.setCursor(7, 1);
        lcd1.print("czuwanie");
        
      }
        
    break;
    
    case 2:
      //STREFA 1
      
      if (digitalRead(CZUJNIK_KUCHNIA) == HIGH) {
       stanAlarmu = 4; //Natychmiast uruchamiamy alarm
      } else if 
      (digitalRead(KONTAKTRON_1) == HIGH) {
       stanAlarmu = 4; //Natychmiast uruchamiamy alarm
      } else if (digitalRead(KONTAKTRON_2) == HIGH) {
        stanAlarmu = 4; //Natychmiast uruchamiamy alarm
   }    
    break;
    
    case 3:
      //Rozbrajanie
      klawisz = klawiatura.getKey();
      if (klawisz) {
        //Czy kolejna podana cyfra jest poprawna?
        if (pinAlarmuPozycja == 1 && klawisz == pinCyfra1) {        //Jesli sprawdzamy 1 pozycje PINu
          pinAlarmuPozycja++;                                       //Cyfra poprawna, mozna sprawdzic na kolejna
        } else if (pinAlarmuPozycja == 2 && klawisz == pinCyfra2) { //Jesli sprawdzamy 2 pozycje PINu
          pinAlarmuPozycja++;                                       //Cyfra poprawna, mozna sprawdzic na kolejna         
        } else if (pinAlarmuPozycja == 3 && klawisz == pinCyfra3) { //Jesli sprawdzamy 3 pozycje PINu
          pinAlarmuPozycja++;                                       //Cyfra poprawna, mozna sprawdzic na kolejna        
        } else if (pinAlarmuPozycja == 4 && klawisz == pinCyfra4) { //Jesli sprawdzamy 4 pozycje PINu
            stanAlarmu = 1;                                         //Wszystkie 4 cyfry kodu sa poprawne
            pinAlarmuPozycja = 1;                                  //Resetujemy informacje o wpisywanym pinie   
            digitalWrite(BUZZER, LOW); 
            digitalWrite(LED_ALARM, LOW); 
        } else {
           stanAlarmu = 4;                                      //Blad w kodzie PIN - wlacz alarm
           pinAlarmuPozycja = 1;                                //Resetujemy informacje o wpisywanym pinie 
        }
      }
      
    break;
    
    case 4:
      //Sygnalizacja alarmu
       
        digitalWrite(LED_ALARM, HIGH);               //Dioda świeci na czerwono 
        digitalWrite(BUZZER, HIGH); 
        lcd1.setCursor(7, 0);
        lcd1.print("uzbrojony    ");              // napis "uzbrojony" na wyświetlaczu LCD 
        lcd1.setCursor(7, 1);
        lcd1.print("intruz       ");
        stanAlarmu = 3;                           //Szansa na rozbrojenie
        wyslijsms();

      
      delay(100);
      
    break;
    
     case 5:
      //STREFA 2
      
      if// (digitalRead(CZUJNIK_SYPIALNIA) == HIGH) {
      // stanAlarmu = 4; //Natychmiast uruchamiamy alarm
     // } else if (digitalRead(CZUJNIK_LAZIENKA) == HIGH) {
    //   stanAlarmu = 4; //Natychmiast uruchamiamy alarm
   //  } else if 
     (digitalRead(CZUJNIK_KORYTARZ) == HIGH) {
       stanAlarmu = 4; //Natychmiast uruchamiamy alarm
      } 
      
      
    break; 
    case 6:
      //STREFA 3
      
      long measurement =TP_init();
      delay(50);

      dht.begin();            // inicjalizacja czujnika
      
      float t = dht.readTemperature();
                
      if (digitalRead(CZUJKA_ZALANIA) == HIGH) {
       stanAlarmu = 4; //Natychmiast uruchamiamy alarm
       } else if (measurement > 1000) {
       stanAlarmu = 4; //Natychmiast uruchamiamy alarm
       }else if (t > 30) {
       stanAlarmu = 4; //Natychmiast uruchamiamy alarm
       } else if(analogRead(smokeA0) > 415) {
       stanAlarmu = 4; //Natychmiast uruchamiamy alarm
     }
    break;  
}
}
