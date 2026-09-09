# AVSEC-Guard

**Plateforme intelligente de cybersurveillance des équipements de sûreté aéroportuaire connectés adaptée au contexte togolais**

Candidature — Concours CNISAI / AVSEC 2026 — Équipe GuardIan-X

## Vue d'ensemble

AVSEC-Guard est une plateforme locale de cybersurveillance conçue comme un outil d'aide à la décision et de sensibilisation pour les équipements de sûreté aéroportuaire connectés. Elle répond aux besoins spécifiques des infrastructures aéroportuaires togolaises avec des ressources limitées, en proposant un outil simple, en français, sans dépendance cloud permanente.

### Besoin opérationnel

Le Togo engage progressivement la modernisation de ses infrastructures aéroportuaires : contrôle d'accès électronique, vidéosurveillance IP, scanners RX et systèmes de supervision numérique se multiplient. Cette numérisation élargit une surface d'attaque qui n'est actuellement ni inventoriée ni surveillée de façon centralisée dans la majorité des infrastructures à ressources limitées.

**Contraintes structurelles du contexte togolais :**
- Peu d'experts dédiés à la cybersécurité OT/aéroportuaire disponibles localement
- Coupures de courant et bande passante variable interdisent une dépendance cloud permanente
- Budgets limités imposant le privilège de l'open source et du matériel réutilisable
- Cadre réglementaire en construction (ANAC Togo, ASECNA, OACI)
- Mélange fréquent de matériel récent et ancien rendant les identifiants par défaut particulièrement sensibles

## Fonctionnalités principales

1. **Inventorier automatiquement** les équipements connectés d'une zone AVSEC
2. **Observer passivement** les flux réseau et journaux sans interférer avec le fonctionnement des équipements
3. **Détecter les écarts** par rapport à un comportement de référence via des règles métier explicables et un modèle léger d'IA
4. **Prioriser les alertes** selon un niveau de risque (faible/moyen/élevé) pour éviter la fatigue d'alerte
5. **Recommander des actions claires** sans jamais agir de façon autonome
6. **Journaliser toutes les alertes et décisions** dans un registre horodaté et difficilement modifiable

### Principe non négociable
AVSEC-Guard n'isole, ne bloque et ne modifie aucun équipement de façon autonome. Toute action proposée est validée par une décision humaine.

## Stack technologique recommandée

```
Python 3.11+
├── FastAPI (backend)
├── Streamlit (interface)
├── scikit-learn (détection d'anomalies)
├── SQLAlchemy + PostgreSQL (stockage)
├── Zeek (capture réseau)
├── paho-mqtt (simulation d'équipements)
├── hashlib (journalisation infalsifiable)
├── Semgrep / Bandit (sécurité du code)
└── Docker Compose (orchestration et déploiement)
```

### Justification de la stack

| Fonction | Outil recommandé (MVP) | Alternative | Pourquoi ce choix pour le contexte togolais |
|---|---|---|---|
| Inventaire d'actifs | Découverte manuelle + scan passif limité | Nmap, arp-scan | Évite le scan actif risqué sur un réseau réel ; suffisant pour un labo simulé |
| Capture et analyse réseau | Zeek | Suricata, Wireshark | Zeek génère des logs structurés faciles à exploiter avec Python, sans expertise IDS avancée |
| Collecte et normalisation des logs | Python (scripts ETL) | Fluent Bit, Filebeat | Léger, sans infrastructure lourde, cohérent avec les compétences Python |
| Stockage des logs et alertes | PostgreSQL | OpenSearch / Elasticsearch | Robuste, open source, faible empreinte mémoire, adapté à un déploiement local |
| Détection d'anomalies | Règles métier (Python) + Isolation Forest (scikit-learn) | Modèles statistiques simples | Explicabilité prioritaire ; pas de GPU nécessaire |
| Backend / API | FastAPI | Flask | Rapide à développer, documentation automatique (Swagger) |
| Interface / tableau de bord | Streamlit (MVP) puis React si le temps le permet | Grafana | Streamlit permet une démonstration crédible sans effort frontend lourd |
| Authentification | JWT + RBAC simple | Keycloak | Suffisant pour un prototype ; Keycloak ajoute une complexité non nécessaire |
| Analyse de sécurité du code | Semgrep, Bandit | Dependency-Check | Cohérent avec les compétences maîtrisées ; renforce l'argument "sécurité par conception" |
| Journalisation infalsifiable | Hash-chain simple (SHA-256) | Solutions de log signé complexes | Facile à implémenter, démontrable, suffisant pour le PoC |
| Simulation d'équipements | Python + MQTT | Docker Compose multi-conteneurs | Simule caméra, badge, scanner sans matériel physique |
| Orchestration / déploiement | Docker Compose | VMs classiques | Reproductible, léger, facile à présenter |

