---
title: "Architecture LoRaWAN Cloud Native expérimentale : collecte, stockage et visualisation des données"
date: 2025-08-19
categories: ["Internet des Objets"]
tags: [IoT, LoRa, LoRaWAN, ESP32, Python, Linux]
math: true
target_blank: true
---

Ce billet fait suite à mes premières expérimentations de communication avec une plateforme LoRaWAN : [LoRaWAN pour les makers – Mon premier nœud avec Heltec ESP32 + DS18B20 vers The Things Network]({{ site.baseurl }}{{ site.url }}/posts/lorawan-heltec-esp32/){:target="_blank"}.

Depuis, j'ai fait évoluer mon système en troquant le DS18B20 pour une sonde de température et d'humidité à base de SHT31 (communication I2C), mais j'en étais resté à constater la transmission des données dans la console de la plateforme [The Things Network](https://console.cloud.thethings.network/){:target="_blank"} :

![Architecture LoRaWAN - TTN](/assets/img/posts/2025-08-19-architecture-lorawan-cloud-native/architecture-lorawan-ttn.jpg)

![Console TTN](/assets/img/posts/2025-08-19-architecture-lorawan-cloud-native/console-ttn.jpg)
*Affichage des données formatées dans la console TTN*

Un *payload* formaté comme `{humidity: 50.6, temperature: 28.79}`est bien restitué dans la console toutes les 12 minutes conformément à la programmation de la carte Heltec LoRa 32 que j'utilise comme nœud de transmission LoRa.

Il s'agit maintenant d'exploiter ces données brutes publiées dans le Cloud et de programmer quelques services Cloud pour l'utilisateur...

Typiquement, le cas d'usage est le suivant :
- supervision environnementale (météo locale avec nœuds de capteurs LoRa), ou plus généralement une plateforme de test pour projets IoT ;
- serveur d’API pour applications web ou mobiles ;
- visualisation et analyse de données en temps réel.

### Le serveur d'applications et les services

L'architecture que je déploie est donc celle du schéma ci-dessous :

![Architecture générale](/assets/img/posts/2025-08-19-architecture-lorawan-cloud-native/architecture.jpg)

Le rectangle gris peut être une carte Raspberry Pi ou un VPS.

Si votre serveur est exposé sur Internet, la sécurité est essentielle :
- accès SSH avec clés ; 
- services exposés derrière un *reverse proxy* (comme Nginx) avec HTTPS ;
- authentification sur InfluxDB (base de données pour séries temporelles) et autres services ;
- Fail2ban pour surveiller les tentatives de connexion et les accès non autorisés.

Dans mon cas, voici les services proposés :

- [InfluxDB v2](https://docs.influxdata.com/influxdb/v2/){:target="_blank"}, un système de gestion de base de données spécialisé dans les séries temporelles. On peut y accéder via une interface Web (bien entendu, en mode administrateur uniquement), et même si ce n'est pas sa spécialité, les données peuvent être visualisées sur des graphiques.
![InfluxDB v2](/assets/img/posts/2025-08-19-architecture-lorawan-cloud-native/data-explorer-influxdb.jpg)
*Data Explorer dans InfluxDB v2*

- une API (programme Python `webhook.py` développé avec [FastAPI](https://fastapi.tiangolo.com/){:target="_blank"}) qui reçoit les données LoRaWAN via un *webhook*, puis les insère dans InfluxDB ;
![Webhook TTN](/assets/img/posts/2025-08-19-architecture-lorawan-cloud-native/webhook.jpg)
*Activation d'un Webhook dans la console TTN*

- Les données stockées dans InfluxDB sont soumises à une politique de rétention limitée volontairement (6 mois, 1 an… à adapter selon le volume). Une seconde API dédiée à l’archivage (programme Python `archivage_s3.py`), exécutée à intervalles réguliers via un processus `cron` (quotidien ou hebdomadaire), extrait les dernières données et les sauvegarde sur un dépôt distant (*Object storage* compatible S3 par exemple) dans un format standard tel que JSON.

- Enfin, une interface publique, sous la forme d’une page Web simple, est proposée à l’utilisateur. Elle repose sur le programme Python `main.py`, développé avec FastAPI, qui sert les fichiers HTML/CSS/JS et permet de visualiser les données en direct dans un navigateur.


> - Les programmes du projet sont sur un dépôt Github : [lorawan-playground](https://github.com/fleb72/lorawan-playground){:target="_blank"}
> - La page Web publique est à l'adresse : `https://iot[dot]techfleb[dot]fr/latest?access=pjLW08`. Cette page ne dépose aucun cookie, ne collecte aucune donnée personnelle, et ne réalise aucun suivi utilisateur.  
{: .prompt-info }

> **En complément :**
>
> - [LoRaWAN pour les makers – Visualiser ses données dans un tableau de bord Node‑RED]({{ site.baseurl }}{{ site.url }}/posts/lorawan-dashboard-nodered/){:target="_blank"})
{: .prompt-info}







