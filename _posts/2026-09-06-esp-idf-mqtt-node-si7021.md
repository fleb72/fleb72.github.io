---
title: "ESP-IDF : Nœud météo MQTT avec ESP32-C6 et Si7021"
date: 2026-09-06
categories: ["Internet des Objets", "domotique"]
tags: [ESP32, "ESP-IDF"]
math: true
target_blank: true
---

Ce projet met en œuvre un nœud météo basé sur un ESP32‑C6 et un capteur de température et humidité Si7021 (i<sup>2</sup>C), avec publication des mesures horodatées en MQTT.
Les données peuvent ensuite être intégrées dans un tableau de bord (Grafana, MQTT Explorer, etc.).
Ce projet sert de prototype pour un petit nœud météo local, et à titre pédagogique pour illustrer l’usage d’un ESP32‑C6, d’un capteur I<sup>2</sup>C, de FreeRTOS, SNTP, et de MQTT.
Il peut être étendu facilement (NVS, interface Web, BLE, autres capteurs…).

> Retrouvez le dépôt du projet sur Github en suivant ce lien : [Nœud météo MQTT avec ESP32‑C6 et capteur SI7021](https://github.com/fleb72/esp32c6-si7021-mqtt-node){:target="_blank"}
{: .prompt-info }

## Matériels et logiciels utilisés

- ESP32-C6 DevKitC-1
- Module Si7021 d'AdaFruit (connexion I<sup>2</sup>C → SDA : GPIO5, SCL : GPIO6)
- Broker MQTT (Mosquitto)
- Environnement de développement Espressif ESP-IDF v5.5
- [Driver Si7021](https://components.espressif.com/components/esp-idf-lib/si7021/){:target="_blank"}

## Objectifs fonctionnels

### Acquisition des données
- Lire périodiquement la température et l’humidité via le capteur Si7021 (I<sup>2</sup>C).
- Gérer la fréquence d’acquisition via un paramètre configurable dans `menuconfig`.
- Assurer une lecture fiable avec gestion des erreurs I<sup>2</sup>C.

### Publication MQTT

- Publier les mesures au format JSON structuré, incluant : température, humidité, et horodatage SNTP.
 
 Exemple : 
```json
    {
      "timestamp": "2026-09-06T11:12:00Z",
      "temperature": 22.4,
      "humidity": 54.1
    }
```

- Permettre la configuration complète du WiFi et du client MQTT via `menuconfig` :
  - identifiant/mot de passe SSID
  - adresse du broker
  - port
  - topic
  - QoS

- Gérer les reconnexions automatiques WiFi et MQTT.

- Un moteur de décision configurable doit déterminer **quand** publier une mesure MQTT, en fonction :
  - des variations significatives de température ou d’humidité (deltas configurables),
  - d’un intervalle minimal entre deux publications pour éviter les rafales,
  - d’un intervalle maximal de publication pour garantir une mise à jour régulière,
  - d’un compromis entre réactivité et charge réseau, afin de ne pas saturer le broker MQTT.

Ce moteur permet de publier uniquement lorsque c’est pertinent, tout en assurant un rythme minimal de publication pour les tableaux de bord.

## Paramétrage via menuconfig

Accès au menu de configuration grâce à la commande : `ìdf.py menuconfig`

![menuconfig](/assets/img/posts/2026-09-06-esp-idf-mqtt-node-si7021/menuconfig.png)
*Commande : idf.py menuconfig*

Voir le fichier `Kconfig.projbuild` pour plus de détails sur les paramètres à configurer.

## Quick Start

Pour compiler et téléverser le nœud météo ESP32‑C6, les commandes essentielles de l’environnement ESP‑IDF sont les suivantes :

```bash
# Configuration du projet (Wi‑Fi, MQTT, fréquences, etc.)
idf.py menuconfig

# Compilation du firmware
idf.py build

# Flash du firmware sur l’ESP32‑C6 (port à adapter)
idf.py -p /dev/ttyUSB0 flash

# Ouverture du moniteur série (port à adapter)
idf.py -p /dev/ttyUSB0 monitor
```

## Architecture logicielle
Le nœud météo repose sur une architecture logicielle organisée autour de plusieurs tâches FreeRTOS indépendantes.
Chaque tâche a une responsabilité claire, ce qui facilite la maintenance, l’extension et la fiabilité du système.

```c
void app_main()
{
    // ...

    xTaskCreatePinnedToCore(si7021_task, "si7021", configMINIMAL_STACK_SIZE * 8, NULL, 5, NULL, APP_CPU_NUM);
    xTaskCreatePinnedToCore(decision_task, "decision", configMINIMAL_STACK_SIZE * 4, NULL, 4, NULL, APP_CPU_NUM);
    xTaskCreatePinnedToCore(wifi_task, "wifi", configMINIMAL_STACK_SIZE * 8, NULL, 3, NULL, APP_CPU_NUM);
    xTaskCreatePinnedToCore(mqtt_task, "mqtt", configMINIMAL_STACK_SIZE * 8, NULL, 4, NULL, APP_CPU_NUM);
}
```

### Tâche d’acquisition (si7021_task, priorité 5)

Tâche responsable de la lecture périodique du capteur Si7021 via I<sup>2</sup>C.

Fonctions principales :
- initialisation du bus I<sup>2</sup>C (`i2cdev`) ;
- lecture de la température et de l’humidité
- mise à disposition des données dans une structure partagée (variable globale `g_si7021_data` protégée également par un *mutex*) ;
- respect de l’intervalle d’acquisition défini dans `menuconfig` (paramètre `CONFIG_MEASURE_INTERVAL_SECONDS`, 10s par défaut).

### Tâche de décision (decision_task, priorité 4)
Tâche qui implémente le moteur de décision configurable.

Rôles :

- déterminer si une publication MQTT est nécessaire ;
- appliquer les règles configurées dans `menuconfig` pour assurer un rythme minimal de publication sans encombrer le réseau de messages MQTT :
  - delta température (paramètre `CONFIG_TEMP_DELTA`, 5 x 1/10è de °C par défaut),
  - delta humidité (paramètre `CONFIG_HUM_DELTA`, 10 x 1/10è %RH par défaut),
  - intervalle minimal entre deux publications (paramètre `CONFIG_MIN_PUBLISH_INTERVAL_SECONDS`, 30s par défaut),
  - intervalle maximal ou *heartbeat* (paramètre `CONFIG_PERIODIC_DELAY_MINUTES`, 10min par défaut) ;
- en cas de décision positive, la donnée prête pour publication est enfilée dans une *queue* FreeRTOS.

### Tâche de publication MQTT (mqtt_task, priorité 4)
Tâche qui gère la communication avec le broker MQTT.

Fonctions :

- connexion au broker (Wi‑Fi + MQTT) ;
- reconnexion automatique en cas de perte de réseau ;
- lecture des données dans la *queue*, et construction du message JSON incluant l'horodatage SNTP (ISO‑8601) ;
- publication sur le topic configuré.

### Tâche de gestion du WiFi (wifi_task, priorité 3)

Tâche responsable de la connexion réseau et de la synchronisation SNTP.

Fonctions :

- connexion au réseau (paramètres SSID / mot de passe depuis `menuconfig`) ;
- initialisation et synchronisation au service SNTP ;
- gestion des erreurs (ex. ESP_ERR_WIFI_SSID), et reconnexion automatique ;
- notification des autres tâches en cas de perte de réseau.

L’usage de FreeRTOS permet de structurer le nœud météo en tâches indépendantes, chacune dédiée à une fonction précise (acquisition, décision, publication, réseau). Ce découplage apporte des bénéfices essentiels :

- chaque tâche avance à son propre rythme sans bloquer les autres (lecture capteur, logique de décision, MQTT) ;

- les problèmes réseau ou MQTT n’interrompent pas l’acquisition, les reconnexions sont gérées séparément ;

- chaque tâche a une responsabilité unique, ce qui simplifie l’évolution du projet (ajout de capteurs, stockage, interface Web).

FreeRTOS assure ainsi un fonctionnement stable, réactif et modulaire, adapté à un nœud météo autonome.

## Exemple de log dans un terminal

```
...
WiFi connected, IP obtained.
I (5019) esp_netif_handlers: sta ip: 192.168.1.131, mask: 255.255.255.0, gw: 192.168.1.254
SNTP started.
Envoi dans la queue -> Temperature: 23.66 °C, Humidity: 59.13 %
MQTT connected
Envoi dans la queue -> Temperature: 26.53 °C, Humidity: 64.61 %
MQTT published at 2026-08-31T19:52:23: {"temperature":26.53,"humidity":64.61,"timestamp":"2026-08-31T19:52:23"}
Envoi dans la queue -> Temperature: 25.45 °C, Humidity: 68.37 %
MQTT published at 2026-08-31T19:52:28: {"temperature":25.45,"humidity":68.37,"timestamp":"2026-08-31T19:52:28"}
...
```


## Conclusion
Ce projet montre qu’il est possible de construire un nœud météo simple, fiable et entièrement configurable en s’appuyant sur l’ESP32‑C6, un capteur I²C et l’environnement ESP‑IDF. L’architecture en tâches FreeRTOS, associée au moteur de décision et au paramétrage via `menuconfig`, permet d’obtenir un système autonome, réactif et peu bavard sur le réseau MQTT.

Les données publiées au format JSON peuvent être intégrées facilement dans n’importe quel tableau de bord (Grafana, MQTT Explorer, Node‑RED…), ce qui ouvre la voie à de nombreuses extensions : ajout d’autres capteurs, stockage local, interface Web, ou intégration dans un système domotique existant.

Le [dépôt GitHub](https://github.com/fleb72/esp32c6-si7021-mqtt-node){:target="_blank"} fournit l’ensemble du code source pour reproduire ou adapter ce nœud météo selon vos besoins.
