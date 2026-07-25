---
title: "Lecture d'un clavier matriciel à membrane 4x3 sur FPGA"
date: 2026-07-25
categories: [FPGA]
tags: [systemVerilog]
math: true
target_blank: true
---

Les claviers à membrane sont très présents dans les systèmes embarqués : téléphones, digicodes, interfaces de contrôle, ou encore dans vos projets Arduino.
Leur simplicité apparente cache une ingénieuse organisation interne : une matrice de lignes et de colonnes qui permet de détecter jusqu’à douze touches avec seulement sept fils.

Dans ce billet, nous allons décortiquer le fonctionnement d’un clavier matriciel 4×3, comprendre comment chaque touche ferme un circuit entre une ligne et une colonne, et voir comment cette logique se traduit en code ou en simulation. Enfin, un clavier à membrane 4x3 sera interfacé avec ma carte FPGA Altera DE0-nano pour une démonstration complète développée en SystemVerilog dans l'environnement Quartus Prime Lite (version 24.1 à ce jour).

![DE0-nano - 4x3 membrane keypad](/assets/img/posts/2026-07-25-fpga-4x3-membrane-keypad/DE0-nano-4x3-membrane-keypad.jpg)

### Organisation interne : lignes et colonnes

Notre clavier comporte douze touches réparties en 4 lignes (R1 à R4) et 3 colonnes (C1 à C3). Pourtant, la nappe qui relie le clavier à la carte ne compte que sept fils : R1, R2, R3, R4, C1, C2 et C3.
Cette économie de câblage est rendue possible grâce à une organisation interne astucieuse : chaque touche ferme simplement un contact entre une ligne et une colonne.

![Keypad structure](/assets/img/posts/2026-07-25-fpga-4x3-membrane-keypad/keypad-structure.png)
*Les trois résistances de tirage ajoutées sur les colonnes (pull‑up) ne font pas partie du clavier : elles sont nécessaires pour que le FPGA puisse détecter correctement les transitions logiques.*

![Assignments pull-up](/assets/img/posts/2026-07-25-fpga-4x3-membrane-keypad/assignments-pullup.png)
*Activation des résistances de tirage pull-up sur les colonnes dans Quartus Prime*

Dans l’exemple ci‑dessus, la touche ‘8’ est enfoncée : le contact interne du clavier établit un chemin électrique entre la ligne R3 et la colonne C2. Ainsi, lorsque le FPGA active la ligne R3 en la forçant à l'état bas, la colonne C2 est elle aussi abaissée à 0, ce qui indique sans ambiguïté que la touche ‘8’ est enfoncée.

### Cycle de scan

Pour pouvoir détecter n’importe quelle touche, le FPGA doit interroger le clavier en activant les lignes **une par une**. Le principe est simple : on force successivement R1, R2, R3 puis R4 à l’état bas, tandis que les colonnes restent en entrée avec une résistance *pull‑up*. À chaque activation de ligne, on lit l’état des trois colonnes. Si une colonne passe à 0, cela signifie qu’un contact est fermé entre la ligne en cours de *scan* et cette colonne.

Ce balayage cyclique des quatre lignes se répète toutes les 4 ms (1 ms par ligne) :

- R1 active (`ROW_OUT = 4'b1110 = 4'hE`) → lecture des colonnes
- R2 active (`ROW_OUT = 4'b1101 = 4'hD`) → lecture des colonnes
- R3 active (`ROW_OUT = 4'b1011 = 4'hB`) → lecture des colonnes
- R4 active (`ROW_OUT = 4'b0111 = 4'h7`) → lecture des colonnes

Ce mécanisme permet au FPGA d’identifier précisément quelle touche est pressée, simplement en observant **quelle colonne s’abaisse pendant l’activation d’une ligne donnée**.

Par exemple, si la ligne R3 est en cours de scan (`ROW_OUT = 4'hB`) et que `COL_IN` vaut `3'b101` (soit `5` en hexadécimal), cela signifie que la colonne C2 est à l'état bas. 

Ce principe — une ligne active à 0 et une colonne qui est abaissée à 0 lorsqu’un contact est fermé — constitue la base de la détection des touches dans un clavier matriciel.

### Cycle de scan en simulation

Pour visualiser concrètement le fonctionnement du scanner, nous pouvons observer le chronogramme obtenu en simulation fonctionnelle (avec le simulateur *Questa FPGA Starter Edition*) :

