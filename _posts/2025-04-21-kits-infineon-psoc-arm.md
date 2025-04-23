---
title: "Les cartes d'évaluation CY8CKIT Infineon et PSoC Creator"
date: 2025-04-21
categories: ['Infineon Kit PSoC']
tags: ['Kits CY8CKIT', 'PSoC Creator']
math: false
target_blank: true
---

Il y a quelques années déjà, je me suis penché sur l'architecture à microcontrôleur des cartes Infineon PSoC (anciennement Cypress) en investissant dans ces kits : [CY8CKIT-042](https://www.infineon.com/cms/en/product/evaluation-boards/cy8ckit-042/){:target="_blank"} et [CY8CKIT-044](https://www.infineon.com/cms/en/product/evaluation-boards/cy8ckit-044/){:target="_blank"}.

![CY8CKIT-042 et CY8CKIT-044](/assets/img/posts/2025-04-21-kits-infineon-psoc-arm/psoc4-cypress.png)
*À gauche, le Pioneer Kit CY8CKIT-042, à droite le Pioneer Kit CY8CKIT-044 peut être enfiché sur une carte Raspberry Pi*

Ces cartes ont même été annoncées à l'époque comme les *tueuses de l'Arduino Uno*. Il n'en a finalement rien été, mais l'architecture si particulières des PSoC (*Programmable System On Chip*) avec ses blocs de fonctionnalités logiques et analogiques configurables et programmables, et son système de routage permettant d'interconnecter les blocs et les entrées-sorties méritent le détour.

En effet, que diriez-vous si au lieu de programmer quelque chose de classique dans un microcontrôleur comme...

```c
if (Input1_Read() && !Input2_Read()){
    LED_Pin_Write(1); // Allumer LED
}
else {
    LED_Pin_Write(0); // Eteindre LED
}
```
 [...] vous pouviez réaliser le schéma simple ci-dessous en câblant des portes logiques et en dirigeant les entrées-sorties dans l'environnement [PSoC Creator](https://www.infineon.com/cms/en/design-support/tools/sdk/psoc-software/psoc-creator/){:target="_blank"} :

![Schéma PSoC Creator](/assets/img/posts/2025-04-21-kits-infineon-psoc-arm/TopDesign_schematic.PNG)

À la génération de ce projet de démonstration, les unités logiques du processeur PSoC seront reconfigurées afin d’activer les portes logiques, et les entrées-sorties seront routées suivant les indications du schéma. Avec ce séquenceur câblé, aucun cycle n’est consommé par le CPU dans la décision d'allumer ou éteindre la LED en sortie.

