## Protokoll 5

**Name:** Sarah Vezonik  

**Datum, Ort:** 04. Juni 2019, Kaindorf  

**Gruppe:** 3  

**Anwesend:** Vezonik, Vollmaier, Wegl, Wesonig, Winter M., Winter T.;  



----------
### **Inhaltsverzeichnis**  

[1]  Wiederholung Modbus  

[2]  Programm Temperatursensor  



----------

### [1] Wiederholung Modbus



Der Kommunikationsablauf beruht auf einem Server/Client Prinzip. Der Client (zum Beispiel ein PC) sendet einen Request zum Server (zum Beispiel ein Aktor oder Sensor). Dieser antworter mit einer Response.


----------

### **Modbus ASCII:**


Rein textuelle byteweise Übertragung von Daten. 

### Modbus ASCII Frame

#### **Protokollaufbau:**

| First Header  | Second Header | Funktion | Daten |LR - Check | Ende
| ------------- | ------------- |
| 1 Zeichen (:) | 2 Zeichen  | 2 Zeichen | n Zeichen | 2 Zeichen |2 Zeichen (CRLF)

Im ASCII-Modus beginnen Nachrichten mit einem vorangestellten Doppelpunkt, das Ende der Nachricht wird durch die Zeichenfolge Carriage return – Line feed (CRLF) markiert.

Die ersten zwei Bytes enthalten zwei ASCII-Zeichen, die die Adresse des Empfängers darstellen. Der auszuführende Befehl ist auf den nächsten zwei Bytes codiert. Über weitere n Zeichen folgen die Daten. Über das gesamte Telegramm (ohne Start- und Ende-Markierung) wird zur Fehlerprüfung ein LRC ausgeführt, dessen Paritätsdatenwort in den abschließenden zwei Zeichen untergebracht wird. Tritt während der Übertragung eines Frames eine Pause von > 1s auf, wird der Frame als Fehlerfall bewertet. Der Benutzer kann ein längeres Timeout konfigurieren.   

 
#### **Datenmodell**

**Discrete Inputs**
    Ein Discrete Input ist ein einzelnes Bit, das nur gelesen werden kann.
    Beispiele: ein Taster, ein Endschalter, ...
    **Coils**
    Eine Coil ist ein Bit das gelesen und beschrieben werden kann.
    Der Name stammt vermutlich von der Spule eines Relais.
    Beispiele: ein Relais, eine LED, ...
    **Input Registers**
    Ein Input-Register ist ein 16-Bit Wert der nur gelesen werden kann.
    Beispiele: ein Temperatursensor, ein ADC, die Geräte-ID, ...
    **Hold-Registers**
    Ein Hold-Register ist ein 16-Bit Wert der gelesen und beschrieben werden kann.
    Beispiele: PWM-Einheit, DAC, ...

