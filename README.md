# Guide d'installation de la suite ELK sur Ubuntu

## Sommaire
1. Prérequis et préparation du système
2. Installer Java (OpenJDK 17)
3. Ajouter le dépôt officiel Elastic
4. Installer et configurer Elasticsearch
5. Installer et configurer Kibana
6. Installer et configurer Logstash
7. Installer Filebeat (recommandé)
8. Vérifier que tout fonctionne ensemble
9. Sécurisation de base
10. Tableau récapitulatif des ports et services
11. Troubleshooting : erreurs courantes et solutions
12. FAQ

## 1. Prérequis et préparation du système
Avant de commencer l'installation de la suite ELK, assurez-vous de disposer des éléments suivants :

| Prérequis | Minimum recommandé |
| --- | --- |
| **Système d'exploitation** | Ubuntu 22.04 LTS ou 24.04 LTS |
| **RAM** | 4 Go minimum (8 Go recommandé pour un usage confortable) |
| **CPU** | 2 cœurs minimum |
| **Espace disque** | 20 Go minimum (les logs consomment rapidement de l'espace) |
| **Java** | OpenJDK 17 |
| **Accès** | Utilisateur avec privilèges sudo |
| **Réseau** | Connexion internet active pour télécharger les paquets |

Mettez à jour votre système et installez les dépendances de base :
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https curl gnupg2 software-properties-common
```

---

## 2. Installer Java (OpenJDK 17)
Elasticsearch et Logstash nécessitent Java pour fonctionner. L'installation d'OpenJDK 17 garantit la compatibilité globale de la stack.

```bash
sudo apt install -y openjdk-17-jre-headless
```

Vérifiez que Java est correctement installé :
```bash
java -version
```

---

## 3. Ajouter le dépôt officiel Elastic
Ajoutez le dépôt officiel Elastic 8.x à votre système Ubuntu.

**Étape 3.1 — Importer la clé GPG Elastic :**
```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
```

**Étape 3.2 — Ajouter le dépôt Elastic 8.x :**
```bash
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
```

**Étape 3.3 — Mettre à jour l'index des paquets :**
```bash
sudo apt update
```

---

## 4. Installer et configurer Elasticsearch

Elasticsearch est le cœur de la stack. Il stocke les données, les indexe et permet les recherches rapides. L'installation se fait via apt :

```bash
sudo apt install -y elasticsearch
```
Conservez le mot de passe ! Lors de la première installation d'Elasticsearch 8.x, un mot de passe est généré pour le superutilisateur elastic. Copiez-le immédiatement. 

Si vous l'avez perdu, vous pouvez le réinitialiser avec : sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic

**Étape 4.1 — Configurer Elasticsearch :**
Éditez le fichier `/etc/elasticsearch/elasticsearch.yml` :

```yaml
sudo nano /etc/elasticsearch/elasticsearch.yml
```

Modifiez ou vérifiez les paramètres suivants :
```yaml
# Nom du cluster (choisissez un nom descriptif)
cluster.name: elk-test

# Nom du nœud
node.name: node-1

# Chemin des données et logs
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch

# Adresse d'écoute (localhost par défaut, sécurisé)
network.host: 127.0.0.1

# Port HTTP
http.port: 9200

# Mode single-node (pas de cluster multi-nœuds)
discovery.type: single-node

#Assurez vous que le paramètre ci-dessous soit commenté comme tel
#cluster.initial_master_nodes
```

**Étape 4.2 — Configurer la mémoire JVM :**

Par défaut, Elasticsearch s'alloue 1 Go de RAM. Ajustez selon votre serveur 

Éditez `/etc/elasticsearch/jvm.options.d/heap.options` :

```bash
sudo nano elasticsearch/jvm.options.d/heap.options
```

Ajoutez les lignes suivantes (exemple pour allouer 2 Go de RAM) :

```text
-Xms2g
-Xmx2g
```
Exemple pour allouer 4 Go de RAM :

```text
-Xms4g
-Xmx4g
```
Il est recommandé de ne pas depasser 50% de la memoire RAM de votre ordinateur

**Étape 4.3 — Démarrer Elasticsearch :**
```bash
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch
```


---

## 5. Installer et configurer Kibana
Kibana est l'interface web de la stack ELK. Elle permet de visualiser les données indexées dans Elasticsearch sous forme de dashboards, graphiques et tableaux interactifs.

Créer un dossier de certificats dédié à Kibana (s'il n'existe pas) et y copier le fichier CA :

```bash
sudo mkdir -p /etc/kibana/certs
sudo cp /etc/elasticsearch/certs/http_ca.crt /etc/kibana/certs/
```
Donner la pleine propriété de ce fichier à l'utilisateur kibana :

```bash
sudo chown -R kibana:kibana /etc/kibana/certs
sudo chmod 644 /etc/kibana/certs/http_ca.crt
```


**Étape 5.1 — Configurer Kibana :**
Éditez `/etc/kibana/kibana.yml` :
```yaml
# Port d'écoute de Kibana
server.port: 5601

# Adresse d'écoute (0.0.0.0 pour accès distant, 127.0.0.1 pour local uniquement)
server.host: "127.0.0.1"

# Nom du serveur Kibana
server.name: "elk-lenidit-kibana"

# URL d'Elasticsearch
elasticsearch.hosts: ["https://localhost:9200"]

# Identifiants pour la connexion Elasticsearch
elasticsearch.username: "kibana_system"
elasticsearch.password: "VOTRE_MOT_DE_PASSE_GENERE_LORS_DE_L'INSTALLATION_DE_ELASTICSEARCH"

# Certificat SSL Elasticsearch
elasticsearch.ssl.certificateAuthorities: ["/etc/kibana/certs/http_ca.crt"]
```

**Étape 5.2 — Démarrer Kibana :**
```bash
sudo systemctl enable kibana
sudo systemctl start kibana
```

**Étape 5.3 — Vérifier :**
```bash
sudo systemctl status kibana
```
Il faut généralement entre 1 et 2 minutes pour que Kibana ait fini de se connecter à Elasticsearch et de charger ses plugins internes après le passage en mode "running".

Kibana démarre sur le port 5601. Ouvrez http://localhost:5601 dans votre navigateur. connectez-vous avec l'utilisateur elastic et le mot de passe généré lors de l'installation d'Elasticsearch.

---

## 6. Installer et configurer Logstash

Logstash est le composant de traitement de la stack. Il collecte les données depuis différentes sources, les transforme (parsing, filtrage, enrichissement) et les envoie vers Elasticsearch.

```bash
sudo apt install -y logstash
```
Créer un dossier de certificats dédié à Logstash (s'il n'existe pas) et y copier le fichier CA :

```bash
sudo mkdir -p /etc/logstash/certs
sudo cp /etc/logstash/certs/http_ca.crt /etc/logstash/certs/
```
Donner la pleine propriété de ce fichier à l'utilisateur kibana :

```bash
sudo chown -R kibana:kibana /etc/kibana/certs
sudo chmod 644 /etc/kibana/certs/http_ca.crt
```
**Étape 6.1 — Créer un pipeline :**
Un pipeline Logstash se compose de trois blocs : input (source des données), filter (transformation) et output (destination). Créez un fichier de configuration :


Créez le fichier `/etc/logstash/conf.d/01-syslog.conf` :

Collez le contenu suivant :

```text
input {
  beats {
    port => 5044
  }
}

output {
  elasticsearch {
    hosts => ["https://localhost:9200"]
    user => "elastic"
    password => "VOTRE_MOT_DE_PASSE"
    ssl_certificate_authorities => "/etc/logstash/certs/http_ca.crt"
    index => "syslog-%{+YYYY.MM.dd}"
  }
}
```

Ce pipeline écoute sur le port 5044 (Beats), et envoie les logs  à Elasticsearch dans un index journalier.

**Étape 6.2 — Tester la configuration :**
```bash
sudo /usr/share/logstash/bin/logstash --config.test_and_exit -f /etc/logstash/conf.d/
```
Si la sortie affiche "Configuration OK", votre pipeline est valide.

**Étape 6.2 — Démarrer Logstash :**

```bash
sudo systemctl enable logstash
sudo systemctl start logstash
```

---

## 7. Installer Filebeat

Pourquoi Filebeat ? Filebeat est un agent léger qui collecte les fichiers de logs et les envoie à Logstash ou directement à Elasticsearch. Il consomme très peu de ressources et est conçu pour tourner sur chaque serveur dont vous souhaitez centraliser les logs.

```


```bash
sudo apt install -y filebeat
```
Créer un dossier de certificats dédié à Filebeat (s'il n'existe pas) et y copier le fichier CA :

```bash
sudo mkdir -p /etc/filebeat/certs
sudo cp /etc/filebeat/certs/http_ca.crt /etc/filebeat/certs/
```
Donner la pleine propriété de ce fichier à l'utilisateur kibana :

```bash
sudo chown -R Filebeat:filebeat /etc/filebeat/certs
sudo chmod 644 /etc/filebeat/certs/http_ca.crt
```

**Étape 7.1 — Configurer Filebeat :**
Éditez `/etc/filebeat/filebeat.yml` :
```yaml
output.logstash:
  hosts: ["localhost:5044"]

setup.kibana:
  host: "localhost:5601"
  protocol: "http"
  username: "elastic"
  password: "VOTRE_MOT_DE_PASSE_ELASTIC"
```
Vérifier les paramètres du module
Le module système utilise un fichier de configuration dédié situé dans le dossier modules.d.
Ouvrez-le :

```bash
sudo nano /etc/filebeat/modules.d/system.yml
```

Assurez-vous qu'elles sont activées (enabled: true) :

```YAML
- module: system
  syslog:
    enabled: true
  auth:
    enabled: true

**Étape 7.2 — Activer le module système et charger les dashboards :**
```bash
sudo filebeat modules enable system
sudo filebeat setup --dashboards
sudo systemctl enable filebeat
sudo systemctl start filebeat
```

---

## 8. Vérifier que tout fonctionne

```bash
# Elasticsearch
curl --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic:VOTRE_MOT_DE_PASSE https://localhost:9200

# Kibana  (doit répondre HTTP 200)
curl -s -o /dev/null -w "%{http_code}" http://localhost:5601/api/status

# Logstash
curl http://localhost:9600?pretty

# Filebeat
sudo systemctl status filebeat
```
Dans Kibana : accédez à http://localhost:5601, puis allez dans Stack Management > Index Management. Vous devriez voir des index syslog-YYYY.MM.dd ou filebeat-* apparaître si les logs sont correctement acheminés.

Rendez-vous ensuite dans Discover pour explorer vos premiers logs en temps réel.

Si vous voyez des données dans Kibana, votre stack ELK est opérationnelle. Vous pouvez désormais centraliser les logs de tous vos serveurs en installant Filebeat sur chacun d'entre eux.

---

## 9. Sécurisation de base

**9.1 — Configurer le pare-feu UFW :**
```bash
# Autoriser SSH
sudo ufw allow 22/tcp

# Autoriser Kibana (uniquement depuis votre IP si accès distant)
sudo ufw allow from VOTRE_IP to any port 5601

# NE PAS exposer Elasticsearch directement (garder 9200 en local)
# NE PAS exposer Logstash (garder 5044 en local)

# Activer le pare-feu
sudo ufw enable
sudo ufw status
```

**9.2 — Vérifier la sécurité d'Elasticsearch :**

Elasticsearch doit écouter uniquement sur 127.0.0.1 (configuration par défaut). Ne passez jamais network.host: 0.0.0.0 sans pare-feu et authentification configurés.

Elasticsearch 8.x active la sécurité par défaut (TLS + authentification). Si vous l'avez désactivée pour les tests, réactivez-la avant toute mise en production :

Dans `/etc/elasticsearch/elasticsearch.yml` :
```yaml
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.http.ssl.enabled: true
```

---

## 10. Tableau récapitulatif des ports et services

| Service | Port | Protocole | Usage |
| --- | --- | --- | --- |
| **Elasticsearch (HTTP)** | 9200 | HTTPS | API REST, requêtes, indexation |
| **Elasticsearch (Transport)** | 9300 | TCP | Communication entre nœuds du cluster |
| **Kibana** | 5601 | HTTP | Interface web de visualisation |
| **Logstash (Beats input)** | 5044 | TCP | Réception des données Filebeat / Beats |
| **Logstash (API monitoring)** | 9600 | HTTP | API de monitoring Logstash |

---

## 11. Troubleshooting : erreurs courantes et solutions

| Problème | Cause probable | Solution |
| --- | --- | --- |
| **Elasticsearch ne démarre pas** | Heap size trop élevé par rapport à la RAM disponible, ou permissions incorrectes sur le répertoire de données | Réduisez `-Xms` et `-Xmx` dans `heap.options`. Ajustez les permissions : `sudo chown -R elasticsearch:elasticsearch /var/lib/elasticsearch` |
| **Port 9200 inaccessible** | network.host configuré sur 127.0.0.1 (accès local uniquement) ou pare-feu bloquant | Vérifiez `elasticsearch.yml` (`network.host`) et le statut du pare-feu (`sudo ufw status`). |
| **Kibana : "server is not ready yet"** | Kibana n'arrive pas à se connecter à Elasticsearch (ES pas démarré, mauvais mot de passe, certificat manquant) | Vérifiez qu'Elasticsearch tourne (systemctl status elasticsearch), que le mot de passe kibana_system est correct, et que le chemin du certificat CA est bon dans kibana.yml. |
| **Logstash : erreur de pipeline** | Syntaxe incorrecte dans le fichier de configuration ou identifiants Elasticsearch invalides | Testez la config : sudo /usr/share/logstash/bin/logstash --config.test_and_exit -f /etc/logstash/conf.d/. Consultez les logs : sudo tail -f /var/log/logstash/logstash-plain.log. |
| **Filebeat ne se connecte pas** | Port Logstash 5044 non écouté ou adresse erronée | Vérifiez que Logstash écoute (`sudo ss -tlnp \| grep 5044`) et contrôlez `filebeat.yml`. |
| **Erreur "max virtual memory areas" `vm.max_map_count`** | Paramètre noyau trop bas | Exécutez : sudo sysctl -w vm.max_map_count=262144. Pour rendre permanent, ajoutez vm.max_map_count=262144 dans /etc/sysctl.conf. |
| **Espace disque insuffisant** | Saturation par accumulation des index | Mettez en place une politique ILM (Index Lifecycle Management) pour purger les anciens index. |

Pour consulter les logs de chaque composant en cas de problème :

# Logs Elasticsearch
sudo tail -f /var/log/elasticsearch/elk-lenidit.log

# Logs Kibana
sudo journalctl -u kibana -f

# Logs Logstash
sudo tail -f /var/log/logstash/logstash-plain.log

# Logs Filebeat
sudo journalctl -u filebeat -f

---

## FAQ (Foire Aux Questions)

* **Quel est l'ordre d'installation recommandé pour ELK ?**  
  L'ordre recommandé est : Elasticsearch en premier, puis Kibana, puis Logstash, et enfin Filebeat ou d'autres Beats.
* **Combien de RAM faut-il pour faire tourner ELK ?**  
  Le minimum absolu est 4 Go de RAM pour un environnement de test. 8 Go sont recommandés pour un usage confortable, et 16 Go ou plus en production.
* **Où se trouvent les fichiers de configuration ELK sur Ubuntu ?**  
  * Elasticsearch : `/etc/elasticsearch/elasticsearch.yml`  
  * Kibana : `/etc/kibana/kibana.yml`  
  * Logstash : `/etc/logstash/logstash.yml` et `/etc/logstash/conf.d/`  
  * Filebeat : `/etc/filebeat/filebeat.yml`
* **Quel est le port par défaut d'Elasticsearch ?**  
  Le port 9200 pour les requêtes HTTP (API REST) et 9300 pour la communication inter-nœuds.
* **Quelle est la différence entre Filebeat et Logstash ?**  
  Filebeat est un agent léger de collecte de logs. Logstash est un moteur de traitement plus puissant permettant de filtrer, transformer et enrichir les données.
* **Comment vérifier qu'Elasticsearch fonctionne correctement ?**  
  Exécutez curl --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic:VOTRE_MOT_DE_PASSE https://localhost:9200. Vous devriez recevoir une réponse JSON contenant le nom du cluster, la version et le statut "green" ou "yellow". Si vous obtenez une erreur de connexion, vérifiez que le service est démarré avec systemctl status elasticsearch.
* **Peut-on installer ELK avec Docker plutôt qu'en natif ?**  
  Oui, Elastic fournit des images Docker officielles pour chaque composant.
* **Comment réinitialiser le mot de passe Elasticsearch si je l'ai perdu ?**  
  Utilisez la commande sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic. Cela génère un nouveau mot de passe pour l'utilisateur spécifié. Vous pouvez également réinitialiser le mot de passe de kibana_system de la même manière.
