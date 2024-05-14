#include "WiFi.h"
#include "ESPAsyncWebServer.h"
#include <Adafruit_Sensor.h>
#include <DHT.h>

const char* ssid = "test";
const char* password = "123456789";

#define DHTPIN 16
#define DHTTYPE    DHT22

DHT dht(DHTPIN, DHTTYPE);

AsyncWebServer server(80);

// Circular buffer to store last 5 temperature readings
float lastTemperature[5] = {0};
// Circular buffer to store last 5 humidity readings
float lastHumidity[5] = {0};
// Index to keep track of the current position in the circular buffer
int currentIndex = 0;

String readDHTTemperature() {
  float t = dht.readTemperature();
  if (isnan(t)) {    
    Serial.println("Failed to read temperature from DHT sensor!");
    return "--";
  } else {
    Serial.println(t);
    lastTemperature[currentIndex] = t;
    currentIndex = (currentIndex + 1) % 5;
    return String(t);
  }
}

String readDHTHumidity() {
  float h = dht.readHumidity();
  if (isnan(h)) {
    Serial.println("Failed to read humidity from DHT sensor!");
    return "--";
  } else {
    Serial.println(h);
    lastHumidity[currentIndex] = h;
    currentIndex = (currentIndex + 1) % 5;
    return String(h);
  }
}

const char index_html[] PROGMEM = R"rawliteral(
<!DOCTYPE HTML>
<html>
<head>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/milligram/1.4.1/milligram.min.css">
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.15.4/css/all.min.css">
  <style>
    html {
      font-family: Arial;
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100%;
      background-color: #e0f7fa; /* Light blue background */
    }
    body {
      text-align: center;
    }
    h2 { 
      font-size: 3.5rem;
      font-weight: bold;
    }
    p { 
      font-size: 2.5rem;
      font-weight: bold;
    }
    .units { 
      font-size: 1.5rem; 
      font-weight: bold;
    }
    .sensor-container {
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-items: center;
      margin-bottom: 20px;
    }
    .sensor-info {
      width: 100%;
    }
    .progress-bar {
      width: 100%;
      background-color: #ddd;
      border-radius: 20px;
      padding: 3px;
    }
    .progress-bar-fill {
      height: 20px;
      text-align: center;
      line-height: 20px;
      color: white;
      width: 0%;
      border-radius: 20px;
    }
    .readings {
      font-size: 2rem;
      text-align: left;
      margin-top: 20px;
    }
    ul {
      list-style-type: none;
      padding: 0;
      margin: 0;
    }
    li {
      background-color: #e0f7fa; /* Background color */
      padding: 10px;
      margin-bottom: 5px;
    }
  </style>
</head>
<body>
  <h2>Teplomer a vlhkomer</h2>
  
  <!-- Temperature and Humidity section -->
  <div class="sensor-container">
    <div class="sensor-info">
      <p>
        <i class="fas fa-thermometer-half" style="color:#059e8a;"></i>
        <span id="temperature">%TEMPERATURE%</span>
        <sup class="units">&deg;C</sup>
      </p>
      <div class="progress-bar">
        <div class="progress-bar-fill" id="temperature-bar"></div>
      </div>
    </div>
    <div class="sensor-info">
      <p>
        <i class="fas fa-tint" style="color:#00add6;"></i>
        <span id="humidity">%HUMIDITY%</span>
        <sup class="units">&percnt;</sup>
      </p>
    </div>
  </div>
  
  <!-- Previous Readings section -->
  <div class="readings">
    <h3>Predchadzajuce hodnoty:</h3>
    <ul id="readings">%READINGS%</ul>
  </div>
</body>
<script>
setInterval(function ( ) {
  var xhttp = new XMLHttpRequest();
  xhttp.onreadystatechange = function() {
    if (this.readyState == 4 && this.status == 200) {
      document.getElementById("temperature").innerHTML = this.responseText;
      var temperature = parseFloat(this.responseText);
      var temperatureBar = document.getElementById("temperature-bar");
      if (temperature >= 0 && temperature <= 20) {
        temperatureBar.style.backgroundColor = "#FBBD06";
      } else if (temperature >= 21 && temperature <= 26) {
        temperatureBar.style.backgroundColor = "#FBAD06";
      } else if (temperature >= 27 && temperature <= 35) {
        temperatureBar.style.backgroundColor = "#FB7C06";
      } else {
        temperatureBar.style.backgroundColor = "#FB4B06";
      }
      temperatureBar.style.width = ((temperature / 50) * 100) + "%";
    }
  };
  xhttp.open("GET", "/temperature", true);
  xhttp.send();
}, 10000 ) ;

setInterval(function ( ) {
  var xhttp = new XMLHttpRequest();
  xhttp.onreadystatechange = function() {
    if (this.readyState == 4 && this.status == 200) {
      document.getElementById("humidity").innerHTML = this.responseText;
    }
  };
  xhttp.open("GET", "/humidity", true);
  xhttp.send();
}, 10000 ) ;

setInterval(function ( ) {
  var xhttp = new XMLHttpRequest();
  xhttp.onreadystatechange = function() {
    if (this.readyState == 4 && this.status == 200) {
      document.getElementById("readings").innerHTML = this.responseText;
    }
  };
  xhttp.open("GET", "/readings", true);
  xhttp.send();
}, 10000 ) ;
</script>
</html>)rawliteral";

// Replaces placeholder with DHT values and last 5 readings
String processor(const String& var){
  if(var == "TEMPERATURE"){
    return readDHTTemperature();
  }
  else if(var == "HUMIDITY"){
    return readDHTHumidity();
  }
  else if(var == "READINGS"){
    String readingsList;
    for (int i = 0; i < 5; ++i) {
      int idx = (currentIndex - i - 1 + 5) % 5; // Compute index in circular buffer
      readingsList += "<li>Teplota: " + String(lastTemperature[idx]) + "°C, Vlhkost: " + String(lastHumidity[idx]) + "%</li>";
    }
    return readingsList;
  }
  return String();
}

void setup(){
  Serial.begin(115200);
  dht.begin();
  
  IPAddress local_ip(192, 168, 1, 1);
  IPAddress gateway(192, 168, 1, 1);
  IPAddress subnet(255, 255, 255, 0);
  WiFi.softAP(ssid, password);
  WiFi.softAPConfig(local_ip, gateway, subnet);
  Serial.print("Connect to My access point: ");
  Serial.println(ssid);

  server.begin();
  Serial.println("HTTP server started");
  Serial.println(WiFi.localIP());

  server.on("/", HTTP_GET, [](AsyncWebServerRequest *request){
    request->send_P(200, "text/html", index_html);
  });
  server.on("/temperature", HTTP_GET, [](AsyncWebServerRequest *request){
    request->send_P(200, "text/plain", readDHTTemperature().c_str());
  });
  server.on("/humidity", HTTP_GET, [](AsyncWebServerRequest *request){
    request->send_P(200, "text/plain", readDHTHumidity().c_str());
  });
  server.on("/readings", HTTP_GET, [](AsyncWebServerRequest *request){
    String readingsList;
    for (int i = 0; i < 5; ++i) {
      int idx = (currentIndex - i - 1 + 5) % 5; // Compute index in circular buffer
      readingsList += "<li>Teplota: " + String(lastTemperature[idx]) + "°C, Vlhkost: " + String(lastHumidity[idx]) + "%</li>";
    }
    request->send(200, "text/html", readingsList);
  });
}

void loop(){
}
