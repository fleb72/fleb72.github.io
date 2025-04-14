---
title: "La pile IOTstack pour Raspberry Pi"
date: 2025-04-14
categories: ['Raspberry Pi', 'domotique']
tags: ['Raspberry Pi', 'IoT', 'domotique', 'Zigbee2MQTT', 'MQTT', 'Zigbee', 'Docker']
math: false
target_blank: true
---

Je poursuis ici [mon projet IoT]({{ site.baseurl }}{{ site.url }}/posts/raspberry-pi-iot-zigbee2mqtt/){:target="_blank"} avec mes capteurs Zigbee et mon dongle USB Zigbee2MQTT.

Le flot des données depuis le capteur Zigbee peut être résumé sur le schéma ci-dessous :

![Flot des données Zigbee](/assets/img/posts/2025-04-14-pile-iotstack/flux-transfo-donnees.png)

À gauche de l’image, un (ou plusieurs) nœuds de capteurs qui émettent des messages au *broker* MQTT.

Dans la pile [IOTstack](https://sensorsiot.github.io/IOTstack/){:target="_blank"}, vous trouverez les logiciels et serveurs nécessaires pour traiter, stocker et présenter les données.
Passons ces outils en revue...

### Broker Mosquitto  {#chapitre-mosquitto}

Voir [IOTstack/Mosquitto](https://sensorsiot.github.io/IOTstack/Containers/Mosquitto/){:target="_blank"}.

MQTT est un protocole léger conçu pour des réseaux à bande passante limitée fonctionne suivant un modèle de publication-abonnement (*publish-suscribe*).
Dans cette architecture MQTT, vous avez les clients qui publient des messages dans une rubrique (un *topic*), par exemple `\home\bedroom` pour les données environnementales de la chambre, et les clients qui s’abonnent au *topic* et recevront les données les concernant. Le dispositif central de l’architecture est le *broker* (courtier) qui relaie les messages publiés aux abonnés.

![Broker MQTT](/assets/img/posts/2025-04-14-pile-iotstack/publish-subscribe-broker-mqtt.png)
*Modèle Publication-Abonnement avec un broker MQTT*

Dans mon cas, le *broker* sera local et embarqué dans la carte Raspberry Pi. Des applications clientes également installées sur la carte pourront s’y connecter pour lire les messages.

### Node-RED  {#chapitre-node-red}

Voir [IOTstack/Node-RED](https://sensorsiot.github.io/IOTstack/Containers/Node-RED/){:target="_blank"}.

Le développement de l’application sur la Raspberry Pi va consister à :
- créer un client MQTT qui va s’abonner au topic des messages délivrés par le nœud de capteurs ;
- traiter, mettre en forme ces données et les insérer dans une base de données.

On pourrait programmer cela avec un langage de programmation, comme Python. Mais pour un développement de prototype, vous pouvez le faire avec très peu de code grâce à [Node-RED](https://nodered.org/){:target="_blank"} qui fait partie des applications retenues dans la pile IOTstack.

Si la pile est démarrée, vous accéderez à Node-RED depuis un navigateur à l’URL `http://localhost:1880` si vous travaillez sur la Raspberry Pi ou `http://<adresse IP de la Raspberry Pi>:1880` d’un poste distant sur le même réseau.

Node-RED est un outil de programmation « visuelle » où le flot des données est dirigé vers des blocs fonctionnels pour traiter, formater les données en fonction des nombreux protocoles du Web.

![Accueil Node-RED](/assets/img/posts/2025-04-14-pile-iotstack/accueil-node-red.png)
*Node-RED dans un navigateur (http://localhost:1880)*

Dans l'interface Web de Node-RED, vous programmez des « flux » à la souris :

![Premier flux Node-RED](/assets/img/posts/2025-04-14-pile-iotstack/flux-mqtt-debug6.png)
*Un premier flux avec un nœud client MQTT pour s'abonner au topic du capteur relié à un nœud Debug pour affichage des données brues dans la console*

Dans la palette des composants de Node-RED vous trouverez une floppée de noeuds pour l'Internet des Objets.

### InfluxDB {#chapitre-influxdb}

Voir [IOTstack/InfluxDB](https://sensorsiot.github.io/IOTstack/Containers/InfluxDB/){:target="_blank"} ou [IOTstack/InfluxDB 2](https://sensorsiot.github.io/IOTstack/Containers/InfluxDB2/){:target="_blank"}.

[InfluxDB](https://www.influxdata.com/products/influxdb-overview/){:target="_blank"} (versions 1.x, 2.x et maintenant en 3.x) est un système de gestion de bases de données spécialisé dans les séries temporelles.

Avec la version 1.8 (un peu ancienne) que j'utilise, on peut manipuler les données avec un langage InfluxQL qui ressemble à du SQL :

```bash
pi@raspberrypi:~ $ docker exec -it influxdb influx -precision rfc3339
Connected to http://localhost:8086 version 1.8.10
InfluxDB shell version: 1.8.10

> show databases
name: databases
name
----
sensors

> use sensors
Using database sensors

> show measurements
name: measurements

name
----
home_measurement

> select time, temperature, humidity from home_measurement
where sensor='SNZB-02P' and location='livingroom' and temperature<17.5
order by time desc limit 10;

name: home_measurement
time                           temperature humidity
----                           ----------- --------
2025-04-10T07:00:04.603777553Z 16.6        45.9
2025-04-10T06:33:19.269946547Z 17.1        45.9
2025-04-10T05:59:55.726420445Z 17.1        45.2
2025-04-02T15:18:52.353334453Z 17.4        46.7
2025-04-02T14:59:49.883111913Z 17.2        46.7
2025-04-02T14:44:25.937167282Z 17.2        47.7
2025-04-02T14:32:15.536493442Z 17.2        46.7
2025-04-02T14:18:43.730871423Z 17.2        47.5
2025-04-02T13:32:06.767120815Z 16.8        47.5
2025-04-02T13:18:34.912019517Z 16.8        48.1
> 
```
Les données insérées sont horodatées automatiquement.

La bonne nouvelle, c'est que la palette des composants de Node-RED comprend aussi des nœuds dédiés pour accéder à une base InfluxDB.

![Flux Node-RED avec nœud InfluxDB](/assets/img/posts/2025-04-14-pile-iotstack/flux-influxdb6.png)
*Flux Node-RED avec un nœud pour insertion de données dans une base InfluxDB*

### Grafana {#chapitre-grafana}

Voir [IOTstack/Grafana](https://sensorsiot.github.io/IOTstack/Containers/Grafana/){:target="_blank"}.

[Grafana](https://grafana.com/grafana/){:target="_blank"} vous permet de créer de véritables tableaux de bord et visualiser vos données sous formes de graphiques de toutes sortes.

Si la pile est démarrée, vous accéderez à Grafana depuis un navigateur à l’URL `http://localhost:3000` si vous êtes sur la Raspberry Pi ou `http://<adresse IP de la Raspberry Pi>:3000` d’un poste distant sur le même réseau.

Grafana permet bien sûr de récupérer vos données grâce à un connecteur InfluxDB :

![Grafana, connecteur InfluxDB](/assets/img/posts/2025-04-14-pile-iotstack/grafana11.jpeg)
*Connexion à une base InfluxDB*

Vous pouvez alors créer votre tableaux de bord avec des graphiques de visualisation :

![Grafana, série temporelles](/assets/img/posts/2025-04-14-pile-iotstack/grafana_time_series.png)
*Graphique des températures, avec la moyenne temporelle sur 1 heure*

![Grafana, tableau de bord](/assets/img/posts/2025-04-14-pile-iotstack/grafana_visualisation.png)
*Tableau de bord avec température et humidité*

### Conclusion

Je vous ai présenté les principaux outils IoT de la pile IOTstack :

- [Mosquitto](#chapitre-mosquitto), broker/client MQTT pour la publication des messages et les abonnements aux *topics* ;
- [Node-RED](#chapitre-node-red) pour la transformation des données ;
- [InfluxDB](#chapitre-influxdb) pour le stockage en local ;
- [Grafana](#chapitre-grafana) pour la visualisation des données.

L’intérêt de déployer les applications dans des conteneurs Docker est que toutes les configurations et dépendances nécessaires au fonctionnement d’une application sont aussi regroupées dans le conteneur. Tout ce qui se passe dans le conteneur n’affectera pas votre système d’exploitation, ce qui permet de créer des environnements de tests idéals. Vous ne risquez pas ainsi de casser des dépendances lors des installations, désinstallations ou mises à jour, et de créer des interactions néfastes au fonctionnement de vos applications (comme des incompatibilités entre versions). Les conteneurs de la pile IOTstack sont aussi configurés pour tourner de concert ([Docker-compose](https://docs.docker.com/compose/){:target="_blank"}) ce qui vous épargnera bien des tracas.

Le projet continue, à suivre...