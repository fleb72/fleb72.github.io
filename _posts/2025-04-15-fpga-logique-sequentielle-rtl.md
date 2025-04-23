---
title: "Logique séquentielle et conception RTL (Register Transfer Level)"
date: 2025-04-15
categories: [FPGA]
tags: [systemVerilog]
math: true
target_blank: true
---

En [logique combinatoire](https://fr.wikipedia.org/wiki/Logique_combinatoire){:target="_blank"}, les sorties d'un système numérique ne dépendent que des états logiques présents aux entrées. Le système n'a donc pas de mémoire, et un système sans mémoire est forcément très limité.
Prenez l'exemple d'un système d'ouverture à code qui donne trois tentatives à l'utilisateur pour saisir le bon code. Dans un système à logique purement combinatoire, chaque saisie de code infructueuse ne peut produire qu'un seul résultat, car sans mémorisation, il ne peut pas tenir compte des tentatives infructueuses précédentes et de leur nombre. Pour une entrée donnée, la sortie produite est toujours la même...

Pour corriger cela, il faut passer à une autre branche de la logique en mathématique : la [logique séquentielle](https://fr.wikipedia.org/wiki/Logique_s%C3%A9quentielle){:target="_blank"} dont les composants dans les circuits, à leur façon, ont des capacités de mémorisation.
Dans un circuit à logique séquentielle, l'état logique de la sortie dépend des états logiques actuels des entrées, **mais aussi de l'état logique précédent de la sortie**. Le circuit séquentiel a donc la capacité de mémoriser l'état logique actuel de la sortie entre deux instants de calcul des sorties, pour calculer l'état logique futur.

Le composant primitif et fondamental de logique séquentielle dans un FPGA est la **bascule D synchrone** (ou *D flip-flop*), conçue avec un câblage particulier de portes logiques élémentaires. Il comprend une donnée d'entrée D et une sortie Q. À l'entrée CLK, on attend un signal d'horloge (*clock*) qui va commander les changements d'états logiques de la sortie Q.
Le petit triangle au niveau du signal d'horloge CLK signifie que cette bascule est sensible au front montant de l'horloge (*edge-triggered D flipflop*).

![Schéma bascule D](/assets/img/posts/2025-04-15-fpga-logique-sequentielle-rtl/dff-schematic.png)
*Bascule D synchrone, sensible au front montant de l'horloge CLK*

Le comportement de la bascule peut être expliqué grâce au chronogramme suivant obtenu par simulation :

![Diagramme Bascule D](/assets/img/posts/2025-04-15-fpga-logique-sequentielle-rtl/dff-diagram.png)

Les fronts montants de l'horloge sont repérés en rouge.
On remarque que :
- sur front montant de l'horloge, la sortie Q prend l'état de l'entrée D, et peut donc basculer à ce moment-là ;
- entre les fronts montants, quel que soit l'état de l'entrée D, la sortie Q maintient son état (mémorisation).

Comment peut-on exploiter ce comportement en pratique ?

### Cas du registre 1-bit

On peut décrire le comportement d'un **registre 1-bit chargeable** très simplement en SystemVerilog.

```verilog
module my_register 
   (
      input logic clk, enable, d,
      output logic q   
   );   
   
   always_ff @(posedge clk) begin
      if (enable) 
               q <= d;
      // sous-entendu, l'état de q est maintenu sinon    
   end   
endmodule
```

Le bloc `always_ff @(posedge clk)` va inférer un système à bascule(s) D sensible(s) au front montant de l'horloge à la synthèse. L'instruction d'affectation (dite non-bloquante) `q <= d` signifie que l'état « courant » de l'entrée d sera « affecté » à la sortie q au front d'horloge suivant.

Avec [Intel Quartus Prime Lite](https://www.intel.com/content/www/us/en/products/details/fpga/development-tools/quartus-prime/resource.html){:target="_blank"}, ce code va inférer un sous-système à base de bascule D avec une nouvelle entrée ENA :

> *When the ENA (clock enable) input is high, the flipflop passes a signal from D to Q. When the ENA input is low, the state of Q is maintained, regardless of the D input.*

> Trad. : Quand l'entrée ENA est active à l'état haut, la bascule dirige le signal D vers Q. Quand l'entrée ENA est à l'état bas, l'état de Q est maintenu, indépendamment de l'état de l'entrée D.
> - extrait documentation Quartus

![bascule D avec ENA](/assets/img/posts/2025-04-15-fpga-logique-sequentielle-rtl/dff-ena-rtl-view.png)
*Vue RTL*

Une simulation comportementale permet de confirmer le bon fonctionnement du registre. Au niveau du front ①, le registre est chargé avec le bit 1 et l'état de la sortie est maintenu. Au front ②, on charge le registre avec le bit 0, puis on charge à nouveau le bit 1 au front ③.

![Diagramme registre 1-bit](/assets/img/posts/2025-04-15-fpga-logique-sequentielle-rtl/waveform-load-register.png)

Regardons plus en détail la structure interne de ce sous-système dans *Quartus Prime* :

![map view registre 1-bit](/assets/img/posts/2025-04-15-fpga-logique-sequentielle-rtl/map-viewer.png)
*Technology Map View (post-Mapping)*

Ce schéma montre deux choses importantes :
- un bloc de logique combinatoire au centre de la figure que j'ai détaillé, dont la sortie est reliée à l'entrée D de la bascule. La structure de ce bloc n'est pas inconnue, c'est celle d'un multiplexeur (ou sélecteur) que nous avons vu dans un billet précédent) ;
- une boucle formée en reliant la sortie Q de la bascule avec l'entrée du bloc logique.

Pour mieux décrire ce qui s'y passe, j'ai préféré élaboré un nouveau schéma de cette structure <sub>avec Word ;-)</sub>

![schéma fonctionnelle registre 1-bit](/assets/img/posts/2025-04-15-fpga-logique-sequentielle-rtl/load-register-schematic.png)


Pour interpréter ce schéma, il faut comprendre que :
1. Le signal Q de la bascule représente l'état de la sortie à un instant présent ;
2. Un bloc de logique combinatoire (ici un multiplexeur) se sert de l'état présent de la sortie et des entrées en cours pour calculer l'état futur de la sortie (ici, maintien de l'état ou chargement du nouvel état selon l'état du signal enable) ;
3. Cet état futur calculé est présenté en entrée D de la bascule pour qu'il soit effectif en sortie au front d'horloge suivant. *On rappelle qu'entre deux fronts montants, les entrées peuvent prendre un nouvel état sans modifier l'état de la sortie de la bascule qui reste stable.*

### Un autre cas : le compteur n bits

Pour expliquer le principe de la description d'un comptage en HDL, on considère un code SystemVerilog montrant un compteur 3 bits qui s'incrémente à chaque front montant de l'horloge, avec remise à zéro possible :

```verilog
module my_counter
   (
      input logic clk, clear,
      output logic [2:0] counter  // compteur 3 bits
   );   
   
   always_ff @(posedge clk) begin
      if (clear) 
         counter <= 0;  // raz
      else
         counter <= counter + 3'b1;
   end   
endmodule
```

Et voici une simulation fonctionnelle de ce compteur :

![Diagramme compteur](/assets/img/posts/2025-04-15-fpga-logique-sequentielle-rtl/waveform-counter.png)

Le schéma RTL (*Register Transfer Level*) généré par ce code sous *Quartus Prime* est le suivant, avec une structure qui n'est pas sans rappeler celle du registre 1-bit :

![Compteur vue RTL](/assets/img/posts/2025-04-15-fpga-logique-sequentielle-rtl/counter-rtl-view.png)
*Les traits en gras représentent des liaisons par bus 3 bits*

On retrouve un registre 3 bits (en fait 3 bascules D, une par bit) pour mémoriser l'état du compteur, et de la logique combinatoire (multiplexeur et additionneur 3 bits) en amont pour calculer la nouvelle valeur du compteur en fonction des entrées courantes et de l'état courant du compteur.

### Conclusion

Dans un environnement de développement sur FPGA, la « compilation » d'un projet comprend plusieurs étapes. Dans les dernières étapes, l'environnement génère un fichier binaire *bitstream* comprenant la configuration de la cible avec le placement et le routage des cellules physiques du circuit électronique de façon la plus optimisée possible.
Mais avant cela, il y a des étapes intermédiaires dont la phase dite de « synthèse ». La synthèse consiste à transformer le code source en une représentation intermédiaire (souvent appelée la *netlist*) qui comprend les éléments logiques et leurs connexions. Tous les schémas logiques de ce billet sont issus de cette phase de synthèse, et ce niveau de description est bien sûr indépendant de la cible FPGA qu'elle soit une puce Cyclone d'Intel, Spartan de Xilinx ou autre.
Les structures logiques comme celles du registre 1-bit ou le compteur vus plus haut sont typiquement des structures RTL (*Register Transfer Level*, en référence aux transferts d'états entre registres et les opérations de logique combinatoire sur leurs signaux), des structures à haut niveau d'abstraction où les briques de base sont les registres, multiplexeurs, additionneurs et autres. Ce haut niveau d'abstraction exploité par les FPGA est plus facile à déduire des descriptions comportementales des HDL (*Hardware Description Language*), et permet d'en dériver des descriptions de plus bas niveau, proches du matériel. 