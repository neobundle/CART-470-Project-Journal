# Week 11
Terence, Jolene, and I finally assembled the Arduino with the speaker and the sensor. Initially, the speaker wasn't working, so we had to try it out with another speaker and the Arduino alone. After a few hours of testing, the speaker worked, and we were able to add the sensor to it.

--------
I added the file we used for the Arduino and the speaker/mini player alone to the sensor file we had. It didn't work out, but after modifying the code, it finally worked.

### Sketch we used for the breadboard and to actually connect everything
<img width="389" height="500" alt="image" src="https://github.com/user-attachments/assets/5b51ca92-a929-4dca-91b3-c2579a57d99a" />
<img width="2728" height="2046" alt="image" src="https://github.com/user-attachments/assets/df24fe97-92dd-4d35-bdfd-2093927a1e34" />


### 1st version (Sensor and Player/Speaker)
It's detecting colors according to brightness and plays the sound files that Namiko made that are also placed depending on brightness

```
/*
  as7265x color + tone classifier v5 (standalone, dfplayer mini)
  track = how light or dark the color is (matches the sound files: 1 = lightest, 4 = darkest)
  still prints hex, hsb and color family for every reading
  white led stays on steady (no flashing)
  plays automatically on power-up, no computer needed

  sd card: 001.mp3 to 004.mp3 (optional 005.mp3 as background)
  play(n) uses the order files were copied, so format the card and copy them one by one in order

  wiring (nano):
  dfplayer vcc -> 5v, gnd -> gnd
  dfplayer rx -> d11 through 1k resistor
  dfplayer tx -> d10
  dfplayer spk_1 / spk_2 -> speaker
  triad sda -> a4, scl -> a5, 3v3 -> 3.3v, gnd -> gnd

  serial commands (type in serial monitor, press enter):
  w = white reference, hold white paper on the sensor first, then paste the printed values below
  r = reset learned brightness
*/

#include "SparkFun_AS7265X.h"
#include <SoftwareSerial.h>
#include "DFRobotDFPlayerMini.h"

AS7265X sensor;
SoftwareSerial mp3Serial(10, 11); //nano rx (d10), tx (d11)
DFRobotDFPlayerMini player;

const int VOLUME = 30;        //0-30
const int AMBIENT_TRACK = 0;  //set to 5 if you add 005.mp3 as background, 0 = repeat last clip

int currentTrack = 0; //track the sensor is on right now, 0 = empty
int lastPlayed = 0;
bool isPlaying = false;            //true while a clip is playing
unsigned long playStart = 0;
const unsigned long MAX_CLIP_MS = 60000; //safety, if the finished message is missed

//how many readings in a row must agree before a track is sent
const int STABLE_COUNT = 2;
int candidateTrack = -1;
int stableReadings = 0;

//brightness cutoffs between tracks, tune these from the serial monitor
//above LIGHT_1 = track 1, above LIGHT_2 = track 2, above LIGHT_3 = track 3, below = track 4
const float LIGHT_1 = 80.0;
const float LIGHT_2 = 65.0;
const float LIGHT_3 = 50.0;

//hue zones in degrees (0 = red, 30 = orange, 60 = yellow, 270 = purple, 330 = pink)
const float RED_ZONE_START = 345.0;   //red zone wraps past 360 to RED_ZONE_END
const float RED_ZONE_END = 12.0;
const float ORANGE_ZONE_END = 45.0;
const float PURPLE_ZONE_START = 255.0;
const float PINK_ZONE_START = 300.0;

//saturation splits inside each zone (0-1)
const float RED_SAT = 0.55;
const float ORANGE_SAT = 0.55;
const float PINK_SAT = 0.40;
const float PURPLE_SAT = 0.35;

//below this saturation it's white / grey / black, counts as unknown
const float NEUTRAL_SAT = 0.08;

//hsb brightness (0-100) below this = dark tone, above = light tone (just for the printout)
const float DARK_B = 45.0;

//absolute minimum brightness, below = nothing under the sensor
const float EMPTY_MIN = 5.0;
//below this fraction of the brightest reading = nothing under the sensor
const float EMPTY_RATIO = 0.10;
//hsb brightness below this = too dark to be a real piece, counts as empty
const float EMPTY_B = 20.0;

const char* familyNames[6] = {"red", "orange", "pink", "peach", "mauve", "purple"};

//white balance, replace with the values printed after sending 'w'
float gainR = 1.0, gainG = 1.0, gainB = 1.0;
float whiteLevel = -1; //-1 = not calibrated, uses the brightest thing seen instead

//fallback if no white reference: brightest channel seen so far
float maxChannelSeen = -1;
//brightest total reading seen, used for empty detection
float maxBright = -1;

void setup()
{
  Serial.begin(115200);
  Serial.println("as7265x color + tone classifier v5 initializing...");

  if (sensor.begin() == false)
  {
    Serial.println("sensor does not appear to be connected. freezing...");
    while (1);
  }

  sensor.disableIndicator(); //turn off the blue status led

  //steady white light only, no uv / ir flashing
  sensor.setBulbCurrent(AS7265X_LED_CURRENT_LIMIT_12_5MA, AS7265x_LED_WHITE);
  sensor.enableBulb(AS7265x_LED_WHITE);

  mp3Serial.begin(9600); //dfplayer only talks at 9600
  if (!player.begin(mp3Serial)) //same as the test sketch that works
  {
    Serial.println("dfplayer not found. check wiring and sd card.");
  }
  player.volume(VOLUME);
  delay(500);

  Serial.println("ready.");
}

void loop()
{
  updateAudio();

  sensor.takeMeasurements(); //white led is already on

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
    setEmpty();
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

  //too dark to be a real piece, treat as empty
  if (bright < EMPTY_B)
  {
    setEmpty();
    return;
  }

  Serial.print("hex: "); Serial.print(hex);
  Serial.print(" | hsb: ");
  Serial.print(hue, 0); Serial.print(", ");
  Serial.print(sat * 100.0, 0); Serial.print(", ");
  Serial.print(bright, 0);

  int family = classifyFamily(hue, sat);
  if (family < 0)
  {
    Serial.print(" | unknown"); //still plays a sound based on brightness
  }
  else
  {
    Serial.print(" | "); Serial.print(familyNames[family]);
    Serial.print(bright < DARK_B ? " dark" : " light");
  }

  //only send a track once it's stable for a few readings
  int track = brightnessToTrack(bright);
  if (track == candidateTrack)
  {
    stableReadings++;
  }
  else
  {
    candidateTrack = track;
    stableReadings = 1;
  }

  if (stableReadings >= STABLE_COUNT)
  {
    triggerAudio(track);
  }
  else
  {
    Serial.println(); //not stable yet
  }

  delay(200);
}

//nothing under the sensor
void setEmpty()
{
  Serial.println("empty");
  currentTrack = 0;
  candidateTrack = -1;
  stableReadings = 0;
  delay(200);
}

int brightnessToTrack(float bright)
{
  if (bright >= LIGHT_1) return 1;
  if (bright >= LIGHT_2) return 2;
  if (bright >= LIGHT_3) return 3;
  return 4;
}

//returns 0-5 for red, orange, pink, peach, mauve, purple, or -1 if unknown
int classifyFamily(float hue, float sat)
{
  if (sat < NEUTRAL_SAT) return -1; //white / grey / black

  if (hue >= RED_ZONE_START || hue < RED_ZONE_END)
  {
    return (sat >= RED_SAT) ? 0 : 2; //red or pink
  }
  if (hue >= RED_ZONE_END && hue < ORANGE_ZONE_END)
  {
    return (sat >= ORANGE_SAT) ? 1 : 3; //orange or peach
  }
  if (hue >= PINK_ZONE_START && hue < RED_ZONE_START)
  {
    return (sat >= PINK_SAT) ? 2 : 4; //pink or mauve
  }
  if (hue >= PURPLE_ZONE_START && hue < PINK_ZONE_START)
  {
    return (sat >= PURPLE_SAT) ? 5 : 4; //purple or mauve
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

    //print values to hardcode for standalone use
    Serial.print("gainR = "); Serial.println(gainR, 4);
    Serial.print("gainG = "); Serial.println(gainG, 4);
    Serial.print("gainB = "); Serial.println(gainB, 4);
    Serial.print("whiteLevel = "); Serial.println(whiteLevel, 4);
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
  currentTrack = trackNumber;
  Serial.print(" | PLAY_TRACK:");
  Serial.println(trackNumber);
}

//plays clips without cutting them off, no silence between scans
void updateAudio()
{
  checkPlayer();
  if (isPlaying) return; //clip still playing, let it finish

  int t = 0;
  if (currentTrack > 0)      t = currentTrack;
  else if (AMBIENT_TRACK > 0) t = AMBIENT_TRACK;
  else if (lastPlayed > 0)    t = lastPlayed;
  if (t == 0) return; //nothing detected yet

  player.play(t);
  if (currentTrack > 0) lastPlayed = currentTrack;
  isPlaying = true;
  playStart = millis();
  Serial.print("dfplayer: playing track "); Serial.println(t);
}

//listens for messages from the dfplayer, mainly "play finished"
void checkPlayer()
{
  if (player.available())
  {
    uint8_t type = player.readType();
    int value = player.read();
    printDetail(type, value);
    if (type == DFPlayerPlayFinished || type == DFPlayerError) isPlaying = false;
  }
  if (isPlaying && millis() - playStart > MAX_CLIP_MS) isPlaying = false;
}

void printDetail(uint8_t type, int value){
  switch (type) {
    case TimeOut:
      Serial.println(F("Time Out!"));
      break;
    case WrongStack:
      Serial.println(F("Stack Wrong!"));
      break;
    case DFPlayerCardInserted:
      Serial.println(F("Card Inserted!"));
      break;
    case DFPlayerCardRemoved:
      Serial.println(F("Card Removed!"));
      break;
    case DFPlayerCardOnline:
      Serial.println(F("Card Online!"));
      break;
    case DFPlayerUSBInserted:
      Serial.println("USB Inserted!");
      break;
    case DFPlayerUSBRemoved:
      Serial.println("USB Removed!");
      break;
    case DFPlayerPlayFinished:
      Serial.print(F("Number:"));
      Serial.print(value);
      Serial.println(F(" Play Finished!"));
      break;
    case DFPlayerError:
      Serial.print(F("DFPlayerError:"));
      switch (value) {
        case Busy:
          Serial.println(F("Card not found"));
          break;
        case Sleeping:
          Serial.println(F("Sleeping"));
          break;
        case SerialWrongStack:
          Serial.println(F("Get Wrong Stack"));
          break;
        case CheckSumNotMatch:
          Serial.println(F("Check Sum Not Match"));
          break;
        case FileIndexOut:
          Serial.println(F("File Index Out of Bound"));
          break;
        case FileMismatch:
          Serial.println(F("Cannot Find File"));
          break;
        case Advertise:
          Serial.println(F("In Advertise"));
          break;
        default:
          break;
      }
      break;
    default:
      break;
  }
  
}
```

