# QA-HANDLER · ARCHI DJIBRIL CHINOIS

> Refonte du bot de creusage pour l'app `djibril-student-app`.
> Front modifié pour afficher : progression (tour X/3), niveau conscience 1-5,
> synthèse en 4 blocs (vraie problématique / vrai besoin / blocage / action).
> Backend (edge function Supabase `qa-handler`) à migrer pour exploiter Hermès chinois.

---

## 1. PIPELINE DE CREUSAGE (5-TIER)

```
USER → /qa/submit
  ├─ Stage 1 : CLASSIF (Tier 3 Qwen3 Turbo, ~$0.0001)
  │   → détecte intention (mental | business | offre | call | mindset)
  │   → décide nombre de tours target (1-4) selon clarté initiale
  │
  ├─ Stage 2 : RELANCE SOCRATIQUE (Tier 1 GLM-5.1 ou Kimi K2.6)
  │   → cap 200 tokens output, prompt cache STABLE_BASE
  │   → 1 question UNIQUE qui creuse :
  │     - déclencheur émotionnel concret
  │     - ou conséquence non-dite
  │     - ou croyance limitante sous-jacente
  │     - ou pattern (depuis quand, à quelle fréquence)
  │
  ├─ Stage 3 : SCORE CONSCIENCE (Tier 2 Step 3.5 Flash)
  │   → relit thread, retourne {niveau_conscience: 1-5}
  │   → 1=vague, 5=précis enjeu+blocage+demande claire
  │
  ├─ [LOOP 1-3 fois] : si niveau<4 ET tours<3 → re-Stage 2
  │
  └─ Stage 4 : SYNTHÈSE FINALE (Tier 1 DeepSeek V4 Pro)
      → output JSON strict :
        {
          niveau_conscience: 4,
          vraie_problematique: "...",
          vrai_besoin: "...",
          blocage_emotionnel: "...",
          action_concrete_proposee: "...",
          tags: ["closing","mindset","peur_refus"],
          qualified_question: "phrase unique pour Djibril"
        }
```

### Council (optionnel sur questions stratégiques)

Si `tags` contient `pricing`, `offre`, `positioning` OU si élève a >5 questions
non répondues → trigger **Council 3-stage** :

- Stage A : 3 brains parallèles donnent leur lecture (DeepSeek V4 Pro / Kimi K2.6 / GLM 5.1)
- Stage B : cross-review adversarial anonyme
- Stage C : DeepSeek V4 Pro chairman synthétise

Helper local : `~/agence-ia/manager/llm_council.sh`

---

## 2. PROMPT GLM-5.1 RELANCE SOCRATIQUE (Stage 2)

**STABLE_BASE_PROMPT** (cacheable, ne change jamais) :

```
Tu es l'IA mentor de Djibril Pardin, coach business chez Reset Ultra.
Cible : barbiers + coachs banlieue 93/IDF/Paris qui veulent passer à 5-10k€/mois en 80 jours.

Ton rôle UNIQUE : creuser la VRAIE problématique d'un élève avant que Djibril ne réponde personnellement.

RÈGLES INVIOLABLES :
1. Tu poses TOUJOURS UNE seule question à la fois. Jamais deux.
2. Tu vises le DÉCLENCHEUR émotionnel CONCRET (« le call de 16h où il a dit X »),
   pas l'abstrait (« comment gérer le stress »).
3. Tu n'apportes JAMAIS de conseil, JAMAIS d'opinion. Tu CREUSES.
4. Tu ne valides JAMAIS (« super question », « c'est intéressant »). Tu vas droit au but.
5. Tu utilises le tutoiement, ton direct, frère/sœur de l'élève, banlieue 93 vibe.
6. Tu privilégies : « depuis quand ? », « quand exactement ? », « concrètement avec qui ? »,
   « qu'est-ce qui s'est passé juste avant ? », « si tu pousses, qu'est-ce qui se passe au pire ? »
7. Output : 1 question, max 25 mots. Pas de préambule.
```

**Variable input** :
```
HISTORIQUE_THREAD:
- Élève (tour 0) : "{initial_question}"
- IA (tour 0) : "{previous_q1}"
- Élève (tour 1) : "{response_to_q1}"
- ...

NIVEAU_CONSCIENCE_ACTUEL: {score}/5
TOUR_ACTUEL: {n}/3

→ Pose la prochaine question qui creuse.
```

---

## 3. PROMPT DEEPSEEK V4 PRO SYNTHÈSE (Stage 4)

**STABLE_BASE_PROMPT** :

