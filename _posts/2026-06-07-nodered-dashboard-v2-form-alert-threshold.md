---
title: "Node‑RED Dashboard V2 : un formulaire pour piloter un seuil d’alerte"
date: 2026-06-07
categories: ["Internet des Objets"]
tags: [IoT, "Node-RED"]
math: false
target_blank: true
---

### Dashboard V1 vs Dashboard V2

Je reviens sur mes petits travaux domotiques — par exemple lorsque j’affichais les [données d’un capteur de température et d’humidité LoRaWAN dans un tableau de bord Node‑RED]({{ site.baseurl }}{{ site.url }}/posts/lorawan-dashboard-nodered/#construction-du-tableau-de-bord){:target="_blank"}.

J’étais déjà assez fier de mon tableau de bord sous *Dashboard V1*, mais il faut reconnaître que cette version (basée sur AngularJS) commence à dater.

> Si vous découvrez Node‑RED, je vous conseille de lire d’abord mes [précédents billets]({{ site.baseurl }}{{ site.url }}/tags/node-red/){:target="_blank"} sur la construction du tableau de bord et la logique d’alerte. Celui‑ci s’adresse plutôt à ceux qui connaissent déjà les bases et veulent explorer les nouveautés de *Dashboard V2*.
{: .prompt-warning }

Je tente donc de rattraper mon retard avec *Dashboard V2*, ou [FlowFuse Dashboard](https://dashboard.flowfuse.com/){:target="_blank"}, une refonte complète reposant sur Vue.js — plus moderne, plus fluide, et surtout bien mieux adaptée aux écrans mobiles.

J’ai donc entrepris de reproduire mon tableau de bord existant avec *Dashboard V2* afin de profiter de cette nouvelle version pour obtenir quelque chose de plus propre, plus moderne et surtout plus agréable à utiliser sur smartphone.

![dashboard v1 vs v2](/assets/img/posts/2026-06-07-nodered-dashboard-v2-form-alert-threshold/dashboard-v1vsv2.png)
*À gauche Dashboard V1, à droite Dashboard V2*

Car c’est là que la différence se voit immédiatement : *Dashboard V2* est pensé mobile‑first. Les widgets s’adaptent mieux, les marges sont plus cohérentes, les formulaires sont plus lisibles, et l’ensemble donne une interface beaucoup plus fluide que celle de *Dashboard V1*.

![Dashboard V2 sur mobile](/assets/img/posts/2026-06-07-nodered-dashboard-v2-form-alert-threshold/dashboard-smartphone.png)
*Dashboard V2 sur mobile*

Dans un précédent billet, j'avais aussi ajouté une fonctionnalité supplémentaire : [faire une notification *push* (ntfy) sur mon smartphone lorsqu'un seuil de température haut était franchi]({{ site.baseurl }}{{ site.url }}/posts/nodered-ntfy/){:target="_blank"}.

Le problème, c’est que les paramètres de cette alerte étaient codés en dur dans le flow :

```javascript
// Seuil haut : déclenchement de l'alerte
let upper_threshold = 30.0;

// Seuil bas : réarmement (hysteresis)
let lower_threshold = 29.0;

...

msg.topic = "techfleb-xxxxxxxxxxxxxx"; // topic ntfy
```

Pas très pratique si l’on veut ajuster le seuil ou changer le topic sans modifier le code.
Je vais donc ajouter un petit formulaire de configuration dans *Dashboard V2* pour piloter ce seuil d’alerte en temps réel.

![formulaire configuration seuil d'alerte](/assets/img/posts/2026-06-07-nodered-dashboard-v2-form-alert-threshold/temperature-alert-settings-page.png)

### Construction du formulaire de configuration Dashboard V2

L’idée est simple : permettre d’ajuster le seuil haut, l’hystérésis, le topic ntfy et l’activation de l’alerte — le tout directement depuis l’interface, sans modifier le flow.

Le page se compose de quatre champs :
- **Upper threshold :** le seuil haut qui déclenche l’alerte, réglé via un *slider* (glissière) pour un ajustement rapide.
- **Hysteresis :** la marge de réarmement, saisie en valeur numérique.
- **Topic :** le topic [ntfy](https://ntfy.sh/){:target="_blank"} utilisé pour la notification.
- **Enabled :** un interrupteur pour activer/désactiver l’alerte.

Mais le nœud `form` ne propose pas de slider pour la saisie d'une valeur. On ajoute donc le slider comme widget séparé.

![nœud 'form'](/assets/img/posts/2026-06-07-nodered-dashboard-v2-form-alert-threshold/noeud-form.png)
*Le slider sera un widget séparé du noeud 'form'*

Le formulaire et le slider envoient chacun leur propre message : il faut donc les fusionner pour obtenir un seul objet de configuration exploitable dans le flow.
Nous verrons le flow complet juste après, mais voici déjà la structure JSON obtenue une fois les messages du `form` et du slider fusionnés :

```json
{
  "upper_threshold": 30,
  "form": {
    "hysteresis": 1,
    "topic_ntfy": "techfleb-xxxxx",
    "enabled": true
  }
}
```

### Présentation du flow

Voici maintenant le flow complet utilisé pour gérer la configuration du seuil d’alerte et la notification ntfy.

On y retrouve :
- le formulaire *Dashboard V2* nommé *Temperature Alert Settings Form*, qui envoie les paramètres saisis,
- le slider, ajouté comme widget séparé pour régler le seuil haut,
- deux nœuds `Change` qui définissent `msg.topic` pour préparer la fusion,
- un nœud `Join` qui assemble les messages du formulaire et du slider en un seul objet JSON. Il est configuré pour combiner chaque `msg.payload`, et créer un objet clé/valeur en utilisant la valeur du `msg.topic` comme clé.
- puis un dernier nœud `Change` qui extrait les valeurs du JSON fusionné et les stocke dans le contexte `flow` :
```javascript
flow.set("upper_threshold", msg.payload.upper_threshold);
flow.set("hysteresis", msg.payload.form.hysteresis);
flow.set("topic_ntfy", msg.payload.form.topic_ntfy);
flow.set("enabled", msg.payload.form.enabled);
```

![flow formulaire de configuration](/assets/img/posts/2026-06-07-nodered-dashboard-v2-form-alert-threshold/temperature-alert-settings-nodered-flow.png)
*Le flow complet : formulaire, slider, fusion JSON, stockage des paramètres et logique d’alerte.*

### Adaptation de la logique d’alerte

La logique d’alerte reste identique à celle présentée dans [mon billet précédent]({{ site.baseurl }}{{ site.url }}/posts/nodered-ntfy/){:target="_blank"}.
La seule différence est que les valeurs ne sont plus codées en dur : elles sont désormais lues depuis le contexte `flow`, alimenté par le formulaire *Dashboard V2*.

Dans les nœuds `function`, les constantes sont simplement remplacées.
Par exemple :
```javascript
let upper_threshold = 30;   // seuil haut;
```
devient :
```javascript
let upper_threshold = flow.get("upper_threshold");   // seuil haut;
```
etc.

Le code du nœud `Temperature Threshold (Hysteresis)` devient :
```javascript
// Seuil haut : déclenchement de l'alerte
let upper_threshold = flow.get("upper_threshold");   // seuil haut;
let hysteresis = flow.get("hysteresis");        // valeur d'hystérésis
let enabled = flow.get("enabled");           // activation ON/OFF

// Seuil bas : réarmement (hysteresis)
let lower_threshold = upper_threshold - hysteresis;

// Température en cours
let temperature = Number(msg.payload);

// État mémorisé : true si le seuil a déjà été franchi
let threshold_crossed = flow.get("threshold_crossed") || false;

// Si l’alerte est désactivée ==> on ne fait rien
if (!enabled) {
    return null;
}

// Si on était en dessous et qu'on passe au-dessus du seuil haut --> alerte
if (!threshold_crossed && temperature >= upper_threshold) {
    flow.set("threshold_crossed", true);
    return {
        alert: true,
        temperature: temperature
    };
}

// Si on était au-dessus et qu'on redescend sous le seuil bas --> réarmement
if (threshold_crossed && temperature <= lower_threshold) {
    flow.set("threshold_crossed", false);
}

// Sinon → aucune alerte
return null;
```

Et le code du nœud `Notification builder` devient :
```javascript
let t = msg.temperature;

msg.topic = flow.get("topic_ntfy");

msg.headers = {
    "Title": "Alerte température",
    "Priority": "4",
    "Tags": "thermometer"
};
msg.payload = `Température élevée : ${t}°C`;

return msg;
```

### Conclusion

Avec cette mise à jour, la configuration du seuil d’alerte devient entièrement dynamique : plus besoin de modifier le flow ou de redéployer Node‑RED pour ajuster le seuil haut, l’hystérésis, le topic ntfy ou l’activation de l’alerte.
Le *Dashboard V2* fournit une interface claire, le flow fusionne les paramètres du formulaire et du slider, et la logique d’alerte s’appuie désormais sur les variables du contexte `flow`.

Cette approche rend le système plus flexible, plus propre et surtout plus simple à utiliser au quotidien.
Il ne reste plus qu’à adapter cette méthode à d’autres types d’alertes ou de réglages — le principe reste exactement le même.