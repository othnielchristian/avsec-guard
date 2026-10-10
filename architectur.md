# AVSEC-Guard — Architecture technique

## 1. Vue d’ensemble

**AVSEC-Guard** est une plateforme locale de cybersurveillance destinée aux équipements de sûreté aéroportuaire connectés. Elle permet de :

- Inventorier les équipements AVSEC connectés.
- Collecter leurs événements et journaux de sécurité.
- Détecter les comportements anormaux ou non conformes.
- Attribuer un niveau de risque aux alertes.
- Proposer des actions de réponse claires aux responsables de sûreté.
- Conserver une traçabilité des incidents et des décisions humaines.

Le prototype repose sur une **zone AVSEC simulée**, composée d’équipements virtuels ou matériels de laboratoire. Il ne remplace pas les systèmes opérationnels d’un aéroport et ne prend aucune décision autonome : il s’agit d’un outil d’aide à la supervision et à la décision.

## 2. Principes de conception

| Principe | Application dans AVSEC-Guard |
|---|---|
| Sécurité par conception | Séparation des rôles, journalisation, validation des entrées, principe du moindre privilège |
| Aide à la décision | Les alertes sont priorisées et accompagnées de recommandations, mais la décision finale reste humaine |
| Supervision passive | Le système privilégie l’observation des événements et flux, sans scan agressif des équipements OT |
| Modularité | Chaque composant peut évoluer séparément : capteurs, moteur de détection, API, interface |
| Déploiement local | Le prototype fonctionne sans dépendance obligatoire au cloud |
| Explicabilité | Chaque alerte indique l’équipement concerné, la cause, le niveau de risque et l’action recommandée |
| Résilience | Les équipements et services critiques sont surveillés pour détecter une indisponibilité ou une perte de télémétrie |

## 3. Architecture logique

```text
┌────────────────────────────────────────────────────────────────┐
│                         Zone AVSEC simulée                     │
│                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  Caméra IP   │  │   Lecteur    │  │   Scanner RX         │  │
│  │   simulée    │  │  de badge    │  │      simulé          │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘  │
│         │                 │                     │              │
│         └─────────────────┼─────────────────────┘              │
│                           │                                    │
│                    ┌──────▼───────┐                            │
│                    │ MQTT Broker  │                            │
│                    │  Mosquitto   │                            │
│                    └──────┬───────┘                            │
│                           │                                    │
│                    ┌──────▼───────┐                            │
│                    │ AVSEC-Guard  │                            │
│                    │   Backend    │                            │
│                    └──────┬───────┘                            │
│                           │                                    │
│         ┌─────────────────┼──────────────────┐                 │
│         │                 │                  │                 │
│  ┌──────▼──────┐  ┌───────▼───────┐  ┌───────▼──────────────┐  │
│  │ PostgreSQL  │  │  Moteur de    │  │  Assistant IA local  │  │
│  │ Données et  │  │  détection    │  │       Ollama         │  │
│  │  alertes    │  │ règles + IA   │  │  Résumé et conseils  │  │
│  └─────────────┘  └───────────────┘  └──────────────────────┘  │
│                                                                │
│                    ┌─────────────────┐                          │
│                    │   Dashboard     │                          │
│                    │   Streamlit     │                          │
│                    └─────────────────┘                          │
└────────────────────────────────────────────────────────────────┘
```

## 4. Composants principaux

### 4.1 Équipements AVSEC simulés

Les équipements simulés produisent des événements de sécurité réalistes. Ils permettent de démontrer le fonctionnement de la plateforme sans utiliser de matériel ou de réseau aéroportuaire réel.

