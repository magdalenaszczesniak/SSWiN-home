# SSWiN-home
Opracowany projekt Systemu Sygnalizacji Włamania i Napadu z założenia zabezpiecza mały domek letniskowy. Schemat ideowy podłączenia wszystkich czujek, modułów oraz komponentów, do których odnosi się przedstawiony w etapach kod. Połączenia zostały dopasowane do makiety i rozmieszczenia na niej elementów. 

![image](https://github.com/user-attachments/assets/2234121b-90b0-4a57-9d46-e135397f52d6)
Tabela przedstawiająca kosztorys systemu
![Lp_page-0001](https://github.com/user-attachments/assets/22013338-29cb-4d57-a038-fc94b7285323)

Krok po kroku kod działania Systemu Sygnalizacji Włamania i Napadu
## Krok 1 
Wczytanie bibliotek pomocniczych do działania programu 
```
#include <Keypad.h>                  //biblioteka od klawiatury
#include <LiquidCrystal_I2C.h>       // biblioteka wyświetlacza
#include "DHT.h"                      //biblioteka moduły DHT
#include <virtuabotixRTC.h> 
#include <SoftwareSerial.h>
#include <SPI.h>                                         
#include <MFRC522.h>                    // biblioteka czytnika RFID 
#include <Servo.h>                      // Servo
```
## Krok 2 
Zdefiniowanie stałych, które określają numery pinów mikrokontrolera, pod które są podłączone poszczególne elementy systemu takie jak moduł czytnika kart RFID, czujnik dymu, czujnik temperatury i wilgotności, buzzer, kontaktron nr 1, kontaktron nr 2, czujnik ruchu w kuchni, czujnik ruchu w sypialni czujnik ruchu w łazience, czujnik ruchu w korytarzu, czujka temperatury, czujka zalania, czujka wibracji oraz LED uzbrojenia i alarmu. 
```
#define SS_PIN 53              // pin internetowy modułu czytnika RFID                        
#define RST_PIN 46             // pin komunikacyjny modułu czytnika RFID                       #define smokeA0 29             // czujka dym
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
```
## Krok 3
Utworzenie sześciu obiektów: 
o	mySerial - obiekt klasy SoftwareSerial do komunikacji przez port szeregowy UART przez piny 39 i 37,
o	mfrc522 - obiekt klasy MFRC522 do komunikacji z modułem czytnika kart RFID
o	myServo - obiekt klasy Servo obsługi serwomechanizmu,
o	lcd1 - obiekt klasy LiquidCrystal_I2C do obsługi wyświetlacza LCD z interfejsem I2C o adresie 0x27, o szerokości 20 znaków i 4 wierszach,
o	lcd2 - obiekt klasy LiquidCrystal_I2C do obsługi wyświetlacza LCD z interfejsem I2C o adresie 0x26, o szerokości 16 znaków i 2 wierszach,
o	dht - obiekt klasy DHT do obsługi czujnika temperatury i wilgotności DHT.
```
SoftwareSerial mySerial(39,37);      //Tx & Rx
LiquidCrystal_I2C lcd1(0x27, 20, 4);
LiquidCrystal_I2C lcd2(0x26, 16,2);
MFRC522 mfrc522(SS_PIN, RST_PIN);   
Servo myServo;                          //  servo nazwa
DHT dht(CZUJKA_TEMPERATURY, DHTTYPE);     // definicja czujnika
```
## Krok 4
Zdefiniowanie funkcji ,,wyslijsms ()”, która będzie realizować wykonanie powiadomienia SMS-a.

```
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
```
## Krok 5
Zdefiniowanie funkcji „updateSerial ()”, która będzie realizować przesyłanie danych między portem szeregowym mikrokontrolera a portem szeregowym software.

```
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
```
## Krok 6
Zdefiniowanie funkcji ,,dateTimeWrite ()”, która służy do inicjalizacji wyświetlacza lcd1 i wyświetlenia na nim aktualnej daty i godziny.

```
// CLK -> 43 , DAT -> 45, Reset -> 47
virtuabotixRTC myRTC(43, 45, 47); 
char timeChar[8];
void dateTimeWrite() {
  lcd1.init();                      // inicjalizowanie LCD
  lcd1.backlight(); 
}
```
## Krok 7
Zdefiniowanie stałych ,,ROWS” i ,,COLS”, określających ilość wierszy 
i kolumn klawiatury oraz tablic ,,rowPins” i ,,colPins”, określających numery pinów mikrokontrolera, pod które są podłączone wiersze i kolumny klawiatury.

```
const byte ROWS = 4;                  // ile wierszy
const byte COLS = 4;                  //ile kolumn
 
byte rowPins[ROWS] = {5, 4, 3, 2};     //piny wierszy
byte colPins[COLS] = {6, 7, 8, 9};    //piny kolumn
```
## Krok 8
Zdefiniowanie tablicy ,,keys”, określającej mapowanie klawiatury oraz obiektu typu Keypad o nazwie ,,keypad”, który będzie służył do obsługi klawiatury.
```
char keys[ROWS][COLS] = {             //mapowanie klawiatury
  {'1','2','3','A'},
  {'4','5','6','B'},
  {'7','8','9','C'},
  {'*','0','#','D'}
};

Keypad klawiatura = Keypad( makeKeymap(keys), rowPins, colPins, ROWS, COLS );     //inicjalizacja klawiatury
```
## Krok 9
Zadeklarowanie zmiennej ,,stanAlarmu”, ,, pinAlarmuPozycja”, ,,pinCyfra1”, pinCyfra2”, pinCyfra3”, pinCyfra4”, ,,ileCzasuMinelo” i przypisanie im odpowiednich 
wartości.
```
volatile int stanAlarmu = 1;
int pinAlarmuPozycja = 1;
char pinCyfra1 = '1';
char pinCyfra2 = '2';
char pinCyfra3 = '3';
char pinCyfra4 = '4';
 
int ileCzasuMinelo = 0;
```
## Krok 10
Zdefiniowanie funkcji ,,setup()”. Funkcja zawiera:
o	ustawienia na zegarze RTC  (Real-Time Clock), który został zainicjowany wcześniej jako obiekt „myRTC”, dane są przedstawione zgodnie z podanymi argumentami (sekundy/minuty/godzina/dzień/dzień/miesiąc/rok),
o	ustawienia trybu pracy dla pinów cyfrowych w mikrokontrolerze jako wejścia bądź wyjścia,
o	ustawione są także początkowe stany poszczególnych pinów,
o	inicjowane są również dwa wyświetlacze LCD, a także moduł RFID oraz serwomechanizm.
```
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
```
## Krok 11
Zdefiniowanie funkcji void RFID(), która obsługuje czytnik RFID. Funkcja ta sprawdza, czy nowa karta została zbliżona do czytnika. Jeśli tak, to zostaje odczytany jej unikalny identyfikator (UID) i porównuje go z zaprogramowanym w kodzie. Jeśli są zgodne, to zmienia stan alarmu na włączony, wyłącza dźwięk alarmu, włącza diody LED i umożliwia wejście na podstawie karty RFID. W przeciwnym wypadku wyświetla odpowiedni komunikat na wyświetlaczu LCD [15].
```
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
```
## Krok 12
Zmienne dotyczące czujki wibracji 
```
long TP_init(){
  delay(10);
  long measurement=pulseIn (CZUJKA_WIBRACJI, HIGH);         //poczekaj, aż pin stanie się WYSOKI i zwróci pomiar
  return measurement;
}
```
## Krok 13
Zdefiniowanie funkcji ,, displayTime()” odnoszącej się do działa modułu czasu rzeczywistego. Krok po kroku, działanie funkcji wygląda następująco:
o	Funkcja ,,updateTime” obiektu ,,myRTC” działa wykorzystuje do aktualizacji informacji o dacie i czasie pobranych ze źródła zegara RTC,
o	Ustawiany jest kursor na pierwszą kolumnę dodatkowego wiersza ekranu LCD.
o	Na ekranie wyświetlacza jest informacja ,,Data:” oraz dzień, miesiąc i rok pobrane z obiektu ,,myRTC”.
o	Ustawiany jest kursor na pierwszą kolumnę drugiego wiersza ekranu LCD.
o	Godzina, minuta i sekunda są pobierane z obiektu ,,myRTC” i sformatowane do postaci 00:00:00. Wynik jest zapisywany we wszystkich znakach o nazwie ,,timeChar”,
o	Na wyświetlaczu wyświetlana jest informacja ,,Czas:” oraz sformatowana godzina, minuta i sekunda, a następnie wprowadzono ,,timeChar”.
```
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
```
## Krok 14
Początek pętli głównej ,,loop()” która działa w nieskończonej pętli. W pierwszej kolejności wczytana jest funkcja ,,RFID()”, która została zdefiniowana wcześniej. Następnie wprowadzono rzeczywisty czas za pomocą funkcji displayTime(). Kolejny krokiem jest stworzenie zmiennej ,,klawisz”, do której przypisywana jest wartość zwrócona przez wykonanie ,,getKey()” z obiektu ,,klawiatura”. Dalej rozpoczyna się instrukcja ,,switch” która służy do podejmowania decyzji wyłącznie na podstawie wartości jednej zmiennej, podana w kodzie zmienna to ,,stanAlarmu”. Jeśli ,,stanAlarmu” ma wartość 1, wykonywana jest ,,case 1”. W tej sekcji sprawdzane są kolejne naciśnięcia klawiszy A, B, C, D na klawiaturze. Każdy z klawiszy określa strefę systemu, klawisz A załącza strefę pierwszą, B jest to strefa, C jest to trzecia strefa zaś D to strefa czwarta, która jest uruchomieniem wszystkich stref. Jeśli któryś z nich zostanie naciśnięty, dioda LED zapala się, a na wyświetlaczu pojawi się napis „uzbrojony” oraz „czuwanie”. Następnie ,,stanAlarmu” przypisywana jest wartość (2, 5 lub 6) i opóźnienie 10 sekund. Jeśli żaden z klawiszy nie zostanie naciśnięty, dioda LED nie zostanie zapalona, a na wyświetlaczu pojawi się napis „nie uzbrojony” oraz „czuwanie”. 

```
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
```
## Krok 15
Jeśli zmienna ,,stanAlarm” ma wartość 2, to znaczy, że alarm w strefie 1 jest uzbrojony 
i oczekuje na rozbrojenie. W takiej sytuacji sprawdzane są stany logiczne pinów połączonych z czujnikiem ruchu w kuchni lub kontaktronów. Jeśli stan na któryś z tych pinów jest wysoki (HIGH), oznacza to, że wystąpił ruch w pomieszczeniu i alarm zostaje natychmiast uruchomiony (stanAlarm ustawiony na 4) 
```
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
```
## Krok 16
Jeśli zmienna ,,stanAlarm” ma wartość 3, to znaczy, że alarm jest uruchomiony i jest możliwość rozbrojenia systemu poprzez wprowadzenie kodu PIN. W takiej sytuacji wczytywany jest zapisany kod. Następnie sprawdzane są kolejne cyfry kodu PIN, czy są zgodne. Jeśli tak, to ,,pinAlarm” pozycja zwiększa się o 1 i sprawdzana jest następna cyfra. Jeśli wszystkie 4 cyfry są równe, alarm jest rozbrojony (stanAlarm ustawiony na 1). Zaś w przypadku, gdy kod nie jest zgodny alarm jest wciąż uruchomiony (stanAlarm ustawiony na 4).
```
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
 ```
## Krok 17
Jeśli zmienna stanAlarm ma wartość 4, to znaczy, że alarm jest uruchomiony i jest włączona sygnalizacja alarmu. W takiej sytuacji uruchamiany jest buzzer oraz dioda LED informująca o stanie alarmu. W tym samym czasie na wyświetlaczu LCD pojawia się napis "uzbrojony" i "intruz". Alarm przechodzi do stanu 3 (oczekuje na wprowadzenie kodu PIN). W tym samym czasie wysyłany jest SMS z informacją o alarmie.
```
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
 ```
## Krok 18
Jeśli zmienna ,,stanAlarm” ma wartość 5, to znaczy, że alarm w strefie 2 jest uzbrojony 
i oczekuje na rozbrojenie. W takiej sytuacji sprawdzane są stany logiczne pinów połączonych z czujnikiem ruchu w sypialni, kuchni oraz korytarzu. Jeśli stan na którymkolwiek z tych pinów jest wysoki (HIGH), oznacza to, że wystąpił ruch 
w pomieszczeniu i alarm natychmiast się uruchamia.
```
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
```
## Krok 19
Jeśli zmienna ,,stanAlarm” ma wartość 6, to znaczy, że alarm w strefie 3 jest uzbrojony 
i oczekuje na rozbrojenie (Rys. 46.). Opis działania kodu:
o	inicjalizowany jest czujnik temperatury i wilgotności (wywoływana jest funkcja ,,TP_init” oraz ,,dht.begin”),
o	odczytywana jest aktualna temperatura za pomocą raportu DHT (zapisywana 
w wyniku t),
o	sprawdzany jest stan logiczny pinu dotyczącego zalania. Jeśli jest wysoki na (HIGH), oznacza to, że wystąpiło zalanie i alarm zostaje natychmiast uruchomiony (stanAlarm ustawiony na 4),
o	następnie sprawdzana jest aktualna temperatura, jeśli wartość temperatury jest większa niż 1000, oznacza to, że utrzymuje się bardzo wysoka temperatura i alarm zostaje natychmiast uruchomiony (stanAlarm ustawiony na 4),
o	kolejno sprawdzana jest aktualna temperatura (odczytana za pomocą czujnika DHT). Jeśli jest ona większa niż 30 stopni, oznacza to, że jest bardzo wysoka temperatura i alarm zostają natychmiast uruchomiony (stanAlarm ustawiony na 4).
```
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
```
