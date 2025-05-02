---
title: "La raquette du Pong sur FPGA"
date: 2025-05-02
categories: [FPGA]
tags: [systemVerilog, VGA]
math: true
target_blank: true
---

Ce billet est la suite logique... Après [la prise de contrôle du port VGA]({{ site.baseurl }}{{ site.url }}/posts/fpga-vga-controler/){:target="_blank"} et [le décodage des signaux en quadrature d'un encodeur rotatif]({{ site.baseurl }}{{ site.url }}/posts/fpga-rotative-encoder/){:target="_blank"}, *nous contrôlons les horizontales et les verticales. Nous pouvons vous noyer sous un millier de chaînes ou dilater une simple image jusqu'à lui donner la clarté du cristal, et même au-delà...*[^footnote]

Euh, non... Plus modestement, je commence un Pong. C'est tellement original comme idée, je n'ai pas pû y résister.
Je commence un Pong, mais j'y vais doucement. Dans ce billet, je vais animer la raquette du Pong et ce sera déjà bien, si.

{% include embed/youtube.html id='0u9VgD1mrtE' %}

### Architecture du projet

![vue RTL, raquette pong](/assets/img/posts/2025-05-02-fpga-raquette-pong/rtlview-pong-raquette.png)

**Les différents blocs :**

- `pll` : un circuit spécialisé disponible dans la bibliothèque de composants de Quartus Prime Lite (*IP Catalog → Library → Basic Functions → Clocks, PLLs and Resets → PLL → ALTPLL*), une boucle à verrouillage de phase ou PLL (*phase-locked loop*) pour asservir la fréquence de sortie sur un multiple de la fréquence d’entrée. L’entrée du bloc est raccordée à l’horloge principale 50MHz de la carte FPGA (entrée `CLOCK_50`). Le ratio de fréquence est fixé à 63/125 pour réduire la fréquence de l’horloge principale : (63/125) x 50 = 25,2MHz. Cette fréquence est nécessaire pour produire une vidéo VGA 640x480 à 60 images par seconde ;

- `freq_div` : un diviseur de fréquence programmé en systemVerilog. Quelques kHz suffisent pour les signaux très lents d'un encodeur rotatif actionné à la main. Une fréquence trop élevée, et l'on pourrait échantillonner des parasites ou les rebonds des contacts mécaniques de l'encodeur.

- `encoder` : le module qui décode les signaux en quadrature *DT* et *CLK* de l'encodeur rotatif. La sortie `dir[1..0]`vaut +1 ou -1 selon le sens de rotation, et 0 s'il n'y a pas de mouvement détecté.

- `encoder_pulse` : le problème avec le module précédent est qu'il produit un signal selon la rotation de l'encodeur (`dir[1..0]` = +1, -1 ou 0) à une fréquence très lente. Ce signal peut être asynchrone par rapport à l'horloge rapide pour gérer l'affichage VGA (25.2MHz), ce qui va poser des problèmes de synchronisation. Ce module `encoder_pulse` produira une impulsion brève à chaque détection de mouvement, active pendant un seul cycle à 25.2MHz, et qui pourra être exploitée facilement dans d'autres modules qui gèrent le VGA.

- `vga_sync`et `drawing` : les modules qui produisent les signaux VGA, avec le dessin de la raquette qui se met à jour en fonction de la rotation de l'encodeur.

### Animation de la raquette

La raquette est un rectangle, c'est ce qu'il y a de plus simple à dessiner.

```verilog
    always_comb begin
        r = 4'h0; // par défaut
        g = 4'h0;
        b = 4'h0;       

        if (inDisplayArea) begin
            r = 4'h0; // Couleur de fond
            g = 4'h0;
            b = 4'h0; 
        
            // Dessin de la raquette à la nouvelle position
            if (x > paddle_x && x < paddle_x + PADDLE_WIDTH &&
                y > paddle_y && y < paddle_y + PADDLE_HEIGHT) begin
                    r = 4'hF; // Couleur de la raquette
                    g = 4'h0;
                    b = 4'h0;
            end
        end
    end
```

Quand la rotation de l'encodeur est détectée, la position de la raquette est simplement décalée d'une valeur fixe. Et quand la raquette  atteint le bord de l'écran, sa position est bloquée.

```verilog
        // Mise à jour de la position, verrouille la position si sortie de l'écran
        if (dir != 0) begin
            if (dir == 1)
                paddle_x <= (paddle_x + speed < SCREEN_WIDTH - PADDLE_WIDTH) ? paddle_x + speed : SCREEN_WIDTH - PADDLE_WIDTH;
            else
                paddle_x <= (paddle_x > speed) ? paddle_x - speed : 0;
        end
```

Le code complet du module *drawing.sv* :

```verilog
module drawing (
    input logic clk25,
    input logic [9:0] x, y,
    input logic signed [1:0] dir,
    input logic inDisplayArea,
    input logic hsync, vsync,
    output logic [3:0] vga_r, vga_g, vga_b,
    output logic vga_hsync, vga_vsync
);

    localparam PADDLE_WIDTH  = 60;
    localparam PADDLE_HEIGHT = 20;
    localparam SCREEN_WIDTH  = 640;
    localparam INIT_POSITION = (SCREEN_WIDTH - PADDLE_WIDTH) / 2; // raquette initialement au centre

    integer paddle_x = INIT_POSITION;
    integer paddle_y = 440;
    integer speed = 10; // Réglage de la vitesse

    always_ff @(posedge clk25) begin

        // Mise à jour de la position, verrouille la position si sortie de l'écran
        if (dir != 0) begin
            if (dir == 1)
                paddle_x <= (paddle_x + speed < SCREEN_WIDTH - PADDLE_WIDTH) ? paddle_x + speed : SCREEN_WIDTH - PADDLE_WIDTH;
            else
                paddle_x <= (paddle_x > speed) ? paddle_x - speed : 0;
        end
    end

// ----- Gestion de l'affichage -----
    logic [3:0] r, g, b;

    always_comb begin
        r = 4'h0; // par défaut
        g = 4'h0;
        b = 4'h0;       

        if (inDisplayArea) begin
            r = 4'h0; // Couleur de fond
            g = 4'h0;
            b = 4'h0; 
        
            // Dessin de la raquette à la nouvelle position
            if (x > paddle_x && x < paddle_x + PADDLE_WIDTH &&
                y > paddle_y && y < paddle_y + PADDLE_HEIGHT) begin
                    r = 4'hF; // Couleur de la raquette
                    g = 4'h0;
                    b = 4'h0;
            end
        end
    end


    always_ff @(posedge clk25) begin
        {vga_hsync, vga_vsync} <= {hsync, vsync};
        {vga_r, vga_g, vga_b}  <= {r, g, b};
    end

endmodule
```

Vous trouverez le projet réalisé avec Intel Quartus Prime sur mon dépôt : [FPGA-paddle-VGA](https://github.com/fleb72/FPGA-paddle-VGA){:target="_blank"}.

Forcément, il y aura une suite...

[^footnote]: [Au-delà du réel : L'aventure continue](https://fr.wikipedia.org/wiki/Au-del%C3%A0_du_r%C3%A9el_:_L%27aventure_continue){:target="_blank"}