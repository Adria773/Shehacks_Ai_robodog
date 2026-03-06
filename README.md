# Shehacks_Ai_robodog
#include <Arduino.h>
#include <Wire.h>
#include <Adafruit_PWMServoDriver.h>
#include <WiFi.h>
#include <WebServer.h>


const char* ssid = "YOUR_SSID";
const char* password = "YOUR_PASSWORD";


WebServer server(80);


Adafruit_PWMServoDriver pwm = Adafruit_PWMServoDriver();
#define SERVO_FREQ 50 


int servos[4] = {0,1,2,3};  

#define TRIG_PIN 5
#define ECHO_PIN 18


QueueHandle_t commandQueue;


int angleToPulse(int angle) {
  return map(angle, 0, 180, 150, 600);
}
void setServoAngle(int channel, int angle) {
  pwm.setPWM(channel, 0, angleToPulse(angle));
}


void sendCommand(const char* cmd) {
  char buffer[20];
  strncpy(buffer, cmd, sizeof(buffer));
  xQueueSend(commandQueue, &buffer, portMAX_DELAY);
}


void locomotionTask(void* pv) {
  char command[20];
  
  for (int i = 0; i < 4; i++) setServoAngle(servos[i], 90);

  for (;;) {
    if (xQueueReceive(commandQueue, &command, portMAX_DELAY)) {
      if (strcmp(command, "WALK") == 0) {
        
        for (int cycle = 0; cycle < 4; cycle++) {
          setServoAngle(servos[0], 70); setServoAngle(servos[2], 110);
          vTaskDelay(pdMS_TO_TICKS(300));
          setServoAngle(servos[1], 70); setServoAngle(servos[3], 110);
          vTaskDelay(pdMS_TO_TICKS(300));
        }
      } else if (strcmp(command, "STOP") == 0) {
        for (int i = 0; i < 4; i++) setServoAngle(servos[i], 90);
      } else if (strcmp(command, "SIT") == 0) {
        for (int i = 0; i < 4; i++) setServoAngle(servos[i], 120);
      }
      Serial.printf("[Locomotion] Executed: %s\n", command);
    }
  }
}


void ultrasonicTask(void* pv) {
  for (;;) {
    
    digitalWrite(TRIG_PIN, LOW);
    delayMicroseconds(2);
    digitalWrite(TRIG_PIN, HIGH);
    delayMicroseconds(10);
    digitalWrite(TRIG_PIN, LOW);

  
    long duration = pulseIn(ECHO_PIN, HIGH, 30000); 
    float distance = duration * 0.034 / 2; // cm

    if (distance > 0 && distance < 20) {
      
      sendCommand("STOP");
      Serial.println("[Ultrasonic] Obstacle detected → STOP");
    }

    vTaskDelay(pdMS_TO_TICKS(500)); // check twice per second
  }
}

void handleRoot() {
  String html = "<html><body><h3>Robo Dog Control</h3>"
                "<button onclick=\"fetch('/cmd/WALK')\">WALK</button>"
                "<button onclick=\"fetch('/cmd/STOP')\">STOP</button>"
                "<button onclick=\"fetch('/cmd/SIT')\">SIT</button>"
                "</body></html>";
  server.send(200, "text/html", html);
}
void handleCmd() {
  String cmd = server.pathArg(0);
  sendCommand(cmd.c_str());
  server.send(200, "text/plain", "OK: " + cmd);
}


void wifiTask(void* pv) {
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  Serial.print("Connecting to WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    Serial.print(".");
    vTaskDelay(pdMS_TO_TICKS(500));
  }
  Serial.printf("\nConnected: %s\nIP: %s\n", ssid, WiFi.localIP().toString().c_str());

  server.on("/", handleRoot);
  server.on("/cmd/{cmd}", HTTP_GET, handleCmd);
  server.begin();
  Serial.println("Web server started");

  for (;;) {
    server.handleClient();
    vTaskDelay(pdMS_TO_TICKS(10));
  }
}


void setup() {
  Serial.begin(115200);
  Wire.begin(); // SDA=21, SCL=22
  pwm.begin();
  pwm.setPWMFreq(SERVO_FREQ);

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);

  commandQueue = xQueueCreate(10, sizeof(char[20]));

  xTaskCreatePinnedToCore(locomotionTask, "Locomotion", 4096, NULL, 2, NULL, 1);
  xTaskCreatePinnedToCore(wifiTask, "WiFi", 6144, NULL, 1, NULL, 0);
  xTaskCreatePinnedToCore(ultrasonicTask, "Ultrasonic", 4096, NULL, 1, NULL, 0);

  
  for (int i = 0; i < 4; i++) setServoAngle(servos[i], 90);
}

void loop() {
  
}
