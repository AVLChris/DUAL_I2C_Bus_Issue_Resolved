Report: Resolving I2C Bus Delays and Conflicts on ESP32
Issue Summary
When setting up an ESP32 to interface with two separate I2C devices on different buses (Wire for I2C1 and Wire1 for I2C2), significant delays and timeouts occurred during I2C bus scans. The devices were either not detected, or the scan on Wire1 (I2C2) was excessively slow. These delays persisted despite confirming proper wiring, voltage levels, and pull-up resistors.

Root Causes
Improper initialization of Wire and Wire1:
The ESP32 requires explicit initialization of both I2C buses using the begin() method, specifying the correct pins for each bus. Without this step, one or both buses can exhibit undefined behavior.
Faulty or insufficiently soldered I2C address selection resistors:
A poorly soldered connection on the OLED module caused address conflicts and unreliable communication.
Conflicting or unverified pin configurations:
Incorrect SDA/SCL pin assignments in code could lead to scanning failures or slow performance.
Solution
The following steps were taken to resolve the issue:

1. Verify Hardware Connections
Power Supply:
Ensure each I2C device and the ESP32 are supplied with the correct voltage (3.3V for the ESP32 and OLEDs).
Confirm that all devices share a common ground.
Pull-Up Resistors:
Use 4.7kΩ pull-up resistors on the SDA and SCL lines for both I2C buses. Measure and confirm the voltage is approximately 3.3V on these lines.
OLED Modules:
Check and fix solder joints on address selection resistors to ensure the proper I2C addresses (0x3C and 0x3D) are configured.
2. Configure Pins and Initialize I2C Buses
Assign unique SDA and SCL pins for each I2C bus. In this case:
I2C1 (Wire): SDA = 21, SCL = 22
I2C2 (Wire1): SDA = 25, SCL = 26
Initialize both buses explicitly in the setup() function:
cpp
Copy code
Wire.begin(21, 22);  // Initialize I2C1 with correct pins
Wire1.begin(25, 26); // Initialize I2C2 with correct pins
3. Implement Correct I2C Scanning Code
Use dedicated functions to scan each bus independently and ensure proper detection of devices. Below is a tested and working example:
cpp
Copy code
#include <Wire.h>

void setup() {
  Serial.begin(115200);
  delay(2000);

  // Initialize I2C buses
  Wire.begin(21, 22);  // SDA, SCL for I2C1
  Wire1.begin(25, 26); // SDA, SCL for I2C2

  // Scan each bus
  Serial.println("---------- Scanning Wire (I2C1) -------------");
  scanI2C(Wire);
  Serial.println("---------- Scanning Wire1 (I2C2) ------------");
  scanI2C(Wire1);
}

void loop() {}

void scanI2C(TwoWire &wire) {
  byte error, address;
  int nDevices = 0;

  Serial.println("Scanning...");
  for (address = 1; address < 127; address++) {
    wire.beginTransmission(address);
    error = wire.endTransmission();

    if (error == 0) {
      Serial.print("I2C device found at address 0x");
      if (address < 16)
        Serial.print("0");
      Serial.print(address, HEX);
      Serial.println("  !");
      nDevices++;
    } else if (error == 4) {
      Serial.print("Unknown error at address 0x");
      if (address < 16)
        Serial.print("0");
      Serial.println(address, HEX);
    }
  }

  if (nDevices == 0)
    Serial.println("No I2C devices found\n");
  else
    Serial.println("done\n");
}
4. Test Without Devices
To rule out hardware issues, test I2C buses without devices connected. Both buses should scan with no delay or timeouts, indicating proper software configuration.
5. Resolve Device-Specific Issues
Reconnect I2C devices one at a time and repeat the scan. Verify that:
Each device is detected at the expected address.
There are no delays or communication failures.
Key Learnings
Always initialize I2C buses explicitly: ESP32 requires both SDA and SCL pins to be defined during initialization of Wire and Wire1.
Verify hardware thoroughly: Issues such as poor soldering or incorrect pull-up resistor values can cause intermittent communication failures.
Use systematic testing: Isolate hardware and software components to pinpoint the root cause of issues.
Future Recommendations
Document Pin Configurations: Clearly label and document SDA/SCL pin assignments for each I2C bus in code and physical diagrams.
Use Debugging Tools: Utilize a logic analyzer or oscilloscope to monitor I2C signals when diagnosing complex issues.
Maintain Clean Connections: Double-check solder joints, wires, and pull-up resistors to prevent intermittent failures.
By following this process, the issue was successfully resolved, and the I2C buses are now functioning reliably.



