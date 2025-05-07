---
title: "Déplacer un sprite sur écran VGA avec un mini-joystick analogique"
date: 2025-05-07
categories: [FPGA]
tags: [systemVerilog, VGA]
math: true
target_blank: true
---

Maintenant que j'ai le contrôle de mon [joystick analogique]({{ site.baseurl }}{{ site.url }}/posts/fpga-adc-joystick-1/){:target="_blank"}, je vais pouvoir déplacer quelque chose sur l'écran VGA.

Quelque chose... un [sprite](https://fr.wikipedia.org/wiki/Sprite_(jeu_vid%C3%A9o)){:target="_blank"} par exemple, et celui-ci en particulier :

![viseur à croix](/assets/img/posts/2025-05-07-fpga-adc-joystick-sprite-reticle/viseur.jpg)

Ce *sprite* reproduit un viseur à croix (ou réticule), un dispositif optique utilisé dans les lunettes de visée pour aider à aligner une cible avec précision. Je vais donc simplement déplacer ce viseur avec le joystick :

![viseur sur écran VGA](/assets/img/posts/2025-05-07-fpga-adc-joystick-sprite-reticle/vga-viseur.jpg)

Pour stocker l'image du *sprite*, on peut utiliser des blocs de mémoire M9K intégrés dans la puce FPGA Cyclone IV de la carte DE0-nano.
Le catalogue des composants IP dans Quartus Prime comprend un contrôleur avec un assistant de configuration pour gérer ces blocs mémoire.

![composant ROM 1-PORT](/assets/img/posts/2025-05-07-fpga-adc-joystick-sprite-reticle/megawizard-rom1port.png)

La ROM sera préchargée avec un fichier texte au format [.mif (*Memory Initialization File*)](https://www.intel.com/content/www/us/en/programmable/quartushelp/17.0/reference/glossary/def_mif.htm){:target="_blank"} comprenant la définition de l'image. Vous trouverez en bas de page le lien vers le dépôt du projet avec le programme Python qui génère ce fichier `.mif` à partir de n'importe quelle image png/jpeg.

Ci-dessous, on voit comment le contrôleur de la ROM est relié au module de dessin de l'écran VGA :
![composant ROM 1-PORT](/assets/img/posts/2025-05-07-fpga-adc-joystick-sprite-reticle/rtlview-rom1port.png)

Dans le même cycle d'horloge, le module de dessin renseigne l'adresse du pixel en cours dans la mémoire ROM et récupère la couleur du pixel du sprite.

```verilog
...
// inSprite=1 si le pixel (x, y) en cours de balayage est à l'intérieur du sprite, inSprite=0 sinon
    assign inSprite = (x >= sprite_x) && (x < sprite_x + SPRITE_WIDTH)
        &&
        (y >= sprite_y) && (y < sprite_y + SPRITE_HEIGHT);

    assign adr_sprite = (y - sprite_y) * SPRITE_WIDTH + (x - sprite_x);

    ...
```

```verilog
   ...
           // dessin du sprite
            if (inSprite) begin
                r = color_pixel_sprite[11:8];
                g = color_pixel_sprite[7:4];
                b = color_pixel_sprite[3:0];
            end
    ...
```

Pour plus de détails sur la configuration de ces blocs de mémoire M9K : [Configuration de la ROM dans Quartus Prime](https://f-leb.developpez.com/tutoriels/fpga/controleur-vga/#LIII-E-2){:target="_blank"}


{% include embed/youtube.html id='WdbvdWVCfMc' %}

En plus du *sprite*, il n'était pas bien difficile d'étendre la croix du viseur.
L'affichage est un peu déformée notamment à cause du format 640x480 étendu sur un écran au format 16/9. Il doit y avoir un réglage sur l'écran, mais je ne le trouve pas. Filmer un écran avec un smartphone n'arrange pas les choses non plus...

Que croyez-vous qu'il me reste à faire maintenant que je sais diriger un réticule ? Une piste : le joystick intègre un bouton-poussoir qui fait peut faire office de bouton de tir ;-)

Lien vers le dépôt du projet : [fpga-adc-joystick-sprite-reticle](https://github.com/fleb72/fpga-adc-joystick-sprite-reticle){:target="_blank"}. L'archive comprend aussi l'image du viseur, le fichier au format `.mif` et le programme Python qui génère le fichier `.mif`.