---
title: "InfluxDB pour les développeurs SQL : petit guide de transition vers le langage Flux"
date: 2025-08-26
categories: ["Internet des Objets"]
tags: [IoT]
math: true
target_blank: true
---

### Présentation d'InfluxDB

[InfluxDB](https://docs.influxdata.com/){:target="_blank"} est un système de gestion de base de données spécialisé dans **les séries temporelles** — c’est-à-dire des données horodatées, comme les relevés de capteurs (température, humidité...), les métriques système, etc. Contrairement aux SGBD relationnels (comme SQL Server, MySQL ou PostgreSQL), InfluxDB est conçu pour ingérer, stocker et interroger des flux de données chronologiques à haute fréquence.
Il est très pratique avec son langage de requêtes pour manipuler des flux de données dans le temps : interroger les dernières 24h, faire des moyennes sur des tranches horaires, etc. 

Le problème est qu'InfluxDB a sorti trois versions 1.x, 2.x et maintenant 3.x, avec à chaque fois des langages différents.

- [InfluxDB v1](https://docs.influxdata.com/influxdb/v1/introduction/download/){:target="_blank"} est toujours disponible sur le site d'InfluxData, mais la maintenance est minimale et il ne présente plus de nouvelles fonctionnalités. Le langage de manipulation des données est [InfluxQL](https://docs.influxdata.com/influxdb/v1/query_language/){:target="_blank"}, un langage simple qui se veut proche de SQL, mais qui reste limité.

- [InfluxDB v2](https://docs.influxdata.com/influxdb/v2/get-started/){:target="_blank"} est toujours maintenu, même s'il ne recevra plus d'innovations majeures. La version 2.7.12 est la dernière à ce jour et est sortie en mai 2025. Le langage de manipulation de données se nomme [Flux](https://docs.influxdata.com/flux/v0/get-started/){:target="_blank"}, un langage de script fonctionnel puissant, mais qui ressemble à SQL de très loin.

- [InfluxDB v3](https://docs.influxdata.com/influxdb3/core/){:target="_blank"} a opéré un revirement stratégique avec un retour au langage [SQL](https://docs.influxdata.com/influxdb3/core/query-data/){:target="_blank"}, finalement bien plus familier des développeurs.


Encore et toujours avec un wagon de retard, je travaille actuellement avec la version 2.7.11, bien suffisante pour mes petits besoins, et toujours d'actualité, même si InfluxData recommande évidemment aujourd'hui le passage à InfluxDB v3.

### Le langage fonctionnel Flux d'InfluxDB v2

Prenons comme base la configuration utilisée dans [mon projet LoRaWAN]({{ site.baseurl }}{{ site.url }}/posts/architecture-lorawan-cloud-native/){:target="_blank"}, qui stocke les mesures de température et d’humidité relevées par une sonde SHT31. En Python (grâce à la bibliothèque [influxdb-client](https://pypi.org/project/influxdb-client/){:target="_blank"}), un enregistrement est effectué à intervalles réguliers :

```python
    ...
    # Création du point
    point = Point("lora_data") \
        .tag("device", "heltec-v3-perso") \
        .field("temperature", round(float(temperature), 1)) \
        .field("humidity", round(float(humidity), 1))
    ...
```

Lors de l'insertion, un champ spécial `time` d'horodatage est automatiquement ajouté à la création de l'enregistrement.

- Un *tag* "device" (une métadonnée de l'enregistrement) est un élément indexé qui permet de filtrer ou regrouper les séries (par capteur, par localisation, etc.). Chaque combinaison différente de tags crée une série distincte : le capteur de température de la chambre, le capteur de température du salon, le capteur de pression dans le jardin, etc.

- "temperature" et "humidity" sont des *fields* pour stocker les valeurs mesurées.

Ce *point* qui fait partie d'une série va être enregistré dans un *measurement*. Disons une table si on fait le parallèle avec SQL.

Les *measurements* sont regroupés dans un *bucket*, le conteneur global.

Schématiquement, cela donne :
```
Bucket
  └── Measurement ("lora_data")
        └── Series ("device" = "heltec-v3-perso")
              └── Point (fields + time)
```

Mon *bucket* ne contient qu'un seul *measurement*, nommé "lora_data", et celui-ci ne contient qu'une seule *serie* (car je n'ai qu'un seul capteur).

Ce modèle permet d’interroger facilement les données par capteur, par période, ou par type de mesure, tout en assurant une grande efficacité en écriture et en lecture.

Pour lancer quelques requêtes en langage Flux avec l'interface d'InfluxDB en ligne de commande, je passe par un petit script exécutable : 

**run_query_influxdb.sh**
```bash
#!/bin/bash

source .env
influx query \
  --host $INFLUXDB_URL \
  --org $INFLUXDB_ORG \
  --token $INFLUXDB_TOKEN \
  --file querytest.flux
```
Les identifiants et *token* de connexion sont dans des variables d'environnement dans un fichier *.env*.

La requête est écrite dans un fichier *.flux*, par exemple :

**querytest.flux**
```
from(bucket: "lora-ttn")
  |> range(start: -2h)
  |> filter(fn: (r) => r._measurement == "lora_data")
  |> keep(columns: ["_time", "_field", "_value", "device"])
```

Et on lance l'exécution du script avec la commande `./run_query_influxdb.sh`, ce qui me renvoie les données suivantes :

```
Result: _result
Table: keys: [_field, device]
         _field:string           device:string                      _time:time        _value:float
----------------------  ----------------------  ------------------------------  ------------------
              humidity         heltec-v3-perso  2025-08-29T14:42:21.774570306Z                65.7
              humidity         heltec-v3-perso  2025-08-29T14:54:19.728220928Z                65.3
              humidity         heltec-v3-perso  2025-08-29T15:06:18.632352441Z                61.8
              humidity         heltec-v3-perso  2025-08-29T15:18:17.004567109Z                63.7
              humidity         heltec-v3-perso  2025-08-29T15:30:15.959644094Z                59.8
              humidity         heltec-v3-perso  2025-08-29T15:42:23.182885912Z                  61
              humidity         heltec-v3-perso  2025-08-29T15:54:12.147956696Z                60.6
              humidity         heltec-v3-perso  2025-08-29T16:06:10.564498597Z                60.4
              humidity         heltec-v3-perso  2025-08-29T16:18:08.341589471Z                58.2
              humidity         heltec-v3-perso  2025-08-29T16:30:07.798405170Z                57.4
Table: keys: [_field, device]
         _field:string           device:string                      _time:time        _value:float
----------------------  ----------------------  ------------------------------  ------------------
           temperature         heltec-v3-perso  2025-08-29T14:42:21.774570306Z                21.8
           temperature         heltec-v3-perso  2025-08-29T14:54:19.728220928Z                22.2
           temperature         heltec-v3-perso  2025-08-29T15:06:18.632352441Z                22.7
           temperature         heltec-v3-perso  2025-08-29T15:18:17.004567109Z                  22
           temperature         heltec-v3-perso  2025-08-29T15:30:15.959644094Z                23.7
           temperature         heltec-v3-perso  2025-08-29T15:42:23.182885912Z                22.3
           temperature         heltec-v3-perso  2025-08-29T15:54:12.147956696Z                21.9
           temperature         heltec-v3-perso  2025-08-29T16:06:10.564498597Z                22.4
           temperature         heltec-v3-perso  2025-08-29T16:18:08.341589471Z                23.2
           temperature         heltec-v3-perso  2025-08-29T16:30:07.798405170Z                23.2
```

### Comment lire une requête Flux

Reprenons la dernière requête Flux :
```
from(bucket: "lora-ttn")
  |> range(start: -2h)
  |> filter(fn: (r) => r._measurement == "lora_data")
  |> keep(columns: ["_time", "_field", "_value", "device"])
```
Et décortiquons cette requête ligne par ligne :

- `from(bucket: "lora-ttn")` : la source de données est le *bucket* "lora-ttn".

- `|> range(start: -2h)` : la taille de la fenêtre temporelle est limitée, en demandant seulement les données des deux dernières heures.

- `|> filter(fn: (r) => r._measurement == "lora_data")` : filtre pour des mesures spécifiques. Ici, on ne restitue que les données dont la mesure (*measurement*) porte le nom "lora_data".

- `|> keep(columns: ["_time", "_field", "_value", "device"])` : réduire le nombre de colonnes à l'essentiel - `_time` pour l'horodatage de la mesure, `_field` pour le nom du champ mesuré (ex. température, humidité…), `_value` pour la valeur mesurée, `device` pour l'identifiant du capteur ou module LoRa.

Le symbole `|>` dans Flux est l’opérateur de pipeline, et il fonctionne un peu comme le `|` en bash. Il permet de chaîner les opérations les unes après les autres, en passant les résultats d’une étape à la suivante.

La partie `fn: (r) => ...` évoque une fonction (anonyme ou *lambda*) avec le paramère `r`, le plus souvent une ligne (*row*) du tableau de données.

En SQL, on aurait un équivalent qui ressemblerait à :
```sql
SELECT time AS _time, field AS _field, value AS _value, device
FROM lora_data
WHERE time >= NOW() - INTERVAL '2 hours'
ORDER BY time DESC;
```

### Agrégation temporelle de données avec Flux

Soit la requête :
```
    from(bucket: "lora-ttn")
      |> range(start: -6h)
      |> filter(fn: (r) => r._measurement == "lora_data")
      |> filter(fn: (r) => r._field == "temperature" or r._field == "humidity")
      |> aggregateWindow(every: 1h, fn: mean, createEmpty: false)
      |> keep(columns: ["_time", "_field", "_value"])
```

Au résultat :
```
Result: _result
Table: keys: [_field]
         _field:string                      _time:time                  _value:float
----------------------  ------------------------------  ----------------------------
              humidity  2025-08-29T12:00:00.000000000Z             62.56666666666666
              humidity  2025-08-29T13:00:00.000000000Z            56.120000000000005
              humidity  2025-08-29T14:00:00.000000000Z                         70.12
              humidity  2025-08-29T15:00:00.000000000Z                          66.6
              humidity  2025-08-29T16:00:00.000000000Z             61.38000000000001
              humidity  2025-08-29T17:00:00.000000000Z             58.32000000000001
              humidity  2025-08-29T17:23:49.397923842Z                         58.45
Table: keys: [_field]
         _field:string                      _time:time                  _value:float
----------------------  ------------------------------  ----------------------------
           temperature  2025-08-29T12:00:00.000000000Z            22.366666666666664
           temperature  2025-08-29T13:00:00.000000000Z                          24.6
           temperature  2025-08-29T14:00:00.000000000Z                         19.72
           temperature  2025-08-29T15:00:00.000000000Z            21.259999999999998
           temperature  2025-08-29T16:00:00.000000000Z                         22.52
           temperature  2025-08-29T17:00:00.000000000Z            22.919999999999998
           temperature  2025-08-29T17:23:49.397923842Z                         22.35
```

La ligne `|> aggregateWindow(every: 1h, fn: mean, createEmpty: false)` est la fonction d'agrégation temporelle. Elle regroupe les données par fenêtres de temps (ici, des tranches d’1 heure) et applique une fonction d’agrégation (ici, la moyenne). Avec `createEmpty: false`, on ignore les tranches sans données.

En résumé, cette requête va regrouper les points par tranches horaires (ex. 14h–15h, 15h–16h…), calculer la moyenne de chaque champ (ex. température, humidité) dans chaque tranche, mais en ignorant les tranches sans données (grâce à `createEmpty: false`).

Un peu plus longue, la requête suivante va compter le nombre de jours à forte chaleur (≥ 30°C) par semaine entre deux dates :

```
from(bucket: "lora-ttn")
  |> range(start: 2025-08-06T00:00:00Z, stop: 2025-08-30T00:00:00Z)
  |> filter(fn: (r) => r._measurement == "lora_data" and r._field == "temperature" and r.device == "heltec-v3-perso")
  |> aggregateWindow(every: 1d, fn: max, createEmpty: false)
  |> filter(fn: (r) => r._value >= 30.0)
  |> aggregateWindow(every: 1w, fn: count, createEmpty: true)
  |> fill(column: "_value", value: 0)
  |> keep(columns: ["_time", "_field", "_value"])
```
Avec le résultat :

```
Result: _result
Table: keys: [_field]
         _field:string                      _time:time        _value:int
----------------------  ------------------------------  ----------------
           temperature  2025-08-07T00:00:00.000000000Z                 0
           temperature  2025-08-14T00:00:00.000000000Z                 5
           temperature  2025-08-21T00:00:00.000000000Z                 5
           temperature  2025-08-28T00:00:00.000000000Z                 1
           temperature  2025-08-30T00:00:00.000000000Z                 0
```
Avec l'option `createEmpty: true` et l'instruction `fill(column: "_value", value: 0)`, on prend en compte les semaines sans fortes chaleurs. 11 journées chaudes quand même, *soif*...


Les développeurs SQL sentent les opérations `COUNT` et `AVG` avec `GROUP BY`, mais des fonctions anonymes, des `filter`, cela ressemble davantage à de la programmation fonctionnelle, non ? Pourquoi pas des `map` ou des `reduce` tant qu'on y est...

### Du SQL à la programmation fonctionnelle

... Puisque vous le demandez, prenez la requête Flux suivante :

```
from(bucket: "lora-ttn")
  |> range(start: -14d)
  |> filter(fn: (r) => r._measurement == "lora_data" and r._field == "temperature" 
                                         and r.device == "heltec-v3-perso")
  |> aggregateWindow(every: 1d, fn: max, createEmpty: false)
  |> map(fn: (r) => ({r with tempLevel: if r._value < 15.0 then "froid"
                                        else if r._value < 30.0 then "modéré"
                                        else "chaud"
                     }))
  |> keep(columns: ["_time", "_field", "_value", "tempLevel"])
```

Requête qui renvoie le résultat suivant :

```
Result: _result
Table: keys: [_field]
         _field:string                      _time:time        _value:float        tempLevel:string
----------------------  ------------------------------  ------------------  ----------------------
           temperature  2025-08-17T00:00:00.000000000Z                29.1                modéré
           temperature  2025-08-18T00:00:00.000000000Z                29.8                modéré
           temperature  2025-08-19T00:00:00.000000000Z                31.1                   chaud
           temperature  2025-08-20T00:00:00.000000000Z                26.6                modéré
           temperature  2025-08-21T00:00:00.000000000Z                21.8                modéré
           temperature  2025-08-22T00:00:00.000000000Z                25.5                modéré
           temperature  2025-08-23T00:00:00.000000000Z                24.6                modéré
           temperature  2025-08-24T00:00:00.000000000Z                25.6                modéré
           temperature  2025-08-25T00:00:00.000000000Z                27.1                modéré
           temperature  2025-08-26T00:00:00.000000000Z                29.9                modéré
           temperature  2025-08-27T00:00:00.000000000Z                30.1                   chaud
           temperature  2025-08-28T00:00:00.000000000Z                29.5                modéré
           temperature  2025-08-30T00:00:00.000000000Z                25.9                modéré
           temperature  2025-08-30T17:23:24.295947920Z                25.4                modéré
```

Avec `map` on applique une fonction pour chaque ligne `r` du tableau. Et avec `{ r with ... }`, on renvoie une nouvelle version de `r`, enrichie d'une nouvelle colonne calculée `tempLevel`.

Plutôt que de chercher un équivalent en SQL, on ferait mieux de se mettre à la programmation fonctionnelle, par exemple en Python :

```python
data = [
    {"_value": 31.5},
    {"_value": 26.2},
    {"_value": 14.5}
]

def classify_temperature(r):
    if r["_value"] < 15.0:
        r["tempLevel"] = "froid"
    elif r["_value"] < 30.0:
        r["tempLevel"] = "modéré"
    else:
        r["tempLevel"] = "chaud"
    return r

result = list(map(classify_temperature, data))
```

### Conclusion

La syntaxe de Flux est inspirée de la programmation fonctionnelle avec des opérateurs comme `map()`, `filter()`, `reduce()` ou `aggregateWindow()`. Flux permet de construire des pipelines évolués de requêtes finalement assez lisibles avec l'habitude.

Pour autant, j'aurais beaucoup de mal à traduire certaines requêtes SQL en Flux : requêtes avec sous-requêtes, agégations conditionnelles, etc.

Si la logique orientée flux vous semble trop déroutante, il vous reste à passer à la version 3 d'InfluxDB avec son langage SQL orienté bases de données temporelles ;-)