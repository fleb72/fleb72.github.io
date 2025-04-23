---
title: "Les niveaux d'abstraction dans les langages de description de matériel (HDL) - Partie 1/2 "
date: 2025-04-10
categories: [FPGA]
tags: [systemVerilog]
math: true
permalink: "/posts/fpga-abstraction-level-1/"
target_blank: true
---

On ne le rappellera jamais assez... Les langages de description de matériel (ou HDL pour *Hardware Description Language*) tels VHDL ou Verilog/SystemVerilog ne sont pas des langages de programmation classiques comme C, C++, Java, etc. Même si on retrouve des séquences procédurales dans la syntaxe de ces langages, il ne s'agit plus de produire des suites d'instructions exécutées séquentiellement par un CPU.

Le but des HDL est de générer des circuits logiques qui répondront aux spécifications, et qui seront finalement implémentés physiquement dans la puce FPGA.
Les HDL permettent à ce titre au développeur de faire des descriptions de matériel à différents niveaux d'abstraction. Les trois niveaux d'abstraction reconnus sont :

- niveau structurel (*structural*) : le plus bas niveau, le circuit est vu comme des blocs de primitives ou fonctions logiques aux entrées-sorties connectées entre elles ;
- niveau flot de données (*flow data*) : un niveau d'abstraction au-dessus, un signal peut être décrit par une équation logique, mais aussi avec une syntaxe un peu plus avancée parfois proche des langages procéduraux comme C ou C++ ;
- niveau comportemental (*behavorial*) : le plus haut niveau d'abstraction, on ne décrit plus la structure du circuit mais comment il doit se comporter. Le langage HDL permet alors des structures de contrôle avancées avec des `if... else` ou des `case`, comme en C, C++, Java, etc. (mais on n'oublie pas la première phrase de ce billet).

Lors de la phase dite de « compilation », le synthétiseur prend en charge le programme en HDL pour générer le circuit logique d'abord, pour finalement produire le fichier binaire *bitstream* qui va configurer la puce FPGA cible avec le placement et le routage des composants de façon la plus optimisée possible.

À titre de démonstration, on propose de découvrir ces trois niveaux d'abstraction à travers la description d'un circuit logique combinatoire simple appelé [multiplexeur](https://en.wikipedia.org/wiki/Multiplexer){:target="_blank"} (parfois appelé *sélecteur*) en SystemVerilog (une extension du langage Verilog).


### Définition d'un multiplexeur (abrégé *MUX*)

L'entrée `a` ou `b` est dirigée vers la sortie `s` suivant la valeur du signal de sélection `sel`. Si `sel = 0`, ce sera le signal `a` qui sera dirigé vers la sortie `s`, et si `sel = 1`, ce sera le signal `b` qui sera dirigé vers la sortie `s`.

![Schéma d'un multiplexeur](/assets/img/posts/2025-04-10-fpga-abstraction-level/mux.png)
*Schéma d'un multiplexeur*

On en déduit l'équation logique de la sortie `s` : 

$$
s = a \cdot \overline{sel} + b \cdot sel
$$

 Voyons quelques descriptions de ce composant logique à des niveaux d'abstraction différents.

### Description structurelle

De l'équation logique, on en déduit la structure avec des portes logiques élémentaires, ici des portes AND, OR et NOT.

![Schéma logique du multiplexeur](/assets/img/posts/2025-04-10-fpga-abstraction-level/mux_logic_cells.png)
*Schéma logique du multiplexeur*

On s'aide du schéma ci-dessus pour, en SystemVerilog, instancier des portes logiques et faire du câblage :

```verilog
module mux21  // multiplexeur 2 entrées - 1 sortie
   (
      output   logic s,
      input    logic a, b, sel      
   );
         
   logic q1, q2, sel_n; 

   not (sel_n, sel);
   and (q1, a, sel_n);  
   and (q2, b, sel);   
   or  (s, q1, q2);  
endmodule
```
Faisons évoluer ce multiplexeur. Par exemple, s'il comporte maintenant quatre entrées, il faut une entrée de sélection `sel` codée sur 2 bits (`sel[1]` `sel[0]`).

![Schéma mux41](/assets/img/posts/2025-04-10-fpga-abstraction-level/mux41.png)

Mais si on reprend l'équation logique (à partir de la table de vérité), on se rend compte que notre multiplexeur à 4 entrées peut être obtenu en câblant astucieusement trois multiplexeurs à deux entrées :

![Schéma mux41 avec 3xmux21](/assets/img/posts/2025-04-10-fpga-abstraction-level/mux41-with-mux21.png)

Là encore, on se retrouve à faire du câblage, mais en réutilisant le module précédent, soit en SystemVerilog :

```verilog
module mux41  // 4 entrées, 1 sortie
   (
      output logic s,
      input  logic a, b, c, d,
      input  logic [1:0] sel      
   );
   
   logic q1, q2; // noeuds intermédiaires
   
   // instanciation de 3 multiplexeurs 2/1
   mux21 U0(q1, a, c, sel[1]);
   mux21 U1(q2, b, d, sel[1]);
   mux21 U2(s, q1, q2, sel[0]);  
endmodule
```
Ce qui donne le schéma structurel suivant :

![mux41 vue RTL](/assets/img/posts/2025-04-10-fpga-abstraction-level/mux41-rtlView.png)

Et que se passe-t-il si l'on veut multiplexer des entrées de type *bus* ? Avec des bus de largeur 4 bits par exemple :

![mux bus 4 bits](/assets/img/posts/2025-04-10-fpga-abstraction-level/mux-bus4bits.png)
*Les deux entrées et la sortie sont des bus 4 bits*

Là encore, on s'en sort structurellement par association de 4 multiplexeurs élémentaires :

```verilog
module mux_bus #(parameter WIDTH = 4)   // WIDTH=4 par défaut
   (
      output   logic[WIDTH-1:0] s,
      input    logic[WIDTH-1:0] a, b,
      input    logic sel      
   );
   
   
   mux21 mux_inst[WIDTH-1:0] (   // tableau d'instances de mux21
      .s,
      .a,
      .b,
      .sel  
   );  
endmodule
```
Le module ici est même paramétré pour une largeur de bus `WIDTH` quelconque. Le schéma structurel devient :

![mux bus 4bits vue RTL](/assets/img/posts/2025-04-10-fpga-abstraction-level/mux-bus4bits-rtlView.png)

Bien entendu, même en se limitant à la logique combinatoire, il est difficile de tout concevoir de cette façon pour des fonctionnalités avancées. De plus, il est difficile de modifier la structure si la fonctionnalité doit évoluer, le code est difficilement maintenable. Il faut alors programmer à un plus haut niveau d'abstraction et faire confiance au synthétiseur logique pour générer la structure.

Dans la deuxième partie de ce billet, nous verrons les derniers niveaux d'abstraction : flot de données et comportemental...

> Ce billet m'a été fortement inspiré par le livre [Designing Digital Systems With SystemVerilog (v2.1) de Brent E. Nelson](https://www.amazon.com/Designing-Digital-Systems-SystemVerilog-v2-1/dp/B091CRDBNN){:target="_blank"}.
Les schémas structurels sont des copies d'écran de vues RTL (*Register Transfer Level*) dans [Intel Quartus Prime](https://www.intel.fr/content/www/fr/fr/products/details/fpga/development-tools/quartus-prime.html){:target="_blank"}.
{: .prompt-info }