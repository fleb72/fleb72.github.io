---
title: "Apprendre à communiquer en Bluetooth LE avec une carte Arduino Uno R4 WiFi"
date: 2025-12-19
categories: ["Arduino"]
tags: ["ArduinoUnoR4WiFi", "Bluetooth"]
math: true
target_blank: true
---

### Bluetooth LE et Arduino Uno R4 WiFi

Le Bluetooth basse consommation ou *Bluetooth Low Energy* est une technique de transmission sans fil basée sur le standard ouvert Bluetooth. Comparé au Bluetooth, le BLE permet un débit du même ordre de grandeur (1 Mbit/s) pour une consommation d'énergie 10 fois moindre. BLE est intégré aux normes Bluetooth par le [Bluetooth SIG](https://www.bluetooth.com/){:target="_blank"} (Le Bluetooth *Special Interest Group*)

L'Arduino Uno R4 WiFi propose du Bluetooth *Low Energy* (BLE) via son module ESP32-S3/NORA-W106. Le Bluetooth « classique » avec un port série émulé comme sur les modules HC-05/HC-06 n'est plus possible, l'Arduino Uno R4 WiFi ne supporte que le **BLE**.

Vous devrez installer la bibliothèque [ArduinoBLE](https://docs.arduino.cc/libraries/arduinoble/){:target="_blank"} en passant par le gestionnaire de bibliothèques de l'IDE Arduino 2.*x*. Vérifiez également que le firmware de la carte est à jour (menu *Tools/Firmware Updater*).

![Arduino Uno R4 WiFi et BLE](/assets/img/posts/2025-12-19-uno-r4-wifi-bluetooth-ble-debuter/arduino-uno-r4-wifi-ble.jpg)
*Illustration générée par IA*

### Le protocole de communication BLE

Pour être à l'aise avec un *sketch* exploitant le BLE, il faut comprendre les principes de base d'un protocole beaucoup plus rigoureux et organisé que celui du Bluetooth classique. Le modèle de données utilisé par le Bluetooth LE pour organiser, structurer et échanger des informations entre appareils est défini par le GATT (*Generic Attribute Profile*) que l'on retrouve dans les [spécifications Bluetooth](https://www.bluetooth.com/specifications/specs/){:target="_blank"}.

#### *Central* et *Peripheral*

##### Rôle du *Peripheral* (ou périphérique)

- Un périphérique (*peripheral*) annonce sa présence et sa disponibilité en envoyant des signaux (*advertising*).
- Il agit comme un serveur : il expose des services et des caractéristiques que l'appareil *Central* peut lire ou écrire.
- Il est optimisé pour une faible consommation d'énergie : il passe la plupart du temps en veille et ne se réveille que pour transmettre ou recevoir des données.

##### Rôle du *Central* (ou appareil central)

- Un appareil central (*Central*) recherche les périphériques en scannant les signaux environnants.
- Il initie la connexion, et c’est lui qui décide de se connecter à un périphérique.
- Il est en général plus puissant, car il peut gérer plusieurs connexions simultanées, agréger et traiter les données.
- Il agit comme un client : il interroge les services exposés par le périphérique.

En bref, un périphérique peut être un capteur qui dit « voici mes données ». L'appareil central peut être un smartphone qui vient lire ces données et les afficher ou les traiter.

![BLE model](/assets/img/posts/2025-12-19-uno-r4-wifi-bluetooth-ble-debuter/ble-bulletin-board-model.png)
*d'après https://docs.arduino.cc/libraries/arduinoble/*

#### Services et caractéristiques

En *Bluetooth Low Energy* (BLE), on parle de services (*services*) et de caractéristiques (*characteristics*) pour décrire la manière dont les données sont organisées et échangées entre appareils.

Un service est un regroupement logique de fonctionnalités. Chaque service est identifié par un UUID (*Universally Unique Identifier*).
Par exemple, on peut regrouper dans un service des données environnementales mesurées par des capteurs avec des caractéristiques (*characteristics*) comme la température, l'humidité et la pression atmosphérique. Chaque caractéristique est aussi identifiée par un UUID.

Chaque caractéristique peut avoir une ou plusieurs propriétés : 
- `Read` : l'appareil central peut lire la valeur d'une caractéristique exposée par le périphérique.
- `Write` : l'appareil central peut écrire une valeur d'une caractéristique exposée par le périphérique.
- `Notify` : le périphérique peut notifier l'appareil central d'un changement qu'il a généré (par exemple si la température change, le périphérique notifie l'appareil central qui peut alors recevoir la nouvelle valeur de température).
- `Indicate` : similaire à `Notify`, mais avec accusé de réception.

