---
title: "Les niveaux d'abstraction dans les langages de description de matériel (HDL) - Partie 2/2"
date: 2025-04-12
categories: [FPGA]
tags: [systemVerilog]
math: true
permalink: "/posts/fpga-abstraction-level-2/"
target_blank: true
---
Lors du [billet précédent]({{ site.baseurl }}{{ site.url }}/posts/fpga-abstraction-level-1/){:target="_blank"}, nous avions vu comment établir la description d'un composant simple appelé multiplexeur :

![Description Mux](/assets/img/posts/2025-04-12-fpga-abstraction-level/mux.png)
*Si `sel=0`, `a` est dirigé vers `s`, et si `sel=1`, c'est `b` qui est dirigé vers `s`*

Au niveau d'abstraction le plus bas, le niveau structurel, ce composant est décrit par le câblage de portes logiques élémentaires (portes AND, OR et NOT dans ce cas précis). Voyons comment pourrait-on établir une description de ce composant à un plus haut niveau d'abstraction.

### Description par *flot de données*

En Verilog/SystemVerilog, on reconnait une description de type *flot de données* par le mot-clé `assign`.

Par exemple, pour le multiplexeur de base :
```verilog
module mux21
   (
      output      logic s,
      input       logic a, b,
      input       logic sel      
   );
   
   assign s = (a & ~sel) | (b & sel);
endmodule
```
Alors que signifie cette « assignation » ? Le signe `=` n'est pas celui d'une « affectation » que l'on rencontre dans les langages de programmation. D'ailleurs ce module générera un circuit logique combinatoire, il n'y a pas d'emplacement mémoire réservé pour stocker une valeur. Ici, les « variables » en jeu sont des nœuds, qui transportent des signaux logiques, sans mémoire.

On souhaite que le signal `s` vérifie une égalité, **continuellement**. Toute évolution du terme à droite du signe `=` due à l'évolution des signaux doit être répercutée sur le signal à gauche du signe `=`. La sémantique de l'assignation est donc plus forte (pensez à l'« assignation à résidence », l'« assignation en justice », etc.)

