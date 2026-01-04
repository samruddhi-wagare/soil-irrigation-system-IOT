#include <ESP8266WiFi.h>
#include <WiFiClientSecure.h>
#include <AdafruitIO_WiFi.h>
#include <DHT.h>

// ========== Wi-Fi Credentials ==========
const char* ssid = "Your wifi name";
const char* password = "your wifi password";

// ========== Adafruit IO Credentials ==========
#define IO_USERNAME  "YOUR_ADAFRUIT_IO_KEY"
#define IO_KEY       "PASTE_YOUR_KEY_HERE"

AdafruitIO_WiFi io(IO_USERNAME, IO_KEY, ssid, password);

// ========== DHT11 Setup ==========
#define DHTPIN D4
#define DHTTYPE DHT11
DHT dht(DHTPIN, DHTTYPE);

// ========== Soil Moisture Sensor ==========
#define SOIL_PIN A0

// ========== Relay/Motor Setup ==========
#define MOTOR_PIN D5
bool motorOverride = false;
int manualMotorState = 0;

// ========== Adafruit IO Feeds ==========
AdafruitIO_Feed *temperatureFeed = io.feed("temperature");
AdafruitIO_Feed *humidityFeed = io.feed("humidity");
AdafruitIO_Feed *soilMoistureFeed = io.feed("soil_moisture");
AdafruitIO_Feed *motorFeed = io.feed("motor");

void setup() {
  Serial.begin(115200);

  pinMode(MOTOR_PIN, OUTPUT);
  digitalWrite(MOTOR_PIN, LOW); // Motor OFF initially

  dht.begin();

  Serial.print("Connecting to WiFi");
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi connected");

  Serial.print("Connecting to Adafruit IO");
  io.connect();
  while (io.status() < AIO_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nConnected to Adafruit IO");

  motorFeed->onMessage(handleMotor);
}

void loop() {
  io.run(); // Adafruit IO connection

  // Read DHT11 sensor
  float humidity = dht.readHumidity();
  float temperature = dht.readTemperature();

  // Read Soil Moisture
  int soilRaw = analogRead(SOIL_PIN);
  int soilMoisture = map(soilRaw, 1023, 0, 0, 100); // 0 = dry, 100 = wet

  // If DHT fails
  if (isnan(humidity) || isnan(temperature)) {
    Serial.println("DHT read failed!");
    return;
  }

  // Send data to Adafruit IO
  humidityFeed->save(humidity);
  temperatureFeed->save(temperature);
  soilMoistureFeed->save(soilMoisture);

  // Print to Serial
  Serial.print("Humidity: "); Serial.print(humidity); Serial.print("%\t");
  Serial.print("Temp: "); Serial.print(temperature); Serial.print("°C\t");
  Serial.print("Soil Moisture: "); Serial.print(soilMoisture); Serial.println("%");

  // Motor control logic
  if (motorOverride) {
    digitalWrite(MOTOR_PIN, manualMotorState);
    Serial.println(manualMotorState ? "Manual Pump ON" : "Manual Pump OFF");
  } else {
    if (soilMoisture < 30) {
      digitalWrite(MOTOR_PIN, HIGH);  // Motor ON
 

      Serial.println("Soil dry. Auto Pump ON");
    } else {
      digitalWrite(MOTOR_PIN, LOW);
      Serial.println("Soil wet. Auto Pump OFF");
    }
  }

  delay(10000); // Delay for 10 seconds
}

// ========== Motor Command Handler ==========
void handleMotor(AdafruitIO_Data *data) {
  int value = data->toInt();
  Serial.print("Motor Command: ");
  Serial.println(value);

  if (value == 1 || value == 0) {
    motorOverride = true;
    manualMotorState = value;
  } else {
    motorOverride = false;
    Serial.println("Auto Mode Enabled");
  }
}Project source code files
