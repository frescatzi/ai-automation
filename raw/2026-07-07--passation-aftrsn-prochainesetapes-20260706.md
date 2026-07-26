---
type: raw
title: "PASSATION_AFTRSN_ProchainesEtapes_20260706"
source_url: "drive:1xok8ml79TGeM25Fmj6YYGo6otI6ry-Sf"
captured: 2026-07-07
vault: brands
brand: AFTRSN
immutable: true
---

# Passation — Prochaines étapes (06.07.2026)

**Statut :** Session W-7 Knowledge-Capture clôturée. **La plateforme AFTRSN est complète (Étapes 1→4 terminées).** Ce document **remplace `PASSATION_AFTRSN_ProchainesEtapes_20260705-NUIT.md`** comme relais courant.

---

## 1 · Boussole — toujours commencer ici
Lire **`FICHE_PROJET_AFTRSN_Automation.md`** (document compas, à jour au 06.07.2026). Le présent document se lit en second.

**Cap :** la construction de toute la plateforme est **terminée**. On bascule désormais en **mode test / amélioration continue**. Le **08.08.2026 = run de validation A→Z** par Karter (création édition → promo → post-event).

## 2 · Ce qui a été fait cette session (06.07.2026)

1. **W-7 Knowledge-Capture construit, testé et PUBLIÉ 🟢** (`AGoADrWzpZDa5b1g`, cron quotidien 09h). Modèle deux temps validé Karter : Phase A ouvre une fiche débrief auto sur `Done` (base Notion **Post-Event Débriefs** `21f26092-…`) ; Phase B capture les insights dans `aftrsn_memory`/`insights` quand la fiche passe en `Prêt`. Phase A + B testées end-to-end.
2. **Panne infra majeure détectée et résolue** : la credential Postgres partagée `Ojp0aTYdIFfNsr4u` est devenue fantôme → tout le cerveau (mémoire + 17 workflows/agents actifs) hors ligne. Karter a recréé **`LUMINA_Postgres` `mlN3noHH3TT9C6Eu`** ; repoint des 24 nodes + republication des 17 actifs. Vérifié : write, read, chat Maestro. **Piège clé** : `/activate` obligatoire après un changement de credential (le PATCH ne touche que le brouillon).
3. **Health-check mémoire** construit et publié (`LUMINA-HEALTHCHECK-MEMOIRE` `M4rvTAqAlhJUzNSy`, 07h30 quotidien, alerte Telegram si mémoire KO).
4. **Données de test nettoyées** : édition `ZZTEST-KC` → Cancelled ; 10 lignes ZZTEST purgées de `aftrsn_memory`.

## 3 · Fichiers produits cette session
| Fichier | Rôle |
|---|---|
| `Fiche_AFTRSN-Knowledge-Capture_20260706.md` | Fiche technique W-7 (as-built) |
| `POS-AFTRSN_Capture-Connaissance-Debrief_20260706.md` + `POS-GENERIQUE_…` | POS paire du pattern capture débrief → mémoire |
| `POS-LUMINA_Reparation-Credential-Postgres-Partagee_20260706.md` | Runbook infra : réparer une credential Postgres partagée (à pousser au canon) |
| `FICHE_PROJET_AFTRSN_Automation.md` | Boussole, à jour (plateforme complète, incident, health-check) |
| `PASSATION_AFTRSN_ProchainesEtapes_20260706.md` | Ce document |

## 4 · Prochaines étapes (mode test/amélioration)
- **Préparer le run A→Z du 08.08** : vérifier chaque processus publié bout-à-bout avec de vraies données (W-1 création → W-3/W-4 coordinators/marketing → W-6 budget → W-7 débrief).
- **Confirmer les 2 artistes du 08.08** (deadline 18.07, booking WhatsApp) et les consigner dans DJs/Artists (Lineup + Confirmed + email) pour déclencher le brief J-7.
- **Remplir la fiche débrief EP4** (déjà ouverte par W-7, statut `À remplir`) pour tester une capture réelle, ou attendre le 08.08.
- **Pousser au canon** (`wiki/` → collection `canon`) : POS credential Postgres, POS token Meta, règle « from day to night ». La mémoire étant réparée, l'ingestion wiki→pgvector refonctionne.

## 5 · Décisions / nettoyages en attente (Karter)
- **DA-010** — Supprimer via l'UI n8n les 2 workflows jetables : `ZZZ_20260705_tmp_mem_probe` (`n6uGOu4O84nmavND`) et `ZZZ_20260705_tmp_sql_runner` (`8RWQRa5tPILPT9DW`) — le DELETE API a renvoyé 400.
- **DA-009** et antérieures (`ZZZ_` divers) — voir Fiche Projet.
- Fiches Notion de test à corbeille : édition `ZZTEST-KC` (Cancelled) + sa fiche `Débrief — ZZTEST-KC` (Capturé).
- Fiche `Débrief — EP4` : réelle, à remplir ou laisser.

## 6 · Rappels transverses (invariants LUMINA OS)
Numérotation depuis 1. PII jamais dans le cloud. Actions critiques (dépense, envoi, publication, suppression) gatées par validation humaine. Embedding figé. Git = source de vérité. n8n fait foi. Aucune suppression/DROP sans Karter. Jamais réactiver un `ZZZ_` sans revue. Toujours HTTP Request pour Claude. L'agent ne saisit jamais de secret. **Nouveau : après tout changement de credential/param sur un workflow actif, republier via `/activate` (le PATCH ne touche que le brouillon).**

---
*Document rédigé le 06.07.2026 après clôture de la session W-7.*