### 2nd version - Random fallback
When it's not detecting a specific family color (pink, mauve, peach, beige, red), it plays a random track
```
/*
  as7265x color + tone classifier v6 (standalone, dfplayer mini, random fallback)
  known color = track by how light or dark it is (1 = lightest, 4 = darkest)
  unknown color or nothing under the sensor = random track, never the same twice in a row
  still prints hex, hsb and color family for every reading
  white led stays on steady (no flashing)
  plays automatically on power-up, no computer needed

  sd card: 001.mp3 to 004.mp3
  play(n) uses the order files were copied, so format the card and copy them one by one in order

  wiring (nano):
  dfplayer vcc -> 5v, gnd -> gnd
  dfplayer rx -> d11 through 1k resistor
  dfplayer tx -> d10
  dfplayer spk_1 / spk_2 -> speaker
  triad sda -> a4, scl -> a5, 3v3 -> 3.3v, gnd -> gnd

  serial commands (type in serial monitor, press enter):
  w = white reference, hold white paper on the sensor first, then paste the printed values below
  r = reset learned brightness
*/

#include "SparkFun_AS7265X.h"
#include <SoftwareSerial.h>
#include "DFRobotDFPlayerMini.h"

AS7265X sensor;
SoftwareSerial mp3Serial(10, 11); //nano rx (d10), tx (d11)
DFRobotDFPlayerMini player;

const int VOLUME = 30;        //0-30
const int NUM_TRACKS = 4;     //number of sound files on the sd card
const int RANDOM_TRACK = -1;  //means "pick a random track"

int currentTrack = RANDOM_TRACK; //track the sensor is on right now, random until a color is found
int lastPlayed = 0;
bool isPlaying = false;            //true while a clip is playing
unsigned long playStart = 0;
const unsigned long MAX_CLIP_MS = 60000; //safety, if the finished message is missed

//how many readings in a row must agree before a track is sent
const int STABLE_COUNT = 2;
int candidateTrack = 0; //0 = nothing yet
int stableReadings = 0;

//brightness cutoffs between tracks, tune these from the serial monitor
//above LIGHT_1 = track 1, above LIGHT_2 = track 2, above LIGHT_3 = track 3, below = track 4
const float LIGHT_1 = 80.0;
const float LIGHT_2 = 65.0;
const float LIGHT_3 = 50.0;

//hue zones in degrees (0 = red, 30 = orange, 60 = yellow, 270 = purple, 330 = pink)
const float RED_ZONE_START = 345.0;   //red zone wraps past 360 to RED_ZONE_END
const float RED_ZONE_END = 12.0;
const float ORANGE_ZONE_END = 45.0;
const float PURPLE_ZONE_START = 255.0;
const float PINK_ZONE_START = 300.0;

//saturation splits inside each zone (0-1)
const float RED_SAT = 0.55;
const float ORANGE_SAT = 0.55;
const float PINK_SAT = 0.40;
const float PURPLE_SAT = 0.35;

//below this saturation it's white / grey / black, counts as unknown
const float NEUTRAL_SAT = 0.08;

//hsb brightness (0-100) below this = dark tone, above = light tone (just for the printout)
const float DARK_B = 45.0;

//absolute minimum brightness, below = nothing under the sensor
const float EMPTY_MIN = 5.0;
//below this fraction of the brightest reading = nothing under the sensor
const float EMPTY_RATIO = 0.10;
//hsb brightness below this = too dark to be a real piece, counts as empty
const float EMPTY_B = 20.0;

const char* familyNames[6] = {"red", "orange", "pink", "peach", "mauve", "purple"};

//white balance, replace with the values printed after sending 'w'
float gainR = 1.0, gainG = 1.0, gainB = 1.0;
float whiteLevel = -1; //-1 = not calibrated, uses the brightest thing seen instead

//fallback if no white reference: brightest channel seen so far
float maxChannelSeen = -1;
//brightest total reading seen, used for empty detection
float maxBright = -1;

void setup()
{
  Serial.begin(115200);
  Serial.println("as7265x color + tone classifier v5 initializing...");

  if (sensor.begin() == false)
  {
    Serial.println("sensor does not appear to be connected. freezing...");
    while (1);
  }

  sensor.disableIndicator(); //turn off the blue status led

  //steady white light only, no uv / ir flashing
  sensor.setBulbCurrent(AS7265X_LED_CURRENT_LIMIT_12_5MA, AS7265x_LED_WHITE);
  sensor.enableBulb(AS7265x_LED_WHITE);

  mp3Serial.begin(9600); //dfplayer only talks at 9600
  if (!player.begin(mp3Serial)) //same as the test sketch that works
  {
    Serial.println("dfplayer not found. check wiring and sd card.");
  }
  player.volume(VOLUME);
  delay(500);

  randomSeed(analogRead(A0)); //unused pin, gives a different random order each power-up

  Serial.println("ready.");
}

void loop()
{
  updateAudio();

  sensor.takeMeasurements(); //white led is already on

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
    setEmpty();
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

  //too dark to be a real piece, treat as empty
  if (bright < EMPTY_B)
  {
    setEmpty();
    return;
  }

  Serial.print("hex: "); Serial.print(hex);
  Serial.print(" | hsb: ");
  Serial.print(hue, 0); Serial.print(", ");
  Serial.print(sat * 100.0, 0); Serial.print(", ");
  Serial.print(bright, 0);

  int family = classifyFamily(hue, sat);
  if (family < 0)
  {
    Serial.print(" | unknown"); //still plays a sound based on brightness
  }
  else
  {
    Serial.print(" | "); Serial.print(familyNames[family]);
    Serial.print(bright < DARK_B ? " dark" : " light");
  }

  //only send a track once it's stable for a few readings
  //unknown colors ask for a random track instead of a brightness one
  int track = (family < 0) ? RANDOM_TRACK : brightnessToTrack(bright);
  if (track == candidateTrack)
  {
    stableReadings++;
  }
  else
  {
    candidateTrack = track;
    stableReadings = 1;
  }

  if (stableReadings >= STABLE_COUNT)
  {
    triggerAudio(track);
  }
  else
  {
    Serial.println(); //not stable yet
  }

  delay(200);
}

//nothing under the sensor
void setEmpty()
{
  Serial.println("empty");
  currentTrack = RANDOM_TRACK; //nothing there, keep playing random clips
  candidateTrack = 0;
  stableReadings = 0;
  delay(200);
}

int brightnessToTrack(float bright)
{
  if (bright >= LIGHT_1) return 1;
  if (bright >= LIGHT_2) return 2;
  if (bright >= LIGHT_3) return 3;
  return 4;
}

//returns 0-5 for red, orange, pink, peach, mauve, purple, or -1 if unknown
int classifyFamily(float hue, float sat)
{
  if (sat < NEUTRAL_SAT) return -1; //white / grey / black

  if (hue >= RED_ZONE_START || hue < RED_ZONE_END)
  {
    return (sat >= RED_SAT) ? 0 : 2; //red or pink
  }
  if (hue >= RED_ZONE_END && hue < ORANGE_ZONE_END)
  {
    return (sat >= ORANGE_SAT) ? 1 : 3; //orange or peach
  }
  if (hue >= PINK_ZONE_START && hue < RED_ZONE_START)
  {
    return (sat >= PINK_SAT) ? 2 : 4; //pink or mauve
  }
  if (hue >= PURPLE_ZONE_START && hue < PINK_ZONE_START)
  {
    return (sat >= PURPLE_SAT) ? 5 : 4; //purple or mauve
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

    //print values to hardcode for standalone use
    Serial.print("gainR = "); Serial.println(gainR, 4);
    Serial.print("gainG = "); Serial.println(gainG, 4);
    Serial.print("gainB = "); Serial.println(gainB, 4);
    Serial.print("whiteLevel = "); Serial.println(whiteLevel, 4);
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
  currentTrack = trackNumber;
  Serial.print(" | PLAY_TRACK:");
  if (trackNumber == RANDOM_TRACK) Serial.println("random");
  else Serial.println(trackNumber);
}

//plays clips without cutting them off, no silence between scans
void updateAudio()
{
  checkPlayer();
  if (isPlaying) return; //clip still playing, let it finish

  //known color = its track, unknown or empty = random
  int t = (currentTrack == RANDOM_TRACK) ? randomTrack() : currentTrack;

  player.play(t);
  lastPlayed = t;
  isPlaying = true;
  playStart = millis();
  Serial.print("dfplayer: playing track "); Serial.println(t);
}

//picks a random track that isn't the one that just played, so it never repeats twice in a row
int randomTrack()
{
  if (NUM_TRACKS < 2) return 1;
  int t = lastPlayed;
  while (t == lastPlayed) t = random(1, NUM_TRACKS + 1);
  return t;
}

//listens for messages from the dfplayer, mainly "play finished"
void checkPlayer()
{
  if (player.available())
  {
    uint8_t type = player.readType();
    int value = player.read();
    printDetail(type, value);
    if (type == DFPlayerPlayFinished || type == DFPlayerError) isPlaying = false;
  }
  if (isPlaying && millis() - playStart > MAX_CLIP_MS) isPlaying = false;
}

void printDetail(uint8_t type, int value){
  switch (type) {
    case TimeOut:
      Serial.println(F("Time Out!"));
      break;
    case WrongStack:
      Serial.println(F("Stack Wrong!"));
      break;
    case DFPlayerCardInserted:
      Serial.println(F("Card Inserted!"));
      break;
    case DFPlayerCardRemoved:
      Serial.println(F("Card Removed!"));
      break;
    case DFPlayerCardOnline:
      Serial.println(F("Card Online!"));
      break;
    case DFPlayerUSBInserted:
      Serial.println("USB Inserted!");
      break;
    case DFPlayerUSBRemoved:
      Serial.println("USB Removed!");
      break;
    case DFPlayerPlayFinished:
      Serial.print(F("Number:"));
      Serial.print(value);
      Serial.println(F(" Play Finished!"));
      break;
    case DFPlayerError:
      Serial.print(F("DFPlayerError:"));
      switch (value) {
        case Busy:
          Serial.println(F("Card not found"));
          break;
        case Sleeping:
          Serial.println(F("Sleeping"));
          break;
        case SerialWrongStack:
          Serial.println(F("Get Wrong Stack"));
          break;
        case CheckSumNotMatch:
          Serial.println(F("Check Sum Not Match"));
          break;
        case FileIndexOut:
          Serial.println(F("File Index Out of Bound"));
          break;
        case FileMismatch:
          Serial.println(F("Cannot Find File"));
          break;
        case Advertise:
          Serial.println(F("In Advertise"));
          break;
        default:
          break;
      }
      break;
    default:
      break;
  }
  
} 
```

