---
title: "Piloter un vérin électrique depuis une page Web en Zigbee"
date: 2025-04-24
categories: ['Raspberry Pi', 'domotique']
tags: ['IoT', 'Zigbee2MQTT', 'MQTT', 'Zigbee']
math: false
target_blank: true
---

Je poursuis ici [mon projet IoT]({{ site.baseurl }}{{ site.url }}/posts/raspberry-pi-iot-zigbee2mqtt/){:target="_blank"} avec mes équipements Zigbee, et mon dongle USB Zigbee2MQTT.

{% include embed/youtube.html id='n40Sm4dQVxI' %}

Voir les informations sur la [carte relais Zigbee]({{ site.baseurl }}{{ site.url }}/posts/relayx4-board-zigbee2mqtt/){:target="_blank"} utilisée ici.

![Architecture réseau zigbee](/assets/img/posts/2025-04-18-relayx4-board-zigbee2mqtt/architecture-rpi-zigbee2mqtt.png)
*Architecture simplifiée du réseau avec Zigbee2MQTT*

Le mini-vérin électrique 12V (course 50mm, 8mm/s, 70N) intègre des rupteurs de fin de course. Il est câblé avec deux relais de la carte, et on active un relais ou l'autre selon le sens de rentrée ou sortie de la tige :

![Pré-actionneur mini-vérin](/assets/img/posts/2025-04-24-pilotage-verin-electrique-zigbee/linear-actuator-relays.png)


Une interface Web avec deux boutons est construite avec Node-RED. Sur clic, un message JSON est publié par MQTT, par exemple pour sortir la tige du vérin:

```json
{"state_l1":"OFF", "state_l2":"ON"} 
```
Je vais pouvoir commander l'ouverture de la trappe de la mini-serre pour l'aérer. Projet à suivre...