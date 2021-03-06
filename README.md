# CarTrack
#include <SoftwareSerial.h>
#include <string.h>
#include <TinyGPS.h>
SoftwareSerial GPRS(7, 8);
byte buffer[64]; // buffer array for data recieve over serial port
int count=0; // counter for buffer array
SoftwareSerial GPS(4, 5);
TinyGPS gps;
unsigned long fix_age;
long lat, lon;
float LAT, LON;
void gpsdump(TinyGPS &gps);
bool feedgps();
char inchar; // Will hold the incoming character from the GSM shield
char incoming_char=0;
void getGPS();
void SIM900power() // software equivalent of pressing the GSM shield "power" button
{
  digitalWrite(9, HIGH);
  delay(1000);
  Sim900_Inti();
}

void setup()
{
  GPRS.begin(19200); // the SIM900 baud rate
  SIM900power();
  GPS.begin(4800); // GPS module baud rate
  Serial.begin(19200); // the Serial port of Arduino baud rate.
  delay(500);
}

void loop()
{
  GPRS.listen();
  if (GPRS.available()) // If data is coming from the GSM shield)
  {
    while(GPRS.available()) // reading data into char array
    {
      buffer[count++]=GPRS.read(); // writing data into array
      if(count == 64)break;
    }
    Serial.write(buffer,count); // if no data transmission ends, write buffer to hardware serial port
    Cmd_Read_Act(); // new password authentication function by Daniel Garrett
    ClearBufferArray(); // call clearBufferArray function to clear the storaged data from the array
    count=0; // set counter of while loop to zero
  }
  else
  {
    Serial.println("GPRS not available");
    delay(1000);
  }
  if (Serial.available()) // if data is available on hardwareserial port ==> data is comming from PC or notebook
  GPRS.write(Serial.read()); // write it to the GPRS shield
}

void ClearBufferArray() // function to clear buffer array
{
  for (int i=0; i<count;i++)
  { buffer[i]=NULL;} // clear all index of array with command NULL
}

void Sim900_Inti(void)
{
  GPRS.println("AT+CMGF=1"); // Set GSM shield to sms mode
  Serial.println("AT CMGF=1");
  delay(500);
  GPRS.println("AT+CNMI=2,2");
  Serial.println("AT CMGF=1");
  delay(500);
}

void Cmd_Read_Act(void)                       //Function reads the SMS sent to SIM900 shield.
{ 
  Serial.println("Cmd_Read_Act initialised");
  char buffer[64];
  for (int i=0; i<count;i++)
  { buffer[i]=char(buffer[i]);}
  Serial.println("listening for passcode");
  delay(20000);
    
  if (strstr(buffer,"dan968"))    //Comparing password entered with password stored in program  
  {
      Serial.println("Password Authenticated.");
      Serial.println("Sending reply SMS. ");
      SendTextMessage();
      delay(1000);
  }
  
  else
  {
    Serial.println("no correct passcode received");
  }
  
}      

void SendTextMessage()
{
  GPRS.print("AT+CMGF=1\r"); //Sending the SMS in text mode
  delay(100);
  GPRS.println("AT + CMGS = \"+447565319881\""); //The predefined phone number
  delay(100);
  GPRS.println("Please wait while Module calculates position"); //the content of the message
  delay(100);
  GPRS.println((char)26);//the ASCII code of the ctrl z is 26
  delay(100);
  GPRS.println();
  delay(5000);
  int counter=0;
  GPS.listen();
  for (;;)
  {
    long lat, lon;
    unsigned long fix_age, time, date, speed, course;
    unsigned long chars;
    unsigned short sentences, failed_checksum;
    long Latitude, Longitude;
    // retrieves /- lat/long in 100000ths of a degree
    gps.get_position(&lat, &lon, &fix_age);
    getGPS();
    Serial.print("Latitude : ");
    Serial.print(LAT/1000000,7);
    Serial.print(" :: Longitude : ");
    Serial.println(LON/1000000,7);
    GPRS.print("AT+CMGF=1\r"); //Sending the SMS in text mode
    delay(100);
    GPRS.println("AT + CMGS = \"+447565319881\""); //The predefined phone number
    delay(100);
    GPRS.print("maps.google.com/maps?q=");
    GPRS.print(LAT/1000000,7);
    GPRS.print("+");
    GPRS.println(LON/1000000,7);//the content of the message
    delay(100);
    GPRS.println((char)26);//the ASCII code of the ctrl z is 26
    delay(100);
    GPRS.println();
    delay(5000);
    counter=0;
    break;
  }
}

void getGPS()
{
  bool newdata = false;
  unsigned long start = millis();
  while (millis() - start < 1000)
  {
    if (feedgps ())
    {
      newdata = true;
    }
  }
  if (newdata)
  {
    gpsdump(gps);
  }
}

bool feedgps()
{
  while (GPS.available())
  {
    if (gps.encode(GPS.read()))return true;
  }
  return 0;
}

void gpsdump(TinyGPS &gps)
{
  gps.get_position(&lat, &lon);
  LAT = lat;
  LON = lon;
  {
    feedgps();
  }
}
