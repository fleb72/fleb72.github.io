---
title: "Un environnement Arduino portable pour les établissements scolaires"
date: 2026-06-27
categories: [Arduino]
tags: [ESP32, ArduinoUnoR4WiFi, vscode, "Arduino-CLI", LoRa]
math: true
target_blank: true
---

Dans de nombreux établissements scolaires, l’utilisation d’Arduino en salle informatique se heurte à des contraintes techniques bien connues : postes verrouillés, droits limités, proxys académiques restrictifs, dossiers utilisateurs redirigés sur le réseau pédagogique…
Pour répondre à ces difficultés, j’ai développé un environnement Arduino portable, prêt à l’emploi, fonctionnant sans installation et sans droits administrateur, à condition d’être placé dans un dossier local.

Ce billet présente les raisons de ce projet et met à disposition le pack complet.

### Pourquoi ce projet existe
L’idée de créer un environnement Arduino portable est née d’un constat simple : dans beaucoup de lycées, l’installation classique de l’IDE Arduino ou de VSCode est difficile, voire impossible. Plusieurs facteurs se cumulent :

- postes élèves verrouillés ;
- dossiers utilisateurs redirigés ;
- proxys académiques restrictifs ;
- antivirus / EDR trop stricts ;
- hétérogénéité des postes ;
- etc.

### Objectif du projet
L’objectif est de fournir un environnement :

- stable ;
- identique sur tous les postes ;
- hors‑ligne ;
- sans installation ;
- sans droits admin ;
- prêt à l’emploi.

> le pack doit être placé dans un dossier local (`C:\Arduino-LMS-portable`).  
> Il ne fonctionne pas depuis une clé USB ou un dossier réseau.
{: .prompt-warning }

### Téléchargement et documentation
Le projet complet avec les instructions est disponible sur GitHub :

> [Arduino-LMS-portable](https://github.com/fleb72/Arduino-LMS-portable){:target="_blank"}
{: .prompt-info }

Vous y trouverez :

- le pack portable ;
- les *cores* et bibliothèques intégrés ;
- les explications détaillées ;
- les mises à jour futures.

### Les outils utilisés dans ce pack

Ce projet repose sur deux composants principaux : [VSCode](https://code.visualstudio.com/){:target="_blank"} portable et [Arduino-CLI](https://docs.arduino.cc/arduino-cli/){:target="_blank"}.
Ces outils ont été choisis pour leur fiabilité, leur portabilité et leur capacité à fonctionner dans des environnements verrouillés.

#### VSCode portable : un IDE moderne, sans installation
VSCode est un éditeur de code puissant, extensible et largement utilisé dans l’enseignement technologique.
Dans ce pack, il est fourni en version portable, ce qui permet :

- un fonctionnement sans installation ;
- aucun accès administrateur ;
- aucune modification du système ;
- aucune écriture dans *Program Files* ou *AppData* ;
- une configuration entièrement contenue dans le dossier du pack.


#### arduino-cli : le moteur de compilation
Le cœur du pack est Arduino-CLI, l’outil officiel en ligne de commande développé par Arduino.
Il permet de compiler et téléverser les programmes sans interface graphique, ce qui présente plusieurs avantages :

- fonctionnement hors‑ligne ;
- absence de dépendances dynamiques ;
- gestion propre des cartes et bibliothèques ;
- logs de compilation plus lisibles ;
- meilleure stabilité dans les environnements verrouillés.

#### Pourquoi ce choix technique ?
Ce duo VSCode portable + Arduino-CLI permet :

- une configuration entièrement contenue dans un dossier local ;
- une utilisation sans droits admin ;
- une compatibilité totale avec les postes verrouillés ;
- une compilation fiable, même hors‑ligne ;
- une maintenance simplifiée : mise à jour du pack = remplacement du dossier.

C’est une solution plus robuste que l’IDE Arduino classique, qui dépend fortement du réseau, des proxys, des dossiers *AppData*, et des droits administrateur.

### Conclusion
Cet environnement Arduino portable vise à simplifier l’usage d’Arduino dans les lycées, en contournant les contraintes techniques les plus courantes.
Il permet de démarrer rapidement une séance, sans dépendre des installations locales ou des restrictions réseau.

> **En complément :**
>
> - [Arduino CLI sous Linux : compiler et téléverser en ligne de commande]({{ site.baseurl }}{{ site.url }}/posts/arduino-cli-linux-workflow-git/){:target="_blank"}
{: .prompt-info}