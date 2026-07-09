---
title: "Arduino CLI sous Linux : compiler et téléverser en ligne de commande"
date: 2026-07-09
categories: [Arduino]
tags: [ESP32, git, "Arduino-CLI"]
math: true
target_blank: true
---

Si vous programmez des cartes dans l’environnement Arduino (Arduino Uno, Nano, Mega, Uno R4 Minima/WiFi, mais aussi ESP8266/ESP32, RP2040, RP2350, etc.), vous êtes sans doute passés par l’IDE Arduino officiel (1.8.*x* ou 2.*x*). C’est l’outil le plus répandu pour débuter : il permet de compiler et téléverser en un seul clic, sans avoir à se soucier de la chaîne de compilation ou des outils sous‑jacents.

L’IDE a aussi un avantage important : il unifie les pratiques. Le même bouton *Build* ou *Upload* fonctionne pour toutes les cartes, alors que les outils utilisés en arrière‑plan sont totalement différents selon la famille. Une Arduino Uno ne se compile pas avec les mêmes toolchains qu’un ESP32, et un RP2040 utilise encore une autre chaîne. L’IDE masque cette complexité et offre une expérience homogène, ce qui est idéal pour débuter.

Arduino‑CLI, développé indépendamment des IDE 1.8.*x* et 2.*x*, est l’outil officiel pour compiler et téléverser des sketches dans l’environnement Arduino, mais sans interface graphique : tout se fait en ligne de commande. Comme l’IDE, il propose des commandes unifiées (`compile`, `upload`) qui fonctionnent pour toutes les familles de cartes, même si les toolchains utilisées en arrière‑plan diffèrent selon le modèle. La différence, c’est que la CLI (*Command Line Interface*) rend ces opérations plus explicites, scriptables et intégrables dans des workflows automatisés.

Je propose dans ce billet d’installer et de découvrir cette interface en ligne de commande et ses avantages. L’objectif est de comprendre comment compiler et téléverser des sketches sans passer par l’IDE Arduino, en utilisant uniquement le terminal. J’ai testé l’installation d’Arduino‑CLI sur une Raspberry Pi avec le *Raspberry Pi OS Lite* officiel, mais les commandes présentées ici fonctionnent de la même manière sur n’importe quelle distribution Linux (Ubuntu, Debian, Mint, etc.).

### Installation d’Arduino CLI

La méthode d'installation standard, et mise en avant par la documentation, fonctionne sur toutes les distributions Linux.

On commence par télécharger et exécuter le script d'installation :
```bash
curl -fsSL https://raw.githubusercontent.com/arduino/arduino-cli/master/install.sh | sh
```

Le script détecte l'architecture (ARM, x86_64...), télécharge la bonne version d'Arduino CLI et extrait le binaire dans un dossier `bin/` local.

Pour pouvoir utiliser `arduino-cli` depuis n'importe quel emplacement, il est conseillé de déplacer ce binaire dans `/usr/local/bin` :
```bash
sudo mv bin/arduino-cli /usr/local/bin/
```

Vous pouvez vérifier votre installation avec la commande : 
```bash
arduino-cli version
```
Si tout s’est bien passé, la commande affiche la version installée, par exemple :
```
arduino-cli  Version: 1.5.1 Commit: 01f3d4f2b Date: 2026-06-05T10:22:11Z
```

> Pour plus de détails sur l'installation, voir [https://docs.arduino.cc/arduino-cli/installation/](https://docs.arduino.cc/arduino-cli/installation/){:target="_blank"}
{: .prompt-info }

### Configuration initiale

En plus du binaire `arduino-cli`, un dossier caché `~/.arduino15/` est créé automatiquement. Il sert à stocker les fichiers temporaires, les *cores* installés, les bibliothèques, les index, ainsi que divers éléments internes nécessaires au fonctionnement de l’outil.

La documentation recommande de générer un fichier de configuration minimal avec la commande :

```bash
arduino-cli config init
```
Cette commande crée un fichier `~/.arduino15/arduino-cli.yaml`. Arduino‑CLI peut fonctionner sans fichier de configuration, mais dès que l’on souhaite installer des *cores* supplémentaires (ESP32, RP2040, etc.) ou personnaliser les chemins, ce fichier devient indispensable.

Cette commande crée un fichier `~/.arduino15/arduino-cli.yaml` minimal. 
```
board_manager:
    additional_urls: []
```

Ce fichier peut rester tel quel, et l'outil fonctionnera avec la configuration par défaut, mais il permet aussi de personnaliser certains aspects de l’outil, comme l’ajout d’URLs supplémentaires pour installer des *cores* non officiels, ou la modification des chemins utilisés par Arduino‑CLI. 


