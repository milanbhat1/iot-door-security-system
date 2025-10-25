# IoT-Based Smart Door Security & Monitoring System (ESP8266 + Web UI + Telegram Alerts)

IoT project that implements password-protected door access with real-time alerts and presence detection using an ESP8266.

---

## Overview
This project provides a secure, networked door control system with:
- Web-based password authentication
- Relay-driven door actuation
- Ultrasonic proximity detection (HC-SR04)
- Buzzer alerts on unauthorized attempts
- Telegram notifications to multiple users (owner, secondary contact, police)
- Automatic lockout after multiple failed attempts

---

## Features
- Password-protected Web Login (HTML UI served by ES P8266)
- Relay control after successful authentication
- Buzzer alarm for incorrect attempts (2 seconds)
- Telegram notifications (HTTPS) via `WiFiClientSecure`
- Proximity alerts when someone is within 30 cm (alerts every 15s while present)
- Lockout after 5 failed attempts (1 minute)

---

## Hardware Required
- ESP8266 (NodeMCU / Wemos / similar)
- Relay Module (for door lock)
- HC-SR04 Ultrasonic Sensor
- Buzzer
- Jumper wires, breadboard/power supply

---

## Pinout / Circuit

| Component | ESP8266 Pin |
|-----------|-------------|
| Relay IN  | D6 (GPIO12) |
| Buzzer    | D0 (GPIO16) |
| Ultrasonic TRIG | D4 (GPIO2) |
| Ultrasonic ECHO | D2 (GPIO4) |

### ASCII Circuit (simple)

```
          +-----------+
          | ESP8266   |
          |           |
   TRIG --| D4     D6 |-- Relay IN
   ECHO --| D2     D0 |-- Buzzer
          |           |
          +-----------+
```

## **System Flowchart**

```
          ┌──────────────────────┐
          │  Start / Power ON   │
          └───────────┬─────────┘
                      │
                Check Proximity (≤30cm)?
                      │
             Yes ─────┘───── No
              │                  │
Send Alert to User1 & User2      │
Wait 15s, Check Authentication    │
              │                  │
If still no login → Send URGENT Alert
              │
   ┌──────────┴────────────┐
   │   User Opens Web UI   │
   └──────────┬────────────┘
              │
      Password Correct?
        │          │
       Yes         No
        │          │
  Grant Access   Buzzer ON (2s)
  Relay Control  + Telegram Alerts
        │          │
     Logout   After 5 failures → 1-minute Lockout
```

---

## **Web Login Workflow**

| Step | Action |
|-------|--------|
| 1 | User opens ESP IP in browser |
| 2 | Enters password |
| 3 | If correct → relay control page opens |
| 4 | If wrong → buzzer + Telegram alert |
| 5 | After logout → access blocked until login again |

---

## **Telegram Alert Logic**

| Event | Telegram Notification |
|---------|---------------------|
| 1st wrong attempt | User1 + User2 |
| 3rd attempt | User1 + User2 + Police (User3) |
| 5th attempt | System Lockout + All 3 notified |
| Person detected within 30cm | User1 + User2 alert |

---

## **How to Use**

1. Flash code to ESP8266
2. Connect to Wi-Fi (SSID & Password in code)
3. Open browser → enter ESP IP (example: `192.168.1.25`)
4. Login using password → control relay

---

## **Software Used**

- Arduino IDE
- ESP8266 Core
- Telegram Bot API
- C++ Embedded Programming

---

