---
title: "Projet IoT avec Raspberry Pi et Zigbee2MQTT"
date: 2025-04-13
categories: ['Raspberry Pi', 'domotique']
tags: ['Raspberry Pi', 'IoT', 'domotique', 'Zigbee2MQTT', 'MQTT', 'Zigbee']
math: false
target_blank: true
---
Vous souhaitez surveiller la température et l'humidité à l'intérieur de chez vous, piloter l'arrosage de vos plantes ou votre chauffage, le tout à distance depuis un smartphone ? Ce projet domotique à base de Raspberry Pi est peut-être fait pour vous !

En cherchant des solutions, je suis tombé sur ce petit cylindre blanc de 45mm de diamètre, un capteur de température et d’humidité de chez Sonoff (référence Sonoff SNZB-02P) :

![Sonoff SNZB-02P](/assets/img/posts/2025-04-13-raspberry-pi-iot-zigbee2mqtt/sonoff-snzb-02p.jpg)
*Capteur de température et d'humidité Sonoff SNZB-02P*

Ce capteur fonctionne en [Zigbee](https://csa-iot.org/fr/toutes-les-solutions/zigbee/){:target="_blank"}, un protocole de communication radio très économe en énergie. Le système est livré avec une pile bouton CR2477, pour une autonomie annoncée de 4 ans !! Le cylindre est aimanté et vous trouverez aussi dans l’emballage un support métallique à visser sur un mur ou une paroi. Le tout pour 20€ environ.

Le capteur présente un autre avantage : il est compatible avec le service [Zigbee2MQTT](https://www.zigbee2mqtt.io/){:target="_blank"}, une interface opensource qui permet la publication des mesures du capteur en passant par le protocole de messagerie [MQTT](https://mqtt.org/){:target="_blank"}. Concrètement, c’est un dongle USB Zigbee2MQTT de la même marque Sonoff (20€ environ) connecté à la carte Raspberry Pi qui établit la passerelle :

![Sonoff USB dongle](/assets/img/posts/2025-04-13-raspberry-pi-iot-zigbee2mqtt/zonoff-zigbee-dongle_usb.jpg)
*Sonoff Zigbee 3.0 USB Dongle Plus (ZB Dongle-P, contrôleur TI CC2652P flashé avec le firmware coordinateur Z-Stack)*

L'architecture de mon installation est la suivante :

![Architecture Raspberry Pi avec Zigbee2MQTT](/assets/img/posts/2025-04-13-raspberry-pi-iot-zigbee2mqtt/architecture-rpi-zigbee2mqtt.png)
*Attention : pour brancher le dongle sur la carte Raspberry Pi, il faut une rallonge USB pour éloigner le dongle des interférences avec le WiFi de la carte (au moins un mètre).*

Pour la configuration logicielle de la Raspberry Pi, j'installe les outils de la pile [IOTstack](https://sensorsiot.github.io/IOTstack/){:target="_blank"}. Les développeurs d’IOTstack proposent une série d’images de services et d’applications IdO pour Raspberry Pi déjà configurées et prêtes à être déployées dans des conteneurs Docker. L’interface textuelle vous permettra d’administrer facilement les conteneurs de la pile, sans avoir à connaître les commandes Docker (mais un peu de lignes de commande ne fera pas de mal).

La pile comprend notamment le broker [MQTT Mosquitto](https://sensorsiot.github.io/IOTstack/Containers/Mosquitto/){:target="_blank"}, qui reçoit les messages des données mesurées et qui les transmettra aux clients abonnés au *topic*.
Il faut aussi installer et configurer le conteneur du service Zigbee2MQTT afin que le dongle USB soit reconnu, voir [la documentation IOTstack](https://sensorsiot.github.io/IOTstack/Containers/Zigbee2MQTT/){:target="_blank"}.

Le conteneur comprend également un petit assistant de configuration en ligne, à l’URL `http://<adresse IP de la Raspberry Pi>:8080`. Autorisez l’appairage depuis l’interface et maintenez le petit bouton sur la paroi cylindrique du capteur appuyé pendant quelques secondes. Un voyant devrait clignoter pendant la phase d’appairage, et le capteur devrait apparaître dans la liste indiquant sa disponibilité *En ligne* :

![Assistant de configuration Zigbee2MQTT](/assets/img/posts/2025-04-13-raspberry-pi-iot-zigbee2mqtt/assistant-zigbee2mqtt-1.png)

Ici, j’ai renommé provisoirement le capteur (*nom simplifié*).
Le capteur devrait commencer à exposer ses données :

![Exposition des données MQTT](/assets/img/posts/2025-04-13-raspberry-pi-iot-zigbee2mqtt/assistant-zigbee2mqtt-4.png)

Les journaux doivent rendre compte des messages publiés :

![Journal messages MQTT publiés](/assets/img/posts/2025-04-13-raspberry-pi-iot-zigbee2mqtt/assistant-zigbee2mqtt-3.png)

Les données sont publiées régulièrement au format JSON sur le topic `zigbee2mqtt/fleb-SNZB-02P` :

```json
{"battery":100,"humidity":57.2,"linkquality":24,"temperature":22.4}
```


Voilà du matériel Zigbee et une interface Zigbee2MQTT concluants pour une utilisation domotique avec une carte Raspberry Pi. Le Zigbee est très pertinent pour un usage domotique avec du matériel bon marché et une faible consommation d’énergie, et le protocole de messagerie MQTT est très simple à exploiter par les développeurs.
Vous pouvez maintenant étendre votre réseau domotique avec les nombreux capteurs et actionneurs disponibles sur le marché (capteurs environnementaux, systèmes d’éclairage, interrupteurs, relais commandés, systèmes d’ouverture/fermeture, d’arrosage automatique, détecteurs de présence, d'ouverture, de fuite... prises connectées, thermostats, etc.)
Le service Zigbee2MQTT est opensource, et de nombreux constructeurs en plus de vendre leur passerelle propriétaire mettent en lumière la compatibilité de leur matériel avec ce protocole. Méfiez-vous toutefois, les modules Zigbee ne sont pas tous compatibles avec Zigbee2MQTT. Référez-vous au site officiel pour voir [la liste des matériels supportés](https://www.zigbee2mqtt.io/guide/supported-hardware.html){:target="_blank"}.

Bien entendu, vous n'allez pas vous contenter de visualiser les données dans les *logs* de l'assistant, il est temps d'ingérer toutes ces données MQTT et les présenter convenablement à l'utilisateur grâce aux autres outils de la pile [IOTstack](https://sensorsiot.github.io/IOTstack/){:target="_blank"}.

Affaire à suivre...