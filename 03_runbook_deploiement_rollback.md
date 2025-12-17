# Runbook — Déploiement, rollback, et incidents (Firebase + Cloud Run)

## 1) Déploiement Cloud Run (stratégie safe)
Objectif : déployer sans risque, avec rollback immédiat.

### Étapes recommandées
1) Déployer une nouvelle révision SANS trafic (no-traffic)
2) Taguer la révision (ex: prod-packs, prod-dashboard-v2)
3) Basculer le trafic progressivement : 0% → 10% → 50% → 100%
4) Garder la commande rollback prête

### Rollback instantané (modèle)
```bash
gcloud run services update-traffic solaire-frontend \
  --region europe-west1 \
  --to-revisions=<REVISION_STABLE>=100
```

## 2) Gestion des révisions et tags
- Toujours noter :
  - la révision stable actuelle (100% trafic)
  - la nouvelle révision candidate
  - les tags appliqués
- Toujours conserver une URL directe taggée pour valider :
  - prod-packs
  - prod-dashboard-v2
  - staging-*

## 3) Incident : “Firebase Error (auth/api-key-expired…)”

Symptôme :
- Les pages web.app (admin/login, client/login) affichent :
  Firebase: Error (auth/api-key-expired.-please-renew-the-api-key.)

Interprétation probable :
- Mauvaise clé Firebase côté build/runtime du frontend web.app
- Clé remplacée/invalidée
- Mauvais projet Firebase
- Restriction API Key (HTTP referrers) trop stricte
- Variable d’environnement VITE_FIREBASE_API_KEY incorrecte sur la build déployée

Check-list corrective (sans blabla)
1. Vérifier la config Firebase côté frontend web.app :
   - VITE_FIREBASE_API_KEY
   - VITE_FIREBASE_AUTH_DOMAIN
   - VITE_FIREBASE_PROJECT_ID
   - etc.
2. Vérifier Google Cloud Console :
   - APIs & Services → Credentials → API keys
   - Restrictions : HTTP referrers incluent bien :
     - https://solaire-frontend.web.app/*
     - https://solaire-frontend.firebaseapp.com/*
     - (et tout domaine custom éventuel)
   - Firebase Auth API activée
3. Vérifier que le build déployé utilise la bonne .env / substitutions :
   - éviter qu’un build “staging” remplace la config “prod”
4. Correctif :
   - remettre une API key valide
   - re-déployer web.app (Hosting) avec les bonnes env
   - retester login

## 4) Validation après déploiement (smoke tests)

Master CRM (Cloud Run)
- Ouvrir : / (dashboard)
- Vérifier :
  - Leads list OK (packs/prix/date)
  - Clients list OK (packs/prix/date)
  - Dossiers list OK (packs/prix/date)
  - Pagination OK (20/page)

Conversion anti-duplication
- Convertir un lead une fois : OK
- Re-cliquer convertir (ou refresh + retry) : doit retourner 409 et UI doit bloquer

Dossier anti-doublon
- Créer dossier pour un client : OK
- Re-créer dossier pour le même client : doit retourner 409 + redirection vers dossier existant

web.app (Firebase)
- admin/login : connexion OK
- client/login : magic link OK
- après login : données visibles (pas de page blanche)

## 5) Politique de merge
- PR séparées recommandées :
  1. packs/pricing alignment
  2. dashboard v2
  3. fixes métier (anti-dup lead→client, anti-doublon dossier)
- Toujours merger “métier critique” avant “cosmétique UI”