```
Tu es le synthétiseur final de Reset Ultra (coach Djibril Pardin).
Tu reçois un thread de creusage IA↔élève.

Ton job : extraire en JSON strict la SUBSTANCE pour que Djibril réponde en 30 secondes.

OUTPUT JSON OBLIGATOIRE (aucun texte autour) :
{
  "niveau_conscience": 1-5,
  "vraie_problematique": "phrase concrète, 15-30 mots, ce qui se passe vraiment",
  "vrai_besoin": "phrase, 10-25 mots, ce que l'élève cherche en réalité",
  "blocage_emotionnel": "peur du refus | doute compétence | imposteur | colère | tristesse | aucun",
  "action_concrete_proposee": "action de 1 phrase que Djibril pourrait recommander",
  "tags": ["closing","mindset","mental","offre","call","positioning","pricing"],
  "qualified_question": "1 phrase finale pour Djibril, ton de l'élève"
}

RÈGLES :
- Si niveau_conscience < 3 → action_concrete_proposee = "creuser encore avant de répondre"
- Tags max 5, en kebab-case ou snake_case
- vrai_besoin DIFFÉRENT de vraie_problematique (problématique=état actuel, besoin=état désiré)
- Pas de "tu", pas de "vous", reste descriptif
```

---

## 4. EDGE FUNCTION SCHEMA (Supabase)

### Table `qa_questions` (extension)

```sql
ALTER TABLE qa_questions ADD COLUMN IF NOT EXISTS consciousness_level INT DEFAULT 1;
ALTER TABLE qa_questions ADD COLUMN IF NOT EXISTS target_turns INT DEFAULT 3;
ALTER TABLE qa_questions ADD COLUMN IF NOT EXISTS synthesis JSONB;  -- {vraie_problematique, vrai_besoin, blocage_emotionnel, action_concrete_proposee, tags}
ALTER TABLE qa_questions ADD COLUMN IF NOT EXISTS llm_cost_eur NUMERIC(10,6) DEFAULT 0;
ALTER TABLE qa_questions ADD COLUMN IF NOT EXISTS council_used BOOLEAN DEFAULT FALSE;
```

### Endpoints à modifier

| Endpoint | Modifs |
|----------|--------|
| `POST /qa/submit` | Stage 1 classif → Stage 2 1ère relance → renvoie `{question: {id, status:'deepening', thread, consciousness_level, target_turns}}` |
| `POST /qa/answer/:id` | Append réponse → Stage 3 score → si <4 et tours<target → Stage 2 next → sinon Stage 4 synth → renvoie `{question: {status:'qualified', synthesis: {...}, qualified_question}}` |
| `POST /qa/reformulate/:id` | Re-Stage 4 avec temperature différente, pour variation |
| `POST /qa/finalize/:id` | Accepte `edited_synthesis: {vraie_problematique, vrai_besoin, blocage_emotionnel, action_concrete_proposee}` du front |

---

## 5. INTÉGRATION HERMÈS (côté Mac Djibril)

L'edge function Supabase ne doit PAS appeler directement les APIs chinoises depuis Deno.
Elle appelle un endpoint **Hermès gateway** exposé via ngrok permanent ou Cloudflare Tunnel.

### Architecture

```
[App tracking.html]
  ↓ (Supabase Auth Bearer)
[Edge function qa-handler]
  ↓ (HMAC signed)
[Hermès Gateway https://hermes.djibril.tld]
  ↓ (tier_dispatch.sh routing)
[GLM-5.1 / DeepSeek V4 Pro / Kimi K2.6 / Council]
  ↓
[Réponse JSON structurée]
```

### Helper côté Mac

`~/agence-ia/manager/qa_pipeline.sh` :

```bash
#!/usr/bin/env bash
# qa_pipeline.sh — pipeline complet creusage Reset Ultra
# Usage: qa_pipeline.sh --stage=relance|score|synth --thread-file=...

set -euo pipefail
STAGE="${1#--stage=}"
THREAD="$(cat "${2#--thread-file=}")"

case "$STAGE" in
  classif)
    ~/agence-ia/manager/hermes_call.sh --model=qwen-turbo --cap=80 \
      --task-type=classif "$THREAD"
    ;;
  relance)
    ~/agence-ia/manager/hermes_call.sh --model=glm-5.1 --cap=200 \
      --task-type=socratic "$THREAD"
    ;;
  score)
    ~/agence-ia/manager/hermes_call.sh --model=step-3.5-flash --cap=50 \
      --task-type=score "$THREAD"
    ;;
  synth)
    ~/agence-ia/manager/hermes_call.sh --model=deepseek-v4-pro --cap=400 \
      --task-type=synth --json-only "$THREAD"
    ;;
  council)
    ~/agence-ia/manager/llm_council.sh --debate=true \
      --models=deepseek-v4-pro,kimi-k2.6,glm-5.1,doubao-2.0,qwen-3.6-max \
      "$THREAD"
    ;;
esac
```