| Équipement | Rôle simulé | Exemples d’événements |
|---|---|---|
| Caméra IP | Surveillance vidéo de la zone de contrôle | Heartbeat, connexion réseau, tentative de communication externe |
| Lecteur de badge | Contrôle d’accès physique | Badge scanné, accès refusé, changement de zone |
| Scanner RX | Inspection des bagages | Scan terminé, maintenance, connexion au poste opérateur |
| Console opérateur | Interface de supervision locale | Connexion utilisateur, échec d’authentification, accès administrateur |
| Équipement inconnu | Appareil non enregistré | Détection d’un nouveau périphérique sur le réseau AVSEC |

### 4.2 Broker MQTT

Le broker MQTT assure la communication entre les équipements simulés et le backend AVSEC-Guard.

- Technologie retenue : **Mosquitto**.
- Protocole : **MQTT**.
- Sujet principal : `avsec/events/device`.
- Format des messages : JSON.
- Avantages : légèreté, simplicité, adapté aux environnements IoT et OT simulés.

Dans le prototype, le broker fonctionne dans un réseau Docker isolé. En production, il devrait utiliser TLS, authentification forte, contrôle d’accès et segmentation réseau.

### 4.3 Backend AVSEC-Guard

Le backend constitue le cœur de la plateforme. Il reçoit les événements, les enregistre, applique les règles de détection, génère les alertes et expose les données au tableau de bord.

- Framework : **FastAPI**.
- Langage : **Python 3.12**.
- Base de données : **PostgreSQL**.
- Communication : API REST.
- Documentation automatique : Swagger UI via `/docs`.

Ses responsabilités sont :

- Recevoir les événements MQTT.
- Valider et normaliser les données.
- Enregistrer les événements dans PostgreSQL.
- Mettre à jour la date de dernière activité des équipements.
- Appliquer les règles de détection.
- Créer des alertes selon le niveau de risque.
- Exposer les équipements, événements et alertes via une API.
- Permettre la mise à jour du statut des alertes.

### 4.4 Moteur de détection

Le moteur de détection combine deux approches :

1. **Règles explicites** : détection déterministe de situations connues.
2. **Détection d’anomalies** : identification de comportements s’écartant d’une baseline.

| Approche | Exemples | Avantages |
|---|---|---|
| Règles métier | Communication externe interdite, équipement inconnu, accès hors horaires | Explicable, fiable, facile à auditer |
| Détection d’anomalies | Pic de trafic, fréquence inhabituelle d’événements, comportement hors profil | Détecte des situations non prévues |
| Assistant IA local | Résumé d’incident, reformulation, proposition de plan de réponse | Améliore la lisibilité et la prise de décision |

Le moteur ne remplace pas les analystes : il priorise les alertes et propose des actions.

### 4.5 Base de données PostgreSQL

PostgreSQL conserve les données nécessaires à la supervision et à l’analyse.

| Table | Contenu principal |
|---|---|
| `devices` | Inventaire des équipements : identifiant, type, zone, adresse IP, criticité, autorisation |
| `security_events` | Événements reçus : type, source, destination, utilisateur, message, horodatage |
| `alerts` | Alertes générées : titre, description, sévérité, score de risque, recommandation, statut |

Les données sensibles doivent rester dans l’environnement local du prototype. Aucune donnée réelle d’aéroport ne doit être utilisée sans autorisation écrite et procédure de protection appropriée.

### 4.6 Tableau de bord

Le tableau de bord est l’interface destinée aux responsables AVSEC, aux équipes informatiques et aux démonstrateurs.

Il présente :

- Le nombre d’équipements supervisés.
- Les alertes ouvertes.
- Les alertes critiques.
- Le nombre d’événements enregistrés.
- La liste des équipements et leur état.
- Les alertes triées par priorité.
- Les recommandations de réponse.
- Le statut de traitement des alertes.

Technologie retenue : **Streamlit**, pour permettre un développement rapide, une interface claire et une démonstration efficace.

### 4.7 Assistant IA local avec Ollama

Ollama est utilisé comme assistant local afin d’éviter l’envoi de données sensibles vers un service externe.

Ses usages possibles dans le MVP :

