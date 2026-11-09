#define BLYNK_TEMPLATE_ID "TMPL3m0NjAiJe"
#define BLYNK_TEMPLATE_NAME "Smart Coal mine Helmet using IOT"
#define BLYNK_AUTH_TOKEN "HygTIJZKYqdiMJk8rpDrJcg7CbH4rFYB"

#define BLYNK_PRINT Serial

#include <ESP8266WiFi.h>
#include <BlynkSimpleEsp8266.h>

#include <Wire.h>
#include <Adafruit_BMP085.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>

// ================= WIFI =================
char auth[] = "YOUR_BLYNK_AUTH_TOKEN";
char ssid[] = "YOUR_WIFI_SSID";
char pass[] = "YOUR_WIFI_PASSWORD";

// ================= MQ2 =================
#define MQ2_PIN A0

// ================= OUTPUTS =================
#define RED_LED D5
#define BUZZER D6
#define HEADLIGHT D7
 
// ================= EMERGENCY BUTTON =================
#define EMERGENCY_BUTTON D8

// ================= BMP180 =================
Adafruit_BMP085 bmp;

// ================= MPU6050 =================
Adafruit_MPU6050 mpu;

BlynkTimer timer;

// ================= VARIABLES =================
float envTemperature;
float bodyTemperature;
float pressure;
float altitude;

int gasValue;

float ax, ay, az;

bool gasAlertSent = false;
bool tempAlertSent = false;
bool bodyTempAlertSent = false;
bool fallDetected = false;
bool emergencySent = false;
 
// ================= SENSOR FUNCTION =================
void sendSensorData()
{
  // ================= MQ2 =================
  gasValue = analogRead(MQ2_PIN);

  // ================= BMP180 =================
  envTemperature = bmp.readTemperature();
  pressure = bmp.readPressure() / 100.0; // hPa
  altitude = bmp.readAltitude();

  // ================= MPU6050 =================
  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);

  ax = a.acceleration.x;
  ay = a.acceleration.y;
  az = a.acceleration.z;

  // MPU6050 internal temperature
  bodyTemperature = temp.temperature;

  // ================= SERIAL MONITOR =================
  Serial.println("========== SENSOR DATA ==========");

  Serial.print("Gas Value: ");
  Serial.println(gasValue);

  Serial.print("Environmental Temperature: ");
  Serial.println(envTemperature);

  Serial.print("Body Temperature: ");
  Serial.println(bodyTemperature);

  Serial.print("Pressure: ");
  Serial.print(pressure);
  Serial.println(" hPa");

  Serial.print("Altitude: ");
  Serial.print(altitude);
  Serial.println(" m");

  Serial.print("Accel X: ");
  Serial.println(ax);

  Serial.print("Accel Y: ");
  Serial.println(ay);

  Serial.print("Accel Z: ");
  Serial.println(az);

  // ================= BLYNK =================
  Blynk.virtualWrite(V0, gasValue);
  Blynk.virtualWrite(V1, envTemperature);
  Blynk.virtualWrite(V2, pressure);
  Blynk.virtualWrite(V3, altitude);
  Blynk.virtualWrite(V4, ay);
  Blynk.virtualWrite(V5, bodyTemperature);


  // ================= GAS ALERT =================
  if (gasValue > 600)
  {
    digitalWrite(RED_LED, HIGH);
    digitalWrite(BUZZER, HIGH);
    delay(2000);
        digitalWrite(RED_LED, LOW);
    digitalWrite(BUZZER, LOW);
    if (!gasAlertSent)
    {
      Blynk.logEvent("gas_alert", "Harmful gasses are Detected!!! Please Evacuate this Place !!!!");
      Blynk.email("USER MAIL ID", "Alert", "Harmful gasses are Detected!!! Please Evacuate this Place !!!!");

      Serial.println("GAS LEAKAGE DETECTED!");
      gasAlertSent = true;
    }
  }
  else
  {
    gasAlertSent = false;
  }
 
  // ================= ENVIRONMENT TEMP ALERT =================
  if (envTemperature > 45)
  {
    digitalWrite(RED_LED, HIGH);
    digitalWrite(BUZZER, HIGH);
   delay(2000);
        digitalWrite(RED_LED, LOW);
    digitalWrite(BUZZER, LOW);
    if (!tempAlertSent)
    {
      Blynk.logEvent("env_temp", "High Environmental Temperature is Detected!!! Please take Some Action!!!!");
      Serial.println("High Environmental Temperature is Detected!!! Please take Some Action!!!");
     Blynk.email("USER MAIL ID ", "Alert", "High Environmental Temperature is Detected!!! Please take Some Action!!!!");

      tempAlertSent = true;
    }
  }
  else
  {
    tempAlertSent = false;
  }

  // ================= BODY TEMP ALERT =================
  if (bodyTemperature > 43)
  {
    digitalWrite(RED_LED, HIGH);
    digitalWrite(BUZZER, HIGH);
   delay(2000);
        digitalWrite(RED_LED, LOW);
    digitalWrite(BUZZER, LOW);
    if (!bodyTempAlertSent)
    {
      Blynk.logEvent("body_temp", "High Body Temperature is Detected!!! Please take Some Rest!!!!");
           Blynk.email("USER MAIL ID ", "Alert", "High Body Temperature is Detected!!! Please take Some Rest!!!!!!");
      Serial.println("HIGH BODY TEMPERATURE!");
      bodyTempAlertSent = true;
    }
  }
  else
  {
    bodyTempAlertSent = false;
  }

  // ================= FALL DETECTION =================
  if (ay > 8 || ay < -8 )
  {
    digitalWrite(RED_LED, HIGH);
    digitalWrite(BUZZER, HIGH);
   delay(2000);
        digitalWrite(RED_LED, LOW);
    digitalWrite(BUZZER, LOW);
    if (!fallDetected)
    {
      Blynk.logEvent("fall_alert", "Fall is Detected!!! Please take Some Action !!!!");
      Blynk.email("USER MAIL ID ", "Alert", "Fall is Detected!!! Please take Some Action !!!!!");
      Serial.println("FALL DETECTED!");
      fallDetected = true;
    }
  }
  else
  {
    fallDetected = false;
  }

  // ================= EMERGENCY BUTTON =================
  int emergencyState = digitalRead(EMERGENCY_BUTTON);

  if (emergencyState == HIGH)
  {
    digitalWrite(RED_LED, HIGH);
    digitalWrite(BUZZER, HIGH);
   delay(2000);
        digitalWrite(RED_LED, LOW);
    digitalWrite(BUZZER, LOW);
    Blynk.virtualWrite(V6, emergencyState);

    if (!emergencySent)
    {
      Blynk.logEvent("eme_alert", "Emergency Button is Pressed!!! Please take Some Action !!!!!");
      Blynk.email("USER MAIL ID ", "Alert", "Emergency Button is Pressed!!! Please take Some Action !!!!!!");

      Serial.println("EMERGENCY BUTTON PRESSED!");
      emergencySent = true;
    }
  }
  else
  {
    Blynk.virtualWrite(V4, 0);
    emergencySent = false;
  }

  // ================= NORMAL CONDITION =================
  if (gasValue <= 600 &&
      envTemperature <= 45 &&
      bodyTemperature <= 43 &&
      emergencyState == LOW)
  {
    digitalWrite(RED_LED, LOW);
    digitalWrite(BUZZER, LOW);
  }
}

