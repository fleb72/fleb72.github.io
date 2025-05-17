---
title: "Prototype d’un jeu de tir sur cible en FPGA avec affichage VGA"
date: 2025-05-11
categories: [FPGA]
tags: [systemVerilog, VGA, fun-tech]
math: true
target_blank: true
---

Je continue [mon projet]({{ site.baseurl }}{{ site.url }}/posts/fpga-adc-joystick-sprite-reticle/){:target="_blank"} en explorant les premières étapes d’un jeu de tir sur cible en FPGA avec affichage VGA. Il permet le contrôle d’un réticule via un joystick et affiche des impacts à l’écran lors d'un appui sur le bouton de tir. L'objectif est encore de montrer des concepts de design de circuits logiques en SystemVerilog dans l'environnement Intel Quartus Prime (*Lite*).

![Ecran visée et tir](/assets/img/posts/2025-05-11-fpga-target-shooting-demo/impacts-cible.jpg)

### Le bouton de tir

Le bouton de tir intégré au mini-joystick est en fait un simple bouton-poussoir à appui momentané que l'on retrouve souvent dans les kits pour débuter sur Arduino, Raspberry Pi et consorts. Quand on appuie sur la manette du joystick, on fait descendre un axe guidé en translation et qui vient en appui sur le poussoir :

![Bouton-poussoir Joystick](/assets/img/posts/2025-05-11-fpga-target-shooting-demo/joystick-bouton-poussoir.jpg)
*L'axe guidé en translation vertical vient en appui sur le poussoir du bouton*

En sortie du module Joystick, sur la broche SW (*SWitch*), vous aurez un signal logique à l'état bas avec le bouton enfoncé. Si le bouton est relâché, l'état est indéterminé, car la broche est « flottante ». Il reste donc à connecter une résistance de tirage (ou *pull-up*) pour lever l'indétermination.
Vous pouvez brancher une résistance de tirage externe sur votre montage, mais vous pouvez aussi activer une résistance interne à la broche de la carte DE0-nano. Pour cela, rendez-vous dans le menu *Assignments → Assignment Editor*, et activez l'option *Weak Pull-Up Resistor* :

![Weak Pull-Up Resistor](/assets/img/posts/2025-05-11-fpga-target-shooting-demo/assignment-editor-switch-pin.png)

### Les signaux du bouton de tir

On sait que les signaux de ce genre de boutons sont parasités par des phénomènes de rebonds mécaniques sur l'appui et même le relâchement du bouton. Chaque rebond qui repasse par l'état bas du bouton pourrait être interprété comme un appui sur le bouton si on ne prend aucune précaution. Il faut donc mettre en oeuvre une solution « anti-rebonds » (*debounce*) comme celle montrée dans la simulation ci-dessous :

![debounce simulation](/assets/img/posts/2025-05-11-fpga-target-shooting-demo/debounce-simulation.png)

Il y a plusieurs variantes de cette solution, mais celle proposée est très simple sur le principe. Elle consiste à détecter le premier front du signal et d'attendre la stabilisation du signal avant de basculer définitivement. Solution simple, mais qui va entraîner un retard (5, 10, 20ms selon la qualité du bouton) dans la détection de l'appui. Retard négligeable ? 

Dans l'archive du projet ci-jointe en fin de ce billet, vous trouverez les deux modules (j'ai préféré séparer les fonctionnalités) pour produire une impulsion synchronisée à chaque « tir » déclenché, à savoir :
- *debounce.sv* : module pour filtrer les parasites et rebonds mécaniques du bouton ;
- *edge-down.sv* : module pour produire l'impulsion synchronisée sur front descendant du signal filtré.

![modules debounce et edge-down](/assets/img/posts/2025-05-11-fpga-target-shooting-demo/debounce-rtlview.png)

Dans le code ci-dessous, les impacts de tir font des « trous » dans l'écran ;-) Lors d'un `click`sur le bouton, on enregistre les coordonnées du trou (`hole_x`, `hole_y`) à la position du viseur au moment du tir, et on incrémente le nombre d'impacts `hole_count` à l'écran. Le valeur binaire du compteur est aussi transférée sur 4 Leds de la carte, surtout pour vérifier que chaque tir est bien enregistré avec une réactivité suffisante et sans rebonds parasites :