- Résumer un incident en langage clair.
- Expliquer pourquoi une alerte a été générée.
- Proposer une checklist de première réponse.
- Rédiger un brouillon de rapport d’incident.
- Aider un opérateur à comprendre une alerte technique.

Le modèle ne doit jamais être autorisé à :

- Isoler automatiquement un équipement.
- Modifier une configuration réseau.
- Supprimer des journaux.
- Prendre une décision de sûreté sans validation humaine.

## 5. Flux de données

```text
1. Un équipement AVSEC simulé génère un événement.
        │
        ▼
2. L’événement est publié sur le sujet MQTT :
   avsec/events/device
        │
        ▼
3. Le backend AVSEC-Guard s’abonne au sujet MQTT.
        │
        ▼
4. Le backend valide et enregistre l’événement dans PostgreSQL.
        │
        ▼
5. Le moteur de détection analyse l’événement.
        │
        ├── Aucun risque identifié → l’événement est conservé.
        │
        └── Risque identifié → création d’une alerte.
                │
                ▼
6. L’alerte reçoit une sévérité, un score de risque
   et une recommandation d’action.
        │
        ▼
7. Le tableau de bord affiche l’alerte au responsable AVSEC.
        │
        ▼
8. L’utilisateur peut consulter, confirmer, rejeter
   ou résoudre l’alerte.
        │
        ▼
9. La décision humaine est enregistrée pour la traçabilité.
```

## 6. Modèle de données

### 6.1 Table `devices`

| Champ | Type | Description |
|---|---|---|
| `id` | UUID | Identifiant interne |
| `device_id` | Texte | Identifiant métier, par exemple `CAM-001` |
| `name` | Texte | Nom lisible de l’équipement |
| `device_type` | Texte | Type : caméra, badge, scanner, console |
| `zone` | Texte | Zone physique ou logique |
| `ip_address` | Texte | Adresse IP simulée ou réelle |
| `criticality` | Texte | Criticité : faible, moyenne, élevée |
| `expected_zone` | Texte | Zone réseau attendue |
| `is_authorized` | Booléen | Indique si l’équipement est autorisé |
| `last_seen_at` | Date/heure | Dernière activité détectée |
| `created_at` | Date/heure | Date d’ajout à l’inventaire |

### 6.2 Table `security_events`

| Champ | Type | Description |
|---|---|---|
| `id` | UUID | Identifiant de l’événement |
| `device_id` | Texte | Équipement concerné |
| `event_type` | Texte | Type d’événement |
| `source_ip` | Texte | Adresse IP source |
| `destination_ip` | Texte | Adresse IP destination |
| `destination_port` | Entier | Port destination |
| `username` | Texte | Utilisateur concerné, si applicable |
| `message` | Texte | Description de l’événement |
| `payload` | Texte | Données complémentaires |
| `occurred_at` | Date/heure | Horodatage de l’événement |

### 6.3 Table `alerts`

| Champ | Type | Description |
|---|---|---|
| `id` | UUID | Identifiant de l’alerte |
| `event_id` | UUID | Événement à l’origine de l’alerte |
| `device_id` | Texte | Équipement concerné |
| `title` | Texte | Titre court de l’alerte |
| `description` | Texte | Explication de l’incident |
| `severity` | Texte | `low`, `medium`, `high` ou `critical` |
| `risk_score` | Entier | Score de risque de 0 à 100 |
| `recommendation` | Texte | Action recommandée |
| `status` | Texte | `open`, `acknowledged`, `resolved` ou `false_positive` |
| `created_at` | Date/heure | Date de création de l’alerte |

## 7. Règles de détection du MVP

