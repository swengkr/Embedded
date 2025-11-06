Arduino Uno 와 HC-SR04 초음파 센서를 기반으로 레이더 데모를 제작했습니다.

- 컴퓨팅 모듈 : Arduino Uno
- 초음파 센서 : HC-SR04
- 180도 회전 : SG-90 Servo
- 수동 제어 : Potentiometer

### Arduino Uno 소스 코드
```cpp
// Arduino Uno 표준 Servo 라이브러리 사용
#include <Servo.h>

// 핀 정의 (Arduino Uno 기준)
#define TRIG_PIN 8      // 초음파 센서 TRIG 핀
#define ECHO_PIN 9      // 초음파 센서 ECHO 핀
#define SERVO_PIN 10    // 서보 모터 제어 핀 (PWM 핀 사용)
#define POT_PIN A0      // 가변 저항(VR) 입력 핀 (아날로그 핀 사용)
#define CONTROL_PIN 11  // 수동/자동 모드 전환을 위한 디지털 입력 핀

#define MAX_DISTANCE 100 // 최대 측정 거리 (cm)

const int minValue = 0;   
const int maxValue = 1024; 

Servo servo;

long duration;
int distance;

void setup() {
  Serial.begin(9600);
  servo.attach(SERVO_PIN);
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  pinMode(POT_PIN, INPUT); 
  
  pinMode(CONTROL_PIN, INPUT_PULLUP);
}

void loop() {
  // 11번 핀 상태 읽음 (LOW: 수동 모드, HIGH: 자동 스캔 모드)
  if (digitalRead(CONTROL_PIN) == LOW) {
    
    int potValue = analogRead(POT_PIN);
    // ADC 값을 서보 각도(0~180)로 변환
    int angle = map(potValue, minValue, maxValue, 180, 0); 
    
    servo.write(angle);
    delay(20);
    
    // 필터링된 거리 측정값 사용
    distance = getDistanceRaw();
    
    if (distance != -1) {
      Serial.print(angle);
      Serial.print(",");
      Serial.println(distance);
    }

    delay(10);

  } else {
    
    // 0° → 180° 스캔
    for (int angle = 0; angle <= 180 && digitalRead(CONTROL_PIN) == HIGH; ++angle) {
      servo.write(angle);
      delay(20);
      distance = getDistanceRaw();
      if (distance != -1) {
        Serial.print(angle);
        Serial.print(",");
        Serial.println(distance);
      }
    }

    // 180° → 0° 스캔
    for (int angle = 180; angle >= 0 && digitalRead(CONTROL_PIN) == HIGH; --angle) {
      servo.write(angle);
      delay(20);
      distance = getDistanceRaw();
      if (distance != -1) {
        Serial.print(angle);
        Serial.print(",");
        Serial.println(distance);
      }
    }
    
    // 잠시 대기
    delay(100);
  }
}

int getDistanceRaw() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  // pulseIn 함수의 timeout을 20ms로 설정 (20000 마이크로초)
  duration = pulseIn(ECHO_PIN, HIGH, 20000); 
  
  // echo 실패 시(timeout 발생) 0 반환
  if (duration == 0) return -1; 

  // 거리를 cm 단위로 변환 (음속: 340m/s = 0.034cm/μs, 즉 58μs/cm)
  int dist = duration / 58; 
  
  // 최대 거리를 초과하면 -1 반환
  if (dist > MAX_DISTANCE) return -1; 
  
  return dist;
}
```

### GUI 소스 코드 (Processing)
```java
import processing.serial.*;

Serial port;
String data = "";
int angle = 0;
int distance = 0;

float scale = 5.0;
int maxDist = 100;

ArrayList<PVector> points = new ArrayList<PVector>();

void setup() {
  size(1000, 1000);
  port = new Serial(this, "COM9", 9600);
  port.bufferUntil('\n');
  background(0);
  frameRate(60);
}

void draw() {
  fill(0, 30);
  noStroke();
  rect(0, 0, width, height);

  translate(width/2, height/2);
  
  stroke(0, 150, 0);
  noFill();
  for (int r = 20; r <= maxDist; r += 20) {
    ellipse(0, 0, r * scale * 2, r * scale * 2);
  
    fill(255); 
    textAlign(CENTER, CENTER);
    textSize(14);
    text(r + " cm", -r * scale + 25, 10);
    noFill();
  }

  for (int a = 0; a < 360; a += 30) {
    float x = cos(radians(a - 90)) * maxDist * scale;
    float y = sin(radians(a - 90)) * maxDist * scale;
    line(0, 0, x, y);
  }

  pushMatrix();
  scale(1, -1);

  noFill();
  stroke(0, 100, 0, 80);
  arc(0, 0, maxDist * scale * 2, maxDist * scale * 2, radians(-90), radians(70));

  float drawAngle = map(angle, 0, 180, 90, 270);

  float scanX = distance * scale * cos(radians(drawAngle - 90));
  float scanY = distance * scale * sin(radians(drawAngle - 90));

  stroke(0, 255, 0);
  line(0, 0, scanX, scanY);

  points.add(new PVector(scanX, scanY, millis()/1000.0));

  float now = millis()/1000.0;
  for (int i = points.size()-1; i >= 0; i--) {
    if (now - points.get(i).z > 10) {
      points.remove(i);
    }
  }

  noStroke();
  for (PVector p : points) {
    float age = now - p.z;
    float alpha = map(age, 0, 10, 255, 0);
    fill(255, 50 + int(205 * (1 - age / 10)), 0, alpha);
    ellipse(p.x, p.y, 6, 6);
  }
  popMatrix();

  fill(0, 255, 0);
  textSize(16);
  textAlign(LEFT);
  text("Angle: " + angle + "°", -width/2 + 20, -height/2 + 30);
  text("Distance: " + distance + " cm", -width/2 + 20, -height/2 + 55);
  text("Scan Range: 0° ~ 180°", -width/2 + 20, -height/2 + 90);
}

void serialEvent(Serial p) {
  try {
    data = p.readStringUntil('\n');
    if (data != null) {
      data = trim(data);
      String[] parts = split(data, ',');
      if (parts.length == 2) {
        angle = int(parts[0]);
        distance = int(parts[1]);
        distance = constrain(distance, 0, maxDist);
      }
    }
  } catch (Exception e) {
    println("Serial Error: " + e.getMessage());
  }
}
```
