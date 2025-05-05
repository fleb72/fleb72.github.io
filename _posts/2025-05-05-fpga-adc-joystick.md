---
title: "Décoder les signaux analogiques d'un mini-joystick avec l'ADC de ma carte FPGA DE0-nano - partie 1"
date: 2025-05-05
categories: [FPGA]
tags: [systemVerilog, VGA]
math: true
target_blank: true
permalink: "/posts/fpga-adc-joystick-1/"
---

La carte FPGA DE0-nano intègre un ADC (*Analog Digital Converter*) ou convertisseur analogique-numérique. La puce ADC128S022 comprend 8 entrées analogiques reliées à un port GPIO 2x13 E/S sur le dessous de la carte.

![ADC Cyclone IV](/assets/img/posts/2025-05-05-fpga-adc-joystick/adc-cycloneIV.png)
*Extrait User Manual DE0-nano*

Il s'agit d'un convertisseur 12 bits avec une tension de référence 3,3V prise sur la carte. On communique avec la puce avec le protocole SPI (*Serial Peripheral Interface*) avec une fréquence d'horloge qui peut aller jusqu'à 3,2MHz (soit 200 ksps).

Comme un c@#, j'avais rédigé un tutoriel [Communication SPI avec un convertisseur Analogique-Numérique, simulation fonctionnelle et analyse des signaux](https://f-leb.developpez.com/tutoriels/fpga/quartus-prime/communication-spi-adc/){:target="_blank"} avant de me rendre compte que la bibliothèque d'*IP cores* (*Intellectual Property*) dans Quartus Prime comprenait déjà un contrôleur pour ce convertisseur A/N ;-)

![IP catalog](/assets/img/posts/2025-05-05-fpga-adc-joystick/ip-adc-controller.png)
*Catalogue des IP cores dans Quartus Prime*

### Mise en œuvre du contrôleur de l'ADC

Et le composant analogique que je vais utiliser pour tester l'ADC est un mini-joystick HW-504. Le joystick est monté sur un mécaniqme articulé avec deux potentiomètres rotatifs à axes perpendiculaires X et Y. Le mouvement gauche-droite entraîne la rotation du potentiomètre autour de l'axe X. Le mouvement avant-arrière entraîne la rotation de l'autre potentiomètre autour de l'axe Y. Et toute autre direction prise par le joystick affectera la rotation des deux potentiomètres. Ansi, les valeurs retournées par les rotations des deux potentiomètres donneront une image de la direction dans le plan X,Y donnée au joystick.

![Joystick HW-504](/assets/img/posts/2025-05-05-fpga-adc-joystick/joystick-hw-504.jpg)
*le Joystick HW-504 relié à deux entrées analogiques de la cartes FPGA*

Pour un premier test, je ne vais relever que le mouvement avant-arrière du joystick et n'actionner qu'un seul potentiomètre câblé sur l'entrée analogique n°0 :

![vue RTL contrôleur ADC](/assets/img/posts/2025-05-05-fpga-adc-joystick/rtlview-adc-controller.png)

L'horloge `CLOCK`à l'entrée du contrôleur de l'ADC est réduite à 3,2MHz (200 échantillons par seconde en théorie). Le contrôleur se charge de la communication avec l'ADC selon le protocle série SPI (signaux `ADC_xxx`). Je devine que le contrôleur va chercher à échantillonner chacune des huits entrées analogiques tour à tour (mode *free running*). L'ADC présente le résultat 12 bits issu de la conversion A/N sur sa sortie série `ADC_DOUT`. Le contrôleur se charge alors de présenter le résultat entre 0 et 4095 sur un bus parallèle 12 bits `CHx[11..0]`pour exploitation.

Ici, je dirige la sortie `CH0[11..0]` vers un autre contrôleur qui pilotera le  ruban de 8 Leds en surface de la carte FPGA :

```verilog
module adc_led_display (
    input logic clk,              // Horloge principale
    input logic [11:0] adc_data,  // Résultat de la conversion A/N sur l'entrée analogique CH0
    output logic [7:0] led_out    // Affichage sur Ledsx8
);

    logic [11:0] neutral_min = 1850;  // valeur neutre constatée : env. 1970
    logic [11:0] neutral_max = 2100;
    
    always_ff @(posedge clk) begin
        // Position neutre
        if ((adc_data >= neutral_min) && (adc_data <= neutral_max)) begin
            led_out <= 8'b00011000; // Allume LED[3] et LED[4] en position neutre
        end else begin
            // Descente du joystick (valeur ADC monte vers 4095)
            led_out[5] <= (adc_data >= 2500);  
            led_out[6] <= (adc_data >= 3000);  
            led_out[7] <= (adc_data >= 3500);  

            // Montée du joystick (valeur ADC descend vers 0)
            led_out[2] <= (adc_data <= 1500);  
            led_out[1] <= (adc_data <= 1000);  
            led_out[0] <= (adc_data <= 500);  
        end
    end

endmodule
```
Lorsque le joystick est rappelé en position neutre, le convertisseur A/N devrait théoriquement retourner la valeur `4096 / 2 = 2048`. En pratique, je suis plutôt autour de 1980. La précision de la mécanique étant ce qu'elle est, j'ai recadré la valeur neutre entre 1850 et 2100, mais cela dépend de votre joystick.

En postion neutre, les deux leds centrales 3 et 4 sont allumées. Un ruban de leds allumées se forme à gauche ou à droite selon la direction avant ou arrière du joystick.

{% include embed/youtube.html id='vEcTH0qlpYU' %}

Essai concluant, le ruban se forme progressivement lorsque je pousse ou tire doucement le joystick...

Mais avec un joystick, et un écran VGA, on peut faire beaucoup mieux. À suivre ;-)