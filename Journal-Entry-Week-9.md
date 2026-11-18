# Week 9
The team couldn't meet this week for various reasons, but we communicated the reasons on Discord. We updated each other on our work. 3D Design has been progressing and Namiko proposed to do the tapestry so that we could have a mini version of it.
For the mini Arduino, we will meet to finally assemble it and try it out with the new code that Terence has been working on. 

------

This is the same table as last week. Nothing was updated, but it will be used during our meeting Wednesday so that we have on hand the tutorials needed to assemble the nano Arduino wiht the needed component.

--------
| Topic | Resources | 
| ------ | --------|
| Adding DFP Mini Player to Arduino | https://www.instructables.com/MP3-Player-With-Arduino-Using-DF-Player-Mini/ <br> https://www.youtube.com/watch?v=47BZahFHy1I <br> https://www.youtube.com/watch?v=u1qIvflHRts <br> https://www.instructables.com/Play-Audio-With-Arduino-DF-Player-Mini/ |
| Powering Arduino with the power bank | https://youtu.be/f0cYIduul2U?t=103 |
| Powering in general (backup plan) |  https://www.youtube.com/watch?v=f0cYIduul2U |
-------

## Accomplished
- Updated each other on what we were doing

## What needs to be done
- Assemble the nano Arduino
- Test it out with the code
- Give the measurements to Bianca and Sophie so that they can finish out the casing
- Try the following code

```
/*
  as7265x color + tone classifier v3
  color family = hue + saturation, tone = hsb brightness (light / dark)
  prints hex and hsb for every reading, unknown colors just print their hex

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

------
< [Week 8](https://github.com/neobundle/CART-470-Project-Journal/blob/week-6/Journal-Entry-Week-8.md)
