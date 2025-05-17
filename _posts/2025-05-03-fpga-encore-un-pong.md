---
title: "Encore un Pong sur FPGA"
date: 2025-05-03
categories: [FPGA]
tags: [systemVerilog, VGA, fun-tech]
math: true
target_blank: true
---

Un Pong ? Comme c'est original...

Mais après avoir [mis en mouvement les raquettes]({{ site.baseurl }}{{ site.url }}/posts/fpga-raquette-pong/){:target="_blank"}, il restait à gérer les déplacements et rebonds de la p'tite balle.

![photo Pong](/assets/img/posts/2025-05-03-fpga-encore-un-pong/yap-fpga.jpg)

Au niveau du matériel, je reprends :
- la carte FPGA DE0-nano ;
- le module PmodVGA, interface entre la carte FPGA et l'écran VGA ;
- deux encodeurs rotatifs avec des signaux en quadrature, surmontés chacun d'un magnifique bouton doré ;
- et cela va de soi, un câble VGA et un écran avec une entrée VGA.

J'ai déjà présenté l'architecture du projet au [billet précédent]({{ site.baseurl }}{{ site.url }}/posts/fpga-raquette-pong/#animer-la-raquette-du-pong-architecture-du-projet){:target="_blank"}, elle reste identique.

![vue RTL - encore un Pong](/assets/img/posts/2025-05-03-fpga-encore-un-pong/rtlview-yap.png)

Les changement interviennent dans le module de dessin et d'animation *drawing.sv* :

```verilog
module drawing (
    input logic clk25, rst,
    input logic [9:0] x, y,
    input logic signed [1:0] dir2,
    input logic signed [1:0] dir1,
    input logic inDisplayArea,
    input logic hsync, vsync,
    output logic [3:0] vga_r, vga_g, vga_b,
    output logic vga_hsync, vga_vsync,
    input logic frame
);


    localparam PADDLE_WIDTH  = 60;
    localparam PADDLE_HEIGHT = 20;
    localparam SCREEN_WIDTH  = 640;
    localparam SCREEN_HEIGHT = 480;
    localparam INIT_POSITION = (SCREEN_WIDTH - PADDLE_WIDTH) / 2; // raquette initialement au centre
    localparam X = 0;
    localparam Y = 1;

    logic [1:0][1:0] dir; // Tableau contenant 2 bus de 2 bits chacun, sens de rotation des encodeurs
    assign dir = '{dir1, dir2};


    logic [9:0] paddle [2][2] = '{
        '{INIT_POSITION, 440},  // coordonnées paddle 1
        '{INIT_POSITION,  20}   // coordonnées paddle 2
    };


    integer speed = 10; // vitesse de déplacement des raquettes


    localparam BALL_SIZE = 10;  // Taille de la balle (carré)
    localparam BALL_X = (SCREEN_WIDTH - BALL_SIZE) / 2;  // Position centrale en X
    localparam BALL_Y = (SCREEN_HEIGHT - BALL_SIZE) / 2; // Position centrale en Y


    logic first_frame = 1;  // Flag pour détecter la première exécution
    logic [9:0] lfsr = 10'b1101010011;  // graine du générateur pseudo-aléatoire

    integer ball_x, ball_y;
    integer ball_dx, ball_dy;


    always_ff @(posedge clk25) begin
        lfsr <= {lfsr[8:0], lfsr[9] ^ lfsr[6]};  // Mise à jour du LFSR
    end


    always_ff @(posedge clk25) begin // mouvement des raquettes
        for (int i = 0; i < 2; i++) begin
            if (dir[i] != 0) begin
                if (dir[i] == 1) begin
                    paddle[i][X] <= (paddle[i][X] + speed < SCREEN_WIDTH - PADDLE_WIDTH) ? paddle[i][X] + speed : SCREEN_WIDTH - PADDLE_WIDTH;
                end else begin
                    paddle[i][X] <= (paddle[i][X] > speed) ? paddle[i][X] - speed : 0;
                end
            end
        end
    end

    always_ff @(posedge clk25) begin
        if (rst==0) begin
            first_frame <= 1;  // active l'initialisation après le premier cycle
        end else
        begin
            if (frame) begin
                if (first_frame) begin
                    ball_x <= lfsr % (SCREEN_WIDTH - BALL_SIZE);
                    ball_y <= SCREEN_HEIGHT / 2; // Position centrale verticale
                    ball_dx <= (lfsr[1]) ? 2 : -2;  // Direction pseudo-aléatoire en X
                    ball_dy <= (lfsr[2]) ? 2 : -2;  // Direction pseudo-aléatoire en Y
                    first_frame <= 0;  // Désactive l'initialisation après le premier cycle
                end else
                begin
                    ball_x <= ball_x + ball_dx;
                    ball_y <= ball_y + ball_dy;

                    if (ball_x > SCREEN_WIDTH - BALL_SIZE) begin   // rebond sur bord droit
                        ball_dx <= ball_dx * (-1);
                        ball_x <= SCREEN_WIDTH - BALL_SIZE;
                    end
                    else if (ball_x < 0) begin                     // rebond sur bord gauche
                        ball_dx <= ball_dx * (-1);
                        ball_x <= 0;
                    end

                    if (ball_y > SCREEN_HEIGHT - BALL_SIZE) begin  // rebond sur bord bas
                        ball_dy <= ball_dy * (-1);
                        ball_y <= SCREEN_HEIGHT - BALL_SIZE;
                    end
                    else if (ball_y < 0) begin                     // rebond sur bord haut
                        ball_dy <= ball_dy * (-1);
                        ball_y <= 0;
                    end

                    // rebond sur raquette
                    for (int i = 0; i < 2; i++) begin
                        if (ball_y + BALL_SIZE >= paddle[i][Y] && ball_y <= paddle[i][Y] + PADDLE_HEIGHT &&
                            ball_x + BALL_SIZE >= paddle[i][X] && ball_x <= paddle[i][X] + PADDLE_WIDTH) begin
                            ball_dy <= ball_dy *(-1);  // Inversion de la direction verticale
                            // Correction de la position pour éviter que la balle reste collée à la raquette
                            ball_y <= (ball_dy > 0) ? paddle[i][Y] - BALL_SIZE - 1 : paddle[i][Y] + PADDLE_HEIGHT + 1;
                        end
                    end

                end
            end
        end
    end

// ----- Gestion de l'affichage -----
    logic [3:0] r, g, b;

    always_comb begin
        r = 4'h0;  // hors zone d'affichage active de l'écran
        g = 4'h0;
        b = 4'h0;

        if (inDisplayArea) begin
            r = 4'h0;  // Couleur de fond par défaut
            g = 4'h0;
            b = 4'h0;
            if ((y < SCREEN_HEIGHT / 2 + 4 && y > SCREEN_HEIGHT /2 - 4) && (x % 4 == 0)) begin
                r = 4'hf;  // Ligne blanche du terrain
                g = 4'hf;
                b = 4'hf;
            end else begin
                // dessin de la balle
                if (x > ball_x && x < ball_x + BALL_SIZE &&
                    y > ball_y && y < ball_y + BALL_SIZE) begin
                    r = 4'hF; // Couleur de la balle (jaune)
                    g = 4'hF;
                    b = 4'h0;
                end else begin // dessin des raquettes
                    for (int i = 0; i < 2; i++) begin
                        if (x > paddle[i][X] && x < paddle[i][X] + PADDLE_WIDTH &&
                            y > paddle[i][Y] && y < paddle[i][Y] + PADDLE_HEIGHT) begin
                            r = 4'hF;  // Couleur de la raquette
                            g = 4'h0;
                            b = 4'h0;
                        end
                    end
                end
            end
        end
    end

    always_ff @(posedge clk25) begin
        {vga_hsync, vga_vsync} <= {hsync, vsync};
        {vga_r, vga_g, vga_b}  <= {r, g, b};
    end

endmodule
```
Dans ce module, les principales difficultés à prendre en compte sont le mouvement de la balle, avec ses rebonds sur le bord du terrain, et sur la raquette.
La position initiale de la balle et sa direction sont pseudo-aléatoires au démarrage ou après appui sur le bouton Reset.

Le jeu est encore incomplet pour simplifier. Les rebonds sont à 45° et la vitesse de la balle est constante, mais rien n'empêche d'ajouter du piment au jeu en accélérant progressivement la balle ou en donnant des effets avec la raquette. Plus loin encore, on pourra compter les points, et donner un vainqueur au bout de 15 points, etc.

{% include embed/youtube.html id='IUM-gnXVNc0' %}
*Pas facile d'une seule main...*

Vous trouverez le projet réalisé avec Intel Quartus Prime sur mon dépôt : [FPGA-YetAnotherPong](https://github.com/fleb72/FPGA-YetAnotherPong/){:target="_blank"}.




