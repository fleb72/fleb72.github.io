---
title: "Mode d'emploi : module Zigbee CC2530"
date: 2025-07-06
categories: ["Internet des Objets", 'domotique']
tags: ['IoT', 'Zigbee2MQTT', 'MQTT', 'Zigbee', 'Hardware']
math: false
target_blank: true
---
Le module Zigbee CC2530 (avec l’amplificateur CC2591) se présente comme une solution SoC pour les applications sans fil 2,4 GHz 802.15.4, ZigBee et RF4CE. Il offre une portée de communication annoncée pouvant atteindre 800 mètres !! Côté tarif, on le trouve aux environs de 12€ sur Amazon, mais il faut prévoir environ 8€ supplémentaires pour acquérir un firmware stable. En effet, la version gratuite du firmware, bien que disponible, provoque souvent de nombreuses déconnexions d’après mon expérience. Cela dit, elle peut valoir la peine d’être testée selon vos besoins.

![Présentation module Zigbee CC2530+cc2591](/assets/img/posts/2025-07-06-configure-zigbee-cc2530-module/cc2530-cc2591-zigbee-module.jpg)

Ce module, équipé du chipset CC2530 de Texas Instruments (TI) et du firmware associé, permet d’assurer la communication Zigbee en tant que *end device* ou *router* avec un coordinateur. 
Le protocole Zigbee est géré par un ensemble de logiciels appelé Z-Stack, développé par Texas Instruments. Cette « stack » ou pile logicielle fournit tous les outils nécessaires pour permettre au module de communiquer selon la norme Zigbee, facilitant ainsi l’intégration, la gestion des réseaux et la compatibilité avec d’autres appareils Zigbee. La solution testée ici s’avère également compatible avec Zigbee2MQTT :

![CC2530 dans réseau Zigbee](/assets/img/posts/2025-07-06-configure-zigbee-cc2530-module/reseauZigbee.PNG)
*Le module CC2530 avec son firmware compatible avec la solution Zigbee2MQTT, tout à gauche, dans mon réseau Zigbee maillé. Le rond avec l’étoile bleue est le coordinateur.*

