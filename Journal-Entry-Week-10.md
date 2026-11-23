# Week 10
Terence, Jolene, and I worked on the Arduino. Sophie worked on the prototype, and Namiko worked on the sound and how it could be used for the project. 

--------
Terence worked on the Arduino with the TCS color sensor since the spectral one was giving readings that were a little harder to read.
I worked with Jolene to try out the code with the Arduino Nano and the spectral sensor.
I had a code that I wanted to try out to see if we could detect the pink colors that Nadia said she would use for her tapestry.


### Previous code that gives HSL values and if it was closer to pink, mauve, or coral.

```
/*
  AS7265x Spectral Triad -> RGB / HEX color reader
  Based on SparkFun Example 2 (Nathan Seidle, MIT license)

  Reads the 12 visible channels (410-705 nm), groups them into
  blue / green / red, normalizes, and prints RGB (0-255) + HEX.
  Infrared channels (T, U, V, W, K, L) are ignored for color.

  Serial Monitor: 115200 baud
*/

#include <Wire.h>
#include "SparkFun_AS7265X.h"

AS7265X sensor;

// Clamp a 0..1 float and scale it to 0..255
uint8_t toByte(float v)
{
  if (v < 0.0) v = 0.0;
  if (v > 1.0) v = 1.0;
  return (uint8_t)(v * 255.0 + 0.5);
}

void setup()
{
  Serial.begin(115200);
  Serial.println("AS7265x Color Reader");

  if (sensor.begin() == false)
  {
    Serial.println("Sensor not detected. Check wiring. Freezing...");
    while (1);
  }

  sensor.disableIndicator(); // turn off the blue status LED
  Serial.println("Ready. Hold a sample against the sensor.");
}

void loop()
{
  sensor.takeMeasurementsWithBulb(); // lights the sample, reads all 18 channels

  // Visible channels only
  float A  = sensor.getCalibratedA(); // 410nm violet
  float B  = sensor.getCalibratedB(); // 435nm blue
  float C  = sensor.getCalibratedC(); // 460nm blue
  float D  = sensor.getCalibratedD(); // 485nm blue-cyan
  float E  = sensor.getCalibratedE(); // 510nm cyan-green
  float F  = sensor.getCalibratedF(); // 535nm green
  float G  = sensor.getCalibratedG(); // 560nm yellow-green
  float H  = sensor.getCalibratedH(); // 585nm yellow-orange
  float Rr = sensor.getCalibratedR(); // 610nm orange
  float I  = sensor.getCalibratedI(); // 645nm red
  float S  = sensor.getCalibratedS(); // 680nm deep red
  float J  = sensor.getCalibratedJ(); // 705nm deep red

  // Group by wavelength
  float blue  = (A + B + C + D) / 4.0;
  float green = (E + F + G) / 3.0;
  float red   = (H + Rr + I + S + J) / 5.0;

  // Normalize so the strongest channel = 1.0
  float maxVal = max(red, max(green, blue));

  uint8_t r8 = 0, g8 = 0, b8 = 0;
  if (maxVal > 0.001)
  {
    r8 = toByte(red   / maxVal);
    g8 = toByte(green / maxVal);
    b8 = toByte(blue  / maxVal);
  }

  char hex[8];
  sprintf(hex, "#%02X%02X%02X", r8, g8, b8);

  // Output
  Serial.print("RGB: ");
  Serial.print(r8); Serial.print(", ");
  Serial.print(g8); Serial.print(", ");
  Serial.print(b8);
  Serial.print("   HEX: ");
  Serial.println(hex);

  delay(500);
}
```

### Code used that would still detect and give us the HEX of other colors. It was modified with Claude.

