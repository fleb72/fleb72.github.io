---
title: "Node‑RED : envoyer une notification de température sur smartphone avec ntfy.sh"
date: 2026-05-14
categories: ["Internet des Objets"]
tags: [IoT, LoRa, LoRaWAN, "Node-RED"]
math: false
target_blank: true
---

Dans [le billet précédent]({{ site.baseurl }}{{ site.url }}/posts/lorawan-dashboard-nodered/){:target="_blank"}, nous avons affiché les données d’un capteur LoRaWAN dans un tableau de bord conçu avec Node‑RED.
Nous allons maintenant déclencher une notification sur smartphone lorsque la température dépasse un seuil — une méthode valable pour tout type de capteur.

Pour la notification, j’utiliserai l’application [ntfy](https://ntfy.sh/){:target="_blank"}, qui permet de recevoir des messages *push* depuis n’importe quelle source (Node‑RED, script, serveur, etc.). C’est gratuit, open‑source, et très simple à intégrer.

### Objectif : alerte de température critique

L’objectif est d’envoyer une notification uniquement lorsque la température dépasse un seuil critique, par exemple 30°C.
Mais il ne faut pas envoyer une alerte à chaque mesure supérieure à 30°C : une seule notification suffit pour signaler le dépassement.

Si la température oscille autour du seuil (29.8 → 30.1 → 29.9 → 30.2), on risque d’envoyer plusieurs alertes successives.
Pour éviter cela, on utilise une technique classique : l’**hystérésis**.

L’hystérésis consiste à utiliser deux seuils :
- `upper_threshold` – un seuil haut (ex. 30°C) pour déclencher l’alerte ;
- `lower_threshold` – un seuil bas (ex. 29°C) pour réarmer le système.

Tant que la température reste au-dessus de 29°C, aucune nouvelle alerte n’est envoyée.
Une nouvelle alerte ne sera possible qu’après être repassé sous 29°C.

C’est simple, efficace, et cela évite les alertes en rafale.

### Le flux Node-RED

Voici comment débuter le flux :

![flow Node-RED hysteresis](/assets/img/posts/2026-05-14-nodered-ntfy/flow-nodered-hysteresis.png)

Le nœud de type `function`, renommé *Temperature Threshold (Hysteresis)*, se charge de détecter le franchissement du seuil et d’émettre un message uniquement en cas d’alerte.
En entrée, on devrait normalement retrouver la température issue du capteur, mais pour mettre au point le flux j’ai utilisé une série de nœuds `inject` permettant d’envoyer des valeurs fictives d’un simple clic.

Le nœud suivant, également de type `function` et renommé *Notification builder*, construit la requête de notification qui sera envoyée vers *ntfy* grâce au dernier nœud *Notify* de type `http request`.

Ci‑dessous, le code du nœud *Temperature Threshold (Hysteresis)*, à saisir dans l’onglet *Message reçu* :

```javascript
// Seuil haut : déclenchement de l'alerte
let upper_threshold = 30.0;

// Seuil bas : réarmement (hysteresis)
let lower_threshold = 29.0;

// Température en cours
let temperature = Number(msg.payload);

// État mémorisé : true si le seuil a déjà été franchi
let threshold_crossed = flow.get("threshold_crossed") || false;

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
Ce nœud émet donc un message uniquement lorsque le seuil critique est franchi, en tenant compte de l’hystérésis.
Dans ce nœud, `flow.get()` et `flow.set()` servent à mémoriser un état entre deux messages. Contrairement aux variables locales du nœud `function`, qui disparaissent après chaque exécution, les valeurs stockées avec `flow.set("threshold_crossed")` restent disponibles pour les messages suivants.
Cela permet au flux de savoir si l’alerte a déjà été envoyée, et donc d’éviter les notifications répétées lorsque la température oscille autour du seuil.

On passe ensuite au nœud *Notification Builder*, dont voici le code à rédiger également dans l'onglet *Message reçu* :

```javascript
let t = msg.temperature;

msg.topic = "techfleb-xxxxxxxxxxxxxx"; // topic ntfy

msg.headers = {
    "Title": "Alerte température",
    "Priority": "4",
    "Tags": "thermometer"
};
msg.payload = `Température élevée : ${t}°C`;

return msg;
```

Le nom du *topic* est celui qu’il faudra saisir dans l’application *ntfy* du smartphone pour s’y abonner et recevoir les notifications associées.
Attention : les topics ne sont pas protégés. Choisissez un nom suffisamment long et difficile à deviner.
Ici, la notification sera envoyée avec la [priorité 4](https://docs.ntfy.sh/publish/#message-priority){:target="_blank"}, une priorité élevée qui déclenche une vibration prolongée du smartphone.

Enfin, le dernier nœud *ntfy* de type `http request` envoie la requête avec la méthode POST à l'URL {% raw %}`https://ntfy.sh/{{{topic}}}`{% endraw %}
où {% raw %}`{{{topic}}}`{% endraw %} sera automatiquement remplacé par la valeur de `msg.topic`.

![http request node](/assets/img/posts/2026-05-14-nodered-ntfy/http-request-node.png)

### Tests avec ntfy

Une fois l’application *ntfy* installée sur votre smartphone, il suffit de vous abonner au topic :

![Topic subscription](/assets/img/posts/2026-05-14-nodered-ntfy/topic-subscription.png)

Le flux Node‑RED étant déployé, vous pouvez tester son fonctionnement en injectant différentes valeurs de température.
Par exemple, avec la séquence 28.7 → 30.2 → 29.8 → 30.3, une seule notification devrait être envoyée :

![Alert ntfy](/assets/img/posts/2026-05-14-nodered-ntfy/alert-ntfy.png)

Une fois le fonctionnement validé, vous pouvez reconnecter l’entrée de votre flux Node‑RED au nœud du capteur qui fournit la température réelle. Le système enverra alors automatiquement une notification dès que le seuil critique sera franchi.

### Conclusion

Nous avons construit un flux Node‑RED modulaire :

- un nœud dédié à la logique d’hystérésis ;
- un nœud dédié à la construction de la notification ;
- un envoi dynamique vers *ntfy* via un topic configurable.

Cette architecture rend le système fiable, lisible et facilement extensible à d’autres capteurs ou d’autres types d’alertes.

Pour aller plus loin, vous pouvez ajouter plusieurs seuils (*warning* / *critical*) ou plusieurs capteurs.

Cette petite application open‑source, gratuite et sans inscription, est bien plus qu’un simple système de notifications : elle permet d’envoyer des messages depuis n’importe quel script, serveur ou objet connecté, de s’abonner à plusieurs topics, de définir des priorités, des pièces jointes, des actions, et même d’être auto‑hébergée pour un usage privé.

Toutes les possibilités sont détaillées dans la documentation officielle :
[https://docs.ntfy.sh](https://docs.ntfy.sh){:target="_blank"}.

> **En complément :**
>
> - [Node‑RED Dashboard V2 : un formulaire pour piloter un seuil d’alerte]({{ site.baseurl }}{{ site.url }}/posts/nodered-dashboard-v2-form-alert-threshold/){:target="_blank"}
{: .prompt-info}