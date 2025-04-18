---
title: "Carte relaisx4 compatible Zigbee2MQTT"
date: 2025-04-18
categories: ['Raspberry Pi', 'domotique']
tags: ['Raspberry Pi', 'IoT', 'domotique', 'Zigbee2MQTT', 'MQTT', 'Zigbee']
math: false
target_blank: true
---

Je reviens à nouveau sur [mon projet IoT]({{ site.baseurl }}{{ site.url }}/posts/raspberry-pi-iot-zigbee2mqtt/){:target="_blank"}, et la carte [Raspberry Pi comme serveur domotique]({{ site.baseurl }}{{ site.url }}/posts/pile-iotstack/){:target="_blank"}.

 Malgré un catalogue de [composants Zigbee](https://www.zigbee2mqtt.io/supported-devices/#){:target="_blank"} déjà bien fourni, il me fallait encore une carte Zigbee prototype avec plusieurs entrées-sorties, réutilisable en fonction des projets menés. C’est là que la carte [Zigbee 4 Channel Relay](https://www.tindie.com/products/a_lab_technology/zigbee-4-channel-relay/){:target="_blank"} intervient :

 ![Zigbee 4 channel relay, Alab Technology](/assets/img/posts/2025-04-18-relayx4-board-zigbee2mqtt/20240720_114621f.jpg)
 *Zigbee 4 channel Relay, Alab Technology*


|1| 	Convertisseur 230 VAC – 5 VDC|
|2| 	Module Zigbee|
|3| 	Bouton d’appairage|
|4| 	Bornier entrées numériques x 4|
|5| 	Bouton-poussoir à appui momentané x 4|
|6| 	LED verte de statut|
|7| 	Relais X 4|

Et voici un schéma de l'architecture du réseau testée :

![Architecture réseau zigbee](/assets/img/posts/2025-04-18-relayx4-board-zigbee2mqtt/architecture-rpi-zigbee2mqtt.png)

La carte est alimentée par le secteur 230 V (bornier à vis pour phase L et neutre N), et on peut déjà regretter de ne pas avoir proposé une alimentation 12 ou 24 V CC.

Le constructeur propose plusieurs versions de son firmware, j’ai pris celle avec l’option : *Independent Push (closing the input switches the status to the opposite)*. Avec cette option, les entrées et la commande des relais fonctionnent de façon indépendante.

Comme la carte est compatible Zigbee2MQTT, la communication se fait par le service de messagerie MQTT, avec des messages publiés au format JSON, du genre :

```json
{"input_state_in1":0, "input_state_in2":0, "input_state_in3":0, "input_state_in4":0, "linkquality":30, "state_l1":"OFF", "state_l2":"OFF", "state_l3":"OFF", "state_l4":"OFF"}
```

Voir les [paramètres exposés](https://www.zigbee2mqtt.io/devices/alab.switch.html){:target="_blank"} (dont certains dépendent du firmware flashé).

### Les entrées

![Inputs zigbee 4 relays channel](/assets/img/posts/2025-04-18-relayx4-board-zigbee2mqtt/20240721_155720.jpg)

Comme dit précédemment, les actions sur les entrées sont sans conséquences sur les commandes des relais (voir les variantes du firmware proposées par le constructeur).
Il suffit de s’abonner au topic `zigbee2mqtt/fleb-relay` (`fleb-relay` : nom simplifié donné au composant) pour récupérer l’état 0 ou 1 des quatre entrées via les propriétés <code>"input_state_in<strong>x</strong>"</code>. Attention, ces quatre entrées se comportent comme des bascules. La fermeture du circuit, que ce soit par un dispositif branché entre les deux bornes de l’entrée ou par un appui sur le bouton-poussoir correspondant, provoque le basculement de l’état en entrée (mais on peut aussi forcer l'entrée en publiant un <code>"input_state_in<strong>x</strong>":0</code> ou <code>"input_state_in<strong>x</strong>":1</code> sur le topic `zigbee2mqtt/fleb-relay/set`).

### Les sorties

Les relais sont des relais miniatures SPDT (*Single Pole Double Throw*), c’est-à-dire qu’il y a trois connecteurs pour chaque relais qui sont repérés sur les borniers à vis : *Common* (COM), *Normally Open* (NO), et *Normally Closed* (NC).

![Relais x 4](/assets/img/posts/2025-04-18-relayx4-board-zigbee2mqtt/20240721_171921.jpg)


Aucune sortie COM, NC ou NO des relais n'est reliée à l’alimentation de la carte, vous pouvez donc les connecter à vos appareils et leur alimentation, et commuter des charges jusqu’à 230V AC / 5A.
Par exemple pour activer le 1er relais (*Channel 1*), il faut publier un message au topic `zigbee2mqtt/fleb-relay/set` avec le contenu `{"state_l1":"ON"}`, et `{"state_l1":"OFF"}` pour le désactiver. Une petite LED rouge à côté de chaque relais s’allume lorsque celui-ci est activé.

Ci-dessous, une petite vidéo de démonstration où je fais claquer les relais et pousse quelques boutons (interface Web avec [Node-RED]({{ site.baseurl }}{{ site.url }}/posts/pile-iotstack/#chapitre-node-red){:target="_blank"}) :

{% include embed/youtube.html id='Lc7ilKVUOzs' %}

![IHM Node-RED](/assets/img/posts/2025-04-18-relayx4-board-zigbee2mqtt/nodered-ihm-Zigbee4ChannelRelay.png)
*Construction de l'interface Web avec Node-RED sur Raspberry Pi*

À suivre pour une application concrète avec cette carte (système d'arrosage automatique, commande d'ouverture de toit d'une serre, allez savoir ;-) 