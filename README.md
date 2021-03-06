# SteeringWheel


// Library
#include <mcp_can.h>
#include <SPI.h>
#include <Wire.h>
/* 
#include <LCD.h>
#include <LiquidCrystal_I2C.h>

#define I2C_ADDR    0x27  // Define I2C Address where the PCF8574A is
#define BACKLIGHT_PIN     3
#define En_pin  2
#define Rw_pin  1
#define Rs_pin  0
#define D4_pin  4
#define D5_pin  5
#define D6_pin  6
#define D7_pin  7 
*/

int n = 1;

//LiquidCrystal_I2C	lcd(I2C_ADDR,En_pin,Rw_pin,Rs_pin,D4_pin,D5_pin,D6_pin,D7_pin);

// CS Pin
const int SPI_CS_PIN = 10;

// Set CS Pin
MCP_CAN CAN(SPI_CS_PIN);

// CAN ID
unsigned long MC_ID = 0x300;       // Motor controller ID
unsigned long Driver_ID = 0x200;   // Driver ID

long CAN_millis = 0;
float CAN_MotorDrive[2];
float CAN_MotorPower[2];

// Define all the pins
const int RX_Pin = 0;      // RX
const int TX_Pin = 1;      // TX
const int Rgn_BrkLt = 2;   // OUTPUT: Turn on break lights when regen is active
const int CC_btn = 3;      // INPUT: Cruise Control on/set button
const int Brake_sw = 4;    // INPUT: Mechanical brakes active
const int Hazard_btn = 5;  // INPUT: Hazard lights button
const int Left_IN = 6;     // INPUT: Left turn signal button
const int Right_IN = 7;    // INPUT: Right turn signal button
const int Left_OUT = 8;    // OUTPUT: Left signal to lights and horn controller
const int Right_OUT = 9;   // OUTPUT: Right signal to the lights and horn controller
const int Rgn_sw = 15;     // INPUT: Regen on/off switch
const int Rev_sw = 17;     // INPUT: Reverse on/off switch
const int Torque_Pot = A0; // A_INPUT: Torque setting for cruise control
const int Rgn_Pot = A6;    // A_INPUT: Regen potentiometer
const int Acc_Pot = A2;    // A_INPUT: Accelerator pedal potentiomenter
const int Speed_Pot = A7;  // A_INPUT: Speed setting for cruise control

// Current state of the buttons
boolean Right_State, Left_State, Hazard_State, CC_State;

// Switch states
boolean Reverse, Regen, Brakes;

// Previous state of the buttons
boolean xRight_State(LOW), xLeft_State(LOW), xHazard_State(LOW), xCC_State(LOW);

// Output
boolean RightSig, LeftSig, Hazard, CC;

// Regen Active
boolean Regen_Active;

// Debouncing variables
const int debounceDelay = 50;
long lastRight(0), lastLeft(0), lastHazard(0), lastCC(0);

// Analog Reads
int Torque_Raw, Rgn_Raw, Acc_Raw, Speed_Raw;
int TPot_min(0), RPot_min(560), APot_min(560), SPot_min(0);
int TPot_max(1009), RPot_max(985), APot_max(995), SPot_max(1009);
float Torque_Float, Rgn_Float, Acc_Float, Speed_Float;

int movAvg[10];
int index(0);
int sumAvg(0);

//long sample_millis(0);

// Values from CAN messages
float BUS_Current(0), BUS_Voltage(0), Car_Speed(0);

// Threshold for sensorless 
const float Sensorless_Speed = 0;  // Set the threshold for sensorless at ??? m/s

// Read inputs and debounce false button presses
void ReadButton(int buttonPin, boolean &lastBtnState, boolean &buttonState, boolean &output, long &lastDebounceTime) {
  int reading = digitalRead(buttonPin);

  if (reading != lastBtnState) 
  {
    // reset the debouncing timer
    lastDebounceTime = millis();
  }

  if ((millis() - lastDebounceTime) > debounceDelay) {
    // if the button state has changed:
    if (reading != buttonState)
    {
      buttonState = reading;
      
      if(buttonState) 
      {
        output = !output;
        Serial.println(buttonPin);
        Serial.println(output);
      }
    }
  }

  lastBtnState = reading;
}

// Read the potentiometers
void PotRead(int Pin, int &Raw, float &Float, int &Min, int &Max) 
{
  Raw = analogRead(Pin);
  
  if (Raw < Min) Raw = Min;
  if (Raw > Max) Raw = Max;
  
  Float = (float)(Raw-Min)/(Max-Min);
}