![Simulation Questa](/assets/img/posts/2026-07-25-fpga-4x3-membrane-keypad/questa-simul.png)

Le signal `ROW_OUT` déroule son cycle de 4 ms : E → D → B → 7, chaque valeur correspondant à l’activation d’une ligne. Pendant l’activation de R3 (`ROW_OUT = 4'hB`), le signal `COL_IN` vaut `3'b101 = 3'h5` : la colonne C2 est donc abaissée à 0, ce qui indique que la touche située à l’intersection R3 × C2 — la touche ‘8’ — est pressée.

Le scanner encode alors cette information sous la forme d’un code numérique : `KEY_CODE = 9` (ce code correspond à la concaténation de l'indice de ligne entre 0 et 3, et de l'indice de colonne entre 0 et 2, soit ici `4'b10_01 = 4'h9`), qui correspond à la position de la touche dans la matrice. Ce code brut est ensuite transmis à une petite machine à états finis (*FSM*) chargée de filtrer les rebonds et de valider la touche lorsqu'elle est relâchée. Lorsque la FSM confirme que la touche est réellement pressée, elle génère une impulsion unique sur `KEY_DOWN`.

### Architecture du design

Le clavier est piloté par trois modules principaux, organisés en pipeline :

```verilog
module top (
    input  logic        CLOCK_50,
    input  logic [2:0]  COL_IN,
    input  logic        KEY,          // reset actif à l'état bas

    output logic        KEY_DOWN,
    output logic [3:0]  KEY_CODE,
    output logic [3:0]  ROW_OUT
);

    // --- signaux internes ---
    logic        scan_tick;
    logic        key_down_raw;
    logic [3:0]  key_code_raw;

    // --- générateur de tick (1 kHz) ---
    scan_tick_gen #(
        .CLOCK (50_000_000),
        .DIV   (1_000)
    ) tick_gen_inst (
        .clk       (CLOCK_50),
        .rst_n     (KEY),
        .scan_tick (scan_tick)
    );

    // --- scanner de clavier ---
    keypad_scan scan_inst (
        .clk       (CLOCK_50),
        .rst_n     (KEY),
        .scan_tick (scan_tick),
        .col_in    (COL_IN),
        .key_down  (key_down_raw),
        .key_code  (key_code_raw),
        .row_out   (ROW_OUT)
    );

    // --- FSM anti-rebonds + validation ---
    keypad_fsm fsm_inst (
        .clk        (CLOCK_50),
        .rst_n      (KEY),
        .scan_tick  (scan_tick),
        .key_down   (key_down_raw),
        .key_code   (key_code_raw),
        .key_action (KEY_DOWN),
        .key_value  (KEY_CODE)
    );

endmodule
```

![Schéma RTL](/assets/img/posts/2026-07-25-fpga-4x3-membrane-keypad/schema-rtl.png)

Je ne montre ici que les extraits nécessaires à la compréhension.


- `scan_tick_gen` : génère une impulsion régulière (1 kHz) qui cadence le balayage des lignes du clavier.
  - On incrémente un compteur `cnt` toutes les 20 ns (horloge principale `clk` à 50 MHz). Quand le compteur atteint la valeur `49999`, 1 ms s'est écoulée, et une impulsion du signal `scan_tick` est générée :
    ```verilog
    always_ff @(posedge clk or negedge rst_n) begin // reset asynchrone
        if (!rst_n) begin	// reset
            cnt       <= 0;
            scan_tick <= 1'b0;
        end else begin
            if (cnt == CNT_MAX-1) begin
                cnt       <= 0;
                scan_tick <= 1'b1;   // tick
            end else begin
                cnt       <= cnt + 1;
                scan_tick <= 1'b0;
            end
        end
    end
    ```

