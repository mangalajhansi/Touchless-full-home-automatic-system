# Touchless-full-home-automatic-system


//ESP32 Embedded Firmware
#define FAN_ENA 18
#define FAN_IN1 23
#define FAN_IN2 19

#define BULB_ENB 25
#define BULB_IN3 26
#define BULB_IN4 27

void setup() {
  Serial.begin(115200);

  pinMode(FAN_IN1, OUTPUT);
  pinMode(FAN_IN2, OUTPUT);
  pinMode(BULB_IN3, OUTPUT);
  pinMode(BULB_IN4, OUTPUT);

  pinMode(FAN_ENA, OUTPUT);
  pinMode(BULB_ENB, OUTPUT);

  digitalWrite(FAN_IN1, HIGH);
  digitalWrite(FAN_IN2, LOW);

  digitalWrite(BULB_IN3, HIGH);
  digitalWrite(BULB_IN4, LOW);
}

void loop() {

  if (Serial.available()) {

    String cmd = Serial.readStringUntil('\n');
    cmd.trim();

    char device = cmd.charAt(0);
    int value = cmd.substring(1).toInt();

    value = constrain(value, 0, 255);

    if (device == 'F') {
      analogWrite(FAN_ENA, value);
      Serial.print("Fan Speed: ");
      Serial.println(value);
    }

    else if (device == 'B') {
      analogWrite(BULB_ENB, value);
      Serial.print("Bulb Brightness: ");
      Serial.println(value);
    }
  }
}



//Python Gesture Recognition Application
import cv2
import serial
import time
import mediapipe as mp
import math
import requests

# ==========================
# ESP32 Serial Connection
# ==========================
esp32 = serial.Serial('COM12', 115200)
time.sleep(2)

# ==========================
# ThingSpeak Configuration
# ==========================
WRITE_API_KEY = "GAXKITFKCSC8P92E"
last_update = time.time()

# ==========================
# MediaPipe Hand Tracking
# ==========================
mp_hands = mp.solutions.hands

hands = mp_hands.Hands(
    static_image_mode=False,
    max_num_hands=2,
    min_detection_confidence=0.7,
    min_tracking_confidence=0.7
)

mp_draw = mp.solutions.drawing_utils

# ==========================
# Webcam
# ==========================
cap = cv2.VideoCapture(0)

fan_value = 0
bulb_value = 0

while True:

    success, img = cap.read()

    if not success:
        continue

    img = cv2.flip(img, 1)

    rgb_img = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)
    results = hands.process(rgb_img)

    if results.multi_hand_landmarks and results.multi_handedness:

        for handLms, handType in zip(
                results.multi_hand_landmarks,
                results.multi_handedness):

            label = handType.classification[0].label

            mp_draw.draw_landmarks(
                img,
                handLms,
                mp_hands.HAND_CONNECTIONS
            )

            h, w, c = img.shape
            lm_list = []

            for lm in handLms.landmark:
                cx = int(lm.x * w)
                cy = int(lm.y * h)
                lm_list.append((cx, cy))

            if len(lm_list) > 8:

                # Thumb tip
                x1, y1 = lm_list[4]

                # Index finger tip
                x2, y2 = lm_list[8]

                cv2.circle(img, (x1, y1), 10, (0, 255, 0), -1)
                cv2.circle(img, (x2, y2), 10, (0, 255, 0), -1)
                cv2.line(img, (x1, y1), (x2, y2), (255, 0, 255), 3)

                distance = math.hypot(x2 - x1, y2 - y1)

                value = int((distance / 200) * 255)

                if value > 255:
                    value = 255

                if value < 0:
                    value = 0

                # RIGHT HAND -> FAN
                if label == "Right":

                    fan_value = value

                    esp32.write(f"F{value}\n".encode())

                # LEFT HAND -> BULB
                elif label == "Left":

                    bulb_value = value

                    esp32.write(f"B{value}\n".encode())

    # ==========================
    # ThingSpeak Update Every 10 Seconds
    # ==========================
    if time.time() - last_update >= 10:

        try:

            response = requests.post(
                "https://api.thingspeak.com/update",
                data={
                    "api_key": WRITE_API_KEY,
                    "field1": fan_value,
                    "field2": bulb_value
                }
            )

            print("ThingSpeak Updated")
            print("Fan Speed:", fan_value)
            print("Bulb Brightness:", bulb_value)

        except Exception as e:

            print("ThingSpeak Error:", e)

        last_update = time.time()

    # ==========================
    # Display Values
    # ==========================
    cv2.putText(
        img,
        f"RIGHT HAND FAN SPEED : {fan_value}",
        (20, 50),
        cv2.FONT_HERSHEY_SIMPLEX,
        0.8,
        (0, 0, 255),
        2
    )

    cv2.putText(
        img,
        f"LEFT HAND BULB BRIGHTNESS : {bulb_value}",
        (20, 100),
        cv2.FONT_HERSHEY_SIMPLEX,
        0.8,
        (255, 0, 0),
        2
    )

    cv2.imshow("ESP32 Fan & Bulb Control + ThingSpeak", img)

    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

cap.release()
cv2.destroyAllWindows()
esp32.close()
