# Analyse du projet JARVIS

> Repository: https://github.com/ethanplusai/jarvis
> Auteur: ethanplusai
> Licence: Gratuit pour usage personnel, licence commerciale requise

---

## 1. Vue d'ensemble

**JARVIS** (*Just A Rather Very Intelligent System*) est un assistant IA vocal conçu exclusivement pour **macOS**, inspiré de l'IA de Tony Stark dans le MCU. L'utilisateur interagit par la voix et reçoit des réponses parlées avec un accent britannique, accompagnées d'une visualisation 3D (orbe de particules réactif à l'audio via Three.js).

---

## 2. Stack technique

| Couche | Technologies |
|--------|-------------|
| **Backend** | Python 3.11+, FastAPI, WebSocket |
| **IA / LLM** | Claude Haiku (réponses rapides), Claude Opus (tâches complexes) |
| **TTS** | Fish Audio (voix personnalisée JARVIS) |
| **STT** | Web Speech API (côté navigateur) |
| **Frontend** | Vite, TypeScript, Three.js |
| **Base de données** | SQLite + FTS5 (full-text search) |
| **Intégrations macOS** | AppleScript (Calendar, Mail, Notes, Terminal, Chrome) |
| **Automatisation web** | Playwright |
| **Communication** | WebSocket sécurisé (JSON + audio binaire) |

**Répartition du code :** Python 81.3% | TypeScript 11.9% | Swift 3.5% | CSS/Shell/JS/HTML ~3.3%

---

## 3. Architecture du système

```
┌─────────────┐     WebSocket      ┌──────────────────┐
│  Frontend   │◄──────────────────►│   server.py      │
│  (Vite/TS)  │   JSON + Audio     │   (FastAPI)      │
│  Three.js   │                    │                  │
│  Web Speech │                    │  ┌────────────┐  │
└─────────────┘                    │  │ Claude API │  │
                                   │  │ Haiku/Opus │  │
                                   │  └────────────┘  │
                                   │                  │
                                   │  ┌────────────┐  │
                                   │  │ Fish Audio │  │
                                   │  │   (TTS)    │  │
                                   │  └────────────┘  │
                                   │                  │
                                   │  ┌────────────┐  │
                                   │  │  SQLite    │  │
                                   │  │  memory.py │  │
                                   │  └────────────┘  │
                                   │                  │
                                   │  ┌────────────┐  │
                                   │  │ AppleScript│  │
                                   │  │ (macOS)    │  │
                                   │  └────────────┘  │
                                   └──────────────────┘
```

### Pipeline vocal
1. L'utilisateur parle → Web Speech API transcrit
2. Texte envoyé via WebSocket au backend
3. Classification d'intention (fast-path par mots-clés ou LLM)
4. Génération de réponse via Claude Haiku
5. Synthèse vocale via Fish Audio
6. Audio MP3 renvoyé au frontend
7. Actions déclenchées en arrière-plan si nécessaire

---

## 4. Structure du projet

```
jarvis/
├── frontend/
│   └── src/
│       ├── main.ts          # Point d'entrée
│       ├── orb.ts           # Visualisation 3D (Three.js)
│       ├── voice.ts         # Gestion audio/voix
│       ├── ws.ts            # Communication WebSocket
│       ├── settings.ts      # Configuration
│       └── style.css        # Styles
├── desktop-overlay/          # Overlay macOS (Swift)
├── helpers/                  # Modules utilitaires
├── templates/prompts/        # Templates de prompts LLM
├── tests/                    # Suite de tests
├── data/                     # Stockage de données
├── server.py                 # Backend principal (~2300 lignes)
├── memory.py                 # Mémoire persistante (SQLite + FTS5)
├── calendar_access.py        # Accès Apple Calendar (AppleScript)
├── mail_access.py            # Accès Mail (lecture seule)
├── notes_access.py           # Accès Apple Notes
├── actions.py                # Système de dispatching d'actions
├── browser.py                # Automatisation Chrome (Playwright)
├── work_mode.py              # Sessions Claude Code
├── planner.py                # Planification de tâches
└── ...                       # Modules learning, tracking, suggestions
```

---

## 5. Fonctionnalités clés

### 5.1 Système d'actions
JARVIS utilise des tags `[ACTION:X]` dans les réponses LLM pour déclencher des actions système :