- `keypad_scan` : active successivement les quatre lignes, lit les colonnes, et produit les signaux bruts `key_down` et `key_code`.

  - Activation des lignes, tour à tour, à chaque impulsion `scan_tick` :
    ```verilog
    // indice de ligne
    logic [1:0] row_idx;

    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            row_idx <= 0;
        else if (scan_tick)
            row_idx <= row_idx + 1;
    end

    // activation des lignes (une seule à 0)
    always_comb begin
        case (row_idx)
            2'd0: row_out = 4'b1110;
            2'd1: row_out = 4'b1101;
            2'd2: row_out = 4'b1011;
            2'd3: row_out = 4'b0111;
        endcase
    end
    ```

  - Lecture et mémorisation de l'état des colonnes, ligne par ligne :

    ```verilog
      // synchronisation des colonnes sur le signal d'horloge
      logic [2:0] col_state;
      always_ff @(posedge clk or negedge rst_n) begin
          if (!rst_n)
              col_state <= 3'b111;
          else
              col_state <= col_in;
      end

      // mémorisation de l'état des colonnes pour chaque ligne
      logic [2:0] row_sample [3:0];

      always_ff @(posedge clk or negedge rst_n) begin
          if (!rst_n) begin
              row_sample[0] <= 3'b111;
              row_sample[1] <= 3'b111;
              row_sample[2] <= 3'b111;
              row_sample[3] <= 3'b111;
          end else if (scan_tick) begin
              row_sample[row_idx] <= col_state;
          end
      end
    ```

  - à la fin du cycle de scan, on repère la touche pressée : 

    ```verilog
     // décision à la fin du cycle (quand row_idx == 3)
      always_ff @(posedge clk or negedge rst_n) begin
          if (!rst_n) begin
              key_down <= 1'b0;
              key_code <= 4'd0;
          end else if (scan_tick && (row_idx == 2'd3)) begin
              r_sel       = -1;
              c_sel       = -1;

			  for (int r = 0; r < 4; r++) begin
				for (int c = 0; c < 3; c++) begin
				  if (row_sample[r][c] == 1'b0) begin
				    // mémoriser la position du premier zéro
					if (r_sel == -1) begin
					  r_sel = r;
					  c_sel = c;
					end
				  end
				end
			  end

			  if (r_sel != -1) begin
					key_down <= 1'b1;
					key_code <= {r_sel[1:0], c_sel[1:0]};
			  end else begin
					key_down <= 1'b0;
					key_code <= 4'd0;
			  end
          end
      end
    ```


- `keypad_fsm` : filtre les rebonds, valide l’appui, et génère l’impulsion `KEY_DOWN` ainsi que la valeur finale `KEY_CODE` .

  - machine à états finis (*FSM*) : IDLE → DEBOUNCE → HELD → RELEASED
   ```verilog
    case (state)

            IDLE: begin
                if (key_down)
                    next_state = DEBOUNCE;
            end

            DEBOUNCE: begin
                if (!key_down)
                    next_state = IDLE; // rebond
                else if (debounce_cnt >= DEBOUNCE_MAX)
                    next_state = HELD; // touche stable
            end

            HELD: begin
                if (!key_down)
                    next_state = RELEASED;
            end

            RELEASED: begin
                key_action = 1'b1;   // impulsion unique
                next_state = IDLE;
            end

    endcase
  ```


> Retrouvez le dépôt du projet sur Github en suivant ce lien : [FPGA Membrane keypad 4x3](https://github.com/fleb72/FPGA-4x3-membrane-keypad){:target="_blank"}
{: .prompt-info }

Pour visualiser rapidement le fonctionnement, la sortie `KEY_CODE` sur 4 bits est dirigée vers 4 Leds du bandeau intégré à la carte DE0-nano :

![Test keyboard KEYCODE](/assets/img/posts/2026-07-25-fpga-4x3-membrane-keypad/test-keyboard-keycode.jpg
)
*ligne 2 - col 1, KEY_CODE=9, la touche '8' est appuyée.*

### Exemple d'application pour aller plus loin

Dans la vidéo de démonstration ci-dessous, on complète avec un module qui déduit le code ASCII de la touche enfoncée, suivi d'un module qui transmet le code ASCII via une liaison série UART Tx (115200-8-O-1) pour une utilisation pratique (voir [programmation d'un transmetteur UART en SystemVerilog](https://f-leb.developpez.com/tutoriels/fpga/quartus-prime/uart-transmetteur/){:target="_blank"}).
La communication UART est validée à l'analyseur logique.

{% include embed/youtube.html id='EZz4cFAfBT0' %}

### Conclusion

Ce projet montre qu’un clavier matriciel, malgré sa simplicité matérielle, nécessite une architecture logique soignée : cycle de scan, synchronisation, mémorisation, détection, filtrage des rebonds et validation.
La DE0‑nano offre un excellent terrain d’expérimentation pour ce type d’interface, et les modules SystemVerilog présentés ici constituent une base robuste pour aller plus loin (interface UART, affichage, menus interactifs, etc.).
