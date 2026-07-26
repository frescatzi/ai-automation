---
type: raw
title: "POS-EXACT_Allocation-modeles-LLM_2026-07-15"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_allocation-modeles-llm_2026-07-15.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT — Allocation des modèles LLM par rôle (Claude rédaction / OpenAI orchestration+analyse)

**Date :** 2026-07-15 · **Projet :** LUMINA OS / AFTRSN · **Objet :** verrouiller quel modèle LLM sert quel rôle, **tout en direct** (OpenAI **et** Anthropic, zéro OpenRouter). Fait suite à l'élimination d'OpenRouter (voir `POS-EXACT_Elimination-OpenRouter-LLM-direct_2026-07-15.md`).
**Statut :** ✅ **VERROUILLÉ & VALIDÉ LIVE.**

---

## 1. Règle d'allocation (validée Karter)

| Rôle | Modèle | Node / Credential | Pourquoi |
|---|---|---|---|
| **AFTRSN-Secretary** `FtPFOKvi2t18DFb1` | **Claude Sonnet 5** | `lmChatAnthropic` · cred « Anthropic account » `2CkrotK34KR7IVQF` · `claude-sonnet-5` | Rédaction (emails partenaires/comms) — qualité + voix |
| **AFTRSN-Marketing** `wyUEojwZjSKFRaRL` | **Claude Sonnet 5** | idem | Garde-voix de marque + plans marketing — était Claude par design |
| **AFTRSN-CHANNEL-CONTENT** `MWUwLi3vu2Ws65lL` | **Claude Sonnet 5** | idem | Rédaction du copy canal |
| **AFTRSN-Maestro** `OlrQO21u178SjgBK` | **OpenAI gpt-5-mini** | `lmChatOpenAi` (node `OpenAI Chat Model (test M)`) · cred « OpenAi account » `fqQvWAsn0tBEhSGU` | Orchestrateur — **Claude "narre" au lieu de clore le JSON** (leçon M7/M8) → surtout PAS Claude |
| **AFTRSN-Qualite-Compliance** `zbW4p5qsBYhkfBdi` | **OpenAI gpt-5-mini** | `lmChatOpenAi` · « OpenAi account » | Vérificateur analytique — aligné sur Maestro (15-07, était gpt-4o-mini) |
| **AFTRSN-Comptable-Finance** `F1wsXbsYZdEkwQ3n` | **OpenAI gpt-5-mini** | idem | Vérificateur analytique — aligné sur Maestro (15-07) |
| **AFTRSN-Culture-Steward** / **Experience-Designer** | **OpenAI** (statu quo) | `lmChatOpenAi` | Advisory/créatif — à réévaluer si besoin |

**Principe :** **Claude direct = rédaction / voix de marque ; OpenAI direct = orchestration + analyse.** Les deux fournisseurs en **direct** (Anthropic + OpenAI), aucun OpenRouter.

---

## 2. Évaluation qui a motivé le choix (live, 15-07)

Même tâche rédactionnelle passée par la Secrétaire, GPT-4o-mini vs Claude Sonnet 5 :
- **gpt-4o-mini** : n'a **pas** consulté la mémoire de marque, a écrit de mémoire et **confondu la marque avec du skincare** (« rituels de soins innovants »). Générique.
- **Claude Sonnet 5** : a **appelé l'outil Knowledge**, récupéré la voix (African Electronic Dance, éditorial/soulful, « cultural without clichés »), écrit dans le ton juste + note utile à Karter. Idem pour Marketing (2 lookups : voix + `forbidden_words`, clichés bannis, dépense signalée pour validation).

→ **Supériorité rédactionnelle nette de Claude** sur les rôles d'écriture/voix.

---

## 3. Latence mesurée (15-07)

- **Maestro direct** (gpt-5-mini, sans délégation) : **~17,5 s** end-to-end.
- **Sous-agent Claude Sonnet 5 standalone** (Marketing, 2 recherches mémoire) : **~16 s** (node LLM ~10 s).
- Chaîne **Maestro → sous-agent Claude** (vraie décision déléguée) estimée **~25-35 s**, dans la cible 30-60 s de Maestro.
- Le sous-agent Claude ≈ **2× la latence de gpt-4o-mini**, mais n'est sollicité que sur un **vrai enjeu** (Maestro plafonne à 1-2 tours de délégation). **Trade-off qualité/coût assumé pour les rôles d'écriture.**

---

## 4. Méthode de bascule (store Pinia)

Node LLM langchain : muter `node.type` → `@n8n/n8n-nodes-langchain.lmChatAnthropic`, `typeVersion` 1.3, `parameters.model={__rl:true,value:'claude-sonnet-5',mode:'list',cachedResultName:'Claude Sonnet 5'}`, `node.credentials={anthropicApi:{id:'2CkrotK34KR7IVQF',name:'Anthropic account'}}`, **garder le nom du node** (connexion `ai_languageModel` préservée). Puis **vrai drag sur le corps d'un node** (pas un connecteur, sinon le panneau d'ajout s'ouvre) → **Publish**.

---

## 5. Difficultés / Solutions / Lessons learned

### Difficultés
- Le drag pour « salir » ouvrait le panneau d'ajout de node quand il partait d'un connecteur.
- Maestro n'a pas délégué à Marketing lors du test end-to-end (règle « vitesse d'abord » → réponse directe), donc le chemin Marketing-Claude a dû être mesuré en standalone.

### Solutions
- Draguer le **corps** d'un node (loin des connecteurs) pour marquer dirty ; `Escape` pour fermer le panneau d'ajout si ouvert.
- Mesurer Marketing en **standalone** pour isoler la latence du sous-agent Claude.

### Lessons learned
- **Le bon modèle dépend du RÔLE** : rédaction/voix → Claude (utilise la mémoire, respecte la voix) ; orchestration → un modèle qui obéit au format (gpt-5-mini ; Claude narre) ; analyse/vérif → mini OpenAI (cheap). Voir [[lumina-maestro-latence-modeles]].
- **« Modèle OpenAI » avait confondu la marque** : sans consultation mémoire, un petit modèle hallucine le positionnement. Claude va chercher le canon de lui-même.
- Cibler Claude **là où l'écriture compte**, pas partout, pour maîtriser la facture Anthropic.

---

*POS-EXACT Allocation modèles LLM — 2026-07-15, verrouillé. Claude Sonnet 5 direct = rédaction/voix (Secretary, Marketing, Channel-Content) ; OpenAI direct = orchestration (Maestro gpt-5-mini) + analyse (Qualité, Comptable gpt-5-mini). Tout en direct, zéro OpenRouter. n8n fait foi.*