## Architecture générale

Le prototype se limite à une zone de contrôle de sûreté simulée composée de cinq équipements :
- Caméra IP simulée
- Lecteur de badge simulé
- Poste opérateur simulé
- Scanner RX simulé
- Serveur de supervision simulé

**Chaîne technique :**
1. Équipements AVSEC simulés → sources des données du labo
2. Capteur passif réseau (Zeek) → observation sans interférence
3. Collecte et normalisation des logs (Python ETL) → uniformisation des données
4. Moteur d'analyse → combinaison de règles métier explicables et détection d'anomalies
5. PostgreSQL → stockage des données et alertes
6. Tableau de bord AVSEC-Guard → restitution des alertes, risques et recommandations

## Détection d'anomalies

### Règles métier (prioritaires)
- Nouvel équipement (adresse MAC/IP inconnue)
- Connexion inter-zones non autorisée (ex. caméra → réseau bureautique)
- Connexion à une adresse externe non whitelistée
- Connexion à des horaires inhabituels
- Identifiant par défaut détecté sur un service exposé

### Isolation Forest (complément)
Application sur des variables simples (volume de trafic, fréquence de connexion, durée des sessions) pour repérer des écarts non couverts par les règles métier.

**Priorité : l'explicabilité** — pouvoir toujours dire pourquoi une alerte a été déclenchée.

## Composants clés

### Capture et analyse réseau — Zeek
- Observe passivement le trafic réseau du labo simulé
- Produit des logs structurés (connexions, protocoles, adresses, volumes)
- Fonctionne en local, aucune connexion internet permanente nécessaire

### Backend — FastAPI
- Expose une API pour l'inventaire, alertes, journal d'audit et statistiques
- Documentation interactive générée automatiquement (Swagger)

### Base de données — PostgreSQL
- Stocke l'inventaire des équipements, les alertes, décisions humaines et journal d'audit
- Fonctionne entièrement en local, aucune dépendance cloud

### Interface — Streamlit
- Tableau de bord affichant équipements, alertes, niveaux de risque et recommandations
- Développement rapide, concentre l'effort sur la logique de détection

### Simulation d'équipements — Python + MQTT
- Reproduit le comportement des cinq équipements sans matériel physique
- Permet de rejouer des scénarios d'incident de façon contrôlée et reproductible

### Sécurité applicative — Semgrep, Bandit
- Analyse du code du projet pour détecter des vulnérabilités courantes
- Démontre une démarche "sécurité par conception"

### Journalisation infalsifiable — Hash-chain simple
- Chaque entrée du journal d'audit inclut le hachage SHA-256 de l'entrée précédente
- Rend toute modification rétroactive détectable

## Bonnes pratiques transversales

| Technique | Application dans le projet |
|---|---|
| Baseline comportementale | Définir le comportement normal de chaque équipement avant de chercher des anomalies |
| Priorisation par score de risque | Éviter la fatigue d'alerte en classant systématiquement faible/moyen/élevé |
| Séparation stricte règles / IA | Toujours pouvoir expliquer une alerte sans se reposer uniquement sur un modèle boîte noire |
| Segmentation réseau simulée (VLAN) | Démontrer visuellement l'isolation entre zones bureautique, invités, AVSEC, maintenance |
| Journalisation systématique | Toute alerte et décision humaine doit être horodatée et conservée |
| Tests en environnement isolé | Aucune interaction avec système réel, réseau public ou équipement tiers sans autorisation écrite |

## Matériel nécessaire

| Matériel | Statut | Remarque |
|---|---|---|
| Ordinateur avec Docker | Indispensable | Suffisant seul pour un MVP entièrement simulé |
| Deux à quatre conteneurs/VM | Indispensable | Simulent les équipements et services du labo |
| Raspberry Pi | Optionnel | Améliore le réalisme de la démonstration (équipement IoT physique) |
| Switch administrable avec VLAN | Optionnel | Utile pour une démonstration de segmentation réseau physique |
| Caméra IP de test | Optionnel | Renforce la crédibilité si disponible, sinon simulation logicielle suffisante |