### 3rd version (Fully random)
It plays any sound file regardless of the colors
````
/*
  as7265x scanner v7 (standalone, dfplayer mini, fully random)
  every clip is random, the color doesn't change what plays (never the same track twice in a row)
  the sensor still prints hex / hsb / color so you can see the readings
  still prints hex, hsb and color family for every reading
  white led stays on steady (no flashing)
  plays automatically on power-up, no computer needed

  sd card: 001.mp3 to 004.mp3
  play(n) uses the order files were copied, so format the card and copy them one by one in order

  wiring (nano):
  dfplayer vcc -> 5v, gnd -> gnd
  dfplayer rx -> d11 through 1k resistor
  dfplayer tx -> d10
  dfplayer spk_1 / spk_2 -> speaker
  triad sda -> a4, scl -> a5, 3v3 -> 3.3v, gnd -> gnd

  serial commands (type in serial monitor, press enter):
  w = white reference, hold white paper on the sensor first, then paste the printed values below
  r = reset learned brightness
*/

#include "SparkFun_AS7265X.h"
#include <SoftwareSerial.h>
#include "DFRobotDFPlayerMini.h"

AS7265X sensor;
SoftwareSerial mp3Serial(10, 11); //nano rx (d10), tx (d11)
DFRobotDFPlayerMini player;

