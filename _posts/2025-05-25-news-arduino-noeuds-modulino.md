---
title: "Faire du prototypage (très) rapide avec Arduino et les noeuds Modulino"
date: 2025-05-25
categories: [Arduino]
tags: ['Hardware', 'ArduinoUnoR4WiFi']
math: false
target_blank: true
---

Quand on veut faire du prototypage sur Arduino, on veut éviter la « planche à pain » et la connectique *spaghetti* :

![Arduino spaghetti](/assets/img/posts/2025-05-25-news-arduino-noeuds-modulino/artistic%20image%20of%20spaghetti%20nodes%20connected%20to%20an%20Arduino%20board.jpg)
*Illustration générée par IA*

Les spaghettis à la bolognaise oui, les spaghettis à l'Arduino non... On peut alors préférer connecter ses composants en les chaînant sur un bus I2C grâce à des connecteurs [Qwicc](https://www.sparkfun.com/qwiic){:target="_blank"}, connecteur intégré notamment sur l'Arduino Uno R4 WiFi :

![Modulino make kit](/assets/img/posts/2025-05-25-news-arduino-noeuds-modulino/arduino-make-kit.jpg)
*Arduino Plug and Make Kit : carte Arduino Uno R4 WiFi chaînée à trois nœuds de capteurs ou actionneurs avec des connecteurs Qwiic*

À cet effet, la *team* Arduino propose ses nœuds [Modulino](https://store.arduino.cc/collections/modulino){:target="_blank"} que l'on peut acheter indépendamment ou regroupés dans un kit.

Ils sont au nombre de sept pour le moment :

![noeuds Modulino](/assets/img/posts/2025-05-25-news-arduino-noeuds-modulino/modulino-nodes.jpg)

- **Modulino Knob** – un encodeur rotatif avec bouton-poussoir ;
- **Modulino Pixels** – huit Leds RGB adressables ;
- **Modulino Distance** – un capteur *time-of-flight* pour mesurer la distance d'un obstacle ;
- **Modulino Movement** – un capteur IMU 6 axes pur mesurer accélération et taux de rotation ;
- **Modulino Buzzer** – pour générer du son ;
- **Modulino Thermo** – capteur de température et humidité ;
- **Modulino Buttons** – Trois boutons-poussoirs avec des Leds.

Pour piloter ces nœuds, la communauté Arduino propose des bibliothèques prêtes à l'emploi : [Programming / Library / Modulino](https://docs.arduino.cc/libraries/modulino/){:target="_blank"}.


![Montage Modulino DIY](/assets/img/posts/2025-05-25-news-arduino-noeuds-modulino/DIYGUYChris.jpg)

- **Source** : [Blog Arduino](https://blog.arduino.cc/2025/05/21/new-arrivals-nano-connector-carrier-7-modulino-nodes-to-supercharge-your-projects/){:target="_blank"}