> Pour plus de détails sur les options de configuration disponibles, la documentation officielle propose une description complète : [https://docs.arduino.cc/arduino-cli/configuration/](https://docs.arduino.cc/arduino-cli/configuration/){:target="_blank"}
{: .prompt-info }


### Installation des cores

Un *core Arduino* est l’ensemble des fichiers nécessaires pour que l’IDE ou Arduino‑CLI puisse compiler et téléverser un *sketch* pour une famille de cartes donnée.
Il contient la chaîne de compilation adaptée au microcontrôleur, les définitions de cartes, les scripts de build, ainsi que les bibliothèques système de bas niveau.
Chaque famille de cartes possède son propre core : les cartes AVR (Uno, Nano), les ESP32/ESP8266, les RP2040, les cartes ARM, etc.

L’Arduino CLI fraîchement installé ne contient encore aucun *core*. Sans *core*, il est impossible de compiler ou téléverser un sketch, car l’outil ne sait pas quelles définitions de cartes, quelles bibliothèques système ou quelle chaîne de compilation utiliser.

Nous allons commencer par installer le *core* officiel destiné aux cartes Arduino basées sur des microcontrôleurs AVR : Arduino Uno, Nano et Mega. Ce core porte le nom `arduino:avr` :

```bash
arduino-cli core install arduino:avr
```
Après quelques instants, le temps de télécharger et d’installer tous les outils nécessaires, vous devriez voir un message de confirmation indiquant que le *core* a bien été installé :

```
...
Platform arduino:avr@1.8.8 installed
```

On peut vérifier que le *core* est bien installé en affichant la liste des plateformes disponibles avec la commande :

```bash
arduino-cli core list
```
Vous devriez y voir apparaître la ligne :

```
ID          Installed Latest Name
arduino:avr 1.8.8     1.8.8  Arduino AVR Boards
```
Ce qui confirme que le core AVR est actif et prêt à être utilisé pour compiler des sketches destinés aux cartes Arduino Uno, Nano ou Mega.

Certaines familles de cartes, comme les ESP32 d’Espressif, ne sont pas incluses dans les sources officielles d’Arduino. Pour installer ces *cores*, il faut d’abord ajouter l’URL fournie par le fabricant, afin qu’Arduino CLI puisse télécharger l’index correspondant.
Par exemple, pour les cartes ESP32, on peut ajouter l’URL Espressif directement en ligne de commande :

```bash
arduino-cli core update-index --additional-urls https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
```
Une fois l’index mis à jour, l’installation du *core* se fait comme pour les autres plateformes :
```bash
arduino-cli core install esp32:esp32
```
Ce *core* permet de compiler pour la plupart des cartes ESP32 (DevKitC, WROOM, WROVER, NodeMCU‑32S, etc.).
Attention, le *core* Espressif ESP32 est un *core* monolithique qui inclut les toolchains pour plusieurs familles de microcontrôleurs (ESP32, ESP32‑S2, ESP32‑S3, ESP32‑C3, etc.). Son installation peut occuper plusieurs gigaoctets. Ce comportement est normal, car Espressif regroupe toutes les variantes dans un seul *core*.


Sur Linux (et Raspberry Pi OS), Arduino CLI installe tous les *cores* dans : `~/.arduino15/packages/`


### Compiler un sketch

Pour tester la compilation, on part d’un sketch minimal `blink.ino` placé dans un sous‑dossier `blink/` :

```cpp
void setup() {
  pinMode(LED_BUILTIN, OUTPUT);
}

void loop() {
  digitalWrite(LED_BUILTIN, HIGH);
  delay(500);
  digitalWrite(LED_BUILTIN, LOW);
  delay(500);
}
```

La commande de compilation s’exécute depuis le dossier parent :
```bash
arduino-cli compile --fqbn arduino:avr:uno blink
```
Le paramètre `--fqbn` (*Fully Qualified Board Name*) indique la carte cible.
Ici, `arduino:avr:uno` correspond au core `arduino:avr` et à la carte Arduino Uno.

Si tout se passe bien, la compilation affiche un résumé du binaire généré :

```
Sketch uses 924 bytes (2%) of program storage space. Maximum is 32256 bytes.
Global variables use 9 bytes (0%) of dynamic memory, leaving 2039 bytes for local variables. Maximum is 2048 bytes.
```
La commande `arduino-cli compile` accepte plusieurs options pratiques :

- `--verbose`
  Affiche les détails de la compilation : commandes `avr-gcc`, chemins, options, etc.
  Utile pour comprendre ce qui se passe ou diagnostiquer un problème.

- `--warnings <level>`
  Niveau d’avertissements du compilateur : `none`, `default`, `more`, `all`.
  Exemple : `--warnings all` pour activer tous les warnings GCC.

- `--clean`
  Supprime le dossier de build avant de compiler.
  Utile si vous changez de carte ou de core.

- `--build-path <dossier>`
  Permet de choisir où générer les fichiers de compilation.
  Par défaut : `blink/build/arduino.avr.uno/`.

- `--libraries <dossier>`
  Ajoute un dossier de bibliothèques personnalisé à la compilation.

- `--export-binaries`
  Copie les fichiers `.hex` ou `.bin` dans le dossier du sketch après compilation.

Exemple de comande de compilation complète, à lancer depuis un dossier parent :
```bash
arduino-cli compile \
  --fqbn arduino:avr:uno \
  --verbose \
  --warnings all \
  --build-path blink/build \
  blink
```
Le dossier du *build* est alors généré dans `blink/build/arduino.avr.uno/`

### Téléverser un sketch
Une fois votre carte Arduino Uno reliée au port USB de la Raspberry Pi, vous pouvez lister les ports COM/tty détectés avec la commande :
```bash
arduino-cli board list
```

On obtient par exemple : 
```
Port         Protocol Type              Board Name  FQBN            Core
/dev/ttyACM0 serial   Serial Port (USB) Arduino UNO arduino:avr:uno arduino:avr
```
L'Arduino Uno est ici détectée sur le port série `/dev/ttyACM0/`.

On peut maintenant téléverser dans la carte Arduino avec la commande suivante (en mode verbeux) :
```bash
arduino-cli upload -p /dev/ttyACM0 --fqbn arduino:avr:uno --verbose blink
```

Si tout se passe bien, le message se termine par :
```
...
Reading 924 bytes for flash from input file blink.ino.hex
in 1 section [0, 0x39b]: 8 pages and 100 pad bytes
Writing 924 bytes to flash
Writing | ################################################## | 100% 0.21s
924 bytes of flash written
Avrdude done.  Thank you.
New upload port: /dev/ttyACM0 (serial)
```
La Led intégrée à la carte devrait se mettre à clignoter.

### Installer une bibliothèque

L’environnement Arduino, c’est aussi une galaxie de bibliothèques, officielles ou tierces, qui ajoutent des fonctionnalités prêtes à l’emploi : gestion de capteurs, modules de communication, afficheurs, actionneurs, protocoles, traitements numériques, etc.

Ces bibliothèques permettent d’étendre facilement les capacités d’un sketch sans réinventer la roue.

Arduino CLI permet de rechercher une bibliothèque dans l'index officiel :
```bash
arduino-cli lib search servo
```
Et il existe une pléthore de bibliothèques disponibles pour piloter un servomoteur. Je ne mets ici que la première détectée (bibliothèque Servo officielle) :

```
Downloading index: library_index.tar.bz2 downloaded                                  
Name: "Servo"
  Author: Michael Margolis, Arduino
  Maintainer: Arduino <info@arduino.cc>
  Sentence: Allows Arduino boards to control a variety of servo motors.
  Paragraph: This library can control a great number of servos.<br />It makes careful use of timers: the library can control 12 servos using only 1 timer.<br />On the Arduino Due you can control up to 60 servos.
  Website: https://www.arduino.cc/reference/en/libraries/servo/
  Category: Device Control
  Architecture: avr, megaavr, sam, samd, nrf52, stm32f4, mbed, mbed_nano, mbed_portenta, mbed_rp2040, renesas, renesas_portenta, renesas_uno, zephyr
  Types: Arduino
  Versions: [1.0.0, 1.0.1, 1.0.2, 1.0.3, 1.1.0, 1.1.1, 1.1.2, 1.1.3, 1.1.4, 1.1.5, 1.1.6, 1.1.7, 1.1.8, 1.2.0, 1.2.1, 1.2.2, 1.3.0]
  ...
```
Le nom de la bibliothèque officielle est `Servo` (avec un 's' majuscule), et on l'installe avec la commande :

```bash
arduino-cli lib install Servo
```

Avec la sortie :
```
Downloading Servo@1.3.0...
Servo@1.3.0 downloaded                                                               
Installing Servo@1.3.0...
Installed Servo@1.3.0
```
La version la plus récente 1.3.0 est installée par défaut dans le dossier `~/Arduino/libraries/`.

Vous pouvez lister les bibliothèques installées avec la commande :
```bash
arduino-cli lib list
```
Avec en sortie :
```
Name  Installed     Available         Location Description
Servo 1.3.0         -                 user     -
```

Les bibliothèques sont souvent accompagnées d'exemples prêts à l'emploi.
```bash
arduino-cli lib examples Servo
```
```
Examples for library Servo
  - /home/fleb/Arduino/libraries/Servo/examples/Knob
  - /home/fleb/Arduino/libraries/Servo/examples/Sweep
```

### Intégration dans un script

Arduino-CLI se prête très bien à l'automatisation. On peut l'intégrer dans un script pour compiler, téléverser ou même injecter des paramètres avant la compilation.

**Exemple : module ESP32 configuré en serveur Web avec SSID et mot de passe externes**
On suppose que ce projet est versionné sur GitHub. Le code source est donc public ou partagé, tandis que les identifiants WiFi restent locaux et ne sont jamais commités.

Dans cet exemple, un script Python lit un fichier `.env` contenant les identifiants WiFi, génère automatiquement un fichier `config.h`, puis lance la compilation et le téléversement.

**Fichier** `server/.env`
```
SSID=xxxxxxxxxxxxx
PASSWORD=yyyyyyyyyyyyy
```

Ce fichier caché ainsi que le fichier `config.h` généré en Python seront dans la liste `.gitignore` afin de ne pas exposer les identifiants dans votre dépôt Github.

**Sketch Arduino** `server/server.ino`
```cpp
#include <WiFi.h>
#include "config.h"

WiFiServer server(80);

void setup() {
  Serial.begin(115200);

  WiFi.begin(ssid, password);
  Serial.print("Connexion au WiFi");

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println("\nConnecté !");
  Serial.print("IP : ");
  Serial.println(WiFi.localIP());

  server.begin();
}

void loop() {
  WiFiClient client = server.available();
  if (!client) return;

  client.println("HTTP/1.1 200 OK");
  client.println("Content-Type: text/plain");
  client.println();
  client.println("Hello World!");
  client.stop();
}
```

**Script Python** `build.py`
```python
#!/usr/bin/env python3
import subprocess
import sys
from dotenv import dotenv_values # pip install dotenv

SKETCH = "server"
FQBN = "esp32:esp32:esp32"
PORT = "/dev/ttyUSB0"

# Chargement du fichier .env
env = dotenv_values(f"{SKETCH}/.env")
SSID = env.get("SSID")
PASSWORD = env.get("PASSWORD")

if not SSID or not PASSWORD:
    print("Erreur : SSID ou PASS manquant dans .env")
    sys.exit(1)

def run(cmd):
    print(f"---> {cmd}")
    result = subprocess.run(cmd, shell=True)
    if result.returncode != 0:
        print("Erreur : commande échouée.")
        sys.exit(1)

# Génération du fichier config.h
config = f'''
#pragma once
const char* ssid = "{SSID}";
const char* password = "{PASSWORD}";
'''

with open(f"{SKETCH}/config.h", "w") as f:
    f.write(config)

print("config.h généré")

# Compilation
run(f"arduino-cli compile --fqbn {FQBN} {SKETCH}")

# Téléversement
run(f"arduino-cli upload --fqbn {FQBN} -p {PORT} {SKETCH}")

print("Téléversement terminé")
```

**Résultat :** une fois le script `build.py` exécuté, l’ESP32 démarre un petit serveur Web accessible à l’adresse IP attribuée par votre Box/routeur. La page renvoie simplement un "Hello World!".

Ce workflow est intéressant parce qu’il sépare clairement le code versionné et la configuration locale.
Le dépôt Git ne contient que le sketch et le script Python : tout ce qui est sensible (`.env`, `config.h`) est généré ou stocké localement et exclu du dépôt.
Dans un projet hébergé sur GitHub, cela évite toute fuite accidentelle d’identifiants et permet à chaque utilisateur de cloner le dépôt puis d’ajouter ses propres paramètres sans modifier le code source.
On obtient ainsi un processus reproductible, sécurisé et évolutif : le code est partagé via Git, tandis que les identifiants restent sur la machine de l’utilisateur.
Le script automatise ensuite la génération de `config.h` et le téléversement, ce qui simplifie les mises à jour et évite d’avoir à éditer les identifiants ou lancer les commandes à la main.


### Conclusion

Arduino CLI offre une manière plus flexible et plus moderne de travailler avec les cartes compatibles avec l'environnement Arduino.
Ce billet a montré comment compiler et téléverser en ligne de commande, puis comment intégrer ces actions dans un workflow Git pour rendre le développement plus fiable et automatisable.
Cette approche constitue la base de mon projet [Arduino LMS portable]({{ site.baseurl }}{{ site.url }}/posts/arduino-cli-vscode-portable/){:target="_blank"}, où l’intégration d’Arduino CLI dans VS Code permet d’obtenir un environnement complet, portable et prêt à l’emploi.