void filterRegenPot() 
{
  movAvg[index] = analogRead(Rgn_Pot);

  if (movAvg[index] < RPot_min) movAvg[index] = RPot_min;  
  if (movAvg[index] > RPot_max) movAvg[index] = RPot_max;
  
  if (index >=9) index = 0;
  else index++;
  
  sumAvg = 0;
  for(int j = 0; j<10; j++) 
  {
    sumAvg += movAvg[j];
  }
  
  Rgn_Raw = sumAvg / 10;
  
  if (Rgn_Raw < RPot_min) Rgn_Raw = RPot_min;  
  if (Rgn_Raw > RPot_max) Rgn_Raw = RPot_max;
  
  Rgn_Float = (float)(Rgn_Raw - RPot_min)/(RPot_max - RPot_min);
}

// Unpack the floats
unsigned long unpackFloat(const unsigned char *buffer) 
{
  const byte *b = buffer;
  long temp = 0;

  temp = (((long)b[3] << 24) | ((long)b[2] << 16) | ((long)b[1] <<  8) | (long)b[0]);

  Serial.println(temp, HEX);

  return *((float *) &temp);
}


void setup() 
{
  // Declare Digital Inputs
  pinMode(CC_btn, INPUT);
  pinMode(Brake_sw, INPUT);
  pinMode(Hazard_btn, INPUT);
  pinMode(Left_IN, INPUT);
  pinMode(Right_IN, INPUT);
  pinMode(Rgn_sw, INPUT);
  pinMode(Rev_sw, INPUT);
  // Declare Analog Inputs
  pinMode(Torque_Pot, INPUT);
  pinMode(Rgn_Pot, INPUT);
  pinMode(Acc_Pot, INPUT);
  pinMode(Speed_Pot, INPUT);  
  // Declare Digital Outputs
  pinMode(Rgn_BrkLt, OUTPUT);
  pinMode(Left_OUT, OUTPUT);
  pinMode(Right_OUT, OUTPUT);
  
  // Initialize Digital Outputs
  digitalWrite(Rgn_BrkLt, LOW);
  digitalWrite(Left_OUT, LOW);
  digitalWrite(Right_OUT, LOW);
  
  /* lcd.begin (20,4);
  
  // Switch on the backlight
  lcd.setBacklightPin(BACKLIGHT_PIN,POSITIVE);
  lcd.setBacklight(HIGH);
  lcd.home ();                   // go home */
  
  // CAN INITIALIZATION
  Serial.begin(115200);
  
  //initialize CAN BUS baud rate at 500kbps
  while (CAN_OK != CAN.begin(CAN_500KBPS))
  {
    Serial.println("CAN BUS Shield init fail");
    Serial.println(" Init CAN BUS Shield again");
    delay(100);
  }
  Serial.println("CAN BUS Shield init ok!");
}