Par exemple, la carte Arduino Uno R4 WiFi reliée à des capteurs environnementaux peut proposer un service « données environnementales » avec des caractéristiques comme la température, l’humidité et la pression atmosphérique. La carte Arduino agit alors comme un périphérique qui expose ces valeurs. Si elle met à jour ses mesures toutes les 10 minutes, elle peut notifier automatiquement le smartphone — en tant qu’appareil Central — qui recevra les nouvelles valeurs dès qu’elles changent. 


### Exemple : allumer ou éteindre une LED en Bluetooth LE

En s'inspirant fortement des [exemples de la bibliothèque ArduinoBLE](https://github.com/arduino-libraries/ArduinoBLE/tree/master/examples){:target="_blank"}, on se propose de configurer la carte Arduino en périphérique BLE, capable d'allumer ou éteindre la LED intégrée à distance depuis un smartphone, avec des commandes textuelles `on` ou `off`.

Pour cela, on commence par créer un service dédié, nommé `ledService` :
```cpp
#include <ArduinoBLE.h>

BLEService ledService("fb32d672-9653-466b-82b3-304a3ba84b7d");
```
Pour des services standards, les UUID codés sur 16 bits seulement sont répertoriés dans les spécifications Bluetooth (voir [Bluetooth - Assigned numbers](https://www.bluetooth.com/specifications/assigned-numbers/){:target="_blank"}). Par exemple, pour un service de température, on prendra l'UUID `0x1809`. Cette liste officielle d'UUID pour les services et caractéristiques courants facilite l'interopérabilité.

Pour notre LED, l'identifiant UUID pour ce service particulier est une séquence hexadécimale pour un code 128 bits généré par un utilitaire de type [UUID Generator](https://www.uuidgenerator.net/){:target="_blank"} conforme à la [RFC4122](https://www.rfc-editor.org/rfc/rfc4122){:target="_blank"}.

On crée selon le même principe une caractéristique à ce service :
```cpp
BLEStringCharacteristic ledCharacteristic("e665e13d-7da4-47ad-b84c-6ea124349881", BLEWrite, 5); // caractéristique de type String, taille=5 maxi
```
La classe [BLEStringCharacteristic](https://github.com/arduino-libraries/ArduinoBLE/blob/master/src/BLEStringCharacteristic.cpp){:target="_blank"} permet de transporter une caractéristique de type `String`. (Voir aussi [src/BLETypedCharacteristics.cpp](https://github.com/arduino-libraries/ArduinoBLE/blob/master/src/BLETypedCharacteristics.cpp){:target="_blank"}). La caractéristique possède la propriété `BLEWrite` pour que l'appareil central (le smartphone) soit autorisé à écrire une valeur dans celle-ci.

Dans la partie `setup()`, on prépare l'*advertising*, la phase de diffusion publique où le périphérique envoie régulièrement de petits paquets radio pour annoncer sa présence et mettre en avant son service principal :

```cpp
void setup() {
  Serial.begin(9600);
  while (!Serial);

  pinMode(LED_BUILTIN, OUTPUT);

  if (!BLE.begin()) {
    Serial.println("Erreur initialisation BLE !");
    while (1);
  }

  // Nom visible du périphérique
  BLE.setLocalName("UNO_R4_LED");

  // Service principal qui sera annoncé lors de l'advertising
  // Si d'autres services sont proposés par le périphérique,
  // ils seront découverts par l'appareil central à la connexion.
  BLE.setAdvertisedService(ledService);

  // Ajout de la caractéristique au service
  ledService.addCharacteristic(ledCharacteristic);

  // Enregistrement du service
  BLE.addService(ledService);

  // Lancement de l'advertising
  BLE.advertise();
  Serial.println("Advertising BLE actif. Connectez-vous depuis le smartphone.");
}
```

Dans la boucle principale `loop()`, on attend la connexion d'un appareil central (smartphone). Si l'appareil central connecté écrit une valeur `on` ou `off` dans la caractéristique du service, la LED est aussitôt allumée ou éteinte :

```cpp
void loop() {
  // Attente connexion
  BLEDevice central = BLE.central();

  if (central) { // Appareil central connecté
    Serial.print("Connecté à : ");
    Serial.println(central.address());

    while (central.connected()) {
      if (ledCharacteristic.written()) {
        String cmd = ledCharacteristic.value();
        Serial.print("Commande reçue : ");
        Serial.println(cmd);

        if (cmd == "on") {
          digitalWrite(LED_BUILTIN, HIGH);
        } else if (cmd == "off") {
          digitalWrite(LED_BUILTIN, LOW);
        }
      }
    }

    Serial.println("Déconnecté");
  }
}
```


### Test de connexion Bluetooth LE

Comme appareil central, j'utilise mon smartphone Android avec une application générique comme *nRF Connect for Mobile* par *Nordic Semiconductor ASA*.

![scan BLE](/assets/img/posts/2025-12-19-uno-r4-wifi-bluetooth-ble-debuter/scan-ble.jpg)
*Le scan a détecté le périphérique nommé UNO_R4_LED*

![Service BLE](/assets/img/posts/2025-12-19-uno-r4-wifi-bluetooth-ble-debuter/service-ble.jpg)
*Une fois l'appareil central connecté, les services et caractéristiques sont affichés. Les services 'Generic' sont obligatoires et automatiquement ajoutés par la bibliothèque ArduinoBLE.*

![Send BLE](/assets/img/posts/2025-12-19-uno-r4-wifi-bluetooth-ble-debuter/send-ble.jpg)
*On écrit la valeur "on" dans la caractéristique du service. La LED en surface de la carte devrait s'allumer aussitôt après l'appui sur le bouton Send.*

![on sent BLE](/assets/img/posts/2025-12-19-uno-r4-wifi-bluetooth-ble-debuter/on-sent-ble.jpg)
*La valeur "on" (codes ASCII `0x6F` et `0x6E`) a bien été écrite.*

### Conclusion

Si vous avez déjà travaillé avec les modules Bluetooth *classiques* comme les HC‑05 ou HC‑06, le *Bluetooth Low Energy* fonctionne de manière assez différente. Les HC‑05/06 utilisent le Bluetooth SPP (*Serial Port Profile*), qui crée une liaison série sans fil : on envoie et on reçoit des octets comme sur un port UART, sans structure particulière. Le BLE, au contraire, repose sur une organisation en services et caractéristiques, avec des permissions de lecture, d’écriture ou de notification. Il ne fournit pas de port série virtuel, mais une communication plus structurée, plus économe en énergie et mieux adaptée aux objets connectés modernes.

Après avoir assimilé les notions essentielles du protocole, vous pourrez développer des projets plus complets : transmettre des mesures de capteurs, configurer un appareil à distance et concevoir de petits objets connectés autonomes.

Pour plus de détails, référez-vous à la documentation de la bibliothèque [ArduinoBLE](https://docs.arduino.cc/libraries/arduinoble/){:target="_blank"}.