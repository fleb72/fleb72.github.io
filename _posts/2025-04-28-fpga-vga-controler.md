---
title: "Ce bon vieux port VGA..."
date: 2025-04-28
categories: [FPGA]
tags: [systemVerilog, VGA]
math: true
target_blank: true
---

Ce bon vieux VGA... Ses composantes analogiques RVB des faisceaux rouge, vert, et bleu, le protocole pour gérer à l'origine le balayage du spot en sortie du canon à électrons des écrans à tube cathodique, ses définitions d'écran comme le 640x480 (60 images par seconde), et son port DE-15 avec le connecteur 15 broches. À l'ère du tout numérique, tout le monde voudrait s'en débarrasser... Tout le monde ? Non. Une poignée d'irréductibles qui veulent conserver la compatibilité avec du matériel ancien, écrans, moniteurs, vidéoprojecteurs, mini-PC, etc., résistent encore à l'envahisseur digital (et son armée de zéros et de [Huns](https://fr.wikipedia.org/wiki/Huns){:target="_blank"}).

![port VGA image artistique](/assets/img/posts/2025-04-28-fpga-vga-controler/VGA-port-artistic.png)
*Illustration générée par IA*

VGA n'est pas (tout à fait) mort, et cela m'arrange, car son protocole rudimentaire est assez simple à assimiler, et son design dans un FPGA promet des expériences intéressantes.

### Le port VGA

![Connecteur DE-15](/assets/img/posts/2025-04-28-fpga-vga-controler/OIP.jpg)
*Câble et connecteur femelle VGA (DE-15)*

- Les broches (1), (2) et (3) permettent de définir respectivement les composantes rouge (*RED*), verte (*GREEN*) et bleue (*BLUE*) du pixel en cours. Les signaux sur ces broches sont analogiques et doivent évoluer entre 0V (composante de couleur éteinte) et 0,7V (composante de couleur illuminée au maximum). Pour éviter le bruit sur les signaux analogiques, ceux-ci sont transmis au moyen de câbles coaxiaux dans les câbles VGA de qualité. Pour la composante rouge par exemple, le signal utile est égal à la différence de tension entre les conducteurs (1) et (6) (*RED Ground*). Ce sera entre (2) et (7) (*GREEN Ground*) pour la composante verte, puis entre (3) et (8) (*BLUE Ground*) pour la composante bleue.
- Les broches (13) (synchronisation horizontale *HSYNC*) et (14) (synchronisation verticale *VSYNC*) sont pilotées avec des signaux numériques (à l’état logique haut ou bas). Il s’agit avec ces signaux d’organiser le balayage (*scan*) des pixels (de gauche à droite, puis du haut vers le bas de l’écran) en le synchronisant avec un signal d’horloge. Une impulsion du signal *HSYNC* indique la fin d’une ligne et prépare au balayage de la ligne suivante. Une impulsion du signal *VSYNC* indique la fin du balayage de toute la zone et prépare à un nouveau balayage en reprenant au coin supérieur gauche.

![scan et synchronisation](/assets/img/posts/2025-04-28-fpga-vga-controler/450px-Raster-scan.svg.png)
*Balayage du spot - Image Wikimedia Commons (domaine public)*


### Interface VGA

Pour générer les signaux analogiques à partir d’un microcontrôleur ou d’un FPGA, il est plus simple de passer par trois convertisseurs numérique-analogique (ou DAC pour *Digital-Analog Converter*) fonctionnant en parallèle. Pour interfacer le FPGA avec le moniteur VGA, un module comme celui de *Digilent* fait très bien l’affaire :

![Interface VGA](/assets/img/posts/2025-04-28-fpga-vga-controler/Pmod-VGA-digilent.jpg)
*Module Digilent PmodVGA*

Ce module comporte 4 broches numériques par composante de couleur R, G ou B :

- R3, R2, R1 et R0 pour la composante rouge (*Red*) ;
- G3, G2, G1 et G0 pour la composante verte (*Green*) ;
- B3, B2, B1 et B0 pour la composante bleue (*Blue*).

Chaque composante de couleur est donc définie avec 4 bits et le convertisseur numérique-analogique associé (un simple réseau de résistances R-2R monté en surface du module PmodVGA) pourra générer 16 niveaux de tension entre 0 et 0.7V, soit une profondeur de couleurs 12 bits (RGB444).

Deux broches supplémentaires HS et VS du module PmodVGA permettent de diriger les signaux de synchronisation horizontale et verticale vers le port VGA.

On peut maintenant sortir le matos avec ma carte FPGA DE0-nano et le module PmodVGA...

![DE0-nano et module PModVGA](/assets/img/posts/2025-04-28-fpga-vga-controler/de0-nano-pmodvga.jpg)
*Au bout du câble VGA, à droite, un écran VGA évidemment...*

> Pour avoir plus de détails sur le protocole VGA et le matériel utilisé, je vous recommande cet article : [Programmer un contrôleur pour écran VGA avec une carte de développement FPGA](https://f-leb.developpez.com/tutoriels/fpga/controleur-vga/){:target="_blank"}.
{: .prompt-warning }

### Design FPGA

#### Module de génération des signaux

On commence par programmer le module `vga_sync` qui va générer les signaux nécessaires au balayage de l’écran :

![Bloc vga-sync](/assets/img/posts/2025-04-28-fpga-vga-controler/vga-sync.png)


Les entrées de ce module sont :

- un signal d’horloge `clk25` à 25,2MHz, fréquence de balayage de l’écran pixel par pixel ;
- un signal `rst` de réinitialisation, actif à l’état bas, et qui pourra être activé sur appui d’un bouton-poussoir intégré en surface de la carte FPGA.

En sortie :

- les signaux logiques de synchronisation horizontale et verticale `hsync` et `vsync` selon un timing bien précis;
- les coordonnées du point en cours de la surface balayée `x` (entre 0 et 799) et `y` (entre 0 et 524) sur 10 bits évoluant selon le sens de balayage à la fréquence de l’horloge 25,2MHz ;
- un signal logique `inDisplayArea` à l’état haut lorsque les coordonnées du pixel en cours sont dans la zone active d’affichage (`x` entre 0 et 639, et `y` entre 0 et 479) et à l’état bas sinon.

```verilog
module vga_sync
    #(  // 640x480 60Hz      
        parameter hpixels = 800,    // nombre de pixels par ligne
        parameter vlines = 525,     // nombre de lignes image
        parameter hpulse = 96,      // largeur d'impulsion du signal HSYNC
        parameter vpulse = 2,       // largeur d'impulsion du signal VSYNC
        parameter hbp = 48,         // horizontal back porch
        parameter hfp = 16,         // horizontal front porch
        parameter vbp = 33,         // vertical back porch
        parameter vfp = 10          // vertical front porch        
    )
    
    (
        input  logic clk25, rst,    // signal horloge 25MHz, signal reset 
        output logic [9:0] x, y,    // coordonnées écran pixel en cours
        output logic inDisplayArea, // inDisplayArea = 1 si le pixel en cours est dans la zone d'affichage, = 0 sinon
        output logic hsync, vsync   // signaux de synchronisation horizontale et verticale
    );
    
    
    // compteurs 10 bits horizontal et vertical
    // counterX : compteur de pixels sur une ligne
    // counterY : compteur de lignes
    logic [9:0] counterX, counterY;
    
    
    always_ff @(posedge clk25 or negedge rst) // sur front montant de l'horloge 25MHz, ou front descendant du signal Reset
    begin
        if (rst == 0) begin // Remise à zéro des compteurs sur Reset
            counterX <= 0;
            counterY <= 0;    
        end else
        begin
            // compter les pixels jusqu'en bout de ligne
            if (counterX < hpixels - 1)
                counterX <= counterX + 10'd1;
            else
            // En fin de ligne, remettre le compteur de pixels à zéro,
            // et incrémenter le compteur de lignes.
            // Quand toutes les lignes ont été balayées,
            // remettre le compteur de lignes à zéro.
            begin
                counterX <= 0;
                if (counterY < vlines - 1)
                    counterY <= counterY + 10'd1;
                else
                    counterY <= 0;
            end
        end
    end

    // Génération des signaux de synchronisations (logique négative)
    // <expression 1> ? <expression 2> : <expression 3>, opérateur ternaire comme en C
    assign hsync = ((counterX >= hpixels - hbp - hpulse) && (counterX < hpixels - hbp)) ? 1'b0 : 1'b1;
    assign vsync = ((counterY >= vlines - vbp - vpulse) && (counterY < vlines - vbp)) ? 1'b0 : 1'b1;

    // inDisplayArea = 1 si le pixel en cours est dans la zone d'affichage, = 0 sinon
    assign inDisplayArea = (counterX < hpixels - hbp - hfp - hpulse)
                            &&
                           (counterY < vlines - vbp - vfp - vpulse);
    
    // Coordonnées écran du pixel en cours
    // (x, y) = (0, 0) à l'origine de la zone affichable
    assign x = counterX;
    assign y = counterY;
    
endmodule
```

## Module de génération des images
Le module *drawing* ci-dessous va synthétiser un circuit capable de définir la couleur du pixel en cours pendant le balayage, et de conduire tous les signaux vers l'interface PmodVGA :

![Module drawing](/assets/img/posts/2025-04-28-fpga-vga-controler/rtlview-vga.png)

Hop, un peu de dessin maintenant, en commençant modestement...

#### Un carré jaune sur fond cyan

Au cours du balayage, si les coordonnées `(x,y)` sont telles que le pixel est à l'intérieur du carré, le pixel est jaune, sinon il est cyan.

```verilog
module drawing
    #(        
        parameter square_width = 200,     // taille du carré en pixels
        parameter screen_width = 640,   // définition VGA 640x480
        parameter screen_height = 480
    )

    (
      input logic clk25,
      input logic [9:0] x, y,
      input logic inDisplayArea,
      input logic hsync, vsync,
      output logic [3:0] vga_r, vga_g, vga_b,
      output logic vga_hsync, vga_vsync
    );
   
   logic [3:0] r, g, b;
   
   localparam YELLOW = {4'hF, 4'hF, 4'h0}; // rouge + vert = jaune
   localparam CYAN   = {4'h0, 4'hF, 4'hF}; // vert + bleu = cyan
   localparam BLACK  = {4'h0, 4'h0, 4'h0}; // noir
   
   
   // ----- dessin du carré -------------------------------------------------
   
   logic inSquare;  // 1 si le pixel (x, y) en cours est dans le carré, 0 sinon 
   assign inSquare = (x > (screen_width - square_width) / 2) && (x < (screen_width  + square_width) / 2)
                        &&
                     (y > (screen_height - square_width) / 2) && (y < (screen_height + square_width) / 2);                      
   
   always_comb begin
        if (inDisplayArea) begin // si coordonnées (x,y) dans l'aire d'affichage 640x480       
          {r, g, b} = inSquare ? YELLOW : CYAN;  // opérateur ternaire comme en C
        end
        else begin
          {r, g, b} = BLACK;
        end
   end
   
   // ----- Fin dessin du carré -------------------------------------------------
   
   always_ff @(posedge clk25) begin
     {vga_hsync, vga_vsync} <= {hsync, vsync};  
     {vga_r, vga_g, vga_b}  <= {r, g, b};
   end
   
endmodule
```
Et voici le résultat sur écran VGA, grande émotion...

![VGA carré jaune](/assets/img/posts/2025-04-28-fpga-vga-controler/vga-carre-jaune.jpg)


#### Des rectangles

Ici, on dessine trois rectangles (qui ne se chevauchent pas) rouge, jaune et bleu sur un fond cyan.

```verilog
module drawing (   
      input logic clk25,
      input logic [9:0] x, y,
      input logic inDisplayArea,
      input logic hsync, vsync,
      output logic [3:0] vga_r, vga_g, vga_b,
      output logic vga_hsync, vga_vsync
    );
   
   
   localparam NUM_RECTANGLES = 3; // Nombre de rectangles
   
    // Définition des position, taille et couleur des rectangles dans des tableaux
   
    typedef logic [9:0] position_t[2];  // Type (x, y)
    typedef logic [9:0] size_t[2];      // Type (largeur, hauteur)
    typedef logic [3:0] color_t[3];     // Type (rouge, vert, bleu)

    localparam position_t rectangle_positions[NUM_RECTANGLES] = '{  '{100, 100}, // x, y
                                                                    '{300, 200}, 
                                                                    '{500, 350} 
                                                                 };

    localparam size_t rectangle_sizes[NUM_RECTANGLES] = '{  '{150, 80}, // largeur, hauteur
                                                            '{120, 200},
                                                            '{130, 50} 
                                                         };

    localparam color_t rectangle_colors[NUM_RECTANGLES] = '{ '{4'hF, 4'h0, 4'h0}, // rouge, vert, bleu
                                                             '{4'h0, 4'hF, 4'h0}, 
                                                             '{4'h0, 4'h0, 4'hF} 
                                                           };

  // ----- dessin des rectangles -------------------------------------------------
       
   logic [3:0] r, g, b;

   always_comb begin
    
     r = 4'h0;
     g = 4'hF;
     b = 4'hF; // cyan par défaut

     if (inDisplayArea) begin
        for (int i = 0; i < NUM_RECTANGLES; i++) begin
          if (x > rectangle_positions[i][0] && x < rectangle_positions[i][0] + rectangle_sizes[i][0] &&
              y > rectangle_positions[i][1] && y < rectangle_positions[i][1] + rectangle_sizes[i][1])
          begin
            r = rectangle_colors[i][0];
            g = rectangle_colors[i][1];
            b = rectangle_colors[i][2];
          end
        end   
     end else begin
        r = 4'h0;
        g = 4'h0;
        b = 4'h0; // Noir en dehors de la zone d’affichage
     end
    
   end
  
   // ----- Fin dessin des rectangles -------------------------------------------------
   
   always_ff @(posedge clk25) begin
     {vga_hsync, vga_vsync} <= {hsync, vsync};  
     {vga_r, vga_g, vga_b}  <= {r, g, b};
   end
   
endmodule
```

![VGA rectangles](/assets/img/posts/2025-04-28-fpga-vga-controler/vga-rectangles.jpg)

#### Un dégradé rouge-jaune
Pour terminer cette séquence, on présente un magnifique dégradé linéaire entre le coin supérieur gauche (0,0) (rouge) et le coin inférieur droit (639,479) (jaune).

```verilog
module drawing (   
      input logic clk25,
      input logic [9:0] x, y,
      input logic inDisplayArea,
      input logic hsync, vsync,
      output logic [3:0] vga_r, vga_g, vga_b,
      output logic vga_hsync, vga_vsync
    );

  // ----- dessin du dégradé rouge-jaune -------------------------------------------------
       
   logic [3:0] r, g, b;

   always_comb begin
    
     if (inDisplayArea) begin
       r = 4'hF;          // Rouge maximal
       g = (x + y) / 80;  // Augmente progressivement selon X et Y
       b = 4'h0;          // Pas de bleu pour rester dans le spectre rouge-jaune
     end else begin
       r = 4'h0;
       g = 4'h0;
       b = 4'h0; // Noir en dehors de la zone d’affichage
     end
    
   end
  
  // ----- Fin dessin dessin du dégradé rouge-jaune --------------------------------------
   
   always_ff @(posedge clk25) begin
     {vga_hsync, vga_vsync} <= {hsync, vsync};  
     {vga_r, vga_g, vga_b}  <= {r, g, b};
   end
   
endmodule
```

![VGA dégradé linéaire](/assets/img/posts/2025-04-28-fpga-vga-controler/vga-degrade.jpg)

Très joli, mais je rajouterais bien un peu de dynamisme avec quelques animations bien choisies. Ce sera pour une prochaine fois...