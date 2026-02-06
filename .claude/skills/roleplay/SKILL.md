---
name: roleplay
description: "Lance des sessions de roleplay immersif en orchestrant les agents. Exemples: (1) '/roleplay @campagne/', (2) '/roleplay --universe=Witcher --character=Geralt', (3) '/roleplay @config.yaml'"
---

# Roleplay

## ARCHITECTURE DES AGENTS

```
╔══════════════════════════════════════════════════════════════════╗
║  AGENTS D'ARRIÈRE-PLAN (Task tool)                              ║
║  → lore-keeper: init + vérification si action suspecte          ║
║  → session-chronicler: init + sur commande [SAVE]/[CHECKPOINT]  ║
║                                                                  ║
║  AGENTS DE JEU (Comportement intégré - PAS de Task)             ║
║  → character-incarnator: Lire instructions, appliquer direct    ║
║  → story-weaver: Lire instructions, appliquer direct            ║
╚══════════════════════════════════════════════════════════════════╝
```

**Pourquoi?** Task à chaque réplique = trop lent. Comportement intégré = fluide.

---

## INITIALISATION (une seule fois au début)

### Étape 1 — Lancer les 4 agents en parallèle

```
Task #1 - lore-keeper (init):
  subagent_type: general-purpose
  model: haiku
  run_in_background: true
  prompt: |
    Tu es lore-keeper. Lis /chemin/.claude/agents/lore-keeper.md
    Charge le contexte depuis [chemin campagne].
    Indexe personnages, relations, lieux, timeline.
    Retourne rapport structuré.

Task #2 - session-chronicler (init):
  subagent_type: general-purpose
  model: haiku
  run_in_background: true
  prompt: |
    Tu es session-chronicler. Lis /chemin/.claude/agents/session-chronicler.md
    Crée/mets à jour sessions/session-XXX/meta.yaml.
    Confirme initialisation.

Task #3 - story-weaver (scène ouverture):
  subagent_type: general-purpose
  model: sonnet
  prompt: |
    Tu es story-weaver. Lis /chemin/.claude/agents/story-weaver.md
    Config: style=[X], pacing=[Y], creativity=[Z]
    Univers: [contexte]

    Écris la scène d'ouverture. NE JAMAIS contrôler le personnage joueur.
    Termine par --- pour marquer fin du tour.

Task #4 - character-incarnator (référence):
  subagent_type: general-purpose
  model: haiku
  prompt: |
    Tu es character-incarnator. Lis /chemin/.claude/agents/character-incarnator.md
    Personnage joueur: [nom]
    Confirme les guidelines pour incarner ce personnage.
```

### Étape 2 — Afficher confirmation

```
[ROLEPLAY: Session initialisée]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
→ Agents: ✓ lore-keeper ✓ session-chronicler ✓ story-weaver ✓ character-incarnator
→ Mode: interactif
→ Univers: [nom]
→ Personnage: [nom]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Étape 3 — Afficher la scène d'ouverture (résultat Task #3)

---

## BOUCLE DE JEU (à chaque tour)

### Quand le joueur fait une action:

**ÉTAPE 1 — Évaluer si lore-keeper nécessaire**

Le lore-keeper s'active SEULEMENT si l'action est:
- ⚠️ Un saut d'intimité (baiser, contact physique intime, déclaration)
- ⚠️ Une contradiction potentielle avec le caractère établi
- ⚠️ Une référence à un événement/personnage canon
- ⚠️ Un changement majeur de dynamique relationnelle

Le lore-keeper NE s'active PAS si l'action est:
- ✓ Dialogue normal/naturel
- ✓ Mouvement simple (s'approcher, s'asseoir, regarder)
- ✓ Continuation logique de la scène
- ✓ Cohérente avec le flux narratif établi

**SI LORE-KEEPER NÉCESSAIRE — Lancer Task:**

```
Task - lore-keeper (vérification):
  subagent_type: general-purpose
  model: haiku
  prompt: |
    Tu es lore-keeper. Vérifie cette action:
    ACTION: [action]
    CONTEXTE: [situation]
    RELATION: [état actuel]

    Réponds UNIQUEMENT:
    [LORE-KEEPER: ✓ Cohérent] ou
    [LORE-KEEPER: Incohérence] → Problème + Suggestion
```

**SI INCOHÉRENCE — Utiliser AskUserQuestion:**

```
AskUserQuestion:
  questions:
    - question: "[LORE-KEEPER] Incohérence: [problème]. Quelle option?"
      header: "Correction"
      options:
        - label: "Option A"
          description: "[suggestion 1]"
        - label: "Option B"
          description: "[suggestion 2]"
        - label: "Ignorer"
          description: "Garder tel quel"
      multiSelect: false
```

**SINON — Répondre directement (comportement intégré)**

**ÉTAPE 3 — Appliquer le bon comportement:**

| Situation | Comportement | Comment |
|-----------|--------------|---------|
| Dialogue/réaction PNJ | **character-incarnator** | Appliquer les guidelines lues au début |
| Avancer l'histoire | **story-weaver** | Appliquer les guidelines lues au début |

**PAS DE TASK** pour les dialogues courants. Répondre directement en appliquant:

**character-incarnator (dialogues):**
- Incarner le PNJ selon son lore (traits, tics, vocabulaire)
- Format: narrative (italique actions, tirets dialogue)
- Répliques courtes si personnage laconique
- NE JAMAIS contrôler le personnage joueur
- Terminer par ---

**story-weaver (avancer l'histoire):**
- Narrer transitions, ellipses, nouvelles scènes
- Respecter le style configuré (romantic, moderate, suggestive)
- NE JAMAIS contrôler le personnage joueur
- Terminer par ---

**ÉTAPE 4 — Afficher et attendre tour joueur**

---

## COMMANDES SPÉCIALES

### [SAVE] ou [CHECKPOINT]

```
Task - session-chronicler:
  subagent_type: general-purpose
  model: haiku
  prompt: |
    Tu es session-chronicler.
    Enregistre ces événements dans [chemin]/sessions/session-XXX/:

    ÉVÉNEMENTS:
    - [liste des événements depuis dernier save]

    Mets à jour: timeline.yaml, events.yaml, characters.yaml
```

### [OOC] message

Répondre hors personnage sans lancer d'agent.

### [END SESSION]

```
Task - session-chronicler:
  subagent_type: general-purpose
  model: haiku
  prompt: |
    Tu es session-chronicler.
    Clôture la session:
    - Résumé final
    - Tous les événements enregistrés
    - meta.yaml: status = completed
```

---

## CHECKLIST OBLIGATOIRE

Avant CHAQUE réponse en mode roleplay:

- [ ] Action joueur reçue?
- [ ] Action nécessite lore-keeper? (saut intimité, contradiction, canon)
  - Si OUI: Task lore-keeper → si incohérent: AskUserQuestion
  - Si NON: passer directement à story-weaver
- [ ] Task story-weaver lancé?
- [ ] Résultat story-weaver affiché?
- [ ] Fin du tour marquée par ---?

---

## RÈGLE POUR TOUS LES CHOIX

**TOUJOURS utiliser AskUserQuestion pour proposer des choix au joueur:**

- Choix de lieu de scène
- Choix de direction narrative
- Corrections suggérées par lore-keeper
- Brainstorming en mode écriture

**JAMAIS de tableaux markdown pour les choix. TOUJOURS AskUserQuestion.**
