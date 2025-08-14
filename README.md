# BINGINE
Designed and built a Smart Dustbin using ultrasonic sensors, GPS, and Wi-Fi modules to monitor fill levels and track real-time location. Integrated with a web dashboard to display bin status and live map location, enabling efficient waste collection through IoT and real-time data updates.


#CODE for html csss javascript and arduno hardware
#include <WiFi.h>
#include <ESP32Servo.h>
#include <WebServer.h>
#include <TinyGPS++.h>
#include <HardwareSerial.h>

#define TRIG_WASTE 5
#define ECHO_WASTE 18
#define IR_LID 21  // Replacing TRIG_LID with IR sensor pin
#define SERVO_PIN 13

// GPS Pin Definitions
#define GPS_RX_PIN 16
#define GPS_TX_PIN 17

const char* ssid = "BINGENIE";
const char* password = "AGTAA12345";

WebServer server(80);
Servo lidServo;

#define MAX_LOGS 10
String lidOpenLogs[MAX_LOGS];
int logIndex = 0;

unsigned long bootTime;
unsigned long lastTrigger = 0;

// Set up GPS module
TinyGPSPlus gps;
HardwareSerial mySerial(1); // Use hardware serial 1

// Refined function to measure distance and improve accuracy
float getDistance(int trigPin, int echoPin) {
  digitalWrite(trigPin, LOW); // ensure the trigger pin is low
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH); // send a pulse
  delayMicroseconds(10); // wait for the pulse to be sent
  digitalWrite(trigPin, LOW); // stop sending pulse
  
  // Read the duration of the pulse
  long duration = pulseIn(echoPin, HIGH, 30000); // Timeout after 30 ms
  
  // Debugging: Print the duration value
  Serial.print("Duration: ");
  Serial.println(duration);

  // Check if no pulse was received or the duration is out of expected range
  if (duration == 0) {
    return -1; // Invalid reading (no pulse received)
  }

  // Convert the duration to distance in cm
  float distance = duration * 0.0343 / 2;

  // Debugging: Print the calculated distance value
  Serial.print("Distance: ");
  Serial.println(distance);

  // Check if the distance is in a valid range (e.g., 2 cm to 400 cm)
  if (distance < 2 || distance > 400) {
    return -1; // Invalid distance
  }

  return distance;
}

// Helper function to log lid open activity with timestamp
void logLidOpenTime() {
  String timestamp = String((millis() - bootTime) / 1000) + "s";
  if (logIndex < MAX_LOGS) {
    lidOpenLogs[logIndex++] = timestamp;
  } else {
    // Shift older logs
    for (int i = 1; i < MAX_LOGS; i++) {
      lidOpenLogs[i - 1] = lidOpenLogs[i];
    }
    lidOpenLogs[MAX_LOGS - 1] = timestamp;
  }
}