| Règle | Condition | Sévérité | Action recommandée |
|---|---|---|---|
| Équipement non autorisé | Un équipement inconnu envoie un événement | Élevée | Vérifier l’identité, isoler et ouvrir un ticket |
| Communication externe non autorisée | Une caméra ou un équipement contacte une destination externe non autorisée | Critique | Vérifier la destination, isoler, contrôler les identifiants |
| Accès administrateur hors horaires | Connexion admin en dehors de la plage autorisée | Élevée | Vérifier l’utilisateur, l’IP source et révoquer la session si nécessaire |
| Échecs d’authentification répétés | Au moins cinq échecs de connexion | Élevée | Bloquer temporairement la source et vérifier les comptes |
| Équipement hors zone attendue | L’équipement apparaît dans une zone réseau non autorisée | Moyenne | Vérifier la configuration et l’emplacement physique |
| Perte de télémétrie | Un équipement cesse d’envoyer ses événements | Moyenne | Vérifier l’alimentation, le réseau et l’état de l’équipement |

## 8. Niveaux de sévérité

| Niveau | Signification | Exemple |
|---|---|---|
| `low` | Information utile, sans risque immédiat | Événement de maintenance |
| `medium` | Anomalie à vérifier | Équipement hors zone attendue |
| `high` | Risque significatif nécessitant une action rapide | Accès administrateur hors horaires |
| `critical` | Risque majeur pouvant affecter la sûreté ou la continuité | Communication externe non autorisée depuis une caméra |

## 9. Architecture réseau simulée

```text
Internet
   │
Pare-feu périmétrique
   │
┌─────────────────────────────────────────────┐
│              Réseau bureautique             │
│  Postes administratifs, e-mail, outils RH   │
└─────────────────────────────────────────────┘
   │
DMZ / services contrôlés
   │
Pare-feu OT / AVSEC
   │
┌─────────────────────────────────────────────┐
│              Zone AVSEC protégée            │
│                                             │
│  Caméra IP      Lecteur de badge            │
│  Scanner RX     Console opérateur            │
│  Serveur de supervision   Serveur de logs   │
└─────────────────────────────────────────────┘
   │
┌─────────────────────────────────────────────┐
│          Zone maintenance restreinte        │
│  Accès temporaire, tracé et contrôlé        │
└─────────────────────────────────────────────┘
```

### Règles de segmentation

| Zone | Accès autorisé | Accès interdit |
|---|---|---|
| Wi-Fi invités | Internet uniquement | Réseau AVSEC, bureautique, maintenance |
| Bureautique | Services autorisés | Accès direct aux équipements AVSEC |
| AVSEC | Communication entre équipements autorisés et serveur de supervision | Internet non contrôlé |
| Maintenance | Accès temporaire aux équipements concernés | Accès permanent et non journalisé |
| Administration | Gestion des équipements et alertes | Accès direct depuis le réseau invité |

## 10. Sécurité du prototype

### 10.1 Contrôles appliqués

- Validation des données entrantes avec Pydantic.
- Utilisation d’identifiants et de mots de passe distincts pour chaque service.
- Stockage des secrets dans des variables d’environnement.
- Journalisation des événements et alertes.
- Séparation des responsabilités entre backend, base de données et interface.
- Environnement Docker isolé pour les tests.
- Documentation des scénarios de test et des limites du système.

### 10.2 Contrôles à prévoir pour une version avancée

- Authentification des utilisateurs avec rôles : opérateur, analyste, administrateur.
- Authentification multifacteur pour les comptes administrateurs.
- TLS pour les communications internes et externes.
- Sauvegarde chiffrée de la base de données.
- Intégrité des journaux par hachage ou chaînage cryptographique.
- Gestion centralisée des identités.
- Contrôle d’accès basé sur les rôles, RBAC.
- Politique de conservation et de suppression des données.
- Tests d’intrusion réalisés uniquement dans un environnement autorisé.

## 11. Intégration d’Ollama

### 11.1 Rôle de l’assistant IA

L’assistant IA local est intégré au backend AVSEC-Guard. Il ne remplace pas le moteur de détection : il aide à interpréter les alertes.