---

## 6. SÉCURITÉ + COÛT

### Rate limits par élève
- Max 5 nouvelles questions / jour (anti-spam)
- Max 3 tours de creusage par question
- Si tour 3 atteint sans qualif → force qualif (fallback)

### Budget par creusage complet
- Tier 3 classif : ~$0.0001
- Tier 1 relance ×2 : ~$0.001 × 2 = $0.002
- Tier 2 score ×2 : ~$0.0002 × 2 = $0.0004
- Tier 1 synth : ~$0.002
- **Total : ~$0.0044 par question** (sans council)
- Council N=3 + chairman : +$0.015 (~$0.02 total avec)

Cap mensuel : 30 élèves × 5 questions × 30 jours × $0.005 = ~$22.5/mois max.
Avec cache Hermès STABLE_BASE actif → -50% en moyenne = ~$11/mois.

### Anti-abuse
- Hash SHA256 de la question initiale → cache 7j (re-pose même question → renvoyer cache)
- Si élève spam (>10 questions/jour) → throttle + alerte Telegram Djibril

---

## 7. CHECKLIST DÉPLOIEMENT

- [ ] Migration SQL `qa_questions` (4 colonnes ajoutées)
- [ ] Refactor edge function `qa-handler` : 4 endpoints
- [ ] Exposer Hermès gateway en HTTPS permanent (ngrok ou cloudflared)
- [ ] Variables env Supabase : `HERMES_GATEWAY_URL`, `HERMES_HMAC_SECRET`
- [ ] Tests bout-en-bout : 5 scénarios élèves typiques
- [ ] Monitoring : log `llm_cost_eur` par question, dashboard Phoenix
- [ ] Doc admin Djibril : vue des `synthesis` JSON pour réponse rapide

---

## 8. MENTAL CHECK (champ ajouté côté front)

Le payload `PUT /me/entries/:date` reçoit maintenant en plus :

```json
"mental_state": {
  "energy": 7,           // 0-10
  "drive": 8,            // 0-10 (envie de pousser plus)
  "stress": 4,           // 0-10
  "mood_word": "tendu mais focus",
  "friction": "Le call de 16h, j'ai senti que je voulais pas pousser"
}
```

### Migration Supabase `entries`

```sql
ALTER TABLE entries ADD COLUMN IF NOT EXISTS mental_state JSONB;
```

### Vue admin (pour Djibril)

```sql
CREATE OR REPLACE VIEW daily_mental_dashboard AS
SELECT
  e.entry_date,
  s.first_name,
  e.mental_state->>'mood_word'   AS mood,
  (e.mental_state->>'energy')::int  AS energy,
  (e.mental_state->>'drive')::int   AS drive,
  (e.mental_state->>'stress')::int  AS stress,
  e.mental_state->>'friction'    AS friction,
  e.hours_worked,
  e.fulfillment,
  e.calls,
  e.ca_eur
FROM entries e
JOIN students s ON s.id = e.student_id
WHERE e.entry_date = CURRENT_DATE
ORDER BY (e.mental_state->>'stress')::int DESC NULLS LAST;
```

→ Djibril voit en 1 vue : qui stresse, qui flatline, qui pousse, qui décroche.

### Alerte auto (cron 18h)

Si `stress >= 8` OU `drive <= 3` OU `energy <= 3` → push Telegram à Djibril
avec `mood_word` + `friction`. Permet d'appeler l'élève le soir même.

---

## 9. ÉVOLUTIONS POSSIBLES PHASE 2

1. **TTS MiniMax** : lecture audio de la synthèse (l'élève écoute "voici ce que tu m'as vraiment dit aujourd'hui")
2. **Vocal input élève** : Doubao ASR → l'élève parle au lieu de taper
3. **Personnalisation profil** : après 10 questions, l'IA connaît les patterns récurrents et creuse direct au bon endroit
4. **Council auto** quand récurrence détectée : même blocage 3× en 2 semaines → council force pour solution stratégique
5. **Predictive bien-être** : score mental + heures + drive → prédit risque de burnout J+7
6. **Lien direct avec Basile** : si problème = "setting", basile sait que cet élève bloque sur X type d'objection
