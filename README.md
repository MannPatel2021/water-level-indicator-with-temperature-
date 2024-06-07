// Author: Mann Patel\

// Water-level-indicator code


#include <WiFi.h>
#include <ESPAsyncWebServer.h>
#include <OneWire.h>
#include <DallasTemperature.h>

// Define Ultrasonic Sensor Pins
const int trigPin = 5; //RX
const int echoPin = 18; //TX


// Define Relay Pin
const int relayPin = 2;

// Define OneWire Bus and Temperature Sensor
const int oneWireBus = 4;
OneWire oneWire(oneWireBus);
DallasTemperature sensors(&oneWire);

// Define WiFi Credentials
const char* ssid = "ESP32_Water_Level";
const char* password = "12345678";

// Create AsyncWebServer Object on Port 80
AsyncWebServer server(80);

// Variables to store the water level, distance, and temperature
float distance = 0;
float waterLevel = 0;
float waterTemperature;
float tankHeight = 100.0; // Default tank height in cm

// Variable to store motor status
bool motorStatus = false;

void setup() {
  // Start Serial Communication
  Serial.begin(115200);

  // Intialize ultrasonic sensor
  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);

  // Set Relay Pin as Output
  pinMode(relayPin, OUTPUT);
  digitalWrite(relayPin, LOW);

  // Initialize temperature sensor
  sensors.begin();

  // Initialize WiFi as Access Point
  WiFi.softAP(ssid, password);
  Serial.println("Access Point Started");
  Serial.print("IP Address: ");
  Serial.println(WiFi.softAPIP());

  // Serve HTML Web Page
  server.on("/", HTTP_GET, [](AsyncWebServerRequest *request){
    String html = "<!DOCTYPE html><html><head><meta name='viewport' content='width=device-width, initial-scale=1.0'>";
    html += "<style>";
    html += "body { font-family: Arial, sans-serif; text-align: center; margin: 0; padding: 0; }";
    html += ".container { max-width: 100%; height: 100vh; display: flex; flex-direction: column; justify-content: center; align-items: center; padding: 20px; box-sizing: border-box; }";
    html += ".water-tank { position: relative; width: 100%; max-width: 300px; height: 300px; background: #00bfff; border: 3px solid #000; border-radius: 10px; overflow: hidden; }";
    html += ".water-level { position: absolute; bottom: 0; width: 100%; background: #0077be; }";
    html += "button { font-size: 20px; padding: 10px 20px; margin: 10px; width: 100%; max-width: 300px; }";
    html += "footer { width: 100%; background: #000; color: #fff; padding: 10px; box-sizing: border-box; margin-top: 20px; }";
    html += "@media (max-width: 600px) {";
    html += "  .water-tank { height: 200px; }";
    html += "  button { font-size: 16px; padding: 8px 16px; }";
    html += "  footer { font-size: 14px; }";
    html += "}";
    html += "</style></head><body><div class='container'>";
    html += "<h1>Water Tank Level</h1>";
    html += "<div class='water-tank'><div class='water-level' style='height: " + String(waterLevel) + "%;'></div></div>";
    html += "<p>Distance: " + String(distance, 2) + " cm</p>";
    html += "<p>Water Level: " + String(waterLevel, 2) + " %</p>";
    html += "<p>Water Temperature: " + String(waterTemperature, 2) + " °C</p>";
    html += "<p>Tank Height: <input type='number' id='tankHeight' value='" + String(tankHeight) + "' style='width: 100%; max-width: 100px;'> cm</p>";
    html += "<button onclick='setTankHeight()'>Set Tank Height</button>";
    html += "<button onclick='toggleMotor()'>Motor ON/OFF</button>";
    html += "<label for='motorSwitch'>Motor Status: </label>";
    html += "<input type='checkbox' id='motorSwitch' onchange='changeMotorStatus()' " + String(motorStatus ? "checked" : "") + ">";
    html += "<footer>Designed by Mann Patel</footer>";
    html += "<script>";
    html += "function toggleMotor() { fetch('/toggle'); }";
    html += "function changeMotorStatus() {";
    html += "  var isChecked = document.getElementById('motorSwitch').checked;";
    html += "  fetch('/toggle?status=' + isChecked);";
    html += "}";
    html += "function setTankHeight() {";
    html += "  var height = document.getElementById('tankHeight').value;";
    html += "  fetch('/setHeight?height=' + height);";
    html += "setInterval(function() { location.reload(); }, 5000);";
    html += "}";
    html += "setInterval(function() { location.reload(); }, 5000);";
    html += "</script></div></body></html>";
    request->send(200, "text/html", html);
  });

  // Toggle Motor Endpoint
  server.on("/toggle", HTTP_GET, [](AsyncWebServerRequest *request){
    if (request->hasArg("status")) {
      String status = request->arg("status");
      if (status == "true") {
        digitalWrite(relayPin, HIGH);
        motorStatus = true;
      } else {
        digitalWrite(relayPin, LOW);
        motorStatus = false;
      }
    } else {
      digitalWrite(relayPin, !digitalRead(relayPin));
      motorStatus = !motorStatus;
    }
    request->send(200, "text/plain", motorStatus ? "Motor Turned On" : "Motor Turned Off");
  });

  // Set Tank Height Endpoint
  server.on("/setHeight", HTTP_GET, [](AsyncWebServerRequest *request){
    if (request->hasArg("height")) {
      tankHeight = request->arg("height").toFloat();
    }
    request->send(200, "text/plain", "Tank Height Set to " + String(tankHeight) + " cm");
  });

  // Start Server
  server.begin();
}

void loop() {
  // Measure distance
  digitalWrite(trigPin, LOW);
  delayMicroseconds(4);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);
  distance = pulseIn(echoPin, HIGH); 
  distance = distance / 29 / 2 ;// Convert pulse duration to distance in cm

  waterLevel = ((tankHeight - distance) / tankHeight) * 100;

  // Measure Temperature
  sensors.requestTemperatures();
  waterTemperature = sensors.getTempCByIndex(0);

  Serial.print(distance);
  Serial.println("distance");
  //Serial.println(waterLevel);
  //Serial.println("level");
  //Serial.println(waterTemperature);
  //Serial.print("temp");
  
  
  // Delay between readings

  delay(1000);
}