> **Liens utiles :**
>
> - [GitHub - RedBearLab/CCLoader](https://github.com/RedBearLab/CCLoader){:target="_blank"} : pour flasher le module zigbee par l’intermédiaire d’une Arduino Uno Rev 3. Il y a d’autres solutions pour flasher ce module, avec une ESP8266 ou Raspberry Pi par exemple.
>
> - [Zigbee Hobbyist. Rock Pi 4 SBC – Zigbee firmwares for routers and DIY devices](https://ptvo.info/){:target="_blank"} : le firmware à flasher. Il y a une version gratuite à tester, on ne sait jamais, mais mon module était instable et se déconnectait souvent. La version Premium (6,95€ HT pour 1 périphérique Zigbee) a résolu le problème chez moi.
>
> - [MinGW - Minimalist GNU for Windows](https://sourceforge.net/projects/mingw/files/MinGW/Base/binutils/){:target="_blank"} : notamment pour l’utilitaire `objcopy` qui permet de convertir les binaires *.hex* en *.bin*.
>
{: .prompt-info }



### Configuration du firmware

Vous n’aurez pas besoin de rentrer dans des lignes de code, le [logiciel de configuration du site ptvo](https://ptvo.info/zigbee-configurable-firmware-features/){:target="_blank"} vous propose de configurer le firmware dans une interface graphique. Je prends un exemple simple :

![firmware configuration ptvo](/assets/img/posts/2025-07-06-configure-zigbee-cc2530-module/ptvo-config.PNG)

Deux sorties numériques sont configurées sur les ports `P1_2` et `P1_6`.
L’entrée numérique 1 sur le port `P1_3` est configurée avec une résistance interne *Pull-down*, et liée à la sortie 1 pour une commande en *On/Off* (*Link to out 1*). Chaque impulsion sur cette entrée fera basculer l’état de la sortie (initialement à l’état bas) comme le ferait un interrupteur à bascule.
L’entrée 2 sur le port `P0_6` est analogique. La valeur brute entre 0 et 2047 (3,3V maxi) issue de la conversion est renvoyée.

**Résumé de la configuration :**
```
Board type: CC2530 + CC2591
Device type: Router
 Transmit enable (TXEN): P12, Receive enable (RXEN): P14
Enable watchdog timer: Yes
Disable resetting of a device by a power on/off cycle: Yes
Set default reporting interval (s): 60

Output pins:
P12: Output 1, GPIO, External pull-up (Role: Generic)
P16: Output 2, GPIO, External pull-up (Role: Generic)

Input pins:
P13: Input 1, GPIO, Pull-down, Link to out 1 (Bind command: On/Off)
P06: Input 2, ADC (raw value, max 3.3V)
```
**Brochage du module CC2530 :**

![Brochage CC2530](/assets/img/posts/2025-07-06-configure-zigbee-cc2530-module/pinout-cc2530.png)

Vous sauvegardez votre configuration pour obtenir un fichier binaire avec l’extension *.hex*. Ce fichier doit être converti en *.bin* avec l’outil `objcopy` et la commande :
```shell
objcopy -I ihex -O binary --gap-fill=0xFF --pad-to=0x040000 mon_firmware.hex mon_firmware.bin
```

- `--gap-fill=0xFF` : remplit les zones non définies avec 0xFF.
- `--pad-to=0x040000` : étend le fichier *.bin* jusqu’à 256 Ko.

### Préparer le loader et flasher le firmware

Normalement, on devrait utiliser une sonde de programmation et de débogage ([CC-DEBUGGER Debug probe - TI.com](https://www.ti.com/tool/CC-DEBUGGER){:target="_blank"}), mais des développeurs ont conçu d’autres solutions à partir d’Arduino, ESP8266 ou Raspberry Pi. La solution avec une Arduino Uno Rev 3 me convenait et a fonctionné chez moi (voir [RedBearLab/CCLoader](https://github.com/RedBearLab/CCLoader){:target="_blank"}).

Le câblage proposé est le suivant (avec un autre module) :

![CCLoader avec Arduino Uno](/assets/img/posts/2025-07-06-configure-zigbee-cc2530-module/ccloader-arduino.jpg)

Sur le module CC2530 :
- `P2_2` : DC (*Debug Clock*)
- `P2_1` : DD (*Debug Data*)
- `RST`: Reset

> Mais attention, notre module Zigbee est alimenté **en 3,3V**.
{: .prompt-warning }

Avec une Arduino Uno Rev 3, il faut obligatoirement un adaptateur/convertisseur 5V-3,3V entre les broches de l’Arduino (D4, D5, D6) et celles du module CC2530 (DD, DC, RESET) :

![Adaptateur 5v-3,3V](/assets/img/posts/2025-07-06-configure-zigbee-cc2530-module/adapt5v-3v.jpg)
*Adaptateur/Convertisseur de niveaux logiques High Voltage (HV) – Low Voltage (LV) bidirectionnel (Level Shifter). Pour l’adaptation 5V (côté Arduino Uno) vers 3,3V (côté module Zigbee CC2530).*

Il reste à téléverser le sketch [CCLoader.ino](https://github.com/RedBearLab/CCLoader/tree/master/Arduino/CCLoader){:target="_blank"} dans l’Arduino.


Puis sur le PC (Windows) en ligne de commande avec l’exécutable fourni *CCLoader_x86_64.exe*, on saisit par exemple dans un terminal :
```shell
CCLoader_x86_64.exe 10 mon_firmware.bin 0
```

- `10` : le numéro de port COM utilisé par l’Arduino sur le PC. À adapter à votre cas.
- `0` : numéro pour désigner l’Arduino Uno.

Si tout se passe bien, la commande doit se terminer par un victorieux *Upload successfully !!*

![](/assets/img/posts/2025-07-06-configure-zigbee-cc2530-module/arduino-ccloader-flash.jpg)

### Test du module

J’alimente mon module CC2530 (en 3,3V) à portée de coordinateur, et j’active l’appairage dans l’interface Zigbee2MQTT, le module finit par être reconnu et appairé :

![Appairage module CC2530](/assets/img/posts/2025-07-06-configure-zigbee-cc2530-module/cc2530-zigbee2mqtt.PNG)

Les entrées-sorties exposées :

![Paramètres exposés](/assets/img/posts/2025-07-06-configure-zigbee-cc2530-module/cc2530-expose.png)

- `l2` : état de l’entrée analogique entre 0 et 2047
- `state_l1` et `state_l2` : les deux sorties numériques, initialement à l’état *OFF*. 

J’ai bricolé un montage à partir de bouton-poussoir, LED et potentiomètre 10k pour faire des tests :


![Montage CC2530](/assets/img/posts/2025-07-06-configure-zigbee-cc2530-module/montage-cc2530.jpg)

On peut publier des messages MQTT pour mettre à l’état haut ou bas les deux sorties numériques :

![Paramètres exposés 2](/assets/img/posts/2025-07-06-configure-zigbee-cc2530-module/cc2530_expose2.PNG)

![Messages MQTT publiés](/assets/img/posts/2025-07-06-configure-zigbee-cc2530-module/publish1-cc2530.png)

J’avais relié l’entrée 1 à la sortie 1 dans la configuration (*link to output 1*). Quand je connecte un bouton-poussoir et une LED au module, chaque appui sur le bouton allume ou éteint la LED comme prévu et un message est publié reflétant le nouvel état.
J’ai relié l’entrée analogique à un potentiomètre, la champ `l2` varie bien entre 0 et 2047, mais la publication de messages ne se fait pas à chaque variation sur l’entrée analogique mais à intervalles réguliers, par exemple toutes les 60s d’après le réglage de ma configuration :


![Reporting interval](/assets/img/posts/2025-07-06-configure-zigbee-cc2530-module/ptvo-config-expert.PNG)

C’est donc à vous de régler l’intervalle de publication selon vos besoins, sans saturer le réseau de messages MQTT.

### Conclusion

Bien que la configuration et l’utilisation des outils de flashage puissent présenter certaines difficultés, cette approche de type *Do It Yourself* offre la possibilité de concevoir un périphérique Zigbee personnalisé, répondant à des besoins spécifiques pour la gestion d’entrées analogiques, de sorties pour LEDs ou relais de puissance, entre autres applications.
