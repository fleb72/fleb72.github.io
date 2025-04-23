---
title: "Un anneau de LEDs WS2812B en feu"
date: 2025-04-20
categories: [FPGA]
tags: ['fun-tech', systemVerilog]
math: false
target_blank: true
---

Avec mon [contrôleur de LEDs adressables WS2812B]({{ site.baseurl }}{{ site.url }}/posts/ws2812b-ring-led-systemverilog/){:target="_blank"}, j'ai décidé de mettre le feu à mon anneau 12 LEDs, rien que ça...

![Anneau de LEDs en feu](/assets/img/posts/2025-04-20-ws2812b-hyperloop-fx/anneau-feu.png)
*Illustration générée par IA*

Pour ce faire, voici le code en systemVerilog :

- **top.sv** :

```verilog
module top (
    input logic CLOCK_50,         // Horloge 50MHz
    output logic GPIO_WS2812B     // À relier à l'entrée IN de l'anneau
);
   
   localparam NB_LEDS = 12;      // Anneau avec 12 LED adressables
   localparam ON      = 8'h20,   // luminosité maxi=8'hff, mais /!\ la consommation si toutes les Leds sont allumées 
              OFF     = 8'h00;
   
   logic [7:0] red, green, blue;
   logic [7:0] address;      // Numéro de la LED entre 0 et NB_LEDS-1
   logic load;               // Chargement du registre si load=1
   logic latch_n;            // Déverrouillage et transfert du registre sur front descendant

   logic [$clog2(NB_LEDS) - 1:0] num_led = 0;    // Numéro de led en cours
  
   logic [23:0] rgb;
   logic [25:0] delay = 0;
    
   assign red    = rgb[23-: 8]; // bits 23 à 16
   assign green  = rgb[15-: 8]; // bits 15 à 8
   assign blue   = rgb[7-: 8];  // bits 7 à 0
   
   // Instanciation du contrôleur WS2812B
   ws2812b_controller #(.NB_LEDS(NB_LEDS)) ws2812b_controller_inst (
        .clk(CLOCK_50),
        .data_ws2812b(GPIO_WS2812B),
        .*
   );

   integer i, c;  
    
   logic [23:0] led[0:NB_LEDS-1]; // État des LEDs
  logic [3:0] bottom_stack, pointer;
   logic [23:0] color[0:5];
  
   initial begin // synthétisable avec Quartus Pro
        
      for (i=0; i<NB_LEDS; i=i+1)  led[i] = {OFF, OFF, OFF};
         
      color[0] = {ON, OFF, OFF};   // red
      color[1] = {OFF, ON, OFF};   // green
      color[2] = {OFF, OFF, ON};   // blue
      color[3] = {ON, ON, OFF};    // yellow
      color[4] = {OFF, ON, ON};    // cyan
      color[5] = {ON, OFF, ON};    // purple
      
      bottom_stack = NB_LEDS - 1;
      pointer = 0;

   end

  // assignation des sorties
  assign load = (num_led < NB_LEDS);
  assign address = load ? num_led : 0;
  assign rgb = load ? led[num_led] : 0;
  assign latch_n = ~(num_led == NB_LEDS);


  // Chargement & animation
  always_ff @(posedge CLOCK_50) begin

      if (num_led <= NB_LEDS) begin // chargement led par led
         num_led <= num_led + 1;
      end else if (num_led==NB_LEDS+1) begin
               if (delay[21]==1) begin // si fin temporisation
               
                  for (i=0; i<=NB_LEDS-1 ; i=i+1) begin
                     if (i <= bottom_stack) begin
                        led[i] <= (i == pointer) ? color[c] : {OFF, OFF, OFF};
                     end else begin
                        led[i] <= color[c];
                     end
                  end
                  
                  pointer <= pointer + 1;
                  
                  if (pointer == bottom_stack) begin
                     pointer <= 0;
                     bottom_stack <= bottom_stack - 1;
                     if (bottom_stack == 1)  begin
                        bottom_stack <= NB_LEDS - 1;
                        c <= c + 1;
                        if (c==5) c<=0;
                     end
                  end

                  num_led <= 0; // retour à l'état initial
                  delay <= 0;
               end else begin
                  delay <= delay + 1;
               end
      end

  end
 
endmodule
```

Feu...

{% include embed/youtube.html id='Tmj0EzdiBvo' %}

Ouais, je sais... Le titre du post est un poil putaclic ;-)
