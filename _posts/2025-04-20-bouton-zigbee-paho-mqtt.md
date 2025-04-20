---
title: "Bouton connecté Zigbee, connexion MQTT en Python avec Eclipse Paho-MQTT"
date: 2025-04-20
categories: ['Raspberry Pi', 'domotique']
tags: ['Raspberry Pi', 'IoT', 'domotique', 'Zigbee2MQTT', 'MQTT', 'Zigbee', 'Python']
math: false
target_blank: true
---

Toujours en phase d'expérimentation avec [mon réseau de capteurs/actionneurs Zigbee et mon dongle Zigbee2MQTT]({{ site.baseurl }}{{ site.url }}/tags/zigbee2mqtt/){:target="_blank"}, je ne vous avais pas encore parlé de mon bouton sans fil.

{% include embed/youtube.html id='M5pLnNkFCJ0' %}

Et je clique, clic, clic...

Le [bouton-poussoir Zigbee SNZB-01P de chez Sonoff](https://sonoff.tech/product/gateway-and-sensors/snzb-01p/){:target="_blank"} est compatible avec le protocole Zigbee2MQTT et peut détecter trois types de pression : appui simple, double-appui et appui long.

![bouton Zigbee Sonoff SNZB-01P](/assets/img/posts/2025-04-20-bouton-zigbee-paho-mqtt/single-double-long-press.png)
*Image Sonoff*

Ces boutons sans fil connectés à votre réseau domestique peuvent servir à allumer ou éteindre des éclairages, ouvrir ou fermer des ouvrants, déclencher des alarmes, allumer un ventilateur, une cafetière, une caméra espion ou des appareils divers, etc. Que des choses souvent très utiles ;-)

![Illustration IA - éclairage sous-marin en Zigbee](/assets/img/posts/2025-04-20-bouton-zigbee-paho-mqtt/underwater-lighting.jpg)
*Illustration générée par IA*

Ils sont alimentés avec une pile CR2477 pour une autonomie annoncée de... 5 ans !! Autonomie théorique certes, mais quand même.

L'architecture simplifiée de mon réseau pour cette démonstration est la suivante :

![Architecture réseau sans fil](/assets/img/posts/2025-04-20-bouton-zigbee-paho-mqtt/zigbeeButton-SNZB01P.png)

Le bouton est configuré pour publier des messages MQTT au format JSON qui seront relayés par le *broker* Mosquitto de la Rapsberry Pi, du type :

```json
{"action":"single","battery":100,"linkquality":81,
 "update":{"installed_version":8704,"latest_version":8704,"state":"idle"},"voltage":3100}
```

C'est le champ **"action"** qui nous intéresse ici, il peut prendre les valeurs **"single"**, **"double"** ou **"long"** selon le type d'appui détecté. Le message est publié sur le topic `zigbee2mqtt/fleb-SNZB-01P-switch`.

Dans la vidéo de démonstration au début de ce billet, j'utilise Python et la bibliothèque [Eclipse Paho MQTT](https://pypi.org/project/paho-mqtt/){:target="_blank"} pour instancier un client MQTT qui va se connecter au broker installé sur la Raspberry Pi (adresse IP fixe) et s'abonner au *topic* `zigbee2mqtt/fleb-SNZB-01P-switch` où les messages du bouton sont publiés à chaque appui.

```python
import paho.mqtt.client as mqtt
import json
 
class ButtonSNZB01PClient:
    def __init__(self, broker, port, topic):
        self.broker = broker
        self.port = port
        self.topic = topic
        self.client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)
 
        # fonctions de rappel
        self.client.on_connect = self.on_connect
        self.client.on_message = self.on_message
 
    def connect(self):
        self.client.connect(self.broker, self.port, keepalive=60)
 
    def on_connect(self, client, userdata, flags, reason_code, properties):
        if reason_code == 0:
            print("Connexion au broker réussie !")
            self.client.subscribe(self.topic)
        else:
            print(f"Echec de la connexion au broker, code d'erreur : {reason_code}")
 
    def on_message(self, client, userdata, msg):
        key = 'action'
        try:
            message = json.loads(msg.payload.decode())
            #print(f"Message reçu: {message} sur le topic {msg.topic}")
            print(f"Action : {message[key]}")
 
        except json.JSONDecodeError:
            print(f"Réception de JSON non conforme sur le topic {msg.topic}")
            self.disconnect()
 
        except KeyError:
            print(f"Erreur de clé : {key}")
            self.disconnect()
 
    def loop_forever(self):
        try:
            self.client.loop_forever()
        except KeyboardInterrupt:
            print("Interruption. Déconnexion...")
            self.disconnect()
 
    def disconnect(self):
        self.client.disconnect()
 
 
if __name__ == "__main__":
    broker = '192.168.0.99' # IP fixe de la Raspberry Pi
    port = 1883
    topic = "zigbee2mqtt/fleb-SNZB-01P-switch"
 
    myButton = ButtonSNZB01PClient(broker, port, topic)
    myButton.connect()
 
    # Tourner en boucle pour recevoir des messages, jusqu'à déconnexion forcée
    myButton.loop_forever()
```

Ce bout de code peut être complété en publiant de nouveaux messages en réaction à un appui sur le bouton qui seront relayés vers une prise connectée du réseau par exemple, et les matériels branchés sur cette prise pourront s'allumer ou s'éteindre (l'éclairage de votre aquarium ou que sais-je encore). Les possibilités sont nombreuses et tous ces logiciels et matériels opensource offrent une très grande flexibilité.