// ================= SETUP =================
void setup()
{
  Serial.begin(115200);

  // ================= PIN MODES =================
  pinMode(RED_LED, OUTPUT);
  pinMode(BUZZER, OUTPUT);
  pinMode(HEADLIGHT, OUTPUT);

  pinMode(EMERGENCY_BUTTON, INPUT_PULLUP);

  // Headlight ON
  digitalWrite(HEADLIGHT, HIGH);

  // ================= BLYNK =================
  Blynk.begin(auth, ssid, pass);

  // ================= BMP180 =================
  if (!bmp.begin())
  {
    Serial.println("❌ BMP180 NOT FOUND!");
    while (1);
  }

  Serial.println("✅ BMP180 Found!");

  // ================= MPU6050 =================
  if (!mpu.begin())
  {
    Serial.println("❌ Failed to find MPU6050 chip!");
    while (1)
    {
      delay(10);
    }
  }

  Serial.println("✅ MPU6050 Found!");

  mpu.setAccelerometerRange(MPU6050_RANGE_8_G);
  mpu.setGyroRange(MPU6050_RANGE_500_DEG);
  mpu.setFilterBandwidth(MPU6050_BAND_5_HZ);

  Serial.println("✅ MPU6050 Configured!");

  // ================= TIMER =================
  timer.setInterval(1000L, sendSensorData);

  Serial.println("=================================");
  Serial.println("COAL MINE SAFETY HELMET READY");
  Serial.println("=================================");
}

// ================= LOOP =================
void loop()
{
  Blynk.run();
  timer.run();
}
