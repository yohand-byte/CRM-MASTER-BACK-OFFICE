# Règles métier officielles — Pipeline & invariants

## 0) Packs & pricing (règle serveur)
Packs autorisés (strict) :
- ESSENTIEL : 169 €
- PRO : 269 €
- SERENITE : 449 €
- FLEX : prix calculé = somme des flexItems autorisés

flexItems autorisés (FLEX) :
- DP_MAIRIE : 89
- CONSUEL_BLEU : 45
- CONSUEL_VIOLET : 65
- RACCORD_ENEDIS : 99

Règles d’API :
- packCode inconnu => 400 invalid_pack
- flexItem inconnu => 400 invalid_flex_item
- createdAt est fixé par le serveur (pas par le frontend)

## 1) Conversion Lead → Client (UNICITÉ ABSOLUE)
### Règle officielle
Un lead ne peut produire qu’un seul client.

- Tant que lead.clientId == null : conversion autorisée
- Dès que lead.clientId != null : conversion interdite (définitive)

### Contrat API
Sur endpoint conversion :
- Si lead.clientId existe :
  - HTTP 409 Conflict
  - { "error": "lead_already_converted", "clientId": "<id>" }

### Contrat UI (Master CRM)
- Bouton “Convertir” désactivé si clientId existe
- Mention “Déjà converti”
- Lien “Voir le client”

## 2) Création de dossier (ANTI-DOUBLON)
### Problème actuel
Le Master CRM permet de créer 2 dossiers pour le même client sans garde-fou.

### Règle officielle (recommandée)
Un client ne doit avoir qu’un seul dossier ACTIF (non clôturé) à la fois.

- Avant création d’un dossier :
  - rechercher dossier “actif” du client (status != termine/annule)
  - si existe :
    - HTTP 409 Conflict
    - { "error": "client_already_has_active_file", "fileId": "<id>" }
  - sinon : créer

### Contrat UI (Master CRM)
- Action “Créer dossier” :
  - si 409 => ouvrir directement le dossier existant
  - afficher un toast clair (“Dossier déjà existant pour ce client”)

## 3) Pipeline dossier (statuts officiels)
Pipeline (4 étapes minimum) — base “lisible client” :

1) EN_COURS — Dossier ouvert
   - objectifs : collecte infos + docs de base
2) MAIRIE — Déclaration préalable
   - sous-statut : a_faire / depose / accepte / refuse
3) CONSUEL — Attestation / visite
   - sous-statut : a_faire / planifie / valide / refuse
4) ENEDIS_EDF — Raccordement + contrat OA
   - sous-statut : a_faire / en_cours / termine

Statuts de fin :
- TERMINE
- ANNULE

## 4) Règles d’affichage (packs/prix/dates)
- Partout (Leads/Clients/Dossiers + détails + modales) :
  - pack affiché = packLabel (fallback pack.code / packCode)
  - prix affiché = packPrice (fallback basePrice / price)
  - date affichée = createdAt (format JJ/MM/AAAA)

## 5) Portails web.app (admin/client)
Règles strictes :
- Pas de conversion lead → client
- Pas de création client
- Pas de création dossier (sauf si explicitement “demande” et validée côté Master CRM)
- Accès aux dossiers filtré par droits (installerId / clientId) via règles Firestore ou API

## 6) Pagination (contrat)
Toutes les listes paginées doivent être cohérentes :
- paramètres : limit + offset (ou cursor)
- réponse : items + total + nextCursor (si cursor)
- UI :
  - affichage “20/page”
  - navigation pages
  - conservation des filtres et de la recherche