```
/*
  as7265x color + tone classifier v3
  color family = hue + saturation, tone = hsb brightness (light / dark)
  prints hex and hsb for every reading; unknown colors just print their hex

  tracks:
  red light 1,    red dark 2
  orange light 3, orange dark 4
  pink light 5,   pink dark 6
  peach light 7,  peach dark 8
  mauve light 9,  mauve dark 10
  purple light 11, purple dark 12
  nothing / unknown = no sound

  serial commands (type in serial monitor, press enter):
  w = white reference, hold white paper on the sensor first (do this at startup, makes hex + light/dark accurate)
  r = reset learned brightness
*/

#include "SparkFun_AS7265X.h"
AS7265X sensor;

//true = use the triad's leds (real objects in the box), false = phone screen tests
const bool USE_BULB = true;

//hue zones in degrees (0 = red, 30 = orange, 60 = yellow, 270 = purple, 330 = pink)
const float RED_ZONE_START = 345.0;   //red zone wraps past 360 to RED_ZONE_END
const float RED_ZONE_END = 12.0;
const float ORANGE_ZONE_END = 45.0;
const float PURPLE_ZONE_START = 255.0;
const float PINK_ZONE_START = 300.0;  //magenta / hot pink side

//saturation splits inside each zone (0-1)
const float RED_SAT = 0.55;     //red zone: above = red, below = pink
const float ORANGE_SAT = 0.55;  //orange zone: above = orange, below = peach
const float PINK_SAT = 0.40;    //pink zone: above = pink, below = mauve
const float PURPLE_SAT = 0.35;  //purple zone: above = purple, below = mauve

//below this saturation it's white / grey / black, counts as unknown
const float NEUTRAL_SAT = 0.08;

//hsb brightness (0-100) below this = dark tone, above = light tone
const float DARK_B = 45.0;

//absolute minimum brightness, below = nothing under the sensor
const float EMPTY_MIN = 5.0;
//below this fraction of the brightest reading = nothing under the sensor
const float EMPTY_RATIO = 0.10;

const char* familyNames[6] = {"red", "orange", "pink", "peach", "mauve", "purple"};

//white balance gains, 1 = off until you send 'w'
float gainR = 1.0, gainG = 1.0, gainB = 1.0;

//level that counts as 100% brightness, set by 'w'
float whiteLevel = -1;
//fallback if no white reference: brightest channel seen so far
float maxChannelSeen = -1;
//brightest total reading seen, used for empty detection
float maxBright = -1;

void setup()
{
  Serial.begin(115200);
  Serial.println("as7265x color + tone classifier v3 initializing...");

  if (sensor.begin() == false)
  {
    Serial.println("sensor does not appear to be connected. freezing...");
    while (1);
  }

  sensor.disableIndicator(); //turn off the blue status led
  Serial.println("ready. hold white paper on the sensor and type w for best results.");
}

void loop()
{
  if (USE_BULB)
  {
    sensor.takeMeasurementsWithBulb();
  }
  else
  {
    sensor.takeMeasurements();
  }

  //visible channels only
  float a = sensor.getCalibratedA(); //410nm
  float b = sensor.getCalibratedB(); //435nm
  float c = sensor.getCalibratedC(); //460nm
  float d = sensor.getCalibratedD(); //485nm
  float e = sensor.getCalibratedE(); //510nm
  float f = sensor.getCalibratedF(); //535nm
  float g = sensor.getCalibratedG(); //560nm
  float h = sensor.getCalibratedH(); //585nm, yellow, split between red and green
  float r = sensor.getCalibratedR(); //610nm
  float i = sensor.getCalibratedI(); //645nm
  float s = sensor.getCalibratedS(); //680nm
  float j = sensor.getCalibratedJ(); //705nm

  //group into rgb
  float blue  = (a + b + c + d) / 4.0;
  float green = (e + f + g + h * 0.5) / 3.5;
  float red   = (r + i + s + j + h * 0.5) / 4.5;

  handleSerial(red, green, blue);

  //apply white balance
  red *= gainR;
  green *= gainG;
  blue *= gainB;

  float brightness = red + green + blue;

  //nothing under the sensor
  if (brightness < EMPTY_MIN || (maxBright > 0 && brightness < maxBright * EMPTY_RATIO))
  {
    Serial.println("empty");
    delay(200);
    return;
  }
  if (brightness > maxBright) maxBright = brightness;

  float maxC = max(red, max(green, blue));
  float minC = min(red, min(green, blue));
  if (maxC > maxChannelSeen) maxChannelSeen = maxC;

  //what counts as 100%: white paper if set, otherwise the brightest thing seen
  float scale = (whiteLevel > 0) ? whiteLevel : maxChannelSeen;

  //hex values (0-255)
  uint8_t r8 = toByte(red / scale);
  uint8_t g8 = toByte(green / scale);
  uint8_t b8 = toByte(blue / scale);
  char hex[8];
  sprintf(hex, "#%02X%02X%02X", r8, g8, b8);

  //hsb: hue 0-360, saturation 0-1, brightness 0-100
  float delta = maxC - minC;
  float sat = (maxC > 0) ? delta / maxC : 0;
  float bright = min(maxC / scale, 1.0) * 100.0;
  float hue = 0;
  if (delta > 0)
  {
    if (maxC == red)        hue = 60.0 * fmod((green - blue) / delta, 6.0);
    else if (maxC == green) hue = 60.0 * ((blue - red) / delta + 2.0);
    else                    hue = 60.0 * ((red - green) / delta + 4.0);
  }
  if (hue < 0) hue += 360.0;

  Serial.print("hex: "); Serial.print(hex);
  Serial.print(" | hsb: ");
  Serial.print(hue, 0); Serial.print(", ");
  Serial.print(sat * 100.0, 0); Serial.print(", ");
  Serial.print(bright, 0);

  int family = classifyFamily(hue, sat);
  if (family < 0)
  {
    Serial.println(" | unknown");
    delay(200);
    return;
  }

  bool dark = bright < DARK_B;
  Serial.print(" | "); Serial.print(familyNames[family]);
  Serial.print(dark ? " dark" : " light");

  triggerAudio(family * 2 + 1 + (dark ? 1 : 0));

  delay(200);
}

//returns 0-5 for red, orange, pink, peach, mauve, purple, or -1 if unknown
int classifyFamily(float hue, float sat)
{
  if (sat < NEUTRAL_SAT) return -1; //white / grey / black

  //red zone: saturated = red, softer = pink
  if (hue >= RED_ZONE_START || hue < RED_ZONE_END)
  {
    return (sat >= RED_SAT) ? 0 : 2;
  }
  //orange zone: saturated = orange, softer = peach
  if (hue >= RED_ZONE_END && hue < ORANGE_ZONE_END)
  {
    return (sat >= ORANGE_SAT) ? 1 : 3;
  }
  //pink zone: saturated = pink, muted = mauve
  if (hue >= PINK_ZONE_START && hue < RED_ZONE_START)
  {
    return (sat >= PINK_SAT) ? 2 : 4;
  }
  //purple zone: saturated = purple, muted = mauve
  if (hue >= PURPLE_ZONE_START && hue < PINK_ZONE_START)
  {
    return (sat >= PURPLE_SAT) ? 5 : 4;
  }
  return -1; //yellow, green, blue, etc.
}

//clamp 0..1 and scale to 0..255
uint8_t toByte(float v)
{
  if (v < 0.0) v = 0.0;
  if (v > 1.0) v = 1.0;
  return (uint8_t)(v * 255.0 + 0.5);
}

//serial commands from the serial monitor
void handleSerial(float red, float green, float blue)
{
  if (!Serial.available()) return;
  char cmd = Serial.read();

  if (cmd == 'w' && red > 0 && green > 0 && blue > 0)
  {
    //make white read as equal r, g, b at 100% brightness
    float avg = (red + green + blue) / 3.0;
    gainR = avg / red;
    gainG = avg / green;
    gainB = avg / blue;
    whiteLevel = avg;
    Serial.println("white reference set");
  }
  else if (cmd == 'r')
  {
    maxChannelSeen = -1;
    maxBright = -1;
    Serial.println("brightness reset");
  }
}

void triggerAudio(int trackNumber)
{
  Serial.print(" | PLAY_TRACK:");
  Serial.println(trackNumber);
}

```

