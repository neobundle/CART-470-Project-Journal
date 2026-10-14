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

I found multiple tutorials for the Arduino, and buying the sensor directly instead of making it ourselves would save a lot of time, especially for testing.

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


//project 1 - RGB color sensor
#include <Wire.h>
#include "Adafruit_TCS34725.h"

// Create an instance of the TCS34725 sensor
Adafruit_TCS34725 tcs = Adafruit_TCS34725(TCS34725_INTEGRATIONTIME_614MS, TCS34725_GAIN_1X);

void setup() {
  Serial.begin(9600);

  if (tcs.begin()) {
    Serial.println("Found TCS34725 sensor");
  } else {
    Serial.println("No TCS34725 found ... check your connections");
    while (1); // halt the program
  }
}

void loop() {
  uint16_t r, g, b, c;
  tcs.getRawData(&r, &g, &b, &c);

  // Check if there is significant color detected
  if (c > 50) { // Adjust the threshold as needed
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
  } else {
    Serial.println("No significant color detected");
  }

  // Wait for 2 seconds before the next reading
  delay(2000);
}





-----------------------------------------
### What was accomplished
- Acquired the first batch of material (Arduino in my case)

### What remains 
- Testing out the prototype
- Trying out the audio with the Arduino
- Adding the audio component to the Arduino
  

