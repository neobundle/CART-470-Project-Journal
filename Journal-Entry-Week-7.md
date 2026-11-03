# Week 7

We met with Nadia, and we were able to get some feedback from her. She would prefer the scanner to be a bit smaller or longer. So we have to look into ways for it to be maybe wireless or to come with a battery.
She also mentioned that the tapestry would be shades of pink and very feminine colours, so we need to modify the code so that the scanner can detect and distinguish all kinds of reds.

--------
| Topic | Resources | 
| ------ | --------|
| Wireless Arduino | https://www.youtube.com/shorts/2cpeT-Y6tsQ <br> https://www.instructables.com/Wireless-upload-program-to-Arduino-without-USB-cab/ <br> https://www.youtube.com/shorts/2cpeT-Y6tsQ | 
| Battery Arduino | https://www.instructables.com/Powering-Arduino-with-a-Battery/ <br> https://forum.arduino.cc/t/simplest-battery-power-to-arduino-nano-solution/530242/2 <br> |

-------

After researching what could be done, the best course of action would be to create a scanner that could connect to a portable bank. Nadia said she could provide the material for us, so I created an Amazon list with the items necessary to make this possible. To use the prototype without a computer and to make it a bit smaller, we need an Arduino Nano, a speaker, an audio player, and a microSD card. 

Link to the list: https://www.amazon.ca/hz/wishlist/ls/BGGTLWL3U7R1?ref_=wl_share.

### Code to distinguish lighter and darker tones (adapted to red). Claude modified with the help of Claude
```
/*
  as7265x color + tone classifier (no calibration needed)
  color family = hue angle, tone = brightness compared to the lightest / darkest seen so far

  tracks:
  coral light 1, coral dark 2
  pink light 3, pink dark 4
  mauve light 5, mauve dark 6
  peach light 7, peach dark 8
  neutral (white / cream / grey) 9
  nothing / unknown = no sound

  optional serial commands (type in serial monitor, press enter):
  w = white balance, hold something white (paper) on the sensor first
  r = reset the light / dark range
*/

#include "SparkFun_AS7265X.h"
AS7265X sensor;

//true = use the triad's leds (real objects in the box), false = phone screen tests
const bool USE_BULB = true;

//hue ranges in degrees (0 = red, 30 = orange, 60 = yellow, 300 = magenta)
const float MAUVE_START = 260.0;
const float PINK_START = 330.0;
const float CORAL_START = 8.0;  //pink wraps around 360 back to here
const float PEACH_START = 25.0;
const float PEACH_END = 60.0;

//below this saturation the color is basically white / cream / grey
const float NEUTRAL_SAT = 0.08;
//before a light/dark range is known, saturation below this counts as light (pastel)
const float PASTEL_SAT = 0.35;

//absolute minimum brightness, below = nothing under the sensor
const float EMPTY_MIN = 5.0;
//below this fraction of the brightest reading = nothing under the sensor
const float EMPTY_RATIO = 0.10;

//white balance gains, 1 = off until you send 'w'
float gainR = 1.0, gainG = 1.0, gainB = 1.0;

//darkest and brightest real readings seen since power on
float minBright = -1;
float maxBright = -1;

void setup()
{
  Serial.begin(115200);
  Serial.println("as7265x color + tone classifier initializing...");

  if (sensor.begin() == false)
  {
    Serial.println("sensor does not appear to be connected. freezing...");
    while (1);
  }

  sensor.disableIndicator(); //turn off the blue status led
  Serial.println("ready. scan a light and a dark object so it learns the tone range.");
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

  //convert to hue (degrees) and saturation (0-1)
  float maxC = max(red, max(green, blue));
  float minC = min(red, min(green, blue));
  float delta = maxC - minC;
  float sat = (maxC > 0) ? delta / maxC : 0;
  float hue = 0;
  if (delta > 0)
  {
    if (maxC == red)        hue = 60.0 * fmod((green - blue) / delta, 6.0);
    else if (maxC == green) hue = 60.0 * ((blue - red) / delta + 2.0);
    else                    hue = 60.0 * ((red - green) / delta + 4.0);
  }
  if (hue < 0) hue += 360.0;

  Serial.print("bright: "); Serial.print(brightness, 1);
  Serial.print(" | hue: "); Serial.print(hue, 0);
  Serial.print(" | sat: "); Serial.print(sat, 2);

  int track = classify(hue, sat, brightness);
  triggerAudio(track);

  delay(200);
}

int classify(float hue, float sat, float brightness)
{
  //nothing under the sensor
  if (brightness < EMPTY_MIN || (maxBright > 0 && brightness < maxBright * EMPTY_RATIO))
  {
    Serial.print(" | empty");
    return 0;
  }

  updateRange(brightness);

  if (sat < NEUTRAL_SAT)
  {
    Serial.print(" | neutral");
    return 9;
  }

  //pick the color family from the hue
  int family;
  if (hue >= PINK_START || hue < CORAL_START)        { family = 1; Serial.print(" | pink"); }
  else if (hue >= CORAL_START && hue < PEACH_START)  { family = 0; Serial.print(" | coral"); }
  else if (hue >= PEACH_START && hue < PEACH_END)    { family = 3; Serial.print(" | peach"); }
  else if (hue >= MAUVE_START && hue < PINK_START)   { family = 2; Serial.print(" | mauve"); }
  else
  {
    Serial.print(" | unknown color");
    return 0;
  }

  bool dark;
  if (maxBright > minBright * 1.5)
  {
    //range known, compare to the middle of darkest and brightest
    dark = brightness < (minBright + maxBright) / 2.0;
  }
  else
  {
    //range not known yet, guess from saturation, pastels are light
    dark = sat > PASTEL_SAT;
  }
  Serial.print(dark ? " dark" : " light");

  return family * 2 + 1 + (dark ? 1 : 0);
}

//remembers the darkest and brightest real readings
void updateRange(float brightness)
{
  if (minBright < 0 || brightness < minBright) minBright = brightness;
  if (maxBright < 0 || brightness > maxBright) maxBright = brightness;
}

//optional commands from the serial monitor
void handleSerial(float red, float green, float blue)
{
  if (!Serial.available()) return;
  char cmd = Serial.read();

  if (cmd == 'w' && red > 0 && green > 0 && blue > 0)
  {
    //make white read as equal r, g, b
    float avg = (red + green + blue) / 3.0;
    gainR = avg / red;
    gainG = avg / green;
    gainB = avg / blue;
    Serial.println("white balance set");
  }
  else if (cmd == 'r')
  {
    minBright = -1;
    maxBright = -1;
    Serial.println("tone range reset");
  }
}

void triggerAudio(int trackNumber)
{
  if (trackNumber == 0)
  {
    Serial.println();
    return; //nothing to play
  }
  Serial.print(" | PLAY_TRACK:");
  Serial.println(trackNumber);
}
```

## Accomplished
- Worked on the oral presentation
- Sent the list of items necessary to Nadia

## What needs to be done
- Start the new prototype
- Test out the code to see if we can differentiate tones
- Test out the new prototype


------
< [Week 6](https://github.com/neobundle/CART-470-Project-Journal/blob/week-6/Journal-Entry-Week-6.md)
