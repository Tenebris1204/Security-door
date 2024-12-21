//Security-door
//uses HC- SR501 PIR Motion Sensor
//Micro Servo Motor
//IR Receiver Module
//LCD 1602 Module
//Power Supply Module
//2 LEDs

#include <IRremote.hpp>
#include <LiquidCrystal.h>
#include <Servo.h>

// Pin definitions
#define IR_RECV 2        // IR receiver pin
#define LCD_RS 6         // LCD RS pin
#define LCD_E 8          // LCD Enable pin
#define LCD_D4 9         // LCD Data pin 4
#define LCD_D5 10        // LCD Data pin 5
#define LCD_D6 11        // LCD Data pin 6
#define LCD_D7 12        // LCD Data pin 7
#define LED_RED 4        // Red LED pin (Access Denied)
#define LED_GREEN 5      // Green LED pin (Access Granted)
#define SERVO_PIN 3      // Servo motor control pin
#define PIR_SENSOR 7     // PIR motion sensor pin
// LCD configuration
#define LCD_COLUMNS 16   // Number of columns on LCD
#define LCD_ROWS 2       // Number of rows on LCD
// Initialize LCD and Servo
LiquidCrystal lcd(LCD_RS, LCD_E, LCD_D4, LCD_D5, LCD_D6, LCD_D7);
Servo myServo;
// Password configuration
const char* CORRECT_PASSWORD = "1234"; // Correct password for access
char enteredPassword[5] = "";          // Buffer for entered password
byte currentIndex = 0;                 // Tracks current index for password entry
// PIR sensor and motion detection
int pirValue = 0;          // Current PIR sensor value
bool motionDetected = false; // Tracks if motion was detected
bool inPasswordMode = false; // Tracks if the system is in password entry mode
// Decode IR remote input to corresponding key
const char* decodeKey(unsigned int code) {
  switch (code) {
    case 0xC: return "1";
    case 0x18: return "2";
    case 0x5E: return "3";
    case 0x8: return "4";
    case 0x1C: return "5";
    case 0x5A: return "6";
    case 0x42: return "7";
    case 0x52: return "8";
    case 0x4A: return "9";
    case 0x16: return "0";
    case 0x45: return "C"; // Clear input
  }
  return "Unknown";
}
void setup() {
  IrReceiver.begin(IR_RECV); // Start IR receiver
  lcd.begin(LCD_COLUMNS, LCD_ROWS); // Initialize LCD
  myServo.attach(SERVO_PIN); // Attach servo to pin
  pinMode(LED_RED, OUTPUT);
  pinMode(LED_GREEN, OUTPUT);
  pinMode(PIR_SENSOR, INPUT);
  // Initial LCD display message
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Waiting for");
  lcd.setCursor(0, 1);
  lcd.print("motion...");
  Serial.begin(9600); // Start serial monitor for debugging
}
void loop() {
  // Default state: Red LED on, Green LED off
  digitalWrite(LED_RED, HIGH);
  digitalWrite(LED_GREEN, LOW);
  // Check PIR sensor for motion
  pirValue = digitalRead(PIR_SENSOR);
  if (pirValue == HIGH && !motionDetected && !inPasswordMode) {
    motion_detected();
  }
  // Handle password input if in password entry mode
  if (inPasswordMode && IrReceiver.decode()) {
    unsigned int code = IrReceiver.decodedIRData.command;
    const char* key = decodeKey(code);
    processKeyInput(key);
    IrReceiver.resume(); // Resume IR receiver for next input
  }
}
// Handles motion detection
void motion_detected() {
  motionDetected = true;
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Motion detected!");
  delay(2000);
  // Transition to password entry mode
  inPasswordMode = true;
  motionDetected = false; // Reset motion flag
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Enter Password:");
  lcd.setCursor(0, 1);
  lcd.print("Attempt: ");
}
// Process key input from IR remote
void processKeyInput(const char* key) {
  if (key[0] >= '0' && key[0] <= '9' && currentIndex < 4) {
    enteredPassword[currentIndex++] = key[0];
    delay(50); // Prevent double inputs from remote
    enteredPassword[currentIndex] = '\0';
    lcd.setCursor(9, 1);
    lcd.print(enteredPassword);
    Serial.println(enteredPassword); // for debugging purposes
  } else if (key[0] == 'C') {
    clearPassword(); // Clear entered password
  }
  // Check if password is complete
  if (currentIndex == 4) {
    validatePassword();
  }
}
// Validate the entered password
void validatePassword() {
  if (strcmp(enteredPassword, CORRECT_PASSWORD) == 0) {
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Access Granted!");
    digitalWrite(LED_RED, LOW);
    digitalWrite(LED_GREEN, HIGH);
    myServo.write(90); // Unlock
    delay(3000);
    myServo.write(0); // Lock
    delay(1500); // Prevents false motion trigger
    lcd.clear();
  } else {
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Access Denied!");
    digitalWrite(LED_RED, HIGH);
    digitalWrite(LED_GREEN, LOW);
    delay(2000);
  }
  clearPassword(); // Reset password buffer
  inPasswordMode = false; // Return to motion detection mode
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Waiting for");
  lcd.setCursor(0, 1);
  lcd.print("motion...");
}
// Clear the entered password
void clearPassword() {
  memset(enteredPassword, 0, sizeof(enteredPassword)); // Reset password buffer
  currentIndex = 0;
  lcd.setCursor(9, 1);
  lcd.print("    "); // Clear LCD input field
}