void handleRoot() {
  float distance = getDistance(TRIG_WASTE, ECHO_WASTE);
  
  if (distance == -1) {
    // If distance is invalid, return an error page
    String page = "<html><body><h1>Error: Invalid Distance Reading</h1><p>Please check the sensor and ensure the dustbin is positioned correctly.</p></body></html>";
    server.send(200, "text/html", page);
    return;
  }

  // Adjusting for a 15 cm dustbin height
  float percentFilled = constrain(100 - (distance / 15.0) * 100, 0, 100);
  
  String levelText, levelColor;
  if (percentFilled <= 5) {
    levelText = "Very Empty"; levelColor = "#00bfff";
  } else if (percentFilled <= 30) {
    levelText = "Low"; levelColor = "green";
  } else if (percentFilled <= 60) {
    levelText = "Medium"; levelColor = "orange";
  } else if (percentFilled <= 85) {
    levelText = "High"; levelColor = "#ff8c00";
  } else {
    levelText = "Full"; levelColor = "red";
  }

  // Get the current GPS data
  String lat = String(gps.location.lat(), 6);
  String lon = String(gps.location.lng(), 6);

  String page = "<!DOCTYPE html><html><head><meta charset='UTF-8'><meta name='viewport' content='width=device-width, initial-scale=1.0'>";
  page += "<title>Smart Dustbin</title><script src='https://cdn.jsdelivr.net/npm/chart.js'></script>";
  page += "<style>body{font-family:sans-serif;background:#f4f4f4;text-align:center;padding:20px;}";
  page += ".card{background:white;padding:20px;border-radius:12px;box-shadow:0 4px 8px rgba(0,0,0,0.1);margin:20px auto;max-width:600px;}";
  page += ".indicator{font-size:1.5em;padding:10px;border-radius:8px;color:white;font-weight:bold;}";
  page += ".box-container{display:flex;justify-content:space-between;margin-top:20px;}";
  page += ".fill-box{width:22%;background:#e0e0e0;border-radius:10px;padding:10px;font-weight:bold;}";
  page += ".color-strip{height:20px;border-radius:5px;margin-top:5px;transition:background 0.5s;}";
  page += ".green{background:green;}.yellow{background:yellow;}.orange{background:orange;}.red{background:red;}";
  page += "table{width:100%;margin-top:10px;border-collapse:collapse;}th,td{padding:8px;border-bottom:1px solid #ddd;}";
  page += "canvas{max-width:100%;margin-top:20px;}</style></head><body>";

  page += "<div class='card'><h2>🗑 Smart Dustbin Dashboard</h2><p>Fill Level: <strong id='fillValue'>" + String(percentFilled, 1) + "%</strong></p>";
  page += "<div class='indicator' id='fillIndicator' style='background:" + levelColor + ";'>" + levelText + "</div>";
  page += "<div class='box-container'>";
  page += String("<div class='fill-box'>0%<div class='color-strip") + (percentFilled <= 30 ? " green" : "") + "'></div></div>";
  page += String("<div class='fill-box'>25%<div class='color-strip") + ((percentFilled > 30 && percentFilled <= 60) ? " yellow" : "") + "'></div></div>";
  page += String("<div class='fill-box'>50%<div class='color-strip") + ((percentFilled > 60 && percentFilled <= 85) ? " orange" : "") + "'></div></div>";
  page += String("<div class='fill-box'>100%<div class='color-strip") + (percentFilled > 85 ? " red" : "") + "'></div></div></div></div>";

  page += "<div class='card'><h3>📈 Fill History (Live)</h3><canvas id='fillChart' width='400' height='200'></canvas></div>";

  page += "<div class='card'><h3>📜 Lid Open Log</h3>";
  page += "<table id='lidLogsTable'><tr><th>#</th><th>Time</th><th>Status</th></tr>"; // Logs will be filled here
  page += "</table></div>";

  page += "<div class='card'><h3>📡 System Info</h3>";
  page += "<p>IP: " + WiFi.localIP().toString() + "</p>";
  page += "<p>Uptime: " + String((millis() - bootTime) / 1000) + " seconds</p>";
  page += "<p>Signal: " + String(WiFi.RSSI()) + " dBm</p>";
  page += "<p>Location: Latitude " + lat + ", Longitude " + lon + "</p></div>"; // Show GPS location

  page += "<script>let fillData = [];let timeLabels = [];let chartCtx = document.getElementById('fillChart').getContext('2d');";
  page += "let fillChart = new Chart(chartCtx, {type: 'line',data: {labels: timeLabels,datasets: [{label: 'Fill %',data: fillData,borderColor: 'green',tension: 0.3}]},options: {responsive: true,scales: {y: {min: 0,max: 100}}}});";
  page += "function getLevelText(percent) {";
  page += "if (percent <= 5) return ['Very Empty', '#00bfff'];";
  page += "if (percent <= 30) return ['Low', 'green'];";
  page += "if (percent <= 60) return ['Medium', 'orange'];";
  page += "if (percent <= 85) return ['High', '#ff8c00'];";
  page += "return ['Full', 'red'];}";
  page += "function updateChart() {fetch('/data').then(res => res.json()).then(d => {if (!d.fill) return; let percent = d.fill; let seconds = d.time;";
  page += "if (timeLabels.length > 20) {timeLabels.shift(); fillData.shift();} timeLabels.push(seconds + 's'); fillData.push(percent); fillChart.update();";
  page += "let [text, color] = getLevelText(percent); document.getElementById('fillValue').innerText = percent.toFixed(1) + '%'; document.getElementById('fillIndicator').innerText = text; document.getElementById('fillIndicator').style.background = color;});} setInterval(updateChart, 2000);</script>";

  page += "<script>function updateLidLogs() {fetch('/logs').then(res => res.json()).then(logs => {let logsTable = document.getElementById('lidLogsTable'); logsTable.innerHTML = \"<tr><th>#</th><th>Time</th><th>Status</th></tr>\"; logs.forEach((log, index) => {let row = \"<tr><td>\" + (index + 1) + \"</td><td>\" + log + \"</td><td>Opened</td></tr>\"; logsTable.innerHTML += row;});});} setInterval(updateLidLogs, 5000);</script>";

  page += "</body></html>";
  server.send(200, "text/html", page);
}

