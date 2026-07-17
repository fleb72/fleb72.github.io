---
title: "Quartus Prime Lite Edition 25.x : disparition silencieuse du simulateur gratuit"
date: 2026-07-16
categories: [FPGA]
tags: ["Quartus Prime", systemVerilog]
math: true
target_blank: true
---

J'ai voulu passer de *Quartus Prime Lite Edition 23.1* à *Quartus Prime Lite Edition 25.1*, la dernière version au moment où j'écris ce billet.
Voir [Altera Download Center](https://www.altera.com/downloads/fpga-development-tools/quartus-prime-lite-edition-design-software-version-25-1-windows/){:target="_blank"}.

C'est important de se mettre à jour, hein ?

Hé bien, si j'avais su...

Depuis que la division FPGA d’Intel est redevenue Altera en 2024–2025 — Intel ayant vendu 51 % de l’activité à Silver Lake tout en conservant 49 % — **les versions récentes de Quartus Prime Lite ont perdu leur simulateur gratuit Questa FPGA Starter Edition**.
La documentation mentionne encore une *Starter Edition*, et le portail de licences permet toujours d’en générer une, mais aucun installeur compatible n’est fourni. Les éditions 25.*x* et supérieures exigent désormais un serveur de licence **payant**.

![Questa Intel Starter FPGA Edition](/assets/img/posts/2026-07-16-altera-quartus-prime-lite-questa-license/questa-intel-starter-fpga-edition.png)
*Simulateur Questa Intel FPGA Starter Edition*


> La dernière version utilisable **gratuitement** pour la simulation avec *Questa FPGA Starter Edition* reste *Quartus Prime Lite 24.1*, publiée juste avant la transition vers Altera.
{: .prompt-warning }

Un changement majeur, non documenté, que beaucoup découvrent en tentant simplement de simuler leur design.

> En complément : [catégorie FPGA]({{ site.baseurl }}{{ site.url }}/categories/fpga/){:target="_blank"}
{: .prompt-info }