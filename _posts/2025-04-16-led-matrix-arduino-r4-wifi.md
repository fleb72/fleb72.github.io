---
title: "Arduino Uno R4 WiFi et la matrice de LEDs"
date: 2025-04-16
categories: [Arduino]
tags: [Arduino, ArduinoUnoR4WiFi]
math: true
target_blank: true
---

L'Arduino Uno R4 WiFi intègre une matrice de 96 LEDs :

![La matrice de LEDS](/assets/img/posts/2025-04-16-led-matrix-arduino-r4-wifi/unoR4wifi-annonce2.jpg)
*Arduino Uno R4 WiFi est sa matrice de 96 LEDs rouges*

En fouillant dans la [documentation](https://docs.arduino.cc/hardware/uno-r4-wifi/){:target="_blank"}, on en apprend davantage sur cette matrice de LEDs.
Elle est constituée par 12 x 8 LEDs rouges, soit 96 LEDs en tout. Dans les schémas fournis par Arduino (Licence CC-BY-SA), on découvre que les LEDs sont câblées d'une façon particulière pour être pilotées par une technique unique de multiplexage appelée « [Charlieplexing](https://en.wikipedia.org/wiki/Charlieplexing){:target="_blank"} » :


![Charlieplexing](/assets/img/posts/2025-04-16-led-matrix-arduino-r4-wifi/charlieplexing1.png)
*Câblage des 96 LEDs de la matrice pour pilotage par la technique du Charlieplexing*

Le but de cette technique est d'économiser des broches du microcontrôleur, seules 11 broches suffisent pour piloter les 96 LEDs, et on pourrait même en piloter davantage. (Avec `n` broches, on pourrait théoriquement piloter jusqu'à `n x (n-1)` broches avec cette technique).

Si je fais un zoom sur quelques LEDs, pour mieux comprendre comment allumer une LED en particulier :

![Détail Charlieplexing](/assets/img/posts/2025-04-16-led-matrix-arduino-r4-wifi/charlieplexing2-detail.png)
*Comment allumer la LED n°5 ?*

Si on veut allumer la 5è LED sur la 1ère ligne de la matrice (en surbrillance sur le schéma ci-dessus), et avoir toutes les autres LEDs éteintes :
- la broche 3 du microcontrôleur, qui est reliée à l'anode de la LED n°5, doit être configurée en sortie et mise à l'état haut ;
- la broche 4 du microcontrôleur, qui est reliée à la cathode de la LED n°5, doit être configurée en sortie et mise à l'état bas ;
- les autres broches doivent être configurées en entrée haute impédance (une logique à trois états stables est nécessaire, donc).

La LED n°6 qui est aussi reliée aux broches 3 et 4 restera éteinte, car elle est montée dans l'autre sens (état bloqué).

Si on veut allumer plusieurs LEDs, il faut alors revenir aux techniques habituelles de multiplexage. Il faut rafraichir l'état de chaque LED, chacune leur tour, et de façon suffisamment rapide pour faire jouer la persistance rétinienne qui fait que l'œil ne percevra pas les clignotements des LEDs.
Par exemple, pour percevoir les deux LEDs n°5 et n°6 de la matrice allumées, il faut allumer alternativement la n°5 seule (broche 3 = HIGH, broche 4 = LOW), puis la n°6 seule (en inversant les états, broche 3 = LOW, broche 4 = HIGH). Si l'alternance est à une fréquence suffisamment grande, l'œil humain est trompé, et percevra les deux LEDs allumées.