void handleData() {
  float distance = getDistance(TRIG_WASTE, ECHO_WASTE);
  
  if (distance == -1) {
    // Return an error message if the distance is invalid
    String json = "{\"error\":\"Invalid Distance Reading\"}";
    server.send(200, "application/json", json);
    return;
  }

  // Adjusting for a 15 cm dustbin height
  float fill = constrain(100 - (distance / 15.0) * 100, 0, 100);
  unsigned long timeNow = (millis() - bootTime) / 1000;

  String json = "{\"time\":" + String(timeNow) + ",\"fill\":" + String(fill, 1) + "}";
  server.send(200, "application/json", json);
}

void handleLogs() {
  String logsJson = "[";
  for (int i = 0; i < logIndex; i++) {
    logsJson += "\"" + lidOpenLogs[i] + "\"";
    if (i < logIndex - 1) logsJson += ",";
  }
  logsJson += "]";
  server.send(200, "application/json", logsJson);
}

void setup() {
  Serial.begin(115200);
  pinMode(TRIG_WASTE, OUTPUT); pinMode(ECHO_WASTE, INPUT);
  pinMode(IR_LID, INPUT);
  lidServo.attach(SERVO_PIN); lidServo.write(0);
  
  // Initialize GPS module
  mySerial.begin(9600, SERIAL_8N1, GPS_RX_PIN, GPS_TX_PIN);  // Baud rate of 9600

  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) { delay(500); Serial.print("."); }
  Serial.println("\nWiFi connected at " + WiFi.localIP().toString());
  
  server.on("/", handleRoot);
  server.on("/data", handleData);
  server.on("/logs", handleLogs); // Endpoint for lid open logs
  server.begin();
  bootTime = millis();
}

void loop() {
  server.handleClient();

  // Reading GPS data
  while (mySerial.available() > 0) {
    gps.encode(mySerial.read());
  }

  bool objectDetected = digitalRead(IR_LID) == LOW;  // LOW = object present
  static bool lidOpen = false;
  static unsigned long objectLastSeen = 0;

  if (objectDetected) {
    if (!lidOpen) {
      Serial.println("IR sensor detected object near lid.");
      lidServo.write(76);
      Serial.println("Servo moved to 76 degrees.");
      lidOpen = true;
      logLidOpenTime(); // log when lid opens
    }
    objectLastSeen = millis();
  } else {
    if (lidOpen && (millis() - objectLastSeen > 2000)) {
      Serial.println("Object removed. Closing lid...");
      lidServo.write(0);
      lidOpen = false;
    }
  }

  delay(100);  // Short delay for stability
}