![Datenmodell](https://lms.at/dotlrn/classes/htl_elektrotechnik/610437.4AHME_LA1SX.18_19/xolrn/download/file/2148F16AC6F2E.symlink/038028026F2A2/590CCAAE32846/en:C28C6FCA82C8B/modbus_addressing_model_png)

#### **Beispiel:**
 

    :|01|04|0000|0001|_ _|


`:`  =  **Start Frame**  

`01` = **Adresse des Geräts am Bus**  


Jedes Modbus-Gerät hat eine eindeutige Adresse im Bereich 1 bis 247. Die Adresse 0 wird für eine Broadcast-Request verwendet, die Adressen 248 bis 255 sind reserviert und dürfen nicht verwendet werden.

`04` = **Read Input Register**    

![enter image description here](https://www.picotech.com/images/uploads/library/topics/_med/modbus-function-codes-examples.png)

`0000`= **Inputregister 1 für die Temperatur**  

`0001` = **Anzahl der gewählten Input Register**  

`_ _` = **LRC/Prüfsumme**  

Am Ende des Frame wird ein 8-Bit LRC (ein Byte als Hexstring = 2 ASCII Zeichen) gesendet.
Dazu werden alle Bytes des Frames exklusive dem Start ':' und dem Ende (CR + LF) mit 8-Bit Additionen ohne Berücksichtigung des Überlaufs zusammenaddiert und am Ende einem Zweierkomplement unterzogen.  

`<CR><LF>`=  **End-Frame**  

Der folgende Datenframe wurde verwendet um die Antwort des Microcontrollers zum PC zu senden.

    :010402xxxx _ _ <CR><LF>


### [1] Programm Temperatursensor

### Aufgabenstellung
Ziel der Stunde war es, die bereits funktionstüchtige Temperaturmesseinheit zu verbessern und neu zu programmieren. Verwendet wurde dazu der integrierte Temperatursensor des Atmega 328p des Arduino Nano Boards.

### **app.c**

```
  24 void app_init (void) {
  25   memset((void *)&app, 0, sizeof(app));
  26   
  27   ADMUX = 8; // Multiplexer ADC8 = Temp.
  28   ADMUX |= (1 << REFS1) | (1 << REFS0); // VREF=1.1V
  29   ADMUX |= (1 << ADLAR); // Left Adj, -> Result in ADCH
  30   
  31   ADCSRA = (1 << ADEN) | 7; // fADC=125kHz
  32   ADCSRB = 0;
  33   
  34 }
```
`ADMUX = 8` Damit setzen wir den Multiplexer ADC auf den wert 8 wo der Temperatursensor liegt.  

`ADMUX |= (1<<REFS0) | (1<<REFS1)` Damit setzten wir die Referenzspannung auf die internen 1.1V.  

`ADMUX |= (1<<ADLAR)` Damit wird das Ergebnis linksbündig ausgegeben.  

`ADCSRA = (1<<ADEN) | 7` Damit setzt man die Frequenz des ADC, in unserem Fall auf 125kHz.  

`ADCSRB = 0;` Wird zur Sicherheit  deaktiviert.  
  
  


```
  39  uint16_t app_inc16BitCount (uint16_t cnt) {
  40     // sys_setLed(1);
  41     return cnt == 0xffff ? cnt : cnt + 1;
  42 }
```
**Counter:** 
Ist der Wert 0xffff gibt er cnt zurück, ansonsten cnt + 1

```
  44 uint8_t hex2int (char h) {
  45     if (h >= '0' && h <= '9') {
  46         return h - '0';
  47     }
  48     if (h >= 'A' && h <= 'F') {
  49         return h - 'A' + 10;
  50     }
  51     return 0xff; // Fehler
  52 }
```
Kontrolle, ob der Wert ein Hexwert ist. ( 0 - F )  
Funktion zum Umwandeln von Hex in Integer, wenn h zwischen 0 und 9 liegt springt man in das if
```
  54 uint8_t app_handleModbusRequest () {
     
  57     // Check if valid modbus frame
  58     char *b = app.modbusBuffer;
  59     uint8_t size = app.bufIndex;
  60     
  61     if (size < 9) { return 1; }
  62     if (b[0] != ':') { return 2; }
  63     if (b[size - 1] != '\n') { return 3; }
  64     if (b[size - 2] != '\r') { return 4; }
  65     if ( (size - 3) % 2 != 0) { return 5; }
  66     for (uint8_t i = 1; i < (size - 2); i++) {
  67         char c = b[i];
  68         if (! ((c >= '0' && c <= '9') || 
  69                (c >= 'A' && c <= 'F')))
  70             return 6;
  71     }
 
```
Verwendung der Variablen aus der Struktur
Kontrolle, ob beim Übertragen ein Fehler passiert ist. Es wird somit kontrolliert ob es dem Frame des ASCII enspricht. (Startbit, Stopbit...)
  
```
  73     uint8_t lrc = 0;
  74     for (uint8_t i = 1; i < (size - 4); i++) {
  75         lrc += b[i];
  76     }
  77     lrc = (uint8_t)( -(int8_t)lrc);
  78     char s[3];
  79     snprintf(s, sizeof s, "%02X", lrc);
  80     if (b[size - 4] != s[0]) { return 7; }
  81     if (b[size - 3] != s[1]) { return 7; }
  82     
  83     // printf("Request richtig\n");
```
**Kontrolle LRC**  

Alle Werte, angefangen beim Startbit werden in der for Schleife gezählt. Anschließend wird das Zweierkompliment der Zahl gebildet und gespeichert. Danach wird der Wert mit dem LRC mit dem Frame verglichen.
```
  84     uint8_t i, j;
  85     for (i = 1, j = 0; i < (size - 4); i += 2, j++ ) {
  86         uint8_t hn = hex2int(b[i]);
  87         uint8_t ln = hex2int(b[i+1]);
  88         if (hn == 0xff || ln == 0xff) {
  89             return 8;
  90         }
  91         uint8_t value = hn * 16 + ln;
  92         b[j] = value;
  93     }
  94     size = j;

``` 
Sollten HighnNible oder LowNible 0xff sein, wird mit `return 8` der Fehler gezeigt





```
 144 void app_handleUartByte (char c) {
 145     if (c == ':') {
 146         if (app.bufIndex > 0) {
 147             app.errCnt = app_inc16BitCount(app.errCnt);
 148         }
 149         app.modbusBuffer[0] = c;
 150         app.bufIndex = 1;
 151     
 152     } else if (app.bufIndex == 0) {
 153         app.errCnt = app_inc16BitCount(app.errCnt);
 154  
 155     } else if (app.bufIndex >= (sizeof(app.modbusBuffer))) {
 156         app.errCnt = app_inc16BitCount(app.errCnt);
 157     
 158     } else {
 159         app.modbusBuffer[app.bufIndex++] = c;
 160         if (c == '\n') {
 161             uint8_t errCode = app_handleModbusRequest();
 162             if (errCode > 0) {
 163                 // printf("Fehler %u\n\r", errCode);
 164                 app.errCnt = app_inc16BitCount(app.errCnt);
 165             }
 166             app.bufIndex = 0;
 167         }
 168     } 
 169 }
```
Abfrage bzw. Kontrolle, ob der Frame zu Ende ist.


Zuerst wird geprüft ob das ertse ankommende Byte ein `Startbyte :` ist. Ist dies nicht der Fall, sollte im ErrorCount hochgezählt werden.
Wenn im Buffer bereits ein Startbyte : vorhanden ist, darf kein Zweites innerhalb der Request geschrieben werden, da dies sonst zu Fehlern führen könnte. Falls ein solcher Fehler auftritt, sollte in einer Funktion ein ErrorCount hochgezählt werden
Tritt beim Beenden der Request mit \n ein Fehler auftritt, so soll auch der ErrorCount hochgezählt werden.
Ist nach dem Übertragen der ErrorCount größer als 0, so soll die Anzahl der Fehler ausgegeben werden
```
 171 void app_main (void) {
 172     
 173     ADCSRA |= (1 << ADSC);
 
 185     float k1 = 1100.061, d1 = -73992.464;
 186     float k2 = 1002.611, d2 = -66870.812;
 187 

 194     float o1 = -17180.9;
 195     float o2 = -15726.956;
 196     
 197     float mbr;
 198     uint8_t adch = ADCH;
 199     // adch = 88;
 200     if (adch <= 88) {
 201         mbr = k1 * adch + d1 + o1;
 202     } else {
 203         mbr = k2 * adch + d2 + o2;
 204     }
 205     int16_t mbInputReg01 = (int16_t)mbr;
 206     int8_t vk = mbInputReg01 / 256;
 207     uint8_t nk = ((mbInputReg01 & 0xff) * 100) / 256;
 208          
 209     app.mbInputReg01 = (uint16_t)mbInputReg01;
 210     
 213     int c = fgetc(stdin);
 214     if (c != EOF) {
 215         // printf("\r\n %02x\r\n", (uint8_t)c);
 216         app_handleUartByte((char) c);
 217     } 
 218     
 219 }
```

    
 **Gerade:**  
 ModbusRegister = k * ADCH + d
 **aus Datenblatt:**
   -45°C -> 242mV -> ADCH=56.79 -> MBR = -45*256 = -11520
   25°C -> 314mV -> ADCH=73.08 -> MBR =  25*256 =   6400
   85°C -> 380mV -> ADCH=88.40 -> MBR =  85*256 =  21760
 daraus ergibt sich für:
   <=25°C -> MBR = k1 * ADCH + d1
   =>25°C  -> MBR = k2 * ADCH + d2


reale Messung bei 22°C -> ADCH=88
**interpoliert:**
22°C -> ?  -> ADCH=72.38 -> MBR = 22*256 = 5632
 Offsetkorrektur um bei ADCH=88 auf 5632 zu kommen
  ADCH<=88 -> MBR = k1 * ADCH + d1 + o1
ADCH>88  -> MBR = k2 * ADCH + d2 + o2

```
 223 void app_task_1ms (void) {
 224     static uint16_t oldErrCnt = 0;
 225     static uint16_t timer = 0;
 226     if (app.errCnt != oldErrCnt) {
 227         oldErrCnt = app.errCnt;
 228         timer = 2000;
 229         sys_setLed(1);
 230     }
 231     if (timer > 0) {
 232         timer--;
 233         if (timer == 0) {
 234             sys_setLed(0);
 235         }
```
**LED Fehler**  

Diese LED sollte aber nicht dauerhaft leuchten, da man sonst nur den ersten Fehler erkennen könnte. Um dies zu erreichen muss im 1ms Task die LED für 2 Sekunden zum leuchten gebracht werden.



### **app.h**


Struktur:
```
   6 struct App
   7 {
   8   uint8_t flags_u8;
   9   char modbusBuffer[32];
  10   uint8_t bufIndex;
  11   uint16_t errCnt;
  12   uint16_t mbInputReg01;
  13 };
  14 
  15 extern volatile struct App app;
```