const int VOLUME = 30;        //0-30
const int NUM_TRACKS = 4;     //number of sound files on the sd card

int lastPlayed = 0;
bool isPlaying = false;            //true while a clip is playing
unsigned long playStart = 0;
const unsigned long MAX_CLIP_MS = 60000; //safety, if the finished message is missed


//hue zones in degrees (0 = red, 30 = orange, 60 = yellow, 270 = purple, 330 = pink)
const float RED_ZONE_START = 345.0;   //red zone wraps past 360 to RED_ZONE_END
const float RED_ZONE_END = 12.0;
const float ORANGE_ZONE_END = 45.0;
const float PURPLE_ZONE_START = 255.0;
const float PINK_ZONE_START = 300.0;

//saturation splits inside each zone (0-1)
const float RED_SAT = 0.55;
const float ORANGE_SAT = 0.55;
const float PINK_SAT = 0.40;
const float PURPLE_SAT = 0.35;

//below this saturation it's white / grey / black, counts as unknown
const float NEUTRAL_SAT = 0.08;

//hsb brightness (0-100) below this = dark tone, above = light tone (just for the printout)
const float DARK_B = 45.0;

//absolute minimum brightness, below = nothing under the sensor
const float EMPTY_MIN = 5.0;
//below this fraction of the brightest reading = nothing under the sensor
const float EMPTY_RATIO = 0.10;
//hsb brightness below this = too dark to be a real piece, counts as empty
const float EMPTY_B = 20.0;

