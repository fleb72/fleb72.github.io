---
title: "Zigbee low-cost : attention aux firmwares"
date: 2026-04-13
categories: ["Internet des Objets", 'domotique']
tags: ['IoT', 'Zigbee2MQTT', 'MQTT', 'Zigbee', 'Hardware']
math: false
target_blank: true
---

### Retour d’expérience avec un module relais Zigbee 4CH

J’ai récemment acheté un module relais Zigbee 4 canaux de marque MHCOZY pour piloter un petit montage en basse tension (12 V CC). On en trouve facilement sur Amazon pour une vingtaine d’euros, et la fiche produit laisse entendre qu’il s’agit d’un module Zigbee tout à fait standard. Avant de l’acheter, j’ai vérifié sur le site officiel que ce modèle était bien supporté par Zigbee2MQTT (voir [www.zigbee2mqtt.io/supported-devices/#v=MHCOZY](https://www.zigbee2mqtt.io/supported-devices/#v=MHCOZY){:target="_blank"}), ce qui semblait confirmer que je pouvais l’intégrer sans mauvaise surprise.

![boitier-mhcozy](/assets/img/posts/2026-04-13-zigbee-firmware-relay4ch-module/boitier-mhcozy.jpg)

J’ai reçu exactement le même modèle que celui présenté dans la documentation : même marque MHCOZY, même référence ZG‑003‑RF. L’appairage avec Zigbee2MQTT s’est déroulé sans difficulté, et j’ai pu activer les relais depuis l’interface web du *front‑end* sans aucun problème.

![expositions-relaisx2](/assets/img/posts/2026-04-13-zigbee-firmware-relay4ch-module/exposition-relaisx2.jpg)
*Relais 1 et 2 activés !*

À ce stade, tout semble parfaitement conforme : le module est reconnu, les relais répondent, et Zigbee2MQTT l’affiche comme un appareil supporté. Bref, je me dis que tout va bien.

Mais c’est justement en allant un peu plus loin — en examinant les *clusters* Zigbee réellement exposés par l’appareil — que les choses deviennent intéressantes.
Dans l’onglet *Expositions* du module, Zigbee2MQTT affiche un point de terminaison (*endpoint*) par relais, chacun associé au *cluster* standard `On/Off (0x0006)` défini dans la *Zigbee Cluster Library* (ZCL). On y retrouve les boutons *On* et *Off* habituels, qui correspondent aux commandes normalisées de ce *cluster*.

![cluster_on_off](/assets/img/posts/2026-04-13-zigbee-firmware-relay4ch-module/expositions.jpg)

Sauf que, dans la documentation de Zigbee2MQTT au paragraphe *exposes* (voir [www.zigbee2mqtt.io/devices/TYWB_4ch-RF.html#exposes](https://www.zigbee2mqtt.io/devices/TYWB_4ch-RF.html#exposes){:target="_blank"}), on peut lire :


> **On with timed off**
>
> When setting the state to ON, it might be possible to specify an automatic shutoff after a certain amount of time. To do this add an additional property on_time to the payload which is the time in seconds the state should remain on. Additionally an off_wait_time property can be added to the payload to specify the cooldown time in seconds when the switch will not answer to other on with timed off commands. Support depends on the switch firmware. Some devices might require both on_time and off_wait_time to work Examples : `{"state" : "ON", "on_time": 300}, {"state" : "ON", "on_time": 300, "off_wait_time": 120}`.
{: .prompt-info }

Super, le relais peut donc être configuré pour se couper automatiquement après un délai donné.
Sauf que… dans l’interface graphique de Zigbee2MQTT, aucun *cluster* ne permet d’activer ce mode, ni de régler la moindre temporisation. Rien dans les *Expositions*, aucune trace de la commande `On with timed off`, et pas le moindre attribut `on_time` ou `off_wait_time` en fouillant les *clusters* exposés.

Et c’est là que je réalise que j’ai peut‑être survolé trop vite la documentation et manqué au passage la petite mention en fin de paragraphe :
**Support depends on the switch firmware.**

Bienvenue dans l’écosystème des modules Zigbee *low‑cost*, notamment ceux basés sur des plateformes génériques comme Tuya. **Dans ce monde-là, les fonctionnalités exposées dépendent entièrement du firmware embarqué. Et ce firmware peut varier d’un lot à l’autre, parfois même sous la même référence commerciale.**

Résultat : deux modules strictement identiques en apparence peuvent se comporter de manière très différente une fois intégrés dans Zigbee2MQTT.

Il est important de préciser que tout cela est parfaitement légal. Les fabricants qui s’appuient sur des plateformes génériques comme Tuya utilisent des firmwares différents selon les lots, les versions matérielles ou les intégrations prévues, sans forcément modifier la référence commerciale du produit. Ce n’est pas une arnaque : simplement, rien n’oblige ces fabricants à garantir que deux modules portant le même nom exposeront rigoureusement les mêmes fonctionnalités Zigbee.
Dans l’univers des modules *low‑cost*, cette variabilité est même assez courante. Et tant que le produit remplit sa fonction de base — ici, activer des relais — il reste conforme à ce qui est annoncé.


Il faut croire que le firmware TS0004 de chez Tuya est minimaliste.
Ce firmware n’expose que :
- le *cluster* `On/Off` (`0x0006`) ;
- un *endpoint* par relais ;
- aucune commande avancée ;
- aucun attribut `on_time` ou `off_wait_time` ;
- aucune commande `On with timed off`. Cette commande permet de gérer une temporisation. Le relais reste activé pendant la durée `on_time`. Avec `off_wait_time`, on précise le délai avant extinction définitive ;
- même l'ID du firmware n'est pas exposé.

![firmwareTS0004](/assets/img/posts/2026-04-13-zigbee-firmware-relay4ch-module/smart-switch.jpg)

### Conclusion

Ce qu’il faut savoir avant d’acheter un module Zigbee :

1. La référence commerciale ne garantit pas le comportement du module. Deux modules identiques en apparence peuvent embarquer des firmwares différents. Le comportement réel (*clusters* exposés, commandes disponibles) dépend du firmware, pas des références inscrites en façade du boîtier.

2. Le comportement Zigbee ne peut être vérifié qu’après inclusion. C’est uniquement lors de l’interview Zigbee (via Zigbee2MQTT, ZHA, etc.) que l’on découvre le *modelID* réel, le *manufacturerName*, les *clusters* exposés, et les commandes supportées.

3. La documentation communautaire décrit des cas testés, et ne couvre pas toutes les déclinaisons possibles d’un même produit. Le support dépend du firmware embarqué, qui peut varier selon les lots.

4. Les modules Zigbee *low-cost* sont souvent opaques sur leur firmware. Le firmware n’est pas indiqué sur l’emballage ni dans les fiches produit.

Vous êtes prévenu... Avant d’acheter un module Zigbee, il est prudent de vérifier non seulement la référence commerciale, mais aussi les retours d’utilisateurs indiquant le *modelID* et le *manufacturerName* réellement exposés. C'est le firmware qui détermine les fonctionnalités.