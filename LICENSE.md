

#include <Adafruit_NeoPixel.h>
#ifdef __AVR__
#include <avr/power.h>
#endif

#define LEDPIN 13 // Pin for the LEDs

// Parameter 1 = number of pixels in strip
// Parameter 2 = Arduino pin number (most are valid)
// Parameter 3 = pixel type flags, add together as needed:
//   NEO_KHZ800  800 KHz bitstream (most NeoPixel products w/WS2812 LEDs)
//   NEO_KHZ400  400 KHz (classic 'v1' (not v2) FLORA pixels, WS2811 drivers)
//   NEO_GRB     Pixels are wired for GRB bitstream (most NeoPixel products)
//   NEO_RGB     Pixels are wired for RGB bitstream (v1 FLORA pixels, not v2)
//   NEO_RGBW    Pixels are wired for RGBW bitstream (NeoPixel RGBW products)
Adafruit_NeoPixel strip = Adafruit_NeoPixel(240, LEDPIN, NEO_GRB + NEO_KHZ800);

// IMPORTANT: To reduce NeoPixel burnout risk, add 1000 uF capacitor across
// pixel power leads, add 300 - 500 Ohm resistor on first pixel's data input
// and minimize distance between Arduino and first pixel.  Avoid connecting
// on a live circuit...if you must, connect GND first.


 //Variables for pressure sensing
 int counter[10];
 const int Cols = 4;
 const int Rows = 6;

 int colPin[] = {3, 4, 5, 6};
 int rowPin[] = {7, 8, 9, 10, 11, 12};

 int tile_number[6][4] = {
  {1, 2, 3, 4},
  {8, 7, 6, 5},
  {9, 10, 11, 12},
  {16, 15, 14, 13},
  {17, 18, 19, 20},
  {24, 23, 22, 21}
 };

 int tiles_pressed[6][4];

 // Variables for Seat and Button ISR
 int seatPin = 2;
 volatile int buttonStart = 0;
 volatile int timerStart = 0;
 long markStart = 0;

 volatile long currentTime;
 volatile long standupTime = 0;
 volatile long buttonTime = 0;
 volatile long overallTime = 0; // overallTime can be modified in ISR --> volatile 
 //volatile long allTime[buttonTime; overallTime]; // array of all the necessary times

 int matlabData = 0;
 
void setup() {
  // put your setup code here, to run once:

  // LED initialisation
  strip.begin();
  strip.show();

 /* //light up two central columns for TUG
  if(Serial.available() >0)
  {
    matlabData = Serial.read();
  }

  if(matlabData == 100)
  {*/
    for(int i = 0; i < 6; i++)
    {
      for(int j = 1; j < 3; j++)
      {
        for(int k = 0; k < 6; k++)
           {
                strip.setPixelColor(6*(tile_number[i][j]-1)+k, strip.Color(0,0,30));
           }
      }
    }
   for(int i = 0; i < 6; i++)
    {
        for(int k = 0; k < 6; k++)
           {
                strip.setPixelColor(6*(tile_number[i][0]-1)+k, strip.Color(30, 0, 0));
           }
    }
  
    for(int i = 0; i < 6; i++)
    {
        for(int k = 0; k < 6; k++)
           {
                strip.setPixelColor(6*(tile_number[i][3]-1)+k, strip.Color(30, 0, 0));
           }
    }
    strip.show();
  //}

   pinMode(seatPin, INPUT);

  //initialise serial communication
  Serial.begin(9600);
  
  do{
    
   //do nothing
    
  } while(digitalRead(seatPin) == HIGH);

  //As soon as the child stands, the code starts to run from here

  markStart = millis(); //notes the start time for TUG

  attachInterrupt(digitalPinToInterrupt(seatPin), tugtime, RISING); // Triggers ISR whenever the seatPin goes from LOW to HIGH

}

void loop() {
  // put your main code here, to run repeatedly:
  int count = 0;

  for(int i = 0; i < Rows; i++)
  {
    for(int j = 0; j < Cols; j++)
    {
      tiles_pressed[i][j] = 0;
    }
  }

  
   //Initialise columns as outputs
   for(int i = 0; i < Cols; i++)
  {
    pinMode(colPin[i], OUTPUT);
  }

  //initialise rows as inputs
  for(int j = 0; j < Rows; j++)
  {
    pinMode(rowPin[j], INPUT);
  }

 //Set Columns to high and read from rows
  for(int i = 0; i < Cols; i++)
  {
    digitalWrite(colPin[i], HIGH); //Sets column i to 5V

    for(int j = 0; j < Rows; j++)
    {
      if(digitalRead(rowPin[j]) == HIGH) //Reads the value of the jth row
      {
        tiles_pressed[j][i] = 1;
      }
    }
    digitalWrite(colPin[i], LOW); //Turns column i off
  }

  //Initialise columns as inputs
   for(int i = 0; i < Cols; i++)
  {
    pinMode(colPin[i], INPUT);
  }

  //initialise rows as outputs
  for(int j = 0; j < Rows; j++)
  {
    pinMode(rowPin[j], OUTPUT);
  }

  //Set rows high and read from columns
  for(int j = 0; j < Rows; j++)
  {
    digitalWrite(rowPin[j], HIGH); //Sets column i to 5V

    for(int i = 0; i < Cols; i++)
    {
      if(digitalRead(colPin[i]) == HIGH) //Reads the value of the ith column
      {
        tiles_pressed[j][i] = 1;
      }
     
    }
    digitalWrite(rowPin[j], LOW); //Turns row j off
  }


  //Send states of tiles to serial monitor
  
 for(int i = 0; i < Rows; i++)
 {
  for(int j = 0; j < Cols; j++)
  {
    Serial.print(tiles_pressed[i][j]);
  }
 }
 Serial.println();

  //Recognise when the child is nearly at the end of the mat
  for(int m = 0; m < 4; m++)
  {
    if((tiles_pressed[4][m] == 1))
    {
      if(buttonStart == 3) //if it is the first time reaching the end of the mat
      {
        buttonStart = 1;
      }
    }
  }

  if(buttonStart == 0) //only check once for initialising timer
  {
    for(int m = 0; m < 4; m++)
    {
      if((tiles_pressed[1][m] == 1))
      {
        buttonStart = 3;
        currentTime = millis();
        standupTime = currentTime - markStart;
      }
    }
  }

 //Recognise that the child is walking and not still trying to stand up
 
  if(timerStart == 0 && buttonStart == 2) //only check once for initialising timer
  {
    for(int m = 0; m < 4; m++)
    {
      if((tiles_pressed[1][m] == 1))
      {
        timerStart = 1;
      }
    }
  }
 
  
}

//ISR for calculating the TUG time
void tugtime(){
  // ISR that records the TUG time
  
  currentTime = millis();

  if(buttonStart == 1) // Only records button time when child is near the end of the mat
  {
    buttonTime = currentTime - markStart; //calculate time from standing up to pressing the button
    buttonStart = 2;
    
    //Serial.println(buttonTime);
    
    for(int i = 0; i < 6; i++)
    {
      for(int j = 1; j < 3; j++)
      {
        for(int k = 0; k < 6; k++)
        {
          strip.setPixelColor(6*(tile_number[i][j]-1)+k, strip.Color(0,30,0));
        }
      }
    }
    strip.show();
  }
  else if(timerStart == 1) // Accounts for the time taken for the child to stand properly
  {
    overallTime = currentTime - markStart; //calculates overall TUG time
  
    timerStart = 2;
  }

  while(timerStart == 2)
    {
      Serial.print(standupTime);
      Serial.print(" ");
      Serial.print(buttonTime);
      Serial.print(" ");
      Serial.print(overallTime);
      Serial.println();
    }
}