| Tag | Fonction |
|-----|----------|
| `[ACTION:BUILD]` | Lance un projet de développement via Claude Code |
| `[ACTION:BROWSE]` | Ouvre une URL dans Chrome |
| `[ACTION:RESEARCH]` | Génère un rapport HTML détaillé via Claude Opus |
| `[ACTION:ADD_TASK]` | Crée une tâche dans le gestionnaire |
| `[ACTION:REMEMBER]` | Stocke un fait en mémoire persistante |
| `[ACTION:PROMPT_PROJECT]` | Envoie un prompt à un projet existant |

### 5.2 Mémoire persistante
- **3 tables SQLite** : `memories`, `tasks`, `notes`
- **FTS5** pour recherche plein-texte performante
- **Types de mémoire** : fact, preference, project, person, decision
- **Extraction automatique** : Claude Haiku analyse les conversations pour extraire les faits importants
- **Contexte tri-couche** : résumé de session + 20 derniers messages + mémoires utilisateur

### 5.3 Gestion de tâches
- Priorités (high/medium/low), statuts (open/in_progress/done/cancelled)
- Dates et heures d'échéance
- Tags JSON, notes, association à des projets
- Planification quotidienne combinant calendrier + tâches

### 5.4 Intégrations macOS
- **Calendar** : lecture des événements via AppleScript
- **Mail** : lecture seule des emails
- **Notes** : accès aux notes Apple
- **Terminal** : lancement de sessions Claude Code
- **Chrome** : navigation et récupération d'infos d'onglets
- **Screen** : détection des applications ouvertes pour contexte

### 5.5 Gestion de projets de développement
- `ClaudeTaskManager` : jusqu'à 3 sous-processus Claude concurrents
- Vérification QA automatique à la fin de chaque tâche
- Génération de dossiers projet sur le Bureau
- Suivi via `DispatchRegistry`

---

## 6. API REST

| Endpoint | Méthode | Description |
|----------|---------|-------------|
| `/ws` | WebSocket | Interface vocale principale |
| `/api/health` | GET | État du serveur |
| `/api/tts-test` | GET | Test de synthèse vocale |
| `/api/usage` | GET | Consommation tokens et coûts |
| `/api/tasks` | GET/POST | Lister/créer des tâches |
| `/api/tasks/{id}` | GET/DELETE | Statut/annulation de tâche |
| `/api/projects` | GET | Liste des projets sur le Bureau |

---

## 7. Points forts

- **Architecture bien pensée** : séparation claire backend/frontend, pipeline vocal non-bloquant
- **Mémoire persistante intelligente** : extraction automatique de faits via LLM + FTS5
- **Système d'actions extensible** : les tags `[ACTION:X]` permettent d'ajouter facilement de nouvelles capacités
- **Gestion de contexte sophistiquée** : cache de contexte (écran, calendrier, météo) rafraîchi en arrière-plan sans bloquer l'event loop
- **Double modèle LLM** : Haiku pour la réactivité, Opus pour les tâches complexes
- **Suivi des coûts** : logging JSONL de tous les appels API

---

## 8. Points d'amélioration identifiés

- **macOS uniquement** : fortement couplé à AppleScript, pas de support Linux/Windows
- **Fichier server.py monolithique** : ~2300 lignes, pourrait bénéficier d'un découpage en modules
- **Dépendance à Fish Audio** : pas d'alternative TTS configurable
- **Pas de plugin system** : les extensions nécessitent de modifier le code source
- **Pas d'app mobile** : uniquement utilisable depuis un navigateur desktop
- **Tests** : couverture à évaluer

---

## 9. Prérequis d'installation

- macOS (requis pour AppleScript)
- Python 3.11+
- Node.js 18+
- Google Chrome
- Clé API Anthropic
- Clé API Fish Audio
- Claude Code CLI installé
- Certificats SSL (génération locale)

---

## 10. Conclusion

JARVIS est un projet ambitieux et bien réalisé qui offre une expérience d'assistant IA vocal immersive sur macOS. L'architecture est solide avec une bonne gestion de la concurrence et du contexte. Le système d'actions par tags LLM est élégant et extensible. Les principaux axes d'amélioration concernent la portabilité (au-delà de macOS), la modularisation du backend, et l'ajout d'un système de plugins pour faciliter les contributions communautaires.
