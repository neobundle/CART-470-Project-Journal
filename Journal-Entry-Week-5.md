## Week 5

Roles were assigned for the project, and we were able to communicate with Nadia.


| Category | Assigned |
|-------|------------|
| 3D Printing/Design |Sophie, Bianca |
| Arduino | Terence, Jolene, Dika |
| Audio output | Namiko |
| Organization | Jolene |

-----------------------------------------
We were able to obtain samples of ceramic beads so that we could accurately test out the scanner. We also decided to meet during reading week to work on the Arduino together.
In the meantime, I was able to search for alternative ways to create the color sensor. Due to time constraints, I felt it would be better to have alternatives in case things didn't work as expected.

I found multiple tutorials for the Arduino, the different sensors it could require, so that we have a lot of options to test when we meet.

### Material
Color sensor
<br>
<img width="425" height="415" alt="image" src="https://github.com/user-attachments/assets/b6479e48-3202-4258-861b-1e5ebdd04a96" />
<br>
https://www.amazon.ca/DAOKI-TCS-34725-TCS34725-Recognition-Arduino/dp/B08SLNXYDZ/ref=sr_1_5?sr=8-5

Audio player
<br>
<img width="425" height="415" alt="image" src="https://github.com/user-attachments/assets/8a066ab4-639d-455f-a1fe-78c0c579a969" />
<br>
https://www.amazon.ca/Aideepen-DFPlay-Player-Support-Arduino/dp/B07BDD8BF3/ref=sr_1_5?sr=8-5

| Type | Resources |
|-------|------------|
| Color detection with sensor | https://www.youtube.com/watch?v=Ey5UhCxlxis <br> https://projecthub.arduino.cc/SurtrTech/color-detection-using-tcs3200230-a1e463 <br> https://www.youtube.com/watch?v=oPFQrKC0nSg <br> https://www.youtube.com/watch?v=wHxPh1QxtKA <br> https://www.instructables.com/Introduction-29/ <br> https://www.tme.com/ca/en/news/library-articles/diy/page/70870/how-to-build-a-colour-sensor-with-arduino/|
| Adding the audio player to the Arduino | https://www.youtube.com/watch?v=UN9XPWHamHw|


---------------------
Code used for TCS34725 : 

```
//project 1 - RGB color sensor 
#include <Wire.h> 
#include "Adafruit_TCS34725.h"

// Create an instance of the TCS34725 sensor 
Adafruit_TCS34725 tcs = Adafruit_TCS34725(TCS34725_INTEGRATIONTIME_614MS, TCS34725_GAIN_1X);

void setup() { 
  Serial.begin(9600);

if (tcs.begin()) { 
  Serial.println("Found TCS34725 sensor"); 
  } 
else { 
    Serial.println("No TCS34725 found ... check your connections"); 
    while (1); // halt the program 
    } 
  }

void loop() { 
  uint16_t r, g, b, c; 
  tcs.getRawData(&r, &g, &b, &c);

// Check if there is significant color detected 
  if (c > 50) {   // Adjust the threshold as needed 
  // Normalize the values 
  float sum = c; 
  float red = r / sum; 
  float green = g / sum; 
  float blue = b / sum;

Serial.print("R: "); Serial.print(r); Serial.print(" ");
Serial.print("G: "); Serial.print(g); Serial.print(" ");
Serial.print("B: "); Serial.print(b); Serial.print(" ");
Serial.print("C: "); Serial.print(c); Serial.println(" ");

// Calculate the color temperature and lux
uint16_t colorTemp = tcs.calculateColorTemperature(r, g, b);
uint16_t lux = tcs.calculateLux(r, g, b);

Serial.print("Color Temp: "); Serial.print(colorTemp, DEC); Serial.print(" K - ");
Serial.print("Lux: "); Serial.println(lux, DEC);

// Determine the color
if (red > green && red > blue) {
  Serial.println("Detected Color: RED");
} else if (green > red && green > blue) {
  Serial.println("Detected Color: GREEN");
} else if (blue > red && blue > green) {
  Serial.println("Detected Color: BLUE");
} else {
  Serial.println("Detected Color: UNKNOWN");
}
} else { Serial.println("No significant color detected"); 
}

// Wait for 2 seconds before the next reading
 delay(2000); 
}
```

