---
title: "Mise à jour du firmware de coordinateur Zigbee"
date: 2025-06-09
categories: ['Raspberry Pi', "Internet des Objets", 'domotique']
tags: ['IoT', 'Zigbee2MQTT', 'MQTT', 'Zigbee', 'Hardware']
math: false
target_blank: true
---

Mon réseau Zigbee fonctionne avec un [coordinateur Zigbee]({{ site.baseurl }}{{ site.url }}/posts/raspberry-pi-iot-zigbee2mqtt/){:target="_blank"}, un dongle USB de la marque Sonoff relié à ma carte Raspberry Pi.

![Zigbee dongle USB](/assets/img/posts/2025-06-09-update-firmware-zigbee-coordinator/sonoff-zigbee-dongle_usb.jpg)

Pour être plus précis, c'est un *Sonoff Zigbee 3.0 USB Dongle Plus (ZB Dongle-P)*, avec le contrôleur TI CC2652P flashé avec le firmware coordinateur Z-Stack. Dans le *frontend* Zigbee2MQTT, vous trouverez la version du firmware dans le menu *À propos* :

![Revision firmware](/assets/img/posts/2025-06-09-update-firmware-zigbee-coordinator/revision-coordinator.jpg)

Ici, j'ai la dernière version à ce jour (20240710), mais à l'achat le firmware en était à une version ancienne qui datait de 2021 ! Et depuis 2021, il y a évidemment eu quelques avancées et des améliorations au niveau de la stabilité du réseau, voir [https://github.com/Koenkk/Z-Stack-firmware/releases](https://github.com/Koenkk/Z-Stack-firmware/releases){:target="_blank"}. S'il se passe des choses bizarres sur votre réseau, une mise à jour est peut-être nécessaire...

En cherchant la documentation sur le site de Sonoff, on tombe sur [cette page](https://dongle.sonoff.tech/sonoff-dongle-flasher/){:target="_blank"} avec le *Sonoff Dongle Flasher*. Et là je me dis que ça va être facile et rapide... Et non !
Il faut quand même installer les pilotes : [Silicon Labs CP2102 Driver](https://www.silabs.com/developer-tools/usb-to-uart-bridge-vcp-drivers?tab=downloads){:target="_blank"}, mais toujours pas... La page se fige et le flashage ne commence même pas, et je n'ai pas d'explication.

### Mise à jour manuelle

Il reste à récupérer le fichier du firmware et à le flasher vous-mêmes dans la puce du coordinateur en passant par un PC (ici sous Windows) et la liaison USB.

Ce dont vous avez besoin :
- Le fichier du firmware à flasher, sur [https://github.com/Koenkk/Z-Stack-firmware/releases](https://github.com/Koenkk/Z-Stack-firmware/releases){:target="_blank"}. Dans mon cas, ce sera le fichier *CC1352P2_CC2652P_other_coordinator_20240710.zip* et en extraire le fichier au format *.hex*.
- Un convertisseur [hex2bin](https://hex2bin.sourceforge.net/){:target="_blank"}, car le flashage avec le fichier *.hex* n'a pas fonctionné chez moi, mais le problème est réglé après conversion en *.bin*. Ne me demandez pas pourquoi...
- Le programme [smartRF Flash Programmer V2](https://www.ti.com/tool/download/FLASH-PROGRAMMER-2/1.8.2){:target="_blank"} chez Texas Instruments.
- Les pilotes pour que le dongle USB soit reconnu sur votre PC : [Silicon Labs CP2102 Driver](https://www.silabs.com/developer-tools/usb-to-uart-bridge-vcp-drivers?tab=downloads){:target="_blank"}.
- Un petit tournevis cruciforme pour extraire la carte du coordinateur de son boîtier. Hé oui, on a besoin d'accéder au bouton [BOOT] de la carte.

![Zigbee Dongle USB démonté](/assets/img/posts/2025-06-09-update-firmware-zigbee-coordinator/zbdongle-usb1.jpg)
*Le ZBdongle USB sorti de son boîtier*

![Zigbee Dongle USB démonté - zoom](/assets/img/posts/2025-06-09-update-firmware-zigbee-coordinator/zbdongle-usb2.jpg)

On voit sur cette dernière photo la puce CC2652P au centre et le driver USB CP2102 à droite. On voit également à droite les deux boutons [RST] et [BOOT].


Une fois que vous avez tous les outils :

- Installez le pilote du driver CP2102 pour que le dongle soit reconnu sur le port USB. Dans le *Gestionnaire de périphériques* de Windows, un nouveau port COM virtuel devrait apparaître. Dans les paramètres du port, réglez la vitesse de transmission série à 115200 bauds (bits par seconde).
![Gestionnaire périphériques port COM](/assets/img/posts/2025-06-09-update-firmware-zigbee-coordinator/gest-periph-cp210x-usb2uart.jpg)

- Utilisez l'utilitaire *hex2bin* pour convertir le fichier *.hex* en *.bin* avec une simple commande `hex2bin CC1352P2_CC2652P_other_coordinator_20240710.hex`.

- Flashez le fichier binaire avec l'utilitaire de Texas Instruments. Pour cela, insérez le dongle USB en maintenant le bouton [BOOT] de la carte appuyé, et maintenez la pression pendant 5s. Le *bootloader* est activé.
Sélectionnez le fichier *.bin* et la cible CC2652P. Appuyez enfin sur le bouton rond avec la flèche de l'utilitaire pour démarrer le processus. Vous devriez avoir quelques secondes après un message de statut indiquant le succès de l'opération.

![TI Flash programmer2](/assets/img/posts/2025-06-09-update-firmware-zigbee-coordinator/ti-flash-programmmer2.jpg)


Votre firmware est à jour.