```verilog
    // gestion des impacts de tir
    logic [9:0] hole_x [MAX_HOLES-1:0];  // pour stocker 16 positions x
    logic [9:0] hole_y [MAX_HOLES-1:0];  // pour stocker 16 positions y
    logic [3:0] hole_count = 0;          // Nombre d'impacts placés


    assign counter = hole_count;  // counter de tirs
    always_ff @(posedge clk25) begin
        if (rst==1) begin
            hole_count <= 4'd0;
        end else

        if (click) begin // tir détecté
            if (hole_count < MAX_HOLES) begin  // 10 tirs maxi
                hole_x[hole_count] <= reticle_x;  // Enregistre position actuelle du tir
                hole_y[hole_count] <= reticle_y;
                hole_count <= hole_count + 4'd1;
            end
        end
    end
```

### Gestion de l'affichage des sprites

Je gère maintenant deux sprites préchargés en ROM :
- le réticule du viseur ;
- l'impact du tir, comme un bris de verre.

![Sprites en ROM](/assets/img/posts/2025-05-11-fpga-target-shooting-demo/viseur-impact.png)

Les deux sprites de taille 60x60 pixels sont superposés dans la même image 60x120 pixels et encodés dans le même fichier [.mif (*Memory Initialization File*)](https://www.intel.com/content/www/us/en/programmable/quartushelp/17.0/reference/glossary/def_mif.htm). L'adresse du deuxième sprite est donc décalée de 3600 mots de 12 bits (codage des couleurs RVB444).
Les options du composant *ROM: 2-PORT* (dans le catalogue des IP de Quartus Prime), permettent de récupérer facilement dans le même cycle d'horloge les couleurs du pixel en cours, celui du réticule ou celui de l'impact selon les coordonnées du pixel en cours et les emplacements des sprites.

![RTL view ROM-2port](/assets/img/posts/2025-05-11-fpga-target-shooting-demo/rtlview-rom2port.png)


La gestion de l'affichage est délicate en termes de synchronisme. On rappelle que les coordonnées (`x`, `y`) balayent la surface de l'écran en partant de l'origine en haut à gauche, puis de gauche à droite en descendant ligne par ligne, le tout à une fréquence normalisée de 25,2MHz. Soit un pixel traité toutes les 40ns environ.
Ceci implique que la couleur du pixel doit être « calculée à la volée » par un circuit de logique combinatoire (bloc `always_comb`) :

```verilog
    // ----- Gestion de l'affichage -----

    // inReticle=1 si le pixel (x, y) en cours de balayage est à l'intérieur du sprite du réticule, inReticle=0 sinon
    logic inReticle;
    assign inReticle = (x >= reticle_x) && (x < reticle_x + SPRITE_WIDTH)
        &&
        (y >= reticle_y) && (y < reticle_y + SPRITE_HEIGHT);


    assign adr_reticle = (y - reticle_y) * SPRITE_WIDTH + (x - reticle_x);

    logic [3:0] r, g, b;

    always_comb begin
        r = 4'h0;  // hors zone d'affichage active de l'écran
        g = 4'h0;
        b = 4'h0;

        adr_impact = 0;

        if (inDisplayArea) begin // pixel dans l'aire visible 640x480

            // dessin des impacts
            for (int i = 0; i < hole_count; i++) begin
                if (i < MAX_HOLES) begin // nécessaire à la compilation, car i est un int et peut potentiellement dépasser MAX_HOLES

                    if ((x >= hole_x[i]) && (x < hole_x[i] + SPRITE_WIDTH)
                        &&
                        (y >= hole_y[i]) && (y < hole_y[i] + SPRITE_HEIGHT)) begin

                        adr_impact = (y - hole_y[i]) * SPRITE_WIDTH + (x - hole_x[i]) + SPRITE_WIDTH * SPRITE_HEIGHT;

                        r = color_pixel_impact[11:8];
                        g = color_pixel_impact[7:4];
                        b = color_pixel_impact[3:0];
                    end
                end
            end

            // dessin du sprite du réticule
            if (inReticle && color_pixel_reticle != 12'h000) begin  // gestion de la transparence avec les pixels noirs
                r = color_pixel_reticle[11:8];
                g = color_pixel_reticle[7:4];
                b = color_pixel_reticle[3:0];
            end

            if ((x == reticle_x + SPRITE_WIDTH / 2)
                || (y == reticle_y + SPRITE_HEIGHT / 2)) begin // les 2 lignes horizontale et verticale de la croix du viseur
                r = 4'h3a;
                g = 4'hd4;
                b = 4'hd4;
            end

        end
    end


    always_ff @(posedge clk25) begin  // transfert des signaux sur le port VGA
        {vga_hsync, vga_vsync}  <= {hsync, vsync} ;
        {vga_r, vga_g, vga_b} <= {r, g, b};
    end
```

On commence par les différents impacts dans une boucle `for`. Une boucle `for` sur FPGA ne permet pas d'exécuter de façon répétitive une séquence d'instructions comme dans les langages de programmation classique Java, C, C++, Python, etc. Ici on synthétise du matériel répétitif. En tous cas, si le pixel en cours est celui d'un impact, on récupère sa couleur dans la ROM.

Si le pixel en cours est aussi celui du réticule de visée, la couleur du pixel du réticule sera prioritaire... sauf si sa couleur est noire, et dans ce cas on gère la transparence en n'écrasant pas le pixel qui se trouve derrière :

```verilog
if (inReticle && color_pixel_reticle != 12'h000) begin...
``` 
De cette façon, l'impact peut apparaître en arrière-plan du réticule.

![Gestion de la transparence](/assets/img/posts/2025-05-11-fpga-target-shooting-demo/transparence-impact-reticule.jpg)
*L'impact du tir apparaît par transparence derrière le réticule du viseur*

Enfin pour terminer, le pixel des lignes rouges horizontale et verticale de la croix du viseur est prioritaire sur le pixel des éventuels autres sprites en arrière-plan.

### Le problème de la transparence

> **Remarque importante**
>
> Quand on vient du monde de la programmation procédurale, il y a des raisonnements à remettre en question. Prenons ce code :
>
>```verilog
> always_comb begin
>
>  if (condition1) begin
>    // des trucs à faire
>    r = ...;  g = ...;  b = ...;
>  end
>
>  if (condition2) begin
>    // d'autres trucs à faire
>    r = ...;  g = ...;  b = ...;
>  end
>
>  // etc.
> ```
>
> `always_comb` introduit un bloc de logique combinatoire. Les deux conditions dans le `if` et les circuits qui seront inférés à la synthèse font que tous ces signaux évoluent **en parallèle**.
>
> - Dire que le premier `if` est évalué d'abord, puis le second ensuite, comme si le code était exécuté séquentiellement est un raisonnement faux.
> - Dire que si les deux conditions sont vraies, le pixel va d'abord s'allumer d'une couleur avant d'être écrasé dans un second temps par une autre couleur est faux.
> - Il n'y a pas de *frame buffer*, de sauvegarde de l'arrière-plan dans ce circuit de logique combinatoire, il n'y a pas de mémoire, tous les signaux sont concurrents.
>
> **Mais**, dire que dans ce cas l'ordre des `if` dans le code n'a aucune importance n'est pas vrai non plus ;-), car l'ordre des instructions dans le code donne des priorités lors de la synthèse. C'est ce qui se passe dans ce bout de code lors de l'affectation des signaux `r = ...;  g = ...;  b = ...;`. Si les deux conditions sont vraies, c'est la dernière affectation qui est prioritaire et qui l'emporte donc.
>
> Tout ça pour vous dire que dans cette démonstration, si j'ai réussi à gérer la transparence entre deux sprites (coup de bol ?), celui de l'impact du tir qui apparait *par transparence* derrière le réticule du viseur, ce n'est pas grâce à un buffer ou une mémoire qui sauvegarderait l'arrière-plan avant de décider d'écraser un pixel par un autre ou non.
> À une position `(x,y)` donnée pendant le balayage de l'écran, le pixel du réticule et celui de l'impact sont évalués « en même temps », et on arrive à donner la priorité à celui de l'impact en arrière-plan si le pixel du réticule est noir.
>
> On ne sait jamais, relisez ces deux dernière phrases encore... ;-)
{: .prompt-warning }

Enfin, pour tout vous avouer, je n'ai pas réussi à gérer la transparence entre les sprites des impacts (*sic*) :

![échec transparence](/assets/img/posts/2025-05-11-fpga-target-shooting-demo/impacts-non-transparence.jpg)

Les sprites des impacts s'affichent superposés dans une boucle `for` et l'effet n'est pas très joli, mais je n'arrive pas à donner une priorité d'un pixel sur un autre. Mais sans doute qu'à un moment, pour aller plus loin, je devrais passer par du séquentiel et enfin mémoriser l'arrière-plan dans un *frame buffer*, surtout si je veux en plus dégommer des cibles avec mes tirs...


### Au final...

{% include embed/youtube.html id='-3PMtaXyLqA' %}


Lien vers le dépôt du projet : [fpga-target-shooting-demo](https://github.com/fleb72/fpga-target-shooting-demo){:target="_blank"}. 