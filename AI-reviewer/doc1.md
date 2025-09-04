

# 📖 README — AI Code Review sur GitLab EE

## 🚀 Objectif

Automatiser les **reviews de code** dans GitLab en utilisant un modèle open-source (ex: [Qwen2.5-Coder](https://huggingface.co/Qwen/Qwen2.5-Coder), [StarCoder2](https://huggingface.co/bigcode/starcoder2), [DeepSeek-Coder](https://huggingface.co/deepseek-ai/deepseek-coder)).
Chaque fois qu’un développeur ouvre ou met à jour une **Merge Request**, une **IA reviewer** lit les changements (*diff*) et publie un commentaire clair avec :

* 🐞 Risques & bugs probables
* 🧹 Dettes techniques & code smells
* 💡 Suggestions concrètes (mini-patchs)
* 🧪 Tests à ajouter
* 🔐 Points de sécurité & perf

---

## 🔧 Mise en place

### 1. Variables GitLab à configurer

Dans **Settings ▸ CI/CD ▸ Variables** :

* `GITLAB_TOKEN` → token personnel (PAT) avec scope `api` pour poster des commentaires.

### 2. Runner nécessaire

* **Docker runner** ou Kubernetes runner.
* Pas besoin de GPU pour petits diffs (CPU suffit).

### 3. Modèle IA

Choisissez un modèle adapté au code :

* `qwen2.5-coder:7b-instruct` (raisonnement solide)
* `starcoder2:7b` (open-source, entraîné sur gros corpus code)
* `deepseek-coder:6.7b` (rapide et compact)

Téléchargés automatiquement par [Ollama](https://ollama.com).

---

## 📜 Exemple `.gitlab-ci.yml`

```yaml
stages: [ai_review]

ai-review:
  stage: ai_review
  image: curlimages/curl:8.8.0
  services:
    - name: ollama/ollama:latest
      alias: ollama
      command: ["serve"]
  variables:
    OLLAMA_HOST: http://ollama:11434
    MODEL_NAME: qwen2.5-coder:7b-instruct
    MAX_INPUT_CHARS: "90000"
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

  before_script:
    - for i in $(seq 1 60); do
        if curl -s ${OLLAMA_HOST}/api/tags >/dev/null; then break; fi
        sleep 2
      done
    - curl -sS -X POST ${OLLAMA_HOST}/api/pull \
        -H 'Content-Type: application/json' \
        -d "{\"name\":\"${MODEL_NAME}\"}" >/dev/null

  script:
    - git fetch origin "$CI_MERGE_REQUEST_TARGET_BRANCH_NAME" --depth=1
    - git diff --unified=0 "origin/${CI_MERGE_REQUEST_TARGET_BRANCH_NAME}...$CI_COMMIT_SHA" > patch.diff || true
    - |
      TRUNCATED_DIFF="$(sed -E 's/[`$]/ /g' patch.diff | head -c ${MAX_INPUT_CHARS})"
      PROMPT=$(cat <<'EOF'
Tu es un **senior code reviewer**. Analyse UNIQUEMENT le diff fourni.
Donne en Markdown :
1) Risques & bugs (citer lignes +/-)
2) Dettes techniques
3) Suggestions avec mini-patchs
4) Tests à ajouter
5) Points sécurité & perf
EOF
)
      PAYLOAD=$(jq -n --arg model "$MODEL_NAME" --arg sys "$PROMPT" --arg diff "$TRUNCATED_DIFF" '
        { "model": $model,
          "messages": [
            {"role":"system","content":$sys},
            {"role":"user","content":("DIFF:\n<<DIFF>>\n" + $diff + "\n<</DIFF>>")}
          ],
          "stream": false }')

      REVIEW=$(curl -sS -X POST "${OLLAMA_HOST}/api/chat" \
        -H "Content-Type: application/json" -d "$PAYLOAD" | jq -r '.message.content')

      NOTE_PAYLOAD=$(jq -n --arg body "$REVIEW" '{body:$body}')
      curl -sS -X POST \
        "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests/${CI_MERGE_REQUEST_IID}/notes" \
        -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
        -H "Content-Type: application/json" \
        -d "$NOTE_PAYLOAD" >/dev/null
```

---

## 👩‍💻 Workflow pour un développeur

1. **Créer une branche** → modifier une brique (`feature/refacto-pricing`).
2. **Commit & push**.
3. **Ouvrir une Merge Request**.
4. La CI se lance automatiquement et :

   * récupère le diff
   * envoie le diff au modèle IA
   * poste un commentaire **AI Code Review** dans la MR.
5. **Lire le commentaire IA** (risques, bugs, suggestions, tests à ajouter).
6. **Corriger et pousser** → la CI relance une review automatiquement.
7. **Finaliser la MR** (revue humaine + merge).

---

## 📊 Exemple de sortie

> **AI Code Review — Brique `pricing/calculator.py`**
> **1) Risques & bugs**
>
> * `calculator.py:84 (+)` : division par `quantity` sans garde → risque `ZeroDivisionError`.
>
> **2) Dettes techniques**
>
> * Duplication de logique d’arrondi avec `invoice/utils.py`.
>
> **3) Suggestions (mini-patchs)**
>
> ```diff
> - total = price * quantity / discount
> + if discount == 0:
> +     return 0  # ou lever ValueError
> + total = (price * quantity) / discount
> ```
>
> **4) Tests à ajouter**
>
> * Cas `discount=0`.
> * Cas `quantity=0`.

---

## 🔒 Notes importantes

* ✅ Le code reste **confidentiel** (modèle exécuté dans la CI).
* 🛑 L’IA ne merge pas → elle **suggère**, la décision finale reste humaine.
* 📏 Les gros diffs sont tronqués (≈90KB max).
* ⚡ Inline comments possibles (API Discussions GitLab).

---

## 🛠️ Roadmap possible

* [ ] Inline comments sur les lignes modifiées.
* [ ] Bouton **“Relancer la review IA”** (job manuel).
* [ ] Ajout d’un score global de qualité.
* [ ] Combinaison avec `reviewdog` (lint + IA).

---

👉 Avec ce setup, un·e dev qui change une brique voit immédiatement un **feedback IA dans la MR**, sans rien faire de plus.