Bien entendu, une bibliothèque officielle [Arduino_LED_Matrix](https://github.com/arduino/ArduinoCore-renesas/tree/main/libraries/Arduino_LED_Matrix){:target="_blank"} est fournie en standard pour piloter la matrice sans avoir à se soucier du câblage des LEDs et du Charlieplexage, avec des [exemples](https://github.com/arduino/ArduinoCore-renesas/tree/main/libraries/Arduino_LED_Matrix/examples){:target="_blank"} d'utilisation prêts à l'emploi.

Un petit parcours rapide du [code](https://github.com/arduino/ArduinoCore-renesas/blob/main/libraries/Arduino_LED_Matrix/src/Arduino_LED_Matrix.h){:target="_blank"} de la bibliothèque montre les couples de broches à piloter pour allumer chaque LED dans un tableau :

```c++
static const uint8_t pins[][2] = {
  { 7, 3 }, // pilotage pour la LED n°1
  { 3, 7 }, // LED n°2
  { 7, 4 }, // ...
  { 4, 7 },
  { 3, 4 }, // broche 3 = HIGH, et broche 4 = LOW pour la LED n°5
  { 4, 3 }, // broche 4 = HIGH, et broche 3 = LOW pour la LED n°6
  { 7, 8 },
  { 8, 7 },
  { 3, 8 },
  { 8, 3 }, // LED n°10
  { 4, 8 }, // ... 
  { 8, 4 },
  { 7, 0 },
  // etc.
```

Pour allumer une LED en particulier, les broches sont toutes réinitialisées, une broche est configurée en sortie et mise à l'état haut, une autre broche est configurée en sortie et mise à l'état bas :

```c++
// les 11 broches sont réparties sur deux ports
#define LED_MATRIX_PORT0_MASK       ((1 << 3) | (1 << 4) | (1 << 11) | (1 << 12) | (1 << 13) | (1 << 15))
#define LED_MATRIX_PORT2_MASK       ((1 << 4) | (1 << 5) | (1 << 6) | (1 << 12) | (1 << 13))
 
static void turnLed(int idx, bool on) {
  R_PORT0->PCNTR1 &= ~((uint32_t) LED_MATRIX_PORT0_MASK);
  R_PORT2->PCNTR1 &= ~((uint32_t) LED_MATRIX_PORT2_MASK);
 
  if (on) {
    bsp_io_port_pin_t pin_a = g_pin_cfg[pins[idx][0] + pin_zero_index].pin;
    R_PFS->PORT[pin_a >> 8].PIN[pin_a & 0xFF].PmnPFS =
      IOPORT_CFG_PORT_DIRECTION_OUTPUT | IOPORT_CFG_PORT_OUTPUT_HIGH; // anode à l'état haut
 
    bsp_io_port_pin_t pin_c = g_pin_cfg[pins[idx][1] + pin_zero_index].pin;
    R_PFS->PORT[pin_c >> 8].PIN[pin_c & 0xFF].PmnPFS =
      IOPORT_CFG_PORT_DIRECTION_OUTPUT | IOPORT_CFG_PORT_OUTPUT_LOW; // cathode à l'état bas
  }
}
```

Pour le multiplexage, un *timer* est démarré dans la méthode `begin()` :

```c++
// ...
 _ledTimer.begin(TIMER_MODE_PERIODIC, type, ch, 10000.0, 50.0, turnOnLedISR, this);
// ...
```

La fréquence est de 10 000 Hz, avec une fonction d'interruption `turnOnLedISR` appelée sur chaque front montant pour mettre à jour l'état d'une LED. En sortant de la fonction ISR, un compteur est incrémenté (modulo 96) pour passer à la LED suivante de la matrice au prochain appel.
Toute la matrice est ainsi rafraichie 10 000 / 96 = 104 fois par seconde.

Pour jouer avec cette matrice de LEDs, vous avez la description de toute l'API ici : [Using the Arduino UNO R4 WiFi LED Matrix](https://docs.arduino.cc/tutorials/uno-r4-wifi/led-matrix/){:target="_blank"}

Les auteurs ont même prévu un éditeur pour dessiner sur la matrice et récupérer les dessins dans des tableaux C : [LED Matrix Editor](https://ledmatrix-editor.arduino.cc/){:target="_blank"}
