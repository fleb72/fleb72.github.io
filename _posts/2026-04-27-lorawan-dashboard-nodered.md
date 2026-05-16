---
title: "LoRaWAN pour les makers – Visualiser ses données dans un tableau de bord Node‑RED"
date: 2026-04-27
categories: ["Internet des Objets"]
tags: [IoT, LoRa, LoRaWAN, MQTT, "Node-RED"]
math: false
target_blank: true
---

Dans un précédent billet (voir [LoRaWAN pour les makers – Mon premier nœud avec Heltec ESP32 + DS18B20 vers The Things Network]({{ site.baseurl }}{{ site.url }}/posts/lorawan-heltec-esp32/){:target="_blank"}), j'avais créé un premier nœud LoRaWAN avec une carte Heltec ESP32 reliée à une sonde de température DS18B20.

Par la suite, j'ai remplacé cette sonde par un capteur SHT31 afin de mesurer à la fois la température et l'humidité (voir [Architecture LoRaWAN Cloud Native expérimentale : collecte, stockage et visualisation des données]({{ site.baseurl }}{{ site.url }}/posts/architecture-lorawan-cloud-native/){:target="_blank"}).

Grâce à la passerelle LoRaWAN publique disponible à quelques kilomètres de [chez moi](https://ttnmapper.org/heatmap/gateway/?gateway=ttnv3-lemans-kerlink-franck&network=NS_TTS_V3://ttn@000013){:target="_blank"}, les *uplinks* de mon capteur sont transmis vers le cloud et apparaissent dans la plateforme [The Things Network](https://www.thethingsnetwork.org/){:target="_blank"}.

![Architecture LoRaWAN](/assets/img/posts/2026-04-27-lorawan-dashboard-nodered/architecture-lorawan-ttn.jpg)

Dans ce premier billet, je m’étais arrêté à la simple observation des données dans la console de [The Things Stack Sandbox](https://www.thethingsindustries.com/docs/concepts/ttn/){:target="_blank"} : un flux JSON brut, très utile pour vérifier que tout fonctionne, mais pas vraiment exploitable au quotidien.

![Console TTN](/assets/img/posts/2026-04-27-lorawan-dashboard-nodered/console-ttn.jpg)
*Affichage des données formatées dans la console TTS*

Alors… que diriez‑vous d’aller plus loin ?

Et si nous affichions ces données dans un **tableau de bord Web** (*dashboard*), avec de **vraies courbes de température et d’humidité**, le tout réalisé **rapidement** et **sans compétences avancées en développement Web** ?

### Node-RED : l'outil idéal pour visualiser les données LoRaWAN

[Node‑RED](https://nodered.org/){:target="_blank"} est un serveur Web qui tourne en tâche de fond sur votre machine (un Raspberry Pi dans mon cas) et qui met à disposition une interface graphique accessible depuis n’importe quel navigateur.

Tout se fait en ligne, directement dans cette interface : vous assemblez des blocs appelés nœuds pour créer des traitements de données, des automatisations ou encore des tableaux de bord Web. Pas besoin de compiler quoi que ce soit ni d’installer un IDE : il suffit d’ouvrir l’URL locale du serveur Node‑RED (`http://[adresse IP du Raspberry Pi]:1880`) pour construire vos *flows* en glissant-déposant les nœuds. Cette approche visuelle en fait un outil idéal pour manipuler des messages JSON, connecter des services IoT et créer rapidement un tableau de bord (ou *dashboard*) personnalisé pour afficher les données de vos capteurs LoRaWAN.

Une fois [Node‑RED installé](https://nodered.org/docs/getting-started/raspberrypi){:target="_blank"} et accessible via son interface Web, il nous reste à répondre à une question essentielle : comment faire arriver les données LoRaWAN dans ce serveur ? 

C’est là qu’interviennent les *intégrations* dans *The Things Stack* (TTS).

### Comment relier The Things Stack à votre serveur : Webhooks ou MQTT ?

Node-RED ne connait pas directement *The Things Stack*. Mais TTS propose plusieurs mécanismes standards d’intégration pour transmettre automatiquement chaque uplink vers un service externe. Les deux plus simples — et les plus courants — sont les *Webhooks* et MQTT.

Un *webhook* est un appel HTTP POST envoyé par *The Things Stack* à chaque uplink. Vous fournissez une URL, et TTS y envoie un JSON complet contenant le payload décodé, les métadonnées radio, le timestamp, etc.

C’est la solution que j’avais utilisée dans mon billet précédent sur l’[architecture LoRaWAN cloud native]({{ site.baseurl }}{{ site.url }}/posts/architecture-lorawan-cloud-native/){:target="_blank"}.

Elle est idéale si vous avez :
- un serveur Web (FastAPI, Flask, Node.js…) ;
- une API maison ;
- un backend qui stocke les données dans une base (InfluxDB, PostgreSQL…) ;
- un environnement cloud (VPS, Kubernetes…).

Le webhook est simple : TTS pousse les données vers vous à chaque uplink.

MQTT fonctionne différemment : à chaque uplink, TTS publie un message MQTT dans un *topic* dirigé vers un broker MQTT, et votre client (activé sur Node‑RED dans mon cas), abonné au *topic*, le reçoit instantanément.
 
MQTT est particulièrement adapté lorsque :
- vous travaillez sur un Raspberry Pi,
- vous utilisez Node‑RED, Home Assistant ou un autre outil domotique,
- vous voulez un flux temps réel,
- vous ne souhaitez pas exposer une API publique sur Internet.

C’est un protocole léger, rapide, pensé pour l’IoT, et parfaitement intégré dans *The Things Stack* qui propose son propre broker MQTT public.

MQTT est idéal pour un serveur local, un Raspberry Pi, ou un dashboard temps réel.

Dans ce billet, nous allons utiliser MQTT, car c’est le moyen le plus simple et le plus naturel pour alimenter un dashboard Node‑RED installé sur mon Raspberry Pi.

Le flux des données de LoRaWAN vers Node-RED est donc le suivant :

```
[Capteur température & humidité]
        ↓
[Carte Heltec ESP32]
        ↓ (radio LoRa)
[Passerelle publique LoRaWAN]
        ↓
[The Things Network / The Things Stack]
        ↓ (publish)
[Broker MQTT public]
↑ (subscribe)  ↓ (publish)
[Node‑RED sur Raspberry Pi]
        ↓
[Dashboard Web : courbes température / humidité]
```

Après avoir sélectionné votre application dans TTS, rendez-vous dans le menu *Other integrations*→MQTT :

![Intégration MQTT](/assets/img/posts/2026-04-27-lorawan-dashboard-nodered/ttn-other-integrations-mqtt.png)
*Intégration MQTT dans The Things Stack*

Puis, générez une nouvelle clé qui vous donnera les accès au broker MQTT.

Voir [la documentation de l'intégration MQTT dans TTS](https://www.thethingsindustries.com/docs/integrations/other-integrations/mqtt/){:target="_blank"} pour plus de détails.

### Construire le flux Node‑RED pour afficher les données LoRaWAN

Une fois l'intégration MQTT configurée dans TTS, il est temps de construire l'application en définissant les traitements de données sous la forme d'un flux Node-RED de manière graphique, en reliant des nœuds entre eux. Chaque nœud joue un rôle précis : communication, transformation, filtrage ou visualisation.  

#### Récupération des données LoRaWAN

Le flux commence par la récupération des données brutes publiées par *The Things Stack*. Pour cela, on dépose simplement un nœud `[mqtt in]` depuis la palette dans l’espace de travail : il s’abonne au topic MQTT et reçoit chaque uplink tel qu’il arrive du broker.
Ces données, encore très verbeuses, sont ensuite envoyées vers un nœud `[function]` chargé d’extraire uniquement les informations utiles pour notre dashboard : la température, l’humidité et l’horodatage.
Pour vérifier que tout fonctionne correctement, des nœuds `[debug]` peuvent être activés ou désactivés d’un clic. Ils affichent les messages dans la console de débogage située dans la barre latérale droite, ce qui permet de suivre en direct le contenu des messages à chaque étape du flux.

Voici une copie d'écran de ce flux Node-RED après renommage des nœuds :

![flux Node-RED MQTT](/assets/img/posts/2026-04-27-lorawan-dashboard-nodered/flux-nodered-1.png)
*Premier flux sous Node-RED*

En double-cliquant sur le nœud `[mqtt in]`, on peut le configurer avec les paramètres fournis par *The Things Stack* : l’adresse du broker, la version du protocole MQTT, le QoS, ainsi que l’identifiant et la clé API générés dans la console TTS (onglet Sécurité). Une fois ces éléments renseignés, il suffit d’indiquer le topic d’uplink (le *sujet* dans Node-RED) auquel Node‑RED doit s’abonner.

![mqtt in propriétés](/assets/img/posts/2026-04-27-lorawan-dashboard-nodered/mqtt-in-broker.png)
*Paramètres du nœud [mqtt in]*

Le nœud `[function]` sert à simplifier le message MQTT reçu. À l’aide de quelques lignes de JavaScript, il extrait uniquement les informations qui nous intéressent — l’horodatage, la température et l’humidité — et reconstruit un `msg.payload` propre, prêt à être affiché dans le dashboard.

![Node-RED function](/assets/img/posts/2026-04-27-lorawan-dashboard-nodered/node-red-function.png)
*Code d'extraction des données utiles*

La console de débogage ci-dessous démontre que les données utiles (*payload*) de température et d'humidité avec l'horodatage à l'heure locale sont bien extraits des données LoRaWAN :

![Node-RED Debug](/assets/img/posts/2026-04-27-lorawan-dashboard-nodered/node-red-debug.png)
*Console de débogage*

#### Construction du tableau de bord

Pour construire le tableau de bord, Node‑RED met à disposition une série de *widgets* prêts à l’emploi dans la palette des nœuds. Ils permettent d’afficher directement les données extraites du flux sans écrire de code supplémentaire.
Dans notre cas, nous allons utiliser trois types de widgets :
- un widget *text* pour afficher la date et l’heure du dernier relevé LoRaWAN ;
- deux widgets *gauge* pour visualiser en un coup d’œil la température et l’humidité ;
- deux widgets *chart* pour tracer l’évolution de ces deux mesures sur les dernières heures (mon capteur LoRaWAN lance des mesures toutes les douze minutes).

Ces éléments constituent la base du dashboard : un affichage simple, lisible, et mis à jour automatiquement à chaque uplink.

![Dashboard Node-RED](/assets/img/posts/2026-04-27-lorawan-dashboard-nodered/dashboard-node-red.png)
*Widgets pour tableaux de bord prêts à l'emploi*

Le flux Node-RED complété avec les widgets du tableau de bord devient ceci :

![Flux Node-RED final](/assets/img/posts/2026-04-27-lorawan-dashboard-nodered/flow-node-red-final.png)

Les widgets sont rassemblés dans un même *Group*, comme on peut le voir dans l’onglet *Layout*, à droite. En cliquant sur le bouton [layout], on organise librement la disposition et la taille de chaque widget à la souris dans l'éditeur :

![Node-RED layout editor](/assets/img/posts/2026-04-27-lorawan-dashboard-nodered/dashboard-layout-editor.png)
*Dashboard layout editor*

Une fois la mise en page définie, chaque widget est relié à son flux de données pour afficher les valeurs en temps réel. Les nœuds intermédiaires [change] (en jaune) servent à isoler chaque valeur du message — horodatage, température, humidité — et à l’acheminer vers le widget correspondant. Chaque élément du tableau de bord reçoit ainsi exactement la donnée dont il a besoin.

Voici, par exemple, la configuration d’un nœud [change] renommé [get temperature] pour extraire la valeur de température :

![Node-RED change](/assets/img/posts/2026-04-27-lorawan-dashboard-nodered/node-red-change-temperature.png)

En double‑cliquant sur un widget, on accède à ses propriétés pour en ajuster le comportement et l’apparence.

![Gauge Node](/assets/img/posts/2026-04-27-lorawan-dashboard-nodered/node-red-gauge.png)
*Paramétrage du nœud Gauge*

![Chart Node](/assets/img/posts/2026-04-27-lorawan-dashboard-nodered/node-red-chart.png)
*Paramétrage du nœud Chart*

Après avoir configuré le flux, cliquez sur [Déployer] pour appliquer les changements. Le tableau de bord peut ensuite être consulté à l’adresse `http://[adresse IP du Raspberry Pi]:1880/ui`.

![Dashboard final](/assets/img/posts/2026-04-27-lorawan-dashboard-nodered/dashboard-nodered-lorattn.png)

### Conclusion

Avec ce tableau de bord Node‑RED, nous disposons désormais d’une visualisation claire et immédiate des données LoRaWAN issues de *The Things Stack*. C’est une première brique fonctionnelle, simple à mettre en place et suffisamment flexible pour évoluer au fil des besoins.


> **En complément :**
>
> - [Node‑RED : envoyer une notification de température sur smartphone avec ntfy]({{ site.baseurl }}{{ site.url }}/posts/nodered-ntfy/){:target="_blank"}
{: .prompt-info}