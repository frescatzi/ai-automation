---
type: raw
title: "POS-EXACT_Iris-qualite_Peupler-canon-memoire-marque_2026-07-20"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_iris-qualite_peupler-canon-memoire-marque_2026-07-20.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT · Peupler la mémoire de marque AFTRSN (canon) depuis les docs Brand + Marketing

Date : 2026-07-20 · Marque : AFTER SUN PEOPLE (AFTRSN) · Auteur : session Claude (Cowork) · Validé par : Karter (approche + tagline + purge)
Contexte : chantier « qualité Iris ». Le tool Knowledge d'Iris renvoyait vide sur la voix. Karter a fourni 2 docs de marque et demandé de stocker les infos utiles dans la mémoire (Knowledge).

## Résultat
Collection `canon` de `aftrsn_memory` passée de 7 à 11 entrées : 2 anciennes purgées, 6 nouvelles curées ajoutées. Vérifié par recherche sémantique (le tool Knowledge remonte désormais voix, tagline et specs en tête).

## Architecture mémoire (relevée sur l'instance)
- Lecture : tool `Knowledge` d'Iris = `POST https://n8n.aftersunpeople.com/webhook/memory-search` body `{question, brand:'aftrsn'}` ; même banque que le MCP `search_brand_memory` (brand `aftrsn`). Table Postgres `aftrsn_memory` (pgvector), colonnes title, content, collection, knowledge_type, source, source_ref, content_hash (md5 du content), embedding.
- Écriture idempotente : workflow `LUMINA-MEMORY-INGEST-TEXT` (`ugINEyyBKRLcDpH2`, executeWorkflowTrigger). Entrée `{brand,title,content,collection,knowledge_type,source,source_ref}`. Chunk 3000/overlap 200 (titre suffixé « [n] »), embed OpenAI `text-embedding-3-small` (cred `fqQvWAsn0tBEhSGU`), INSERT ... ON CONFLICT (content_hash) DO NOTHING dans `{brand}_memory`.
- ATTENTION : `AFTRSN-MEMORY-INGESTION/DRIVE→AFTRSN_MEMORY` (`krWBy2Sj9k47H24z`) fait un TRUNCATE de la table avant reconstruction (rebuild complet depuis Drive), à ne PAS utiliser pour un ajout.
- Pas d'outil SQL direct exposé et les workflows mémoire sont en executeWorkflowTrigger (non déclenchables via l'API test webhook/form/chat).

## Décisions Karter
1. Approche : entrées canon CURÉES (pas de dump brut du doc marketing de 24k mots).
2. Tagline canonique : « Good People. Great Vibes. Beautiful Energy. » (Marketing Concept, juin 2026). Ancienne « Good People. Beautiful Vibes. After Sun. » (Corporate Brand Identity, février 2026) marquée OBSOLÈTE dans l'entrée Brand Facts.
3. Purge des anciens chunks « Corporate Brand Identity » avant ajout.

## Procédure exécutée
1. Utilitaire temporaire A `ZZZ_tmp_AFTRSN-CANON-ADMIN_0720` (webhook POST → Postgres executeQuery `{{$json.body.sql}}`, cred `LUMINA_Postgres` `mlN3noHH3TT9C6Eu`), activé, déclenché via n8n_test_workflow.
   - Inventaire : canon=7 (episodic 39, insights 22, skills 10). Les 7 canon = Corporate Brand Identity ids 1-2 (source Drive, kt standard) + SYSTEM-CANON LUMINA/AFTRSN ids 16-20.
   - Purge : `DELETE FROM aftrsn_memory WHERE collection='canon' AND title ILIKE 'Corporate Brand Identity%' RETURNING id,title` → supprime exactement ids 1 et 2.
2. Utilitaire temporaire B `ZZZ_tmp_AFTRSN-CANON-INGEST_0720` (webhook → Split → Chunk → Embed OpenAI → Build Insert → Postgres Insert), réplique exacte de LUMINA-MEMORY-INGEST-TEXT. Ingestion des 6 entrées (payload dans le brief) → 6 insérées (ids 94-99).
3. Vérification : canon=11 ; recherche « brand voice + tagline » → tagline 0.72 / voix 0.70 en tête ; recherche specs → Content Specs by Channel en tête. Ancien Corporate Brand Identity absent.
4. Nettoyage : suppression des 2 workflows utilitaires temporaires (aucun résidu sur l'instance).

## Les 6 entrées curées (collection canon, knowledge_type canon, source brand-docs, en anglais)
1. AFTRSN Brand Voice and Tone (id 94)
2. AFTRSN Tagline, Brand Facts and Naming (id 95)
3. AFTRSN Positioning, Essence and Differentiation (id 96)
4. AFTRSN Audience and Personas (id 97)
5. AFTRSN Content Specs by Channel (id 98)
6. AFTRSN Visual Identity and Event Experience (id 99)
Contenu détaillé : voir PROPOSITION_Iris-qualite_Entrees-canon-memoire-marque_2026-07-20.

## Incohérences sources réconciliées
- Tagline : février « Good People. Beautiful Vibes. After Sun. » vs juin « Good People. Great Vibes. Beautiful Energy. » → la juin fait autorité, l'ancienne épinglée obsolète.
- Genre : février « Melodic House » vs juin « Melodic Afro House » (AEDM) → retenu « Melodic Afro House ».

## À signaler / suite
- Stale repéré (hors périmètre, non touché) : SYSTEM-CANON v1.0 (ids 16-17) encore présent alors que v1.1 (ids 18-20) « remplace v1.0 ». Purge possible sur accord.
- Chantier Iris (voix/specs/variété/langue) : à reprendre. La voix et les specs sont désormais dans le canon ET dans le brief M2 ; reste surtout l'anti-répétition (axe C), le préambule français d'Iris et le « default French » du systemMessage.

## Invariants respectés
Rien publié (mémoire interne uniquement). Aucun workflow LUMINA de production modifié (utilitaires temporaires créés puis supprimés). Secrets non exposés. Pas de tiret cadratin.