// This is is a test project to troubleshoot the ESP1.
// The current issue is that the ESP1 is experiencing conflict when using 2 separate OLED displays.
// OLED1 is connected to the ESP1 via I2C1 and OLED2 is connected to the ESP1 via I2C2.
// OLED1 has the address 0x3C and OLED2 has the address 0x3D.
// I rebuilt a test board to troubleshoot the issue. 
// The test board has 2 OLED displays and 1 ESP1 module.
// The OLEDS are powered by a USB-C breakout board with a 3.3V voltage regulator.
// The ESP1 is powered by a USB breakout board which provides the 3.3V voltage needed.
// The ESP1 is connected to the computer via USB to upload code and monitor the serial output.
// All deviced are connected to a breadboard and have a common ground.
// For I2C1 and I2C2, each of the I2C lines (SDA and SCL) have a 4.7k pull-up resistor.
// I have confirmed that the voltage is correct for both the 5V and the 3.3 V lines.


/**
 * Parts list:
 * OLED                 - https://www.amazon.com/gp/product/B08VNRH5HR/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&th=1
 * ESP32                - https://www.amazon.com/gp/product/B0CDRN1WF7/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1
 * ESP32 Breakout Board - https://www.amazon.com/gp/product/B0CTBGLL9Z/ref=ox_sc_act_title_4?smid=A3S807LE0L63AP&th=1
 * USB-C Breakout Board - https://www.amazon.com/gp/product/B09KC1SMGD/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1
 * 4.7K Ohm Resistors   - https://www.amazon.com/gp/product/B07QJB3LGN/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&th=1
 * Voltage Regulator    - https://www.amazon.com/gp/product/B08R6337QY/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1
 */

#include <Arduino.h>
#include <Wire.h>


void I2C_ScannerWire() {
  byte error, address;
  int nDevices;

  Serial.println("Scanning...");

  nDevices = 0;
  for (address = 1; address < 127; address++) {
    Wire.beginTransmission(address);
    error = Wire.endTransmission();

    if (error == 0) {
      Serial.print("I2C device found at address 0x");
      if (address < 16) Serial.print("0");
      Serial.print(address, HEX);
      Serial.println("  !");
      nDevices++;
    } else if (error == 4) {
      Serial.print("Unknown error at address 0x");
      if (address < 16) Serial.print("0");
      Serial.println(address, HEX);
    }
  }
  if (nDevices == 0) {
    Serial.println("No I2C devices found\n");
  } else {
    Serial.println("done\n");
  }
}

void I2C_ScannerWire1() {
  byte error, address;
  int nDevices;

  Serial.println("Scanning...");

  nDevices = 0;
  for (address = 1; address < 127; address++) {
    Wire1.beginTransmission(address);
    error = Wire1.endTransmission();

    if (error == 0) {
      Serial.print("I2C device found at address 0x");
      if (address < 16) Serial.print("0");
      Serial.print(address, HEX);
      Serial.println("  !");
      nDevices++;
    } else if (error == 4) {
      Serial.print("Unknown error at address 0x");
      if (address < 16) Serial.print("0");
      Serial.println(address, HEX);
    }
  }
  if (nDevices == 0) {
    Serial.println("No I2C devices found\n");
  } else {
    Serial.println("done\n");
  }
}
void setup() {
  Serial.begin(115200);
  delay(3000); // Allow time for Serial Monitor to initialize

  // Initialize I2C1 and I2C2 with specified pins
  Wire.begin(21, 22);       // I2C1: SDA 21, SCL 22
  Wire1.begin(25, 26);      // I2C2: SDA 25, SCL 26

  Serial.println("---------- Scanning Wire (I2C1) -------------");
  I2C_ScannerWire();

  Serial.println("---------- Scanning Wire1 (I2C2) ------------");
  I2C_ScannerWire1();
}

void loop() {
  delay(10000); // Delay to prevent excessive scanning
}


