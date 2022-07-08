# weighing-and-packaging-system 


#include <Wire.h>
// #include <SoftwareSerial.h>
#include <Servo.h>
#include "HX711.h"
#include <Ultrasonic.h>
#include <LiquidCrystal.h>

const int LOADCELL_DOUT_PIN = A3;
const int LOADCELL_SCK_PIN = A2;

Ultrasonic ultrasonic(5,6);
HX711 scale_left;
HX711 scale_right;
Servo servo;
// SoftwareSerial ss(2,3);
LiquidCrystal lcd(10,11,12,13,A0,A1);

int amount = 0;
const int motor = 9;
float prev_num = 0;

void truncate(String* str, int columns) {
  *str = str->substring(0, columns);
}
void print_lcd(String str, int row) {
  static const int COLUMNS = 16;
  if (str.length() > COLUMNS) truncate(&str, COLUMNS);
  int pre_spaces = (COLUMNS - str.length()) / 2;
  lcd.setCursor(0, row); lcd.print("                ");
  lcd.setCursor(pre_spaces, row); lcd.print(str);
}
void print_lcd(String str0, String str1) {
  static const int COLUMNS = 16;
  if (str0.length() > COLUMNS) truncate(&str0, COLUMNS);
  if (str1.length() > COLUMNS) truncate(&str1, COLUMNS);
  int pre_spaces0 = (COLUMNS - str0.length()) / 2;
  int pre_spaces1 = (COLUMNS - str1.length()) / 2;
  lcd.clear();
  lcd.setCursor(pre_spaces0, 0); lcd.print(str0);
  lcd.setCursor(pre_spaces1, 1); lcd.print(str1);
}
void default_lcd_text(int del) {
  print_lcd("grain packaging","and weighing");
  delay(del);
}
void initialize_scale_left() {
  // unsigned long scale_left_t = millis();
  // scale_left.begin(LOADCELL_DOUT_PIN, LOADCELL_SCK_PIN);
  // while (!scale_left.is_ready() & millis() - scale_left_t < 4000) {
  //   print_lcd("left scale","not ready");
  //   delay(500);
  // }
  // if(millis() - scale_left_t < 4000) {
  //   print_lcd("right scale","is OK");
  //   delay(1200);
  // }
}
void initialize_scale_right() {
  // unsigned long scale_right_t = millis();
  // scale_right.begin(7, 8);
  // while (!scale_right.is_ready() & millis() - scale_right_t < 4000) {
  //   print_lcd("right scale","not ready");
  //   delay(500);
  // }
  // if(millis() - scale_right_t < 4000) {
  //   print_lcd("left scale","is OK");
  //   delay(1200);
  // }
}
void wait_for_order() {
  print_lcd("waiting for","order");
  while(1) {
    Serial.println("order");
    unsigned long t = millis();
    while(millis() - t < 10000 & !Serial.available());
    if(Serial.available()) {
      String str = Serial.readStringUntil('\r');
      float num = str.toInt();
      print_lcd(str,"");
      delay(3000);
      if(num != prev_num & num > 0) {
        lcd.begin(16,2);
        print_lcd("received:", String(num));
        amount = num;
        prev_num = num;
        return;
      }

    }
    else {
      print_lcd("WiFi module","unresponsive");
      delay(1200);
    }
  }
  // while(!ss.available());
  // String str = ss.readStringUntil('\r');
  // String response = ss.readStringUntil('\r');
  // amount = response.toInt();
  // print_lcd("received", String(amount) + " TShs");
}
bool bag_present() {
  static const int min_distance = 6;
  int distance = ultrasonic.distanceRead();
  if(distance < 0) distance = 0;
  if(distance < min_distance) return true;
  return false;
}
float measure_weight() {
  static const float scale_left_constant = 120000.0 / 1000.0; //per kg
  long reading = scale_left.read();
  float weight = reading / scale_left_constant;
  return weight;
}
void pour_grain() {
  static const float kg_per_tshs = 0.001;
  float required_grain_weight = float(amount) * kg_per_tshs;
  float system_weight = measure_weight();
  print_lcd("pouring grain","....");
  servo.write(120);
  delay(3000);
  // while(measure_weight() < system_weight + required_grain_weight) delay(100);
  print_lcd("done pouring","grain");
  servo.write(0);
  delay(2000);
  print_lcd("moving","packge");
  delay(100);
  digitalWrite(motor, 1);
  delay(4000);
  digitalWrite(motor, 0);
  lcd.begin(16,2);
  print_lcd("done!","");
}
void setup() {
  Serial.begin(9600);
  // ss.begin(9600);
  servo.attach(3);
  servo.write(0);
  pinMode(motor, OUTPUT);
  lcd.begin(16,2);
  // initialize_scale_left();
  // initialize_scale_right();
  default_lcd_text(1500);
  servo.write(120);
  delay(1000);
  servo.write(0);
  // digitalWrite(motor, 1);
  // delay(4000);
  // digitalWrite(motor, 0);
  delay(10000);
  Serial.readString();
}
void loop() {
  wait_for_order();
  if(amount > 0) {
    // print_lcd("place a bag","to begin");
    // while(!bag_present());
    pour_grain();
  }
  default_lcd_text(0);
}