const char* familyNames[6] = {"red", "orange", "pink", "peach", "mauve", "purple"};

//white balance, replace with the values printed after sending 'w'
float gainR = 1.0, gainG = 1.0, gainB = 1.0;
float whiteLevel = -1; //-1 = not calibrated, uses the brightest thing seen instead

//fallback if no white reference: brightest channel seen so far
float maxChannelSeen = -1;
//brightest total reading seen, used for empty detection
float maxBright = -1;

void setup()
{
  Serial.begin(115200);
  Serial.println("as7265x color + tone classifier v5 initializing...");

  if (sensor.begin() == false)
  {
    Serial.println("sensor does not appear to be connected. freezing...");
    while (1);
  }

  sensor.disableIndicator(); //turn off the blue status led

  //steady white light only, no uv / ir flashing
  sensor.setBulbCurrent(AS7265X_LED_CURRENT_LIMIT_12_5MA, AS7265x_LED_WHITE);
  sensor.enableBulb(AS7265x_LED_WHITE);

  mp3Serial.begin(9600); //dfplayer only talks at 9600
  if (!player.begin(mp3Serial)) //same as the test sketch that works
  {
    Serial.println("dfplayer not found. check wiring and sd card.");
  }
  player.volume(VOLUME);
  delay(500);

  randomSeed(analogRead(A0)); //unused pin, gives a different random order each power-up

  Serial.println("ready.");
}

