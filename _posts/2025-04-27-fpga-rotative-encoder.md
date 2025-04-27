---
title: "Décoder les signaux en quadrature d'un encodeur rotatif"
date: 2025-04-27
categories: [FPGA]
tags: [systemVerilog]
math: false
target_blank: true
---

Les encodeurs rotatifs sont des dispositifs électromécaniques qui convertissent la position angulaire d'un axe en signaux électriques. Les encodeurs optiques qui fonctionnent en quadrature proposent deux sorties A et B en décalage de phase (90°). Le nombre d’impulsions par tour de l’axe étant une caractéristique spécifique de l’encodeur, compter les impulsions donne une image du décalage angulaire (et le nombre d’impulsions par seconde donne une image de la vitesse de rotation). L’avance ou le retard de phase d’une sortie par rapport à l’autre renseigne sur le sens de rotation.

![Signaux en quadrature](/assets/img/posts/2025-04-27-fpga-rotative-encoder/quadrature.png)
*Sens de rotation horaire (ClockWise), signal A en avance de phase sur B. Sens de rotation antihoraire (CounterClockWise), signal B en avance de phase sur A*

Dans ce projet, j'exploite ces signaux avec ma carte FPGA DE0-nano pour augmenter ou diminuer la valeur d'un compteur 8 bits selon le sens de rotation de l'axe de l'encodeur HW-040. Dans la vidéo ci-dessous, les 8 Leds sont l'image en binaire de la valeur de ce compteur :

{% include embed/youtube.html id='11spxMg_nNU' %}

On fournit les chronogrammes d’une simulation ci-dessous :

![Simulation encoder waveform](/assets/img/posts/2025-04-27-fpga-rotative-encoder/simul-encoder-waveform.png)
*Simulation fonctionnelle avec les signaux DT et CLK en quadrature*

- *clock* : horloge du système ;
- *CLK* et *DT* : les signaux en quadrature simulés ;
- *pos[7:0]* : le compteur de position 8 bits. Dans la simulation il croît d'abord jusqu'à 20, puis décroît ;
- *edge_detection* : impulsion sur front montant ou descendant des signaux DT et CLK en quadrature. Tous les fronts sont repérés pour une meilleure résolution ;
- *up_down* : il faut regarder lorsque *edge_detection* est à l'état haut. *up_down* est à l'état haut ou à l'état bas selon le sens de rotation.

Le *design*  du projet est le suivant (*RTL view* dans Intel Quartus Prime) :

![encoder RTL view](/assets/img/posts/2025-04-27-fpga-rotative-encoder/encoder-rtl-view.png)

Un composant PLL (boucle à verrouillage de phase) est activé pour réduire la fréquence d'horloge de la carte (50MHz) à 10kHz. Un bouton *Reset* en surface de la carte permet de remettre le compteur de position à zéro. En sortie, les 8 Leds intégrées constitueront l'image en binaire de la valeur du compteur de position.

**encoder.sv** :

```verilog
module encoder  #(
            parameter int WIDTH = 8
           )
           (
            input logic clock,          // horloge principale
            input logic DT, CLK,        // signaux de l'encodeur en quadrature
            output logic [WIDTH-1:0] pos,   // position (relative)
            input reset                     // reset asynchrone, position=0
);

   logic [2:0] chA, chB; // pipelines channelA, channelB
   logic edge_detection;  // détection de front
   logic up_down;         // avance ou retard de phase ?
   
   always_ff @(posedge clock) begin
    chA <= {chA[1:0], DT};  // channel A, pipeline signal DT
    chB <= {chB[1:0], CLK}; // channel B, pipeline signal CLK
   end
   
   assign edge_detection = (chA[1] ^ chA[2]) || (chB[1] ^ chB[2]); // ^ : OU exclusif
   assign up_down = chA[1] ^ chB[2];
   
   always @(posedge clock or posedge reset) begin
     if (reset) begin
       pos <= 0;
     end
     else if (edge_detection) begin
       pos <= up_down ? pos + 1 : pos - 1; // incrémentation ou décrémentation de la position
     end   
   end
   
endmodule
```

**top.sv** :

```verilog
module top #(
            parameter int WIDTH = 8
        )
        (
          input logic CLOCK,
          input logic DT,
          input logic CLK,
          input logic RESET_n,
          output logic [7:0] LED
        );
    
  logic [WIDTH-1:0] pos;
  
  // Instanciation d'un encoder
  encoder #(.WIDTH(WIDTH)) my_encoder (
        .clock(CLOCK),
        .DT(DT),
        .CLK(CLK),
        .reset(~RESET_n), // pull-up sur le bouton reset, actif sur niveau bas
        .pos(pos)
  );

  assign LED = pos[7:0]; // compteur de position dirigé ves les 8 Leds

endmodule
```

Intel Quartus Prime Lite intègre un outil nommé *Signal Tap Logic Analyser* permettant d’implémenter un module supplémentaire dans votre projet qui va se connecter aux signaux souhaités et faire remonter les données vers le PC de développement par l’interface JTAG puis le câble USB. On pourra ainsi visualiser les signaux sur son ordinateur comme le ferait un analyseur logique connecté aux signaux de la puce FPGA.

![Signal Tap Logic Analyser](/assets/img/posts/2025-04-27-fpga-rotative-encoder/Signal-tap.png)
*Analyseur logique intégré dans Quartus Prime*


- [Lien vers le projet](https://github.com/fleb72/FPGA-encoder){:target="_blank"}.