```text
Alerte générée par le moteur de détection
        │
        ▼
Contexte envoyé à Ollama
- Titre de l’alerte
- Description
- Équipement concerné
- Sévérité
- Événements récents
        │
        ▼
Réponse générée localement
- Résumé de l’incident
- Niveau de gravité
- Premières actions recommandées
- Informations à vérifier
```

### 11.2 Exemple de prompt système

```text
Tu es un assistant de cybersécurité pour la sûreté aéroportuaire.
Tu aides un responsable AVSEC à comprendre une alerte générée par
la plateforme AVSEC-Guard.

Règles :
- Réponds uniquement en français.
- Sois concis, factuel et opérationnel.
- Ne prétends jamais avoir accès au réseau réel.
- Ne recommande jamais une action automatique irréversible.
- Propose toujours une vérification humaine.
- Structure ta réponse en trois parties :
  1. Résumé de l’incident
  2. Risque potentiel
  3. Actions recommandées
```

## 12. Technologies retenues

| Couche | Technologie | Justification |
|---|---|---|
| Langage principal | Python | Écosystème riche, adapté à l’équipe et à l’IA |
| API | FastAPI | Rapide, moderne, documentation automatique |
| Base de données | PostgreSQL | Robuste, relationnelle, adaptée aux alertes et journaux |
| Communication IoT | MQTT / Mosquitto | Léger et adapté aux équipements connectés |
| Détection | Règles Python + scikit-learn | Approche explicable et évolutive |
| IA locale | Ollama | Exécution locale, confidentialité, pas de dépendance cloud |
| Interface | Streamlit | Développement rapide et démonstration efficace |
| Conteneurisation | Docker Compose | Déploiement reproductible et simple |
| Versioning | Git | Suivi du code, collaboration et historique |
| Environnement | WSL + VS Code | Environnement Linux pratique sous Windows |

## 13. Limites du prototype

- Les équipements sont simulés et ne représentent pas des équipements aéroportuaires certifiés.
- Le prototype ne doit pas être connecté à un réseau de production sans autorisation formelle.
- Les modèles d’anomalie nécessitent une période d’apprentissage et des données représentatives.
- L’assistant Ollama peut produire des réponses incomplètes ou imprécises ; ses conseils doivent être vérifiés.
- La plateforme ne remplace pas un SIEM, un SOC, un pare-feu industriel certifié ou une équipe de réponse aux incidents.
- Aucune action automatique de blocage, redémarrage ou modification de configuration ne doit être activée sans validation humaine.

## 14. Évolutions prévues

| Version | Fonctionnalités |
|---|---|
| MVP | Inventaire, collecte MQTT, règles de détection, alertes, dashboard |
| V1 | Détection d’anomalies, rapports PDF, gestion des statuts d’alerte, authentification |
| V2 | Intégration Syslog, SNMP, NetFlow, supervision de vrais équipements autorisés |
| V3 | Réponse assistée, playbooks d’incident, intégration SIEM, tableaux de bord multi-zones |
| V4 | Déploiement progressif en environnement pilote, sous supervision opérationnelle |

## 15. Schéma de déploiement Docker

```text
docker-compose.yml
        │
        ├── postgres
        │     └── Base AVSEC-Guard
        │
        ├── mosquitto
        │     └── Broker MQTT
        │
        ├── backend
        │     ├── FastAPI
        │     ├── Moteur de détection
        │     ├── Client MQTT
        │     └── Client Ollama
        │
        └── dashboard
              └── Streamlit
```

## 16. Points de contrôle de la démonstration

Pour la finale, la démonstration doit montrer :

1. Le tableau de bord AVSEC-Guard.
2. L’inventaire des équipements simulés.
3. Le trafic normal généré par les équipements.
4. Le déclenchement d’un scénario d’incident.
5. La détection automatique de l’anomalie.
6. L’affichage d’une alerte priorisée.
7. La recommandation de réponse.
8. La validation humaine de l’alerte.
9. La traçabilité de l’incident.
10. Les limites assumées et la feuille de route de déploiement.