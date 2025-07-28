---
title: "LoRaWAN pour les makers – Mon premier nœud avec Heltec ESP32 + DS18B20 vers The Things Network"
date: 2025-07-25
categories: ["Internet des Objets"]
tags: [IoT, LoRa, LoRaWAN, ESP32]
math: true
target_blank: true
---

LoRaWAN (*Long Range Wide Area Network*) est un protocole de communication sans fil conçu pour les objets connectés à bas débit. Contrairement au Wi-Fi ou à la 4G/5G, LoRaWAN permet de transmettre des données sur plusieurs kilomètres avec une consommation énergétique ultra faible. Ce réseau est idéal pour les capteurs autonomes en extérieur, les projets DIY (*Do It Yourself*) répartis dans des environnements ruraux ou urbains, ou encore les systèmes distribués sans infrastructure internet locale.

![Archtecture LoRaWAN](/assets/img/posts/2025-07-25-lorawan-heltec-esp32/architecture-lorawan.jpg)
*Architecture LoRaWAN - Image sous licence CC BY-SA 4.0*

Grâce à des plateformes collaboratives comme [The Things Network](https://www.thethingsnetwork.org/){:target="_blank"}, n’importe quel *maker* peut créer son propre nœud LoRaWAN et intégrer ses données dans le Cloud, sans frais ni abonnement grâce aux passerelles LoRaWAN publiques réparties sur tout le territoire.

> Avant de poursuivre, commencez par vous assurer sur [TTN Mapper](https://ttnmapper.org/heatmap/){:target="_blank"} que vous êtes à proximité d'une passerelle LoRaWAN publique. La couverture n'est pas complète au niveau national.
{: .prompt-warning }

### Nœud LoRaWAN avec la carte Heltec WiFi LoRa 32 v3 et une sonde de température DS18B20

La [Heltec WiFi LoRa 32 v3](https://docs.heltec.org/en/node/esp32/wifi_lora_32/index.html){:target="_blank"} est une carte de développement compacte intégrant un microcontrôleur ESP32, une radio LoRa SX1262 et un module Wi-Fi/Bluetooth. Elle est pensée pour les projets IoT longue portée, compatibles LoRaWAN, et idéale pour les *makers* habitués à l'environnement Arduino et du prototypage rapide. Pour profiter pleinement des performances LoRa, il est recommandé d'utiliser une antenne externe raccordée via le connecteur IPEX présent sur la carte. De plus, son écran OLED 0,96" intégré permet d’afficher localement des données clés (température, état du réseau, messages LoRa, etc.) sans avoir recours à un périphérique supplémentaire.

Pour cette démonstration, j'ai relié la carte Heltec avec une sonde de température DS18B20, la version encapsulée dans un petit cylindre étanche en acier inoxydable et un câble de longueur 1m :

![Montage prototype Heltec](/assets/img/posts/2025-07-25-lorawan-heltec-esp32/montage-heltec-lora.jpg)
*Prototype sur breadboard avec carte Heltec, sonde DS18B20 et alimentation*

La sonde est alimentée directement via les broches VCC (+5V) et GND. Le fil de données DATA est connecté à la broche GPIO47 (*pin* 13), et tiré à l'alimentation VCC via une résistance 4,7kΩ :

### La plateforme The Things Network : création de l'application

Avant de passer à la programmation, rejoignez la console de la plateforme LoRaWAN [The Things Network](https://console.cloud.thethings.network/){:target="_blank"} afin de créer votre application et d'y associer votre carte Heltec comme nœud LoRaWAN (appareil ou *end device*).


![Création de l'application](/assets/img/posts/2025-07-25-lorawan-heltec-esp32/ttn-add-application1.jpg)

Renseignez les champs généraux de l'application comme ci-dessous et enregistrez votre application :

![Validation création de l'application](/assets/img/posts/2025-07-25-lorawan-heltec-esp32/ttn-create-application1.jpg)

L'interface vous propose d'associer un appareil à votre application, c'est-à-dire la carte Heltec :

![Associer end device -1](/assets/img/posts/2025-07-25-lorawan-heltec-esp32/ttn-register-enddevice1.jpg)

Renseignez la bande de fréquence, la version de LoRaWAN et la version des paramètres régionaux comme ci-dessous :

![Associer end device -2](/assets/img/posts/2025-07-25-lorawan-heltec-esp32/ttn-register-enddevice2.jpg)

Pour la suite, on laissera les options par défaut, notamment le mode d'activation OTAA (mode de connexion au réseau avec une requête `JoinRequest` envoyée par le nœud), et la classe A (le mode de communication le plus répandu et le plus économe en énergie):

![Associer end device -3](/assets/img/posts/2025-07-25-lorawan-heltec-esp32/ttn-register-enddevice3.jpg)

Saisissez le JoinEUI (symboles hexadécimaux), un identifiant envoyé avec la requête `JoinRequest` pendant le processus de jonction OTAA qui permet de faire le lien avec le serveur pendant l'authentification de l'appareil.
Cliquez sur *Confirm* :

![Associer end device -4](/assets/img/posts/2025-07-25-lorawan-heltec-esp32/ttn-register-enddevice4.jpg)

Génerez successivement les codes DevEUI, AppKey, puis saisissez un identifiant pour l'appareil.
DevEUI (*Device Extended Unique Identifier*) est un identifiant unique du nœud LoRaWAN.
AppKey (*Application Key*) est une clé de chiffrement AES-128 bits qui sert à chiffrer et vérifier l’intégrité des messages lors de l’activation. Cette clé est secrète et unique pour chaque appareil.

En bas de page, enregistrez le *end device* :

![Associer end device -5](/assets/img/posts/2025-07-25-lorawan-heltec-esp32/ttn-register-enddevice5.jpg)

Votre *end device* est enregistré, mais il est encore inactif puisqu'il n'a pas encore rejoint le réseau :

![Associer end device -6](/assets/img/posts/2025-07-25-lorawan-heltec-esp32/ttn-register-enddevice6.jpg)

### Transmettre en LoRaWAN, ce n’est pas bavarder

On pourrait penser qu’envoyer des données textuelles comme `"Température=23.75°C"` est logique et plus lisible. Mais sur le réseau LoRaWAN, **chaque octet compte** : on ne parle pas de haut débit, mais d’un canal partagé ultra-contraint, où le temps de transmission radio (*airtime*) est la ressource la plus précieuse.

Le débit et la fréquence de transmission sont bien sûr limités par des considérations physiques et réglementaires selon les régions du monde, mais quand vous accédez à une passerelle LoRaWAN **publique**, il y a en plus une **politique de bonne conduite communautaire** ([LoRaWAN fair usage policy](https://www.thethingsnetwork.org/docs/lorawan/duty-cycle/#fair-use-policy){:target="_blank"}) dont il faut tenir compte si vous ne voulez pas voir le réseau saturé d'ondes radio.

Sur *The Things NetWork*, le bon usage recommande les limites suivantes :
- 30 secondes d’*airtime* uplink par jour ;
- 10 messages downlink/jour.

*airtime* : temps d’antenne, la durée pendant laquelle un appareil LoRaWAN occupe le canal radio pour transmettre un message.

Si on regarde la *datasheet* du DS18B20, on voit que la température est codée sur deux octets (correspondant à la température multipliée par 16, codage par complément à 2 pour les températures négatives):

![Extrait datasheet DS18B20](/assets/img/posts/2025-07-25-lorawan-heltec-esp32/ds18b20-temperature-data-relationship.jpg)
*Extrait datasheet DS18B20*

Le respect du bon usage communautaire nous oblige donc à transmettre la température brute avec ces deux octets. Le formatage en une température plus lisible pour l'utilisateur viendra après.

Si vous allez sur ce [calculateur en ligne](https://avbentem.github.io/airtime-calculator/ttn/eu868/2){:target="_blank"} avec un *payload* de 2 octets, vous obtenez cet extrait de tableau : 


| Data Rate | Spreading Factor | Bande passante | Airtime par message | Fair Access Policy (avg/hour) |
|-----------|------------------|----------------|----------------------|-------------------------------|
| DR5       | SF7              | 125 kHz        | 46.3 ms              | 27.0                       |

Soit en moyenne 27 messages *uplink* par heure.
En effet avec 30s d'*airtime* uplink par jour, et un *airtime* de 46.3ms par message, cela nous donne : 30/0.0463 = 647.94 messages par jour, soit 647.94/24 = 27 messages par heure en moyenne, ou encore un message envoyé toutes les 2min13s...

Et encore, le DataRate DR5 (SF7/BW125) est utilisé pour des capteurs proches de la passerelle. Si la qualité du lien radio est mauvaise le système peut adapter le DataRate (*Adaptive Data Rate*) ce qui va augmenter l'*airtime* et faire chuter le nombre de messages par heure :

| Data Rate | Spreading Factor | Bande passante | Airtime par message | Fair Access Policy (avg/hour) |
|-----------|------------------|----------------|----------------------|-------------------------------|
| DR5       | SF7              | 125 kHz        | 46.3 ms              | 27.0                          |
| DR4       | SF8              | 125 kHz        | 92.7 ms              | 13.5                          |
| DR3       | SF9              | 125 kHz        | 164.9 ms             | 7.6                           |

Dans le pire cas, DR0 (SF12/BW125), vous ne devrez pas envoyer plus d'un message par heure. Deux octets par heure, c'est pas de bol, vous êtes, hélas, bien loin de la passerelle ou il y a trop d'obstacles...

Dans ma démonstration, avec un lien radio de bonne qualité, la température transmise toutes les 12 minutes (5 messages par heure) conviendra déjà très bien, tout en respectant les bons usages communautaires.

### Le sketch Arduino

La carte Heltec WiFi LoRa 32 sera programmée dans l'environnement Arduino. Depuis l'EDI Arduino, il vous faut d'abord installer le framework et les bibliothèques d'Heltec : voir [Installing development framework and library via Arduino IDE](https://docs.heltec.org/en/node/esp32/esp32_general_docs/quick_start.html#via-arduino-ide){:target="_blank"}.

Pour cette première application, je suis parti de l'exemple d'Heltec [examples/LoRaWAN/LoRaWanOLED/LoRaWanOLED.ino](https://github.com/HelTecAutomation/Heltec_ESP32/blob/master/examples/LoRaWAN/LoRaWanOLED/LoRaWanOLED.ino){:target="_blank"} avec quelques adaptations et compléments.

Pour acquérir la température du capteur DS18B20, j'ai installé la bibliothèque [esp32-ds18b20 par junkfix](https://github.com/junkfix/esp32-ds18b20){:target="_blank"}, une bibliothèque minimale compatible avec la carte Heltec.

Le *sketch* final se contente d'acquérir la température toutes les 12 minutes et la transmet via la passerelle LoraWAN au serveur TTN. Quelques affichages sur l'écran OLED rendront compte de l'état sur le réseau (en cours de jonction, envoi des données, accusé de réception...).

Dans le code, il faudra saisir les paramètres du nœud que vous avez enregistrés précédemment sur la plateforme *The Things Network*, entre autres le mode d'activation OTAA et les codes d'authentification de la carte Heltec :

```cpp
/*OTAA ou ABP*/
bool overTheAirActivation = true;  // Activation On-The-Air

// mode OTAA
// Identifiants uniques générés dans TTN. Ils permettent au module de s’enregistrer sur le réseau.
uint8_t devEui[] = { 0x70, 0xB3, 0xD5, 0x7E, 0xD0, 0x06, 0x53, 0xC8 };
uint8_t appEui[] = { 0x12, 0x34, 0x56, 0x78, 0x9A, 0xBC, 0xDE, 0xFF }; // ou joinEUI
uint8_t appKey[] = { 0x74, 0xD6, 0x6E, 0x63, 0x45, 0x82, 0x48, 0x27, 0xFE, 0xC5, 0xB7, 0x70, 0xBA, 0x2B, 0x50, 0x45 };
```
Dans le menu *Tools* de l'EDI Arduino, sélectionnez la région LoRaWAN active : EU-868

![Région LoRaWAN](/assets/img/posts/2025-07-25-lorawan-heltec-esp32/tools-region-eu868.jpg)

Les autres paramètres à modifier éventuellement sont commentés dans le code complet ci-dessous :

**lorawan-test.ino**
```cpp
#include "LoRaWan_APP.h"

#include "temperature.h"

#include <Wire.h>
#include "HT_SSD1306Wire.h"

// mode OTAA
// Identifiants uniques générés dans TTN. Ils permettent au module de s’enregistrer sur le réseau.
uint8_t devEui[] = { 0x70, 0xB3, 0xD5, 0x7E, 0xD0, 0x07, 0x20, 0x9B };
uint8_t appEui[] = { 0xC8, 0xB9, 0x6A, 0xD4, 0xF0, 0x21, 0xF7, 0x9B }; // ou joinEUI
uint8_t appKey[] = { 0x5F, 0x21, 0x81, 0x38, 0x29, 0x11, 0x28, 0xD5, 0xF6, 0xBB, 0x9C, 0xC4, 0x79, 0x58, 0x3C, 0xD0 };


// Variables ABP obligatoires pour éviter erreur de linkage
uint8_t nwkSKey[16] = { 0 };
uint8_t appSKey[16] = { 0 };
uint32_t devAddr = 0x00000000;


/*LoraWan channelsmask, default channels 0-7*/
// Le module utilise uniquement les canaux 0 à 7,
// ce qui est la configuration par défaut pour la bande EU868 utilisée en France.
// Les passerelles TTN écoutent généralement sur les canaux 0 à 7.
uint16_t userChannelsMask[6] = { 0x00FF, 0x0000, 0x0000, 0x0000, 0x0000, 0x0000 };

/*LoraWan region, à selectionner dans le menu Tools*/
LoRaMacRegion_t loraWanRegion = ACTIVE_REGION;

/*LoraWan Class, Classe A et Classe C supportées*/
// classe A, qui est la plus courante et la plus économe en énergie dans le protocole LoRaWAN.
DeviceClass_t loraWanClass = CLASS_A;

/* Intervalle entre deux transmissions de message, valeur en millisecondes */
uint32_t appTxDutyCycle = 12 * 60000; // soit 12 minutes

/*OTAA ou ABP*/
bool overTheAirActivation = true;  // Activation On-The-Air

/*ADR enable*/
// Adaptive Data Rate. En mettant loraWanAdr = true, on demande au module
// de laisser le réseau TTN optimiser automatiquement la vitesse de transmission et la puissance d’émission.
bool loraWanAdr = true;

/* Indicates if the node is sending confirmed or unconfirmed messages */
// Ddétermine si le module LoRaWAN demande un accusé de réception (ACK) pour chaque message envoyé.
// En mettant isTxConfirmed = true, on active le mode uplink confirmé (uplink : module Heltec vers réseau TTN),
// ce qui implique une communication bidirectionnelle avec le réseau TTN.
bool isTxConfirmed = true;

/* Application port */
// Définit le port d’application LoRaWAN utilisé pour transmettre le payload.
// En LoRaWAN, chaque message envoyé peut être associé à un port logique (appelé FPort)
// qui permet au serveur de savoir comment interpréter les données.
uint8_t appPort = 2;


/*!
* Nombre d'essais pour transmettre la trame, au cas où la couche LoRaMAC ne
* reçoit pas d'acquittement. La couche MAC opère une adaptation du datarate,
* selon les spécifications LoRaWAN V1.0.2, chapitre 18.4, d'après le tableau
* ci-dessous :
*
* Transmission nb | Data Rate
* ----------------|-----------
* 1 (first)       | DR
* 2               | DR
* 3               | max(DR-1,0)
* 4               | max(DR-1,0)
* 5               | max(DR-2,0)
* 6               | max(DR-2,0)
* 7               | max(DR-3,0)
* 8               | max(DR-3,0)
*
* Note : si le nombre d'essais est défini à 1 ou 2, la couche MAC ne diminuera
* pas le datarate si la couche LoRaMAC ne reçoit pas d'acquittement.
*/
// Configure le nombre de tentatives de retransmission pour les uplinks confirmés en LoRaWAN.
// Autrement dit, si le module envoie un message et ne reçoit pas d’accusé de réception (ACK) de TTN,
// il va réessayer jusqu’à 4 fois en diminuant le datarate avant d’abandonner.
uint8_t confirmedNbTrials = 4;

/* Préparation du payload de la trame */
static void prepareTxFrame(uint8_t port) {
  /*appData size is LORAWAN_APP_DATA_MAX_SIZE which is defined in "commissioning.h".
  *appDataSize max value is LORAWAN_APP_DATA_MAX_SIZE.
  *if enabled AT, don't modify LORAWAN_APP_DATA_MAX_SIZE, it may cause system hanging or failure.
  *if disabled AT, LORAWAN_APP_DATA_MAX_SIZE can be modified, the max value is reference to lorawan region and SF.
  *for example, if use REGION_CN470, 
  *the max value for different DR can be found in MaxPayloadOfDatarateCN470 refer to DataratesCN470 and BandwidthsCN470 in "RegionCN470.h".
  */
  appDataSize = 2;

  uint16_t t = getRawTemperature(); // Acquisition température ds18b20
  appData[0] = (t >> 8) & 0xFF;  // octet poids fort
  appData[1] = t & 0xFF;         // octet poids faible
  //Serial.print(t / 16.);
  //Serial.println("°C");
}

RTC_DATA_ATTR bool firstrun = true;

extern SSD1306Wire display;


void setup() {

  display.init();
  display.setContrast(255);  // valeur max : 0 à 255
  display.clear();
  display.drawString(0, 0, "LoRa Init...");
  display.display();


  initTemperature();

  Serial.begin(115200);
  Mcu.begin(HELTEC_BOARD, SLOW_CLK_TPYE);

  if (firstrun) {
    LoRaWAN.displayMcuInit();
    firstrun = false;
  }
}

void loop() {
  switch (deviceState) {
    case DEVICE_STATE_INIT:
      {
#if (LORAWAN_DEVEUI_AUTO)
        LoRaWAN.generateDeveuiByChipID();
#endif
        LoRaWAN.init(loraWanClass, loraWanRegion);
        //both set join DR and DR when ADR off
        LoRaWAN.setDefaultDR(3);
        break;
      }
    case DEVICE_STATE_JOIN:
      {
        LoRaWAN.displayJoining();
        LoRaWAN.join();
        break;
      }
    case DEVICE_STATE_SEND:
      {
        LoRaWAN.displaySending();
        prepareTxFrame(appPort);
        LoRaWAN.send();

        deviceState = DEVICE_STATE_CYCLE;
        break;
      }
    case DEVICE_STATE_CYCLE:
      {
        // Schedule next packet transmission
        txDutyCycleTime = appTxDutyCycle + randr(-APP_TX_DUTYCYCLE_RND, APP_TX_DUTYCYCLE_RND);
        LoRaWAN.cycle(txDutyCycleTime);
        deviceState = DEVICE_STATE_SLEEP;
        break;
      }
    case DEVICE_STATE_SLEEP:
      {
        LoRaWAN.displayAck();
        LoRaWAN.sleep(loraWanClass);
        break;
      }
    default:
      {
        deviceState = DEVICE_STATE_INIT;
        break;
      }
  }
}
```

**temperature.h**
```cpp
#ifndef TEMPERATURE_H
#define TEMPERATURE_H
#include <stdint.h>
#include <OneWireESP32.h>

// GPIO 47, broche n°13
#define PIN_DATA_DS18B20 47

extern OneWire32 ds;
extern uint64_t addr[1];

uint8_t initTemperature(void);
int16_t getRawTemperature(void);

#endif
```

**temperature.cpp**
```cpp
#include "temperature.h"

OneWire32 ds(PIN_DATA_DS18B20);
uint64_t addr[1];

uint8_t initTemperature() {
  uint8_t devices = ds.search(addr, 1); // Renvoie le nombre de capteurs détectés, 1 au maximum
  return devices;
}

int16_t getRawTemperature() {
  float temperature;
  ds.request();  // Lance une mesure
  delay(750);    // durée de conversion

  uint8_t err = ds.getTemp(addr[0], temperature);  // Lecture à l'adresse du capteur
  if (err) {
    return 0x8000;
  } else {
    return round(temperature * 16);
  }
}
```

Il vous reste à flasher le programme et faire un un *reset* de la carte. Vous devriez voir la mention *STARTING*, puis *JOINING* apparaître sur l'écran OLED, suivi de *SENDING*, *ACK RECEIVED* une fois la donnée transmise et confirmée.


### Les données en Live

Plutôt de voir les deux octets transmis `01 BB` qui sont l'image de la température, l'utilisateur préfèrera sans doute voir un *payload* formaté de façon plus sympathique, comme `{temperature: 27.69}`.

Pour cela,rendez-vous dans le menu *Payload formatters > Uplink* de *The Things Network*, et remplacez le code JavaScript par défaut par celui ci-dessous :

![Payload formatter](/assets/img/posts/2025-07-25-lorawan-heltec-esp32/payload-formatters-uplink.jpg)

Pour obtenir la valeur de température en °C, il faut diviser la valeur brute par 16 (ou la multiplier par 0.0625).

Si vous vous rendez ensuite dans le menu *Live Data*, les valeurs transmises devraient défiler en *preview* toutes les 12 minutes :

![Live Data](/assets/img/posts/2025-07-25-lorawan-heltec-esp32/live-data.jpg)
*Données transmises en preview avec mon capteur en plein soleil !*

Votre premier nœud LoRaWAN est fonctionnel...

### Conclusion

Si vous avez réussi à transmettre vos données par le réseau LoRaWAN, le projet n'est pas terminé pour autant. Même avec les données formatées, la restitution n'est pas très jolie et la persistance des données est limitée (vous avez quelques options avec le menu *Message storage*, mais cela reste encore insuffisant).
Pour l'utilisateur final qui voudrait voir ses températures dans un tableau de bord avec des graphiques, il faut encore transmettre ces données en *preview* à un service tiers spécialisé. Ce sera l'objet d'un futur billet...