Vous touchez du doigt ici la technologie des architectures PSoC (qui n'est pas sans rappelée celle des FPGA finalement). Vous disposez de circuits mixtes logiques/analogiques configurables avec un microcontrôleur embarqué. Vous pouvez configurer votre propre circuit en définissant quelles fonctionnalités logiques ou analogiques utiliser et comment les interconnecter. De plus, les circuits sont reconfigurables dynamiquement ce qui fait que vous pouvez également définir quand utiliser les fonctionnalités et mettre en œuvre sur le même circuit des configurations différentes à d'autres moments.

![Architecture Infineon PSoC](/assets/img/posts/2025-04-21-kits-infineon-psoc-arm/Image36.jpeg)
*Illustration Infineon : PSoC, une puce programmable et configurable autour d'un MCU ARM-Cortex selon Infineon*

Pour vous convaincre encore, analysez ce schéma dessiné dans PSoC Creator
![Bouton bascule](/assets/img/posts/2025-04-21-kits-infineon-psoc-arm/Button_toggle-Led.PNG) :
*Comment réaliser un interrupteur à bascule pour allumer/éteindre une LED avec un bouton à appui momentané ?*

Avec le kit PSoC, un *Debouncer*, une bascule T *toggle* et pas une ligne de code pour sa gestion, puisque tout est géré matériellement. Faites la comparaison avec l'exemple qui suit sous Arduino, avec toute la logique de détection de front et pause antirebonds codée en langage C/C++ Arduino :

- voir dans la documentation Arduino : [Built-in Examples / Debounce on a Pushbutton](https://docs.arduino.cc/built-in-examples/digital/Debounce/){:target="_blank"}

```c++
/*
  Debounce

  Each time the input pin goes from LOW to HIGH (e.g. because of a push-button
  press), the output pin is toggled from LOW to HIGH or HIGH to LOW. There's a
  minimum delay between toggles to debounce the circuit (i.e. to ignore noise).

  The circuit:
  - LED attached from pin 13 to ground through 220 ohm resistor
  - pushbutton attached from pin 2 to +5V
  - 10 kilohm resistor attached from pin 2 to ground

  - Note: On most Arduino boards, there is already an LED on the board connected
    to pin 13, so you don't need any extra components for this example.

  created 21 Nov 2006
  by David A. Mellis
  modified 30 Aug 2011
  by Limor Fried
  modified 28 Dec 2012
  by Mike Walters
  modified 30 Aug 2016
  by Arturo Guadalupi

  This example code is in the public domain.

  https://www.arduino.cc/en/Tutorial/BuiltInExamples/Debounce
*/

// constants won't change. They're used here to set pin numbers:
const int buttonPin = 2;  // the number of the pushbutton pin
const int ledPin = 13;    // the number of the LED pin

// Variables will change:
int ledState = HIGH;        // the current state of the output pin
int buttonState;            // the current reading from the input pin
int lastButtonState = LOW;  // the previous reading from the input pin

// the following variables are unsigned longs because the time, measured in
// milliseconds, will quickly become a bigger number than can be stored in an int.
unsigned long lastDebounceTime = 0;  // the last time the output pin was toggled
unsigned long debounceDelay = 50;    // the debounce time; increase if the output flickers

void setup() {
  pinMode(buttonPin, INPUT);
  pinMode(ledPin, OUTPUT);

  // set initial LED state
  digitalWrite(ledPin, ledState);
}

void loop() {
  // read the state of the switch into a local variable:
  int reading = digitalRead(buttonPin);

  // check to see if you just pressed the button
  // (i.e. the input went from LOW to HIGH), and you've waited long enough
  // since the last press to ignore any noise:

  // If the switch changed, due to noise or pressing:
  if (reading != lastButtonState) {
    // reset the debouncing timer
    lastDebounceTime = millis();
  }

  if ((millis() - lastDebounceTime) > debounceDelay) {
    // whatever the reading is at, it's been there for longer than the debounce
    // delay, so take it as the actual current state:

    // if the button state has changed:
    if (reading != buttonState) {
      buttonState = reading;

      // only toggle the LED if the new button state is HIGH
      if (buttonState == HIGH) {
        ledState = !ledState;
      }
    }
  }

  // set the LED:
  digitalWrite(ledPin, ledState);

  // save the reading. Next time through the loop, it'll be the lastButtonState:
  lastButtonState = reading;
}
```

Et finalement, voici comment on conçoit un simple clignotement de LED à la sauce PSoC, c'est-à-dire sans consommation du CPU, dans l'Environnement de Design Intégré (EDI) PSoC Creator :

![Générateur PWM pour un blink](/assets/img/posts/2025-04-21-kits-infineon-psoc-arm/designPWM2.PNG)
*Configuration d'un générateur de signaux PWM pour faire clignoter une LED*

![Configuration du PWM](/assets/img/posts/2025-04-21-kits-infineon-psoc-arm/ConfigPWM1.PNG)

Un petit assistant graphique dans l'onglet de configuration du composant PWM redessine le chronogramme en fonction des valeurs saisies dans les champs. On y voit le décompte de 10 000 à 0 à chaque coup d'horloge (10KHz ici, mais c'est aussi configurable) sur chaque période de durée 1s.

Dans le code du fichier `main.c`, il faut juste activer le générateur PWM :

```c
#include <project.h>

int main()
{
    CyGlobalIntEnable; /* Enable global interrupts. */

    /* Place your initialization/startup code here (e.g. MyInst_Start()) */
    PWM_1_Start();
    
    for(;;)
    {
        /* Place your application code here. */
    }
}

/* [] END OF FILE */
```
En résumé, avec un microcontrôleur classique, les opérations suivantes doivent être réalisées pour faire un *blink* :

- repérer les registres du générateur PWM ;
- calculer les valeurs à écrire dans les registres selon la fréquence et le rapport cyclique du signal souhaité ;
- écrire les lignes de code nécessaires pour configurer la broche de sortie et démarrer le générateur PWM, sachant que la plupart des microcontrôleurs n'offrent pas d'alternatives sur le choix de la broche de sortie.

Avec l'architecture PSoC et son environnement de développement PSoC Creator, la conception d'un circuit avec un générateur PWM est réalisée à la souris en quelques clics. Vous disposez vos composants dans la fenêtre *design*, vous les paramétrez, vous interconnectez les composants et routez les entrées-sorties vers les broches GPIO de votre choix. Génial !

Il y a beaucoup à dire sur ces kits Infineon et la technologie des puces PSoC4/PSoc6. Si vous voulez débuter avec ces kits CY8CKIT PSoC4, j'ai préparé quelques activités *Getting started* pour jouer avec des LEDs, des boutons-poussoirs, la liaison série, etc. : [PSOC4-Débuter](https://github.com/fleb72/PSOC4-Debuter){:target="_blank"}