void loop() 
{
  
  
  //lcd.setBacklight(HIGH);
  
  // Read and the debounce the four buttons 
  ReadButton(CC_btn, xCC_State, CC_State, CC, lastCC);
  ReadButton(Hazard_btn, xHazard_State, Hazard_State, Hazard, lastHazard);
  ReadButton(Right_IN, xRight_State, Right_State, RightSig, lastRight);
  ReadButton(Left_IN, xLeft_State, Left_State, LeftSig, lastLeft);
  
  // Read the state of the switches
  Reverse = digitalRead(Rev_sw);
  Regen = digitalRead(Rgn_sw);
  Brakes = digitalRead(Brake_sw);

  // Read and process the potentiometers
  PotRead(Torque_Pot, Torque_Raw, Torque_Float, TPot_min, TPot_max);
  //PotRead(Rgn_Pot, Rgn_Raw, Rgn_Float, RPot_min, RPot_max);
  filterRegenPot();
  PotRead(Speed_Pot, Speed_Raw, Speed_Float, SPot_min, SPot_max);
  PotRead(Acc_Pot, Acc_Raw, Acc_Float, APot_min, APot_max);
  
  if(Rgn_Float>0 && Regen) Regen_Active = true;
  else Regen_Active = false;
  
  // Output the state of the turn signals, both are on if the hazard button is active
  digitalWrite(Right_OUT, (Hazard || RightSig));
  digitalWrite(Left_OUT, (Hazard || LeftSig));

/*  lcd.setCursor(0,0);
  lcd.print("CC:");
  lcd.setCursor(3,0);
  if (CC) lcd.print("Y");
  else lcd.print("N");
  
  lcd.setCursor(0,1);
  lcd.print("<>:");
  lcd.setCursor(3,1);
  if (Hazard) lcd.print("Y");
  else lcd.print("N");
  
  lcd.setCursor(0,2);
  lcd.print(">>:");
  lcd.setCursor(3,2);
  if ((Hazard || RightSig)) lcd.print("Y");
  else lcd.print("N");
  
  lcd.setCursor(0,3);
  lcd.print("<<:");
  lcd.setCursor(3,3);
  if ((Hazard || LeftSig)) lcd.print("Y");
  else lcd.print("N");
  
  lcd.setCursor(6,0);
  lcd.print("Rg:");
  lcd.setCursor(9,0);
  if (Regen) lcd.print("Y");
  else lcd.print("N");
  
  lcd.setCursor(6,1);
  lcd.print("Rv:");
  lcd.setCursor(9,1);
  if (Reverse) lcd.print("Y");
  else lcd.print("N");

  lcd.setCursor(13,0);
  lcd.print("Ac:");
  lcd.setCursor(16,0);
  lcd.print(Acc_Float);
  
  lcd.setCursor(13,1);
  lcd.print("Rg:");
  lcd.setCursor(16,1);
  lcd.print(Rgn_Float);
  
  lcd.setCursor(13,2);
  lcd.print("Tq:");
  lcd.setCursor(16,2);
  lcd.print(Torque_Float);
  
  lcd.setCursor(13,3);
  lcd.print("Sp:");
  lcd.setCursor(16,3);
  lcd.print(Speed_Float);
*/
  // Turn on the brake light if the regen or mechanical brakes are active
  digitalWrite(Rgn_BrkLt, (Brakes || Regen_Active));
  
  // Cruise control is on if it is switched on the the car is not in reverse or braking
  CC = CC && !(Brakes || Regen_Active || Reverse);
  
  if (Regen_Active) 
  {
    CAN_MotorDrive[0] = (float)0;    // Set the motor desired speed to 0 m/s
  }
  else if (Reverse)
  {
    CAN_MotorDrive[0] = (float)-100; // Set the motor desired speed to -100 m/s
  }
  else 
  {
    CAN_MotorDrive[0] = (float)100;  // Set the motor desired speed to 100 m/s
  }

  if (Brakes) {

CAN_MotorDrive[1] = (float)0;   // Set the throttle to 0%
  }
  else if (Regen_Active) {
    CAN_MotorDrive[1] = Rgn_Float;  // Set the throttle to the position of the regen pedal
  }
  else {
    CAN_MotorDrive[1] = Acc_Float;   // Set the throttle to the postion of the accelerator pedal
  }
  
  if (CC) {
    //if (Car_Speed >= Sensorless_Speed) {
      CAN_MotorDrive[0] = Speed_Float*100.0;  // Set the max speed according to the CC Speed Pot
      CAN_MotorDrive[1] = Torque_Float; // Set the max torque according to the CC Torque Pot
    //
    }  
  }
  
  CAN_MotorPower[0] = (float)0;      // These bits are reserved
  CAN_MotorPower[1] = (float)1.0;    // Set controller to use 100% of the available BUS current 

/*  lcd.setCursor(5,2);
  lcd.print("V:");
  lcd.setCursor(7,2);
  lcd.print("      ");
  lcd.setCursor(7,2);
  lcd.print((int)CAN_MotorDrive[0]);
  
  lcd.setCursor(5,3);
  lcd.print("I:");
  lcd.setCursor(7,3);
  lcd.print("      ");
  lcd.setCursor(7,3);
  lcd.print(CAN_MotorDrive[1]);
*/
  if ((millis() - CAN_millis) > 5) 
  {
    CAN_millis = millis();
    
    CAN.sendMsgBuf(Driver_ID+1, 0, 8, (unsigned char *)CAN_MotorDrive);
    CAN.sendMsgBuf(Driver_ID+2, 0, 8, (unsigned char *)CAN_MotorPower);
    
    // Read the relevant CAN messages to determine speed and current draw
    unsigned char len = 0;
    unsigned char buf[8];

    if(CAN_MSGAVAIL == CAN.checkReceive()) {            // check if data coming
      CAN.readMsgBuf(&len, buf);    // read data,  len: data length, buf: data buf

      unsigned int canId = CAN.getCanId();
        
      if (canId == MC_ID+2) {
        BUS_Current = unpackFloat(&buf[4]);
        BUS_Voltage = unpackFloat(&buf[0]);
        
        Serial.println(BUS_Current);
        Serial.println(BUS_Voltage);
      }
      if (canId == MC_ID+3) {
        Car_Speed = unpackFloat(&buf[4]);
        
        Serial.println(Car_Speed);
      }
    }
  }
  
}