<img width="903" height="685" alt="image" src="https://github.com/user-attachments/assets/72601b0f-cb17-4381-987f-50bf2228477d" />

Arduino worked but it didn't detect the color properly so we were provided the triad spectroscopy sensor that would do the job better. 


We used the code from the manufacturer github : https://github.com/sparkfun/SparkFun_AS7265x_Arduino_Library/tree/main/examples

#### Readings from the spectral triad 
```
2526.01,177.28,133.32,63.31,62.56,35.24,46.35,22.16,16.92,28.63,
90.56,396.00,238.07,255.89,151.69,449.94,286.62,373.41,2528.31,177.28,142.21,63.31,62.56,35.24,46.35,22.16,16.92,28.63,
91.54,396.96,238.99,256.80,151.69,449.94,286.62,373.92,2530.61,177.28,134.31,63.31,63.45,35.24,46.35,22.16,16.92,28.63,
91.54,396.96,238.99,257.71,152.50,450.64,290.32,376.49,2536.36,178.16,134.31,63.73,63.45,35.24,47.25,22.16,16.92,28.63,
91.54,397.91,239.92,258.62,154.10,452.03,292.96,378.03,2542.11,179.48,134.31,64.16,63.45,35.24,47.25,22.16,16.92,28.63,
69.14,251.21,131.03,226.75,114.77,383.18,245.45,326.22,1817.76,148.17,94.81,54.75,45.58,23.49,34.77,17.50,13.66,24.68,
78.88,375.86,240.84,251.34,146.07,500.01,306.68,388.28,2603.04,174.19,146.16,66.30,61.67,36.08,46.35,24.50,16.92,29.62,
88.62,409.42,251.91,247.69,155.71,493.75,303.51,383.15,2565.10,179.92,137.28,65.45,61.67,36.08,47.25,24.50,17.57,29.62,
2.92,15.34,19.38,0.00,4.82,0.00,0.00,0.00,0.00,2.20,5.93,0.00,1.79,1.68,1.78,2.33,1.30,0.00,
0.00,0.00,0.00,0.00,0.00,0.00,0.00,0.00,0.00,0.00,0.00,0.00,0.89,1.68,1.78,3.50,0.00,0.00,
0.00,0.00,0.00,0.00,0.00,0.00,0.00,0.00,0.00,0.00,0.00,0.00,1.79,1.68,0.89,3.50,0.65,0.00,
68.17,392.16,288.82,234.03,165.34,549.39,279.76,373.41,2474.27,159.20,166.90,82.13,47.37,33.56,50.81,25.66,20.82,21.72,
84.72,452.57,300.82,287.76,178.98,632.84,308.26,452.40,2467.37,183.45,160.98,87.26,48.26,33.56,49.03,25.66,21.47,26.66,
70.11,385.45,278.67,296.87,160.52,705.16,349.96,489.84,2949.12,160.96,173.82,99.24,57.20,39.44,50.81,29.16,19.52,29.62,
87.64,464.07,310.97,0.00,185.40,700.29,349.44,489.33,2957.17,188.74,190.61,98.38,58.09,39.44,59.73,29.16,22.12,29.62,
87.64,
```

I was able to find a video that turns the readings into colors with an ESP32. It gave this [link](https://how2electronics.com/diy-colorimeter-using-as7265x-spectroscopy-sensor-esp32/) in the description and on there was present the code used in the video.

I modified, with the help of Claude, the existing code of the spectral sensor with the code provided in the link. Only the first 12 channels can be divided into RGB,since the others ones are not visible to the eye. The "press key" to enter was removed so it would detect color directly. Brightness isn't really considered in the code, so after the test we'll have to modify the code again if needed to distinguish light and dark.

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

-----------------------------------------
### What was accomplished
- Acquired the first batch of material (Arduino in my case)
- Met up with Jolene and Terence to build the sensor
- Tried two different sensors
- Settled on the sensor to use (Triad Spectroscopy)

### What remains
- Trying out the sensor with the new code
- Mapping out the colors to the audio 
- Trying out the audio with the Arduino
- Adding the audio component to the Arduino
  

