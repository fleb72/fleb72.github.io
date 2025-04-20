---
title: "Une extension Visual Studio Code officielle pour programmer votre Raspberry Pi Pico"
date: 2025-04-20
categories: ['Raspberry Pi', 'Raspberry Pi Pico']
tags: ['Raspberry Pi', 'Raspberry Pi Pico']
math: false
target_blank: true
---

Il y a maintenant une extension officielle VSCode pour développer sur la carte à microcontrôleur Raspberry Pi Pico. Voir  [Github - The official VS Code extension for Raspberry Pi Pico development](https://github.com/raspberrypi/pico-vscode){:target="_blank"}.
L'extension est encore en phase de développement, mais on espère gagner en confort avec la préparation de l'environnement de développement et quelques icônes pour compiler, flasher, exécuter, déboguer, etc. aussi bien en C/C++ ([SDK C/C++](https://www.raspberrypi.com/documentation/microcontrollers/c_sdk.html#raspberry-pi-pico-cc-sdk){:target="_blank"}), mais aussi en [MicroPython](https://www.raspberrypi.com/documentation/microcontrollers/micropython.html){:target="_blank"}.

### Installation de l'extension

Je tente donc l'expérience en installant cette nouvelle extension dans VSCode :

![Pi Pico VSCode extension](/assets/img/posts/2025-04-20-rpi-pico-vscode-extension/rpi-pico-official-vscode-extension.png)

Une nouvelle icône **Raspberry Pi Pico Project** apparait tout à gauche avec des raccourcis vers les commandes principales. Je commence évidemment avec un *blink* en C depuis le menu **New Project From Example** :

![Pi Pico, projet blink](/assets/img/posts/2025-04-20-rpi-pico-vscode-extension/pipico-blink-project.png)

### Compilation / Run

Je compile le projet (**Compile Project**) avec succès. Tous les fichiers du projet sont dans un dossier *blink*. Je relie la Pico en USB sur mon PC tout en appuyant sur le bouton *BOOTSEL*, et je flashe le binaire du projet (**Run Project USB**) :

![Run Blink Project USB](/assets/img/posts/2025-04-20-rpi-pico-vscode-extension/pipico-blink-run-usb.png)

Test réussi ! La LED intégrée clignote...

### Sonde de programmation avec une seconde carte Pi Pico

J'essaie maintenant de flasher la Pico par l'intermédiaire d'une sonde qui servira aussi pour le débogage (la [sonde du pauvre](https://www.raspberrypi.com/products/debug-probe/){:target="_blank"}, constituée par une deuxième Raspberry Pi Pico avec le firmware *debugprobe*)). Avec cette sonde, je n'ai plus besoin de faire un *Reset* ou de débrancher/rebrancher le câble USB pour flasher à nouveau la carte. Il faut cette fois passer par le menu **Flash Project SWD**, et... cela fonctionne aussi !

![Pico sonde debug](/assets/img/posts/2025-04-20-rpi-pico-vscode-extension/pico-sonde-debug.jpg)
*À gauche, la sonde de programmation/débogage. À droite la Raspberry Pi Pico cible.*

### Débogage

Je tente maintenant une session de débogage avec ma sonde (menu **Debug Project**) :

![Pico test Debug](/assets/img/posts/2025-04-20-rpi-pico-vscode-extension/pico-test-debug.png)
*Session de débogage*

### Et en Micropython...

Enfin, je fais un dernier essai *blink* en MicroPython (menu **New MicroPython Project**). L'outil me propose de flasher le firmware MicroPython dans un premier temps, très bien. Je reconnecte la carte, et l'interpréteur MicroPython est bien reconnu. J'exécute le programme *blink.py* :

![Pi Pico MicroPython test](/assets/img/posts/2025-04-20-rpi-pico-vscode-extension/pipico-micropython.png)

Et ça se met à clignoter... Jusqu'à ce que j'interromps le programme.

### Documentation intégrée

La documentation de l'API C/C++ est aussi présente dans l'interface (les différents menus **Documentation**) :

![Documentation API C/C++](/assets/img/posts/2025-04-20-rpi-pico-vscode-extension/pico-doc-api.png)

### Conclusion

Je n'ai fait que des tests basiques pour l'instant, mais voilà une extension prometteuse qui va faire gagner beaucoup de temps... Affaire à suivre.

### À lire :

- [Boost Your Pico Projects with the new Pico VS Code Extension](https://www.raspberrypi.com/news/pico-vscode-extension/){:target="_blank"}
- [Github - The official VS Code extension for Raspberry Pi Pico development](https://github.com/raspberrypi/pico-vscode){:target="_blank"}
- [Getting started with Raspberry Pi Pico-series](https://www.raspberrypi.com/documentation/microcontrollers/c_sdk.html#raspberry-pi-pico-cc-sdk){:target="_blank"} (mis à jour avec l'utilisation de l'extension VSCode)