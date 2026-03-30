# Smart-Letter-Box-ARDVINO-HALF
#include <SoftwareSerial.h>
#include <Servo.h>

SoftwareSerial espSerial(2, 3); // for connection with ESP 32-CAM

Servo lockServo; //Servo Motor
const int SERVO_PIN = 6;

const int TRIG_PIN = 9; // Echo sensor
const int ECHO_PIN = 10;

const int LETTER_DISTANCE_CM = 8; // Distance threshold in cm

bool onCooldown = false;
unsigned long lastTriggerTime = 0; // for Cooldown timer -> Prevents multiple triggers from the same letter
const int COOLDOWN_MS = 3000; // 10 seconds between detections


long measureDistance() // For working of Echo sensor
{
  digitalWrite(TRIG_PIN, LOW); // Ensure that sensor sends no pulse
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH); // Send a pulse
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW); // Ensure that sensor sends no pulse so it can listen to the wave that comes back

  long duration = pulseIn(ECHO_PIN, HIGH, 30000); // Listen for echo (30ms timeout)-Receiving the pulse 
  
  if (duration == 0) 
  return 999; // No echo -> giving  back large dummy value
  return (duration * 0.034) / 2; // Convert time to cm
}


void setup() 
{
  Serial.begin(9600);
  espSerial.begin(9600);

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);

  lockServo.attach(SERVO_PIN); // Servo library native function  lockServo.write(0); // 0 degrees = LOCKED position
  Serial.println("=== Smart Letter Box Ready ==="); // For our referance on display screen
}


void loop() {  // We check forst if we are ready to detect the next letter 
  
  
  if (onCooldown && (millis() - lastTriggerTime > COOLDOWN_MS)) // onCooldown is to check if true is recorded then dont have multiple true for the same letter and increase letter count  
  // millis() is stopwatch and measures time in ms 
  {
    
    onCooldown = false;
    Serial.println("Ready to detect again.");
  }

  long dist = measureDistance(); // Disstance from echo sensor returned in cm 

  if (dist > 0 && dist < LETTER_DISTANCE_CM && !onCooldown) // Detecting Letter
  {
    
    Serial.print("Letter detected! Distance: ");
    Serial.print(dist);
    Serial.println(" cm");
    
    espSerial.println("LETTER");  
    // Tell the ESP32-CAM a letter has arrived -> THIS IS NOT MY REGULAR FUNCTION -> sends data to esp board

    onCooldown = true; // Start cooldown so we don't trigger again immediately
    lastTriggerTime = millis();
  }


 // NEXT SEGMENT BEGINS -> SERVO MOTOR UNLOCK MECHANISM :)
  
  
  if (espSerial.available()) // RECEIVING COMMAND FROM ESP TO OPEN OR CLOSE THE DOOR
  {
    
    String command = espSerial.readStringUntil('\n');
    command.trim(); // Remove any extra spaces or newline characters

      if (command == "UNLOCK") // UNLOCK command from Blynk app button
      {
        Serial.println("Unlock command received from app!");
        lockServo.write(90);    // Rotate to 90 degrees = UNLOCKED
        Serial.println("Servo at 90 degrees — UNLOCKED");
  
        delay(5000);            // Stay unlocked for 5 seconds
  
        lockServo.write(0);     // Return to 0 degrees = LOCKED
        Serial.println("Servo at 0 degrees — LOCKED");

        // Tell ESP32-CAM the box has re-locked
        espSerial.println("LOCKED");
      }
  }

  delay(100); // Small pause to avoid flooding the sensor
}
