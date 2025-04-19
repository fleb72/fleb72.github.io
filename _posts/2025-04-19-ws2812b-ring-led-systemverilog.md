---
title: "Piloter un anneau de LEDs WS2812B adressables"
date: 2025-04-19
categories: [FPGA]
tags: [FPGA, systemVerilog]
math: false
target_blank: true
---

 Les *Arduinautes* connaissent bien ce genre de LEDs programmables que l'on retrouve souvent sous la forme de [rubans souples de longueur 1m, 3m, ... 10m et plus](https://duckduckgo.com/?t=h_&q=ruban+led+ws2812b&iax=shopping&ia=shopping){:target="_blank"}, et que l'on utilise pour créer des effets lumineux personnalisés. Voir les bibliothèques [FastLED](https://docs.arduino.cc/libraries/fastled/){:target="_blank"} ou [NeoPixel par Adafruit](https://docs.arduino.cc/libraries/adafruit-neopixel/){:target="_blank"}.

Pour la démonstration, j'utiliserai un anneau de 12 LEDs WS2812B (quelques euros l'anneau en cherchant bien), mais que je connecterai à une carte FPGA [DE0-Nano](https://www.intel.com/content/www/us/en/partner/showcase/offering/a5b3b0000004cb9AAA/de0nano-development-and-education-board.html){:target="_blank"}.

![DE0-Nano et anneau 1 LEDs WS2812B](/assets/img/posts/2025-04-19-ws2812b-ring-led-systemverilog/20230802_130816.jpg)
*La carte FPGA, en grand seigneur des ténèbres, contrôle les pouvoirs de l'anneau Unique forgé par Sauron...*

### Principe

Ces rubans ou anneaux de LEDs sont constitués de LEDs multicolores RGB (*Red-Green-Blue*) adressables connectées en cascade, c’est à dire que l’on peut définir la luminosité et la couleur de chaque LED indépendamment. Le contrôleur intégré à chaque LED récupère et traite les données série en entrée (sur un seul fil) pour allumer sa LED ou communiquer les données à la LED suivante en sortie selon un timing et un protocole série bien précis.

![LEDs WS2812B en cascade](/assets/img/posts/2025-04-19-ws2812b-ring-led-systemverilog/cascad-method.jpg)
*Principe avec trois LEDs en cascade d'après WS2812B datasheet*

### Protocole de transmission série

Au niveau des données à communiquer, c'est très simple, la couleur de la LED est définie par les 24 bits des composantes Rouge-Vert-Bleu selon le schéma ci-dessous :


![1 LED, 24 bits](/assets/img/posts/2025-04-19-ws2812b-ring-led-systemverilog/data-24bits.jpg)
*D'après WS2812B datasheet*

Le bit de poids fort est transmis en premier, avec les couleurs suivant l'ordre : *Green* (G7 à G0), *Red* (R7 à R0) puis *Blue* (B7 à B0). C'est tout...

La transmission en cascade est aussi simple sur le principe. Au démarrage (ou après un signal *Reset*), le contrôleur de la LED PIX1 se sert des 24 premiers bits reçus pour allumer sa LED à la bonne couleur. Les signaux des 24 bits suivants seront reconditionnés par PIX1 et transmis sur sa sortie vers PIX2 qui s'en servira pour allumer sa LED PIX2. Et on poursuit... PIX1 reçoit les 24 bits suivants, qui seront transmis à PIX2, et PIX2 transmettra à PIX3 pour allumer sa LED PIX3, etc.
Chaque LED reçoit donc 24 bits de données et transmet le reste des données à la LED suivante. Pour piloter les 12 LEDs de l'anneau, il faut donc transmettre 12 x 24 = 288 bits. Pour redémarrer le cycle à partir de la première LED, on envoie un signal *Reset* et on recommence.

### Les signaux physiques

Comment générer physiquement des `0` et des `1` sur un fil ? En suivant le codage des diagrammes suivants, extraits de la documentation :

![Signaux physiques](/assets/img/posts/2025-04-19-ws2812b-ring-led-systemverilog/sequence-chart.jpg)
*D'après WS2812B datasheet
T0H = 0,4 µs, T0L = 0,85 µs, T1H = 0,8 µs, T1L = 0,45 µs, avec une tolérance ± 150 ns.
Treset doit être supérieur à 50 µs.*



Ce codage consiste donc à envoyer des impulsions électriques de durée variable pour représenter les bits `0` ou `1`. Un état haut pendant 0,4 µs suivi d'un état bas pendant 0,85 µs représente un bit `0`. Un état haut pendant 0,8 µs suivi d'un état bas pendant 0,45 µs représente un bit `1`.
Ainsi, la durée de transmission d'un bit est : TH + TL = 1,25 µs ±300 ns.

### Le contrôleur FPGA

 Le contrôleur à implémenter dans la puce FPGA est donc le circuit qui génèrera en sortie le signal à transmettre au ruban ou à l'anneau.

![Contrôleur WS2812B](/assets/img/posts/2025-04-19-ws2812b-ring-led-systemverilog/rtl-view-ws2812b-controller.jpg)
*Contrôleur WS2812B*

- Entrées :

    - `clk` : à connecter à l'horloge principale 50 MHz. La durée d'un bit correspond à 60 périodes de l'horloge (60 / 50.106 = 1,2 µs).
    address[7..0] : bus 8 bits, adresse de la LED en cours. La 1re LED est à l'adresse 0.
    - `red[7..0]`, `green[7..0]`, `blue[7..0]` : bus 8 bits, composantes rouge, verte et bleue de la LED en cours.
    - `load` : lorsque cette entrée est active, et sur front montant de l'horloge, un registre interne est chargé avec les composantes rouge, verte et bleue présentes en entrée pour la LED dont l'adresse est présente en entrée.
    - `latch_n` : entrée active à l'état bas, le registre interne est déverrouillé sur front descendant de l'horloge, et les données série sont transmises en sortie. 

- Sortie :

    - `data_ws2812b` : les données série à transmettre en entrée du ruban ou de l'anneau. La transmission débute au front montant de l'horloge qui suit le front descendant de l'entrée latch_n. Une fois les 12 x 24 bits transmis, la sortie est à l'état bas. Un signal Reset est donc transmis si l'état bas est maintenu pendant au moins 50 µs.

 Ci-dessous, on montre des chronogrammes obtenus par simulation avec seulement deux LEDs :

 ![Simulation contrôleur WS2812B](/assets/img/posts/2025-04-19-ws2812b-ring-led-systemverilog/waveform.jpg)

- 1re LED à l'adresse 0 : Rouge=0x33, Vert=0x44, Bleu=0x55
- 2nd LED à l'adresse 1 : Rouge=0x66, Vert=0x77, Bleu=0x88


On voit le chargement du registre interne `colors` (2 x 24 bits) sur front montant de l'horloge lorsque l'entrée `load` est active.
Une fois le registre chargé, les données commencent à être transmises, un cycle d'horloge après le front descendant du signal `latch_n`.

Si on fait un zoom arrière sur ces chronogrammes, on visualise le début de la trame envoyée (les huit premiers bits) juste après l'impulsion du signal `latch_n` (entourée en rouge) :

 ![Simulation contrôleur WS2812B, zoom](/assets/img/posts/2025-04-19-ws2812b-ring-led-systemverilog/waveform2.jpg)
*Le premier octet transmis : 0x44, la composante verte de la 1re LED à l'adresse 0*

Voici le code commenté du contrôleur en langage systemVerilog :

- **ws2812b_controller.sv** :

```verilog
module ws2812b_controller #(
    parameter int NB_LEDS = 12,	// Nbre Leds du ruban
    parameter int CLK_MHZ = 50	// Fréquence horloge en MHz
) (
    input logic clk,              // Horloge
    input logic [7:0] red,
    input logic [7:0] green,
    input logic [7:0] blue,
    input logic [7:0] address,    // Numéro de la LED entre 0 et NB_LEDS-1
    input logic load,             // Chargement du registre interne (colors)
    input logic latch_n,          // Déverrouille les données et débute la transmission sur front descendant
    output logic data_ws2812b	// Sortie série
);

    logic [24 * NB_LEDS - 1 : 0] colors = 0;
    logic latch_n_previous = 1;

    localparam int T_HIGH  = 0.4 * CLK_MHZ;  // 20 périodes à 50 MHz = 20 x 20ns = 0,4 us
    localparam int T_LOW   = 0.8 * CLK_MHZ;  // 40 périodes à 50 MHz = 40 x 20ns = 0,8 us
    localparam int T       = T_HIGH + T_LOW;

    logic [$clog2(T) - 1:0] t_counter = 0;	// Compteur pour le temps

    logic [$clog2(24 * NB_LEDS) - 1 : 0] rgb_data_index; // Indice de position entre 0 et 24*NB_LEDS-1
    
    logic load_state, transfer_state;
    logic transfer_state_reg = 0;
    assign load_state = (load == 1) && (address < NB_LEDS);
    assign transfer_state = (transfer_state_reg == 1) && (!load_state);

    always_ff @(posedge clk) begin
        if (load_state) begin	// Chargement du registre 24 bits
            colors[(24 * (NB_LEDS - address) - 1) -: 24] <= {green, red, blue};
        end
    end

    always_ff @(posedge clk) begin
        if (transfer_state) begin	// Si transfert en cours
            t_counter <= t_counter - 1;

            if (t_counter == 0) begin	// Si bit transmis, préparer le bit suivant
                t_counter <= T;
                rgb_data_index <= rgb_data_index - 1;

                if (rgb_data_index == 0) begin	// si dernier bit transmis
                    t_counter <= T;
                    rgb_data_index <= 24 * NB_LEDS - 1;
                    transfer_state_reg <= 0;
                end
            end
        end else if (!latch_n && (latch_n != latch_n_previous)) begin	// Si déverrouillage
            rgb_data_index <= 24 * NB_LEDS - 1;
            t_counter <= T;
            transfer_state_reg <= 1;	// commencer le transfert vers la sortie
        end

        latch_n_previous <= latch_n;
    end

    assign data_ws2812b = (transfer_state == 1) && (colors[rgb_data_index] ? (t_counter > (T - T_LOW)) : (t_counter > (T - T_HIGH)));

endmodule
```

### Exemple : une roue multicolore

Pour animer l'anneau de LED, il faut maintenant mettre en œuvre la description principale (fichier `top.sv` ci-dessous) qui instancie le contrôleur, raccorde ses entrées à des signaux de contrôle et sa sortie vers l'entrée `IN` de l'anneau (lignes 24 à 29). Pour cette démonstration, le code ci-dessous va charger les couleurs initiales de chacune des 12 LEDs (lignes 34 à 47) puis, à intervalle régulier, on décale le motif initial d'une LED (lignes 61 à 64) pour donner l'illusion d'une roue multicolore qui se met à tourner.

- **top.sv** :

```verilog
module top (
    input logic CLOCK_50,         // Horloge 50MHz
    output logic GPIO_WS2812B     // À relier à l'entrée IN de l'anneau
);

    localparam NB_LEDS = 12;  
    localparam logic [7:0] ON  = 8'h20;  // Luminosité max, attention à la consommation
    localparam logic [7:0] OFF = 8'h00;

    logic [7:0] red, green, blue;
    logic [7:0] address;      // Numéro de la LED entre 0 et NB_LEDS-1
    logic load;               // Chargement du registre si load=1
    logic latch_n;            // Déverrouillage et transfert du registre sur front descendant

    logic [$clog2(NB_LEDS) - 1:0] num_led = 0;    // Numéro de led en cours
  
    logic [23:0] rgb;
    logic [25:0] delay = 0;

    assign red    = rgb[23-:8]; // Bits 23 à 16
    assign green  = rgb[15-:8]; // Bits 15 à 8
    assign blue   = rgb[7-:8];  // Bits 7 à 0

    // Instanciation du contrôleur WS2812B
    ws2812b_controller #(.NB_LEDS(NB_LEDS)) ws2812b_controller_inst (
        .clk(CLOCK_50),
        .data_ws2812b(GPIO_WS2812B),
      .*
    );

    integer i;
    logic [23:0] led[0:NB_LEDS-1]; // État des LEDs

    initial begin
        led[0]   = {ON, OFF, OFF}; // Led 0, Rouge
        led[1]   = {OFF, ON, OFF}; // Led 1, Vert
        led[2]   = {OFF, OFF, ON}; // Led 2, Bleu
        led[3]   = {ON, ON, OFF}; // Led 3, Jaune
        led[4]   = {ON, OFF, ON}; // Led 4, Violet
        led[5]   = {OFF, ON, ON}; // Led 5, Cyan
        led[6]   = {ON, ON, ON}; // Led 6, Blanc
        led[7]   = {OFF, ON, ON}; // Led 7, Cyan
        led[8]   = {ON, OFF, ON}; // Led 8, Violet
        led[9]   = {ON, ON, OFF}; // Led 9, Jaune
        led[10]  = {OFF, OFF, ON}; // Led 10, Bleu
        led[11]  = {OFF, ON, OFF}; // Led 11, Vert
    end

    // Assignation des sorties
    assign load = (num_led < NB_LEDS);
    assign address = load ? num_led : 0;
    assign rgb = load ? led[num_led] : 0;
    assign latch_n = ~(num_led == NB_LEDS);

    // Chargement & rotation de la roue
    always_ff @(posedge CLOCK_50) begin
      if (num_led <= NB_LEDS) begin  // chargement led par led
        num_led <= num_led + 1;
      end else if (num_led == NB_LEDS+1) begin
        if (delay[23] == 1) begin // fin de la temporisation
          led[0] <= led[NB_LEDS - 1];
          for (i = 1; i <= NB_LEDS-1; i = i + 1) begin
            led[i] <= led[i-1]; // simule la rotation de la roue
          end
          num_led <= 0;
          delay <= 0;
        end else begin
          delay <= delay + 1;
        end
      end
    end

endmodule
```

{% include embed/youtube.html id='G7Js7aaKNZ8' %}

La vidéo filmée avec mon smartphone ne rend pas bien les couleurs, mais je suis quand même très fier du résultat ;-)