Le terme de droite, et vous l'aurez reconnue, est l'équation logique du multiplexeur (voir [billet précédent]({{ site.baseurl }}{{ site.url }}/posts/fpga-abstraction-level-1/), où on avait abouti à l'équation $$ s = a \cdot \overline{sel} + b \cdot sel $$). Une équation logique écrite avec les opérateurs binaires comme en langage C, tout cela est bien rassurant, mais ne perdez jamais de vue l'interprétation de cette syntaxe *C-like* dans un HDL.

Bien entendu, les concepteurs de HDL ont prévu toutes les situations avec des opérateurs en abondance. Par exemple, si les entrées du multiplexeur sont des bus, disons de largeur 4 bits, au lieu d'écrire bit par bit :

```verilog
assign s[0] = (a[0] & ~sel) | (b[0] & sel);
assign s[1] = (a[1] & ~sel) | (b[1] & sel);
assign s[2] = (a[2] & ~sel) | (b[2] & sel);
assign s[3] = (a[3] & ~sel) | (b[3] & sel);
```
On peut écrire grâce à l'opérateur de réplication de bits : `{n{...}` :

```verilog
assign s = (a & {4{~sel}}) | (b & {4{sel}});
```
On y perd quand même en lisibilité, mais tout cela peut encore être remplacé de façon équivalente par :

```verilog
assign s = sel ? b : a;
```
Encore une syntaxe *C-like* avec l'opérateur ternaire `condition ? signal_si_vrai : signal_si_faux`. À ce niveau d'abstraction, on s'éloigne déjà bien de l'électronique numérique et de ses circuits à logique combinatoire.

Mais il y a encore un niveau d'abstraction au-dessus...

### Description comportementale

Au niveau comportemental, un bloc de logique combinatoire débute avec l'instruction `always_comb`. On peut repartir de l'équation logique du multiplexeur pour écrire un bloc séquentiel :

```verilog
module mux21
   (
      output      logic s,
      input       logic a, b,
      input       logic sel      
   );
        
   logic q1, q2, sel_n;

   always_comb begin
      sel_n = ~sel;
      q1 = a & sel_n;
      q2 = b & sel;
      s = q1 | q2;   
   end
endmodule
```
Mais il semble plus naturel de décrire le comportement du multiplexeur à partir de sa définition (Si `sel=1` alors l'entrée `b` est dirigée vers la sortie `s`, sinon...). On en oublierait (presque) toute notion d'électronique numérique.

```verilog
module mux21
   (
      output logic s,
      input  logic a, b,
      input  logic sel      
   );
      
   always_comb begin
      if (sel) 
         s = b;
      else
         s = a;
   end
endmodule
```

Avec `always_comb`, vous introduisez donc un bloc de logique combinatoire. Au niveau sémantique, on signifie que la logique du bloc doit être évaluée continuellement en fonction de l'évolution des signaux en entrée.

Si le multiplexeur a quatre entrées, l'emploi de `case` est également possible :

```verilog
module mux41  // 4 entrées, 1 sortie
   (
      output logic s,
      input  logic a, b, c, d,
      input  logic [1:0] sel  // 2 bits pour la sélection    
   );
     
   always_comb begin
      case(sel)
         0 : s = a;
         1 : s = b;
         2 : s = c;
         3 : s = d;     
      endcase
   end   
endmodule
```
Alors vous pouvez toujours établir votre description de façon structurelle ou par flot de données (avec `assign` et une belle équation qui doit tenir sur une ligne), mais le niveau comportemental permet des descriptions beaucoup plus riches et de façon plus naturelle pour un développeur, avec des calculs, des `if...else` que l'on peut imbriquer, des `case`, etc.

Par exemple, si on veut rajouter une entrée d'activation au multiplexeur 4 entrées précédent, on pourra écrire de façon comportementale le code suivant :

```verilog
module mux41  // 4 entrées, 1 sortie
   (
      output logic s,
      input  logic a, b, c, d, 
      input logic enable_n, // entrée d'activation, active à l'état bas
      input  logic [1:0] sel      
   );
     
   always_comb begin
      if (enable_n)  // si inactif
         s = 0;
      else begin
         case(sel)
            0 : s = a;
            1 : s = b;
            2 : s = c;
            3 : s = d;     
         endcase
      end
   end  
endmodule
```

![multiplexeur 4 entrées avec entrée d'activation](/assets/img/posts/2025-04-12-fpga-abstraction-level/mux41-enable-rtlView.png)
*Analyse par le synthétiseur du multiplexeur 4 entrées avec entrée d'activation*

Il faut juste, une fois de plus, ne pas oublier que malgré cette lecture séquentielle du code dans les blocs `always_comb`, un circuit à logique combinatoire sera généré par le synthétiseur, où tous les signaux évoluent « en même temps ».

### Conclusion

LES HDL permettent de synthétiser des systèmes complexes à différents niveaux d'abstraction. À un niveau élevé d'abstraction, le développeur se soucie moins des détails de l'électronique numérique et de son implémentation physique. L'écriture du code en est également facilitée, sa mise au point et sa réutilisation sont favorisées.
Pour autant, dans un projet un peu consistant, vous allez certainement mixer les niveaux d'abstraction. Les fonctionnalités du projet seront découpées en modules logiques plus petits qui seront reliés entre eux ou avec les entrées-sorties du système (description structurelle). À l'intérieur des modules, vous élevez le niveau d'abstraction en fonction de la complexité.

![chrono avec 7-segments](/assets/img/posts/2025-04-12-fpga-abstraction-level/seg7.png)
*Exemple de structure d'un projet de chronomètre (au 1/10è de seconde avec afficheur 7-segments). Un module pour le comptage, un pour l'affichage du compteur, un autre pour filtrer les rebonds d'un bouton-poussoir, etc.*

> Ce billet m'a été fortement inspiré par le livre [Designing Digital Systems With SystemVerilog (v2.1) de Brent E. Nelson](https://www.amazon.com/Designing-Digital-Systems-SystemVerilog-v2-1/dp/B091CRDBNN){:target="_blank"}.
Les schémas structurels sont des copies d'écran de vues RTL (*Register Transfer Level*) dans [Intel Quartus Prime](https://www.intel.fr/content/www/fr/fr/products/details/fpga/development-tools/quartus-prime.html){:target="_blank"}.
{: .prompt-info }