### Output of the readings
<img width="1164" height="527" alt="image" src="https://github.com/user-attachments/assets/a4dc9e06-98b6-42b5-9cfb-d9e408971ae6" />
<img width="3000" height="4000" alt="TEMP1795403520356" src="https://github.com/user-attachments/assets/454f4dcb-1174-4904-9408-ff7c76020ed9" />

### Connecting the player and the speaker to the Arduino Nano 
- https://www.youtube.com/watch?v=AWpRGvP-9Lo
- https://www.circuito.io/app?components=9442,11022,336411
<img width="1351" height="722" alt="image" src="https://github.com/user-attachments/assets/77205a4f-01d6-436c-824b-358c5b8fc12f" />

Basic code for DPF mini player
```

// Include Libraries
#include "Arduino.h"


// Pin Definitions
#define DFPLAYER_PIN_TX	10
#define DFPLAYER_PIN_RX	11



// Global variables and defines

// object initialization


// define vars for testing menu
const int timeout = 10000;       //define timeout of 10 sec
char menuOption = 0;
long time0;

// Setup the essentials for your circuit to work. It runs first every time your circuit is powered with electricity.
void setup() 
{
    // Setup Serial which is useful for debugging
    // Use the Serial Monitor to view printed messages
    Serial.begin(9600);
    while (!Serial) ; // wait for serial port to connect. Needed for native USB
    Serial.println("start");
    
    
    menuOption = menu();
    
}

// Main logic of your circuit. It defines the interaction between the components you selected. After setup, it runs over and over again, in an eternal loop.
void loop() 
{
    
    
    if(menuOption == '1')
    {
    // Disclaimer: The DFPlayer - Mini MP3 Player is in testing and/or doesn't have code, therefore it may be buggy. Please be kind and report any bugs you may find.
    }
    
    if (millis() - time0 > timeout)
    {
        menuOption = menu();
    }
    
}



// Menu function for selecting the components to be tested
// Follow serial monitor for instrcutions
char menu()
{

    Serial.println(F("\nWhich component would you like to test?"));
    Serial.println(F("(1) DFPlayer - Mini MP3 Player"));
    Serial.println(F("(menu) send anything else or press on board reset button\n"));
    while (!Serial.available());

    // Read data from serial monitor if received
    while (Serial.available()) 
    {
        char c = Serial.read();
        if (isAlphaNumeric(c)) 
        {   
            
            if(c == '1') 
    			Serial.println(F("Now Testing DFPlayer - Mini MP3 Player - note that this component doesn't have a test code"));
            else
            {
                Serial.println(F("illegal input!"));
                return 0;
            }
            time0 = millis();
            return c;
        }
    }
}

/*******************************************************

*    Circuito.io is an automatic generator of schematics and code for off
*    the shelf hardware combinations.

*    Copyright (C) 2016 Roboplan Technologies Ltd.

*    This program is free software: you can redistribute it and/or modify
*    it under the terms of the GNU General Public License as published by
*    the Free Software Foundation, either version 3 of the License, or
*    (at your option) any later version.

*    This program is distributed in the hope that it will be useful,
*    but WITHOUT ANY WARRANTY; without even the implied warranty of
*    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
*    GNU General Public License for more details.

*    You should have received a copy of the GNU General Public License
*    along with this program.  If not, see <http://www.gnu.org/licenses/>.

*    In addition, and without limitation, to the disclaimers of warranties 
*    stated above and in the GNU General Public License version 3 (or any 
*    later version), Roboplan Technologies Ltd. ("Roboplan") offers this 
*    program subject to the following warranty disclaimers and by using 
*    this program you acknowledge and agree to the following:
*    THIS PROGRAM IS PROVIDED ON AN "AS IS" AND "AS AVAILABLE" BASIS, AND 
*    WITHOUT WARRANTIES OF ANY KIND EITHER EXPRESS OR IMPLIED.  ROBOPLAN 
*    HEREBY DISCLAIMS ALL WARRANTIES, EXPRESS OR IMPLIED, INCLUDING BUT 
*    NOT LIMITED TO IMPLIED WARRANTIES OF MERCHANTABILITY, TITLE, FITNESS 
*    FOR A PARTICULAR PURPOSE, NON-INFRINGEMENT, AND THOSE ARISING BY 
*    STATUTE OR FROM A COURSE OF DEALING OR USAGE OF TRADE. 
*    YOUR RELIANCE ON, OR USE OF THIS PROGRAM IS AT YOUR SOLE RISK.
*    ROBOPLAN DOES NOT GUARANTEE THAT THE PROGRAM WILL BE FREE OF, OR NOT 
*    SUSCEPTIBLE TO, BUGS, SECURITY BREACHES, OR VIRUSES. ROBOPLAN DOES 
*    NOT WARRANT THAT YOUR USE OF THE PROGRAM, INCLUDING PURSUANT TO 
*    SCHEMATICS, INSTRUCTIONS OR RECOMMENDATIONS OF ROBOPLAN, WILL BE SAFE 
*    FOR PERSONAL USE OR FOR PRODUCTION OR COMMERCIAL USE, WILL NOT 
*    VIOLATE ANY THIRD PARTY RIGHTS, WILL PROVIDE THE INTENDED OR DESIRED
*    RESULTS, OR OPERATE AS YOU INTENDED OR AS MAY BE INDICATED BY ROBOPLAN. 
*    YOU HEREBY WAIVE, AGREE NOT TO ASSERT AGAINST, AND RELEASE ROBOPLAN, 
*    ITS LICENSORS AND AFFILIATES FROM, ANY CLAIMS IN CONNECTION WITH ANY OF 
*    THE ABOVE. 
********************************************************/
```

-------

## Accomplished
- Tried out the spectral sensor with the Arduino Nano

## What needs to be done
- Add the speaker to the Arduino with the mini player since it needs to be soldered. We also don't have any female-to-female wires, so we need those, or we'll have to use a breadboard
- Communicate the measurements to Sophie and Bianca, so they can incorporate the portable battery.
------
< [Week 9](https://github.com/neobundle/CART-470-Project-Journal/blob/week-9/Journal-Entry-Week-9.md)&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp; &emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[Week 11](https://github.com/neobundle/CART-470-Project-Journal/blob/week-11/Journal-Entry-Week-11.md) >