## Compétences par rôle

| Compétence | Niveau requis | Statut |
|---|---|---|
| Python (scripts, API, ML léger) | Intermédiaire à avancé | À maîtriser |
| Réseau (bases TCP/IP, VLAN, capture de trafic) | Débutant à intermédiaire | À renforcer en équipe |
| Sécurité applicative (Semgrep, Bandit) | Intermédiaire | À maîtriser |
| Bases de données relationnelles | Débutant à intermédiaire | À confirmer |
| Développement frontend léger (Streamlit) | Débutant | Facilement acquis |
| Rédaction de dossier / pitch | — | Rôle à attribuer |

## Équipe et répartition des rôles

| Rôle | Responsabilité |
|---|---|
| Chef de projet | Cas d'usage, cohérence métier, rapport, pitch devant le jury |
| Développeur backend | API, base de données, gestion des alertes et journal d'audit |
| Spécialiste cybersécurité / réseau | Labo, capteurs Zeek/Suricata, règles de détection, segmentation |
| Data / IA | Modèle de détection d'anomalies, scoring de risque, priorisation |
| Frontend / UX | Tableau de bord, alertes, rapports, ergonomie en français |
| Documentation / qualité | Modèle de menace, procédures, tests, préparation du pitch |

*Un même membre peut cumuler deux rôles si l'équipe compte moins de six personnes.*

## Plan de développement

| Période | Jalon | Livrable |
|---|---|---|
| Candidature | Définition du problème, de l'équipe, de l'architecture | Dossier de candidature |
| Sept.–Oct. 2026 | Montage du labo, simulation des 5 équipements | Environnement de test fonctionnel |
| Nov.–Déc. 2026 | Inventaire, règles de détection, premier tableau de bord | Version alpha |
| Janv. 2027 | Détection d'anomalies, scénarios d'incident, journal d'audit | Version bêta |
| Fév. 2027 | Tests, scénarisation de la démonstration, documentation | Prototype stable |
| Mars 2027 | Finalisation, vidéo de démonstration, dossier d'impact | Livraison finale |

## Setup

*Section à compléter une fois la stack technologique finalisée.*

## Risques et limites assumées

- **Périmètre trop large** → limité à 5 équipements simulés et une seule zone
- **Confusion IT/OT** → priorité donnée à la disponibilité, supervision passive et validation humaine
- **Excès d'alertes** → priorisation par score de risque et recommandations précises
- **Modèle IA peu explicable** → combinaison de règles métier explicables et IA légère
- **Dépendance cloud** → architecture déployable localement, sans connexion internet permanente
- **Perception d'un outil offensif** → communication centrée sur défense, détection et réponse

### Limite assumée du prototype

AVSEC-Guard est un outil d'aide à la supervision et à la décision. Il ne remplace ni les procédures opérationnelles existantes, ni les responsables de sûreté, ni les solutions certifiées nécessaires à un déploiement réel en production. Aucune démonstration n'est réalisée sur un système réel, un réseau public ou un équipement appartenant à un tiers sans autorisation écrite.

## Bénéficiaires

| Bénéficiaire | Valeur apportée |
|---|---|
| ANAC Togo | Outil de sensibilisation et base de réflexion pour une politique de cybersécurité AVSEC nationale |
| Exploitants aéroportuaires | Visibilité sur les équipements connectés et réduction du délai de détection d'incident |
| Équipes informatiques locales | Outil simple ne nécessitant pas d'expertise SIEM avancée |
| Responsables et agents AVSEC | Alertes claires avec recommandation d'action, sans jargon technique |
| Écosystème régional (ASECNA, OACI) | Exemple reproductible d'une approche low-cost adaptée aux infrastructures à ressources limitées |

## Contribution

Nous accueillons les contributions ! Veuillez consulter les directives ci-dessous :

- **Code Style** : Maintenez un style de code cohérent pour la lisibilité
- **Documentation** : Assurez-vous que le code est bien documenté pour une collaboration efficace
- **Testing** : Testez complètement vos modifications avant de soumettre une pull request
- **Issue Tracker** : Consultez le suivi des problèmes pour les tâches
- **Code Review** : Toutes les contributions subissent un processus d'examen du code
- **Licensing** : Les contributions sont sous licence selon les termes du projet

---

**Contact & Support** : Pour toute question ou suggestion, veuillez ouvrir une issue sur ce dépôt.
