---
title: "Affichage en temps réel des mesures de température et humidité du capteur Zigbee SNZB-02P avec FastAPI et MQTT"
date: 2025-06-12
categories: ['Raspberry Pi', 'domotique']
tags: ['IoT', 'Zigbee2MQTT', 'MQTT', 'Zigbee', 'Python']
math: false
target_blank: true
---

Encore et toujours avec [mon réseau de capteurs/actionneurs Zigbee et mon dongle Zigbee2MQTT]({{ site.baseurl }}{{ site.url }}/tags/zigbee2mqtt/){:target="_blank"}, il s'agit cette fois de servir une page Web avec les données renvoyées par un capteur Zigbee de température et humidité (réf. : [Sonoff SNZB-02P]({{ site.baseurl }}{{ site.url }}/posts/raspberry-pi-iot-zigbee2mqtt/){:target="_blank"}). 

![Page web capteur SNZB-02P](/assets/img/posts/2025-06-12-fastapi-mqtt-snzb02p-zigbee-sensor/pageweb-capteur-snzb02p.jpg)

Bien entendu, le contenu de la page est mis à jour en temps réel et sans intervention de l'utilisateur à chaque fois que le capteur publie de nouvelles données.

Pour servir la page Web, j'utiliserai le langage Python et le framework [FastAPI](https://fastapi.tiangolo.com/){:target="_blank"}. En complément, le projet [fastapi-mqtt](https://github.com/sabuhish/fastapi-mqtt){:target="_blank"} gèrera la communication asynchrone pour MQTT.

### Le code

Le code Python du fichier *main.py* ci-dessous configure un serveur FastAPI pour récupérer les données du capteur Zigbee SNZB-02P via MQTT.

Grâce à *fastapi-mqtt*, le topic `zigbee2mqtt/fleb-SNZB-02P` est sur écoute, et on peut récupèrer les informations envoyées par le capteur (température, humidité, et heure de dernière mise à jour).
Les données du capteur sont stockées dans un objet `SensorSNZB02P`, ce qui permet de les mettre à jour à chaque nouveau message MQTT publié sur le topic.

Les données sont envoyées aux clients connectés via WebSocket, pour un affichage en temps réel. Une route FastAPI `/ws` gère les connexions WebSocket des clients et leur permet de recevoir les mises à jour automatiquement.

Enfin, le programme sert un fichier HTML (*test.html*) pour l'affichage des données sur une interface web.

**main.py :**
```python
from fastapi import FastAPI, WebSocket
from fastapi.responses import FileResponse
from fastapi_mqtt import FastMQTT, MQTTConfig

import os

app = FastAPI()

# Configuration MQTT
mqtt_config = MQTTConfig(
    host="192.168.0.40", # @IP du broker
    port=1883,
    keepalive=60
)

TOPIC_MQTT = "zigbee2mqtt/fleb-SNZB-02P"

fast_mqtt = FastMQTT(config=mqtt_config)
fast_mqtt.init_app(app)

# Stockage des données du capteur
class SensorSNZB02P:
    def __init__(self):
        self.data = {}

    def update(self, new_data):
        self.data = {
            "temperature": new_data["temperature"],
            "humidity": new_data["humidity"],
            "last_seen": new_data.get("last_seen"), # pas toujours présent
        }

    def get_data(self):
        return self.data

my_sensor = SensorSNZB02P()

websockets = []  # Stockage des connexions WebSocket

@fast_mqtt.subscribe(TOPIC_MQTT)
async def message_handler(client, topic, payload, qos, properties):
    data = payload.decode("utf-8")

    # Filtrer et extraire les champs requis
    import json
    new_data = json.loads(payload.decode("utf-8"))
    my_sensor.update(new_data)
       
    # Envoyer les données aux clients WebSocket
    for ws in websockets:
        await ws.send_json(my_sensor.get_data())

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    websockets.append(websocket)
    try:
        while True:
            await websocket.receive_text()  # Garde la connexion ouverte
    except Exception:
        websockets.remove(websocket)  # Supprimer la connexion de la liste si elle se ferme

@app.get("/")
async def serve_html():
    return FileResponse(os.path.join(os.getcwd(), "test.html"))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

**test.html :**
```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <title>Température et humidité</title>
</head>
<body>
    <h2>Données du capteur SNZB-02P</h2>
    <p>Température : <span id="temperature">--.-</span>°C</p>
    <p>Humidité : <span id="humidity">--.-</span>%</p>
    <p>Dernière mise à jour : <span id="last_seen">------------------------</span></p>
   
    <script>
        // Connexion WebSocket avec le serveur FastAPI
        const ws = new WebSocket("ws://192.168.0.21:8000/ws");
        
        // Réception des données envoyées par le serveur et mise à jour de la page Web
        ws.onmessage = (event) => {
            const data = JSON.parse(event.data);
            document.getElementById("temperature").innerText = data.temperature;
            document.getElementById("humidity").innerText = data.humidity;
            document.getElementById("last_seen").innerText = data.last_seen;
        };
    </script>
    
</body>
</html>
```

### Analyse des trames réseau

Pour comprendre les échanges, je vous montre des copies d'écran de captures de trames réseau avec Wireshark.

Les trois acteurs de mon réseau local pour cette démonstration sont :
- à l'adresse 192.168.0.16, le client HTTP avec un navigateur ;
- à l'adresse 192.168.0.21, le serveur ASGI (*Asynchronous Server Gateway Interface*) créé par FastAPI, et qui servira la page Web aux clients, mais aussi le client MQTT ;
- à l'adresse 192.168.0.40, le broker MQTT sur la Raspberry Pi.

![Diagramme de flux](/assets/img/posts/2025-06-12-fastapi-mqtt-snzb02p-zigbee-sensor/wireshark-flux.jpg)
*Diagramme des flux*

Au démarrage du programme, le serveur se connecte au broker MQTT et s'abonne au topic `zigbee2mqtt/fleb-SNZB-02P` (les quatre premières lignes en bleu).

Les six lignes qui suivent, en vert : Le client depuis un navigateur envoie une requête HTTP (`GET /`) et se fait servir la page Web en retour (`text/html`). Une connexion Websocket est établie et maintenue. le client et le serveur peuvent maintenant envoyer des messages WebSocket sans passer par des requêtes HTTP classiques.

Les dernières lignes, en bleu : un message est publié sur le topic, du type `{"battery":100,"humidity":53.9,"last_seen":"2025-06-11T16:09:27.335Z","linkquality":131,"temperature":23.8}`, et intercepté par le serveur. Un nouveau message est alors envoyé via Websocket au client, la page est rafraîchie avec les nouvelles mesures de température et humidité. Un peu plus tard, un nouveau message MQTT est publié, et la page est rafraîchie une seconde fois.

![Message websocket](/assets/img/posts/2025-06-12-fastapi-mqtt-snzb02p-zigbee-sensor/wireshark-websocket.jpg)
*Message envoyé au client via Websocket*

### Conclusion

Ce petit code est déjà fonctionnel sur les principes, mais il y a encore beaucoup de choses à améliorer et à compléter, comme :
- l'interface utilisateur avec HTML/CSS/JavaScript. Des graphiques peuvent être tracés en temps réel ;
- la persistance des données, avec un SGBD pour conserver l'historique des données ;
- la validation des données avec le module `pydantic`, bien intégré avec FastAPI ;
- la gestion des connexions MQTT et websocket pour éviter les crashs ;
- etc.