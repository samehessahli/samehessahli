#include "Arduino.h"
#include "PubSubClient.h"
#include "WiFi.h"
#include "esp_wpa2.h"
#include <HX711_ADC.h>
#include "EEPROM.h"
#include "HX711.h"

On déclare les librairies utilent à notre programme. 

HX711 scale;

// Définir les broches de connexion au HX711
const int LOADCELL_DOUT_PIN = 4;
const int LOADCELL_SCK_PIN = 5;

// Paramètres MQTT Broker (A COMPLETER)
//const char *mqtt_broker = "broker.hivemq.com"; 
const char *mqtt_broker = "139.124.208.196"; // Identifiant du broker (Adresse IP)


Sur la ligne précédente on entre l'adresse IP de la page Node-Red


const char *topic = "FORCE"
; // Nom du topic sur lequel les données seront envoyés. 
const char *mqtt_username = ""; // Identifiant dans le cas d'une liaison sécurisée
const char *mqtt_password = ""; // Mdp dans le cas d'une liaison sécurisée
const int mqtt_port = 1883; // Port : 1883 dans le cas d'une liaison non sécurisée et 8883 dans le cas d'une liaison cryptée
WiFiClient espClient; 
PubSubClient client(espClient); 


// Paramètres EDUROAM (A COMPLETER)
#define EAP_IDENTITY "ihab.bouri.1@etu.univ-amu.fr"
#define EAP_PASSWORD "*Anonymous007" 
#define EAP_USERNAME "ihab.bouri.1@etu.univ-amu.fr" 
const char* ssid = "eduroam"; // eduroam SSID

On connecte notre capteur au réseau Wi-Fi Eduroam.


// Fonction réception du message MQTT 
void callback(char *topic, byte *payload, unsigned int length) { 
  Serial.print("Le message a été envoyé sur le topic : "); 
  Serial.println(topic); 
  Serial.print("Message:"); 
  for (int i = 0; i < length; i++) { 
    Serial.print((char) payload[i]); 
  } 
  Serial.println(); 
  Serial.println("-----------------------"); 
}
// Initialisation du programme
void setup() { 
 Serial.begin(38400);
 
On établit la communication série à une vitesse précise. 
 
// Initialisation du capteur 
  scale.begin(LOADCELL_DOUT_PIN, LOADCELL_SCK_PIN);
  Serial.println("Avant la calibration :");
  Serial.print("valeur: \t\t");
  Serial.println(scale.read());     // print a raw reading from the ADC
  Serial.print("read average: \t\t");
  Serial.println(scale.read_average(20));   // print the average of 20 readings from the ADC
  Serial.print("get value: \t\t");
  Serial.println(scale.get_value(5));   // print the average of 5 readings from the ADC minus the tare weight (not set yet)
  Serial.print("get units: \t\t");
  Serial.println(scale.get_units(5), 1);  // print the average of 5 readings from the ADC minus tare weight (not set) divided
  Les lignes serial.print permettent l'affichage des valeurs brutes, donc les valeurs avant la calibration. 
// FACTEUR DE CALIBRATION A MODIFIER AU BESOIN
  scale.set_scale(-1200.f);
  
Facteur de calibration
  
  scale.tare();               // reset the scale to 0
On reinitialise la balance à 0.
  Serial.println("After setting up the scale:");
  Serial.print("read: \t\t");
  Serial.println(scale.read());                 // print a raw reading from the ADC
  Serial.print("read average: \t\t");
  Serial.println(scale.read_average(20));       // print the average of 20 readings from the ADC
  Serial.print("get value: \t\t");
  Serial.println(scale.get_value(5));   // print the average of 5 readings from the ADC minus the tare weight, set with tare()
  Serial.print("get units: \t\t");
  Serial.println(scale.get_units(5), 1);        // print the average of 5 readings from the 
  
On affiche les valeurs après calibration. 
  
  ADC minus tare weight, divided
            // by the SCALE parameter set with set_scale
  Serial.println("Readings:");
 
// Connexion au réseau EDUROAM 
  WiFi.disconnect(true);
  WiFi.begin(ssid, WPA2_AUTH_PEAP, EAP_IDENTITY, EAP_USERNAME, EAP_PASSWORD); 
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(F("."));
  }
  Serial.println("");
  Serial.println(F("L'ESP32 est connecté au WiFi !"));
  
// Connexion au broker MQTT  
  
  client.setServer(mqtt_broker, mqtt_port); 
  client.setCallback(callback); 
  while (!client.connected()) { 
    String client_id = "esp32-client-"; 
    client_id += String(WiFi.macAddress()); 
    Serial.printf("La chaîne de mesure %s se connecte au broker MQTT", client_id.c_str()); 
 
    if (client.connect(client_id.c_str(), mqtt_username, mqtt_password)) { 
      Serial.println("La chaîne de mesure est connectée au broker."); 
    } else { 
      Serial.print("La chaîne de mesure n'a pas réussi à se connecter ... "); 
      Serial.print(client.state()); 
      delay(2000); 
    } 
  }
  }
 
void loop(){
Serial.print("one reading:\t");
  Serial.print(scale.get_units(), 1);
Affichage du poids. 
  Serial.print("\t| average:\t");
  Serial.println(scale.get_units(10), 1);
  
Affichage de la moyenne sur 10 mesures. 
  
  float weight = scale.get_units(10);
  client.publish(topic, String(weight).c_str()); // Publication de la masse sur le topic
  
On publie la valeur moyenne du poids sur le topic MQTT. 
  
  client.subscribe(topic); // S'abonne au topuc pour recevoir des messages
  
On souscrit au topic MQTT afin de recevoir les messages sur celui-ci. 
  
  client.loop(); // Gère les messages MQTT (pour lire la valeur de la masse sur le moniteur série de plateformIO)
 
 // Eteint et rallume le capteur
  scale.power_down();             
  delay(500);
  scale.power_up();
}