void loop()
{
  updateAudio();

  sensor.takeMeasurements(); //white led is already on

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
    setEmpty();
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

  //too dark to be a real piece, treat as empty
  if (bright < EMPTY_B)
  {
    setEmpty();
    return;
  }

  Serial.print("hex: "); Serial.print(hex);
  Serial.print(" | hsb: ");
  Serial.print(hue, 0); Serial.print(", ");
  Serial.print(sat * 100.0, 0); Serial.print(", ");
  Serial.print(bright, 0);

  int family = classifyFamily(hue, sat);
  if (family < 0)
  {
    Serial.print(" | unknown"); //still plays a sound based on brightness
  }
  else
  {
    Serial.print(" | "); Serial.print(familyNames[family]);
    Serial.print(bright < DARK_B ? " dark" : " light");
  }

  Serial.println(); //audio is random, the reading doesn't pick the track

  delay(200);
}

//nothing under the sensor
void setEmpty()
{
  Serial.println("empty");
  delay(200);
}

//returns 0-5 for red, orange, pink, peach, mauve, purple, or -1 if unknown
int classifyFamily(float hue, float sat)
{
  if (sat < NEUTRAL_SAT) return -1; //white / grey / black

  if (hue >= RED_ZONE_START || hue < RED_ZONE_END)
  {
    return (sat >= RED_SAT) ? 0 : 2; //red or pink
  }
  if (hue >= RED_ZONE_END && hue < ORANGE_ZONE_END)
  {
    return (sat >= ORANGE_SAT) ? 1 : 3; //orange or peach
  }
  if (hue >= PINK_ZONE_START && hue < RED_ZONE_START)
  {
    return (sat >= PINK_SAT) ? 2 : 4; //pink or mauve
  }
  if (hue >= PURPLE_ZONE_START && hue < PINK_ZONE_START)
  {
    return (sat >= PURPLE_SAT) ? 5 : 4; //purple or mauve
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

    //print values to hardcode for standalone use
    Serial.print("gainR = "); Serial.println(gainR, 4);
    Serial.print("gainG = "); Serial.println(gainG, 4);
    Serial.print("gainB = "); Serial.println(gainB, 4);
    Serial.print("whiteLevel = "); Serial.println(whiteLevel, 4);
  }
  else if (cmd == 'r')
  {
    maxChannelSeen = -1;
    maxBright = -1;
    Serial.println("brightness reset");
  }
}