## Full Source Code
```cpp
#include <ESP8266WiFi.h>
#include <ESP8266WebServer.h>
#include <WiFiClientSecure.h>

const char* ssid = "OnePlus12R";
const char* password = "milan...";

const char* botToken = "7898287016:AAGGqNLcCah3EHrYElQ-nLtr8fMiUHLUFW8";
const char* user1 = "1298633019";  // Bank Security
const char* user2 = "1364482563";  // User 2
const char* user3 = "1235314210";  // Police

ESP8266WebServer server(80);
WiFiClientSecure client;

uint8_t relayPin = 12;
uint8_t buzzerPin = 16;
uint8_t trigPin = 2;
uint8_t echoPin = 4;
bool isAuthenticated = false;
int failedAttempts = 0;
unsigned long lockoutEndTime = 0;

const String correctPassword = "1234";

void setup() {
  Serial.begin(115200);
  pinMode(relayPin, OUTPUT);
  pinMode(buzzerPin, OUTPUT);
  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);
  digitalWrite(relayPin, LOW);

  Serial.println("Connecting to WiFi...");
  WiFi.begin(ssid, password);
  client.setInsecure();

  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.print(".");
  }
  Serial.println("\nWiFi connected!");
  Serial.print("IP Address: ");
  Serial.println(WiFi.localIP());

  server.on("/", handleLoginPage);
  server.on("/login", handleLogin);
  server.on("/toggleRelay", toggleRelay);
  server.on("/logout", handleLogout);
 // server.on("/checkDistance", checkDistance);

  server.begin();
  Serial.println("HTTP server started");
}

void loop() {
  server.handleClient();
  checkProximity();
}

void handleLoginPage() {
  if (millis() < lockoutEndTime) {
    server.send(200, "text/html", "<h2 style='color:red; text-align:center;'>Too many failed attempts. Try again in 10 minutes.</h2>");
  } else {
    String html = "<!DOCTYPE html><html><head><title>ESP8266 Secure Login</title>"
                  "<style>body { font-family: Arial, sans-serif; text-align: center; padding: 50px; }"
                  ".container { width: 350px; background: white; padding: 20px; border-radius: 8px; box-shadow: 0 0 10px rgba(0,0,0,0.1); margin: auto; }"
                  "h2 { color: #333; }"
                  "input[type='password'] { width: 100%; padding: 10px; margin: 10px 0; }"
                  "button { width: 100%; padding: 10px; background: #28a745; color: white; border: none; cursor: pointer; }"
                  "button:hover { background: #218838; }"
                  "</style></head><body>"
                  "<div class='container'><h2>Enter Password</h2>"
                  "<form action='/login' method='GET'>"
                  "<input type='password' name='password' placeholder='Enter Password' required>"
                  "<button type='submit'>Login</button>"
                  "</form></div></body></html>";
    server.send(200, "text/html", html);
  }
}

void handleLogin() {
  if (millis() < lockoutEndTime) {
    server.send(200, "text/html", "<h2 style='color:red; text-align:center;'>Too many failed attempts. Try again in 10 minutes.</h2>");
    return;
  }

  if (server.hasArg("password")) {
    String inputPassword = server.arg("password");

    if (inputPassword == correctPassword) {
      isAuthenticated = true;
      failedAttempts = 0;
      server.send(200, "text/html", "<h2 style='color:green; text-align:center;'>Access Granted! <a href='/toggleRelay'>Toggle Relay</a> | <a href='/logout'>Logout</a></h2>");
    } else {
      failedAttempts++;
      digitalWrite(buzzerPin, HIGH);
      delay(2000);
      digitalWrite(buzzerPin, LOW);

      if (failedAttempts == 1) {
        sendTelegramNotification(user1, "WARNING: Incorrect password attempt detected.");
        sendTelegramNotification(user2, "WARNING: Incorrect password attempt detected.");
      }
      if (failedAttempts == 3) {
        sendTelegramNotification(user1, "ALERT: Intruder alert! Police have been contacted.");
        sendTelegramNotification(user2, "ALERT: Intruder alert! Police have been contacted.");
        sendTelegramNotification(user3, "INTRUDER ALERT: Multiple failed attempts detected.");
      }
      if (failedAttempts == 5) {
        lockoutEndTime = millis() + (1 * 60 * 1000);
        sendTelegramNotification(user1, "LOCKDOWN: Too many failed attempts. Police have been informed.");
        sendTelegramNotification(user2, "LOCKDOWN: Too many failed attempts. Police have been informed.");
        sendTelegramNotification(user3, "LOCKDOWN: Multiple unauthorized login attempts detected.");
      }

      server.send(200, "text/html", "<h2 style='color:red; text-align:center;'>Incorrect Password! <a href='/'>Try Again</a></h2>");
    }
  }
}

void toggleRelay() {
  if (!isAuthenticated) {
    server.send(403, "text/html", "<h2 style='color:red; text-align:center;'>Unauthorized Access! <a href='/'>Login First</a></h2>");
    return;
  }
  digitalWrite(relayPin, !digitalRead(relayPin));
  server.send(200, "text/html", "<h2 style='color:green; text-align:center;'>Relay Toggled! <a href='/'>Go Back</a></h2>");
}

void handleLogout() {
  isAuthenticated = false;
  server.send(200, "text/html", "<h2 style='color:blue; text-align:center;'>Logged Out! <a href='/'>Login Again</a></h2>");
}

void checkProximity() {
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);

  long duration = pulseIn(echoPin, HIGH);
  int distance = duration * 0.034 / 2;

  if (distance > 0 && distance <= 30 && !isAuthenticated) {
    sendTelegramNotification(user1, "ALERT: Suspicious activity detected near the door.");
    sendTelegramNotification(user2, "ALERT: Suspicious activity detected near the door.");
    delay(15000);

    if (!isAuthenticated) {
      sendTelegramNotification(user1, "URGENT: No password entered in 15s. Possible break-in.");
      sendTelegramNotification(user2, "URGENT: No password entered in 15s. Possible break-in.");
    }
  }
}

void sendTelegramNotification(const char* user, String message) {
  String url = "https://api.telegram.org/bot" + String(botToken) + "/sendMessage?chat_id=" + String(user) + "&text=" + message;
  client.connect("api.telegram.org", 443);
  client.print(String("GET ") + url + " HTTP/1.1\r\n" + "Host: api.telegram.org\r\n" + "Connection: close\r\n\r\n");
  delay(500);
}

```

## Project Photos

| Photos | Description |
|------------|------------|
| <img src="images/group_photo.jpg" width="350"/> | Team / Group Photo |
| <img src="images/telegram_alert.png" width="350"/> | Telegram Intrusion Alert |



## Contributors

- Ballambettu Milan Shankar Bhat (4NI23EC019)
- Pranav Maruti Shanbhag (4NI24EC407)
- M B Nishanth (4NI23EC048)
- Anirudha Jayaprakash (4NI23EC014)
- Adithya Y (4NI23EC005)
- Daivik I Vinayaka (4NI23EC023)
- Karthik K Prasad (4NI23EC039)