//plays random clips back to back, each one finishes before the next
void updateAudio()
{
  checkPlayer();
  if (isPlaying) return; //clip still playing, let it finish

  int t = randomTrack(); //color doesn't matter, always random

  player.play(t);
  lastPlayed = t;
  isPlaying = true;
  playStart = millis();
  Serial.print("dfplayer: playing track "); Serial.println(t);
}

//picks a random track that isn't the one that just played, so it never repeats twice in a row
int randomTrack()
{
  if (NUM_TRACKS < 2) return 1;
  int t = lastPlayed;
  while (t == lastPlayed) t = random(1, NUM_TRACKS + 1);
  return t;
}

//listens for messages from the dfplayer, mainly "play finished"
void checkPlayer()
{
  if (player.available())
  {
    uint8_t type = player.readType();
    int value = player.read();
    printDetail(type, value);
    if (type == DFPlayerPlayFinished || type == DFPlayerError) isPlaying = false;
  }
  if (isPlaying && millis() - playStart > MAX_CLIP_MS) isPlaying = false;
}

void printDetail(uint8_t type, int value){
  switch (type) {
    case TimeOut:
      Serial.println(F("Time Out!"));
      break;
    case WrongStack:
      Serial.println(F("Stack Wrong!"));
      break;
    case DFPlayerCardInserted:
      Serial.println(F("Card Inserted!"));
      break;
    case DFPlayerCardRemoved:
      Serial.println(F("Card Removed!"));
      break;
    case DFPlayerCardOnline:
      Serial.println(F("Card Online!"));
      break;
    case DFPlayerUSBInserted:
      Serial.println("USB Inserted!");
      break;
    case DFPlayerUSBRemoved:
      Serial.println("USB Removed!");
      break;
    case DFPlayerPlayFinished:
      Serial.print(F("Number:"));
      Serial.print(value);
      Serial.println(F(" Play Finished!"));
      break;
    case DFPlayerError:
      Serial.print(F("DFPlayerError:"));
      switch (value) {
        case Busy:
          Serial.println(F("Card not found"));
          break;
        case Sleeping:
          Serial.println(F("Sleeping"));
          break;
        case SerialWrongStack:
          Serial.println(F("Get Wrong Stack"));
          break;
        case CheckSumNotMatch:
          Serial.println(F("Check Sum Not Match"));
          break;
        case FileIndexOut:
          Serial.println(F("File Index Out of Bound"));
          break;
        case FileMismatch:
          Serial.println(F("Cannot Find File"));
          break;
        case Advertise:
          Serial.println(F("In Advertise"));
          break;
        default:
          break;
      }
      break;
    default:
      break;
  }
  
}
````
### Picture of what it looks like 
<img width="3000" height="4000" alt="TEMP1795618209844" src="https://github.com/user-attachments/assets/36b1fbc8-cdd3-465f-8f21-a9530d3d1afe" />

-------

## Accomplished
- Assembled the prototype and tried it out with the cardboard box

## What needs to be done
- Show Nadia what it looks like and the different versions
  
------
< [Week 10](https://github.com/neobundle/CART-470-Project-Journal/blob/week-6/Journal-Entry-Week-10.md)
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp; &emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp; [Week 12](https://github.com/neobundle/CART-470-Project-Journal/blob/week-10/Journal-Entry-Week-12.md)>
