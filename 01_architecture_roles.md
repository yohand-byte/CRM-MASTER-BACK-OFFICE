# Architecture & rôles — Solaire Admin Facile

## 1) Applications (3 frontends)

### A. Master CRM (Admin interne — source de vérité)
- URL : https://solaire-frontend-828508661560.europe-west1.run.app/
- Rôle : administration complète
  - Gestion Leads / Clients / Dossiers
  - Conversion Lead → Client
  - Création/édition des dossiers
  - Export CSV, filtres, recherche, pagination
- Auth : accès interne uniquement

### B. Espace Admin (installateur / équipe opérationnelle)
- URL : https://solaire-frontend.web.app/admin/login
- Rôle : gestion opérationnelle (limité)
  - Accès aux dossiers selon droits (ex: installerId)
  - Mise à jour d’avancement (statuts, dates, pièces)
  - Pas de création client depuis un lead
- Auth : Firebase (email/password ou magic link selon implémentation)

### C. Espace Client final (client B2C / B2B)
- URL : https://solaire-frontend.web.app/client/login
- Rôle : visibilité + dépôt de pièces
  - Consultation des étapes, statuts, prochaines actions
  - Dépôt de documents (selon règles)
  - Pas d’actions de conversion, pas de création d’entités métier
- Auth : Firebase (magic link recommandé)

## 2) Backend API (Cloud Run)
- Base : https://solaire-api-828508661560.europe-west1.run.app
- Responsabilités :
  - Validation stricte des packs (ESSENTIEL/PRO/SERENITE/FLEX)
  - Calcul packPrice côté serveur (FLEX = somme flexItems)
  - Persistance createdAt côté serveur
  - Règles métier critiques (unicité conversions, unicité dossiers, etc.)

## 3) Données (modèle)
### Lead
- id
- company
- name
- email
- phone
- volume
- packCode, packLabel, packPrice
- flexItems[]
- source
- createdAt
- clientId (nullable)  ✅ pivot anti-duplication

### Client
- id
- company
- name
- email
- phone
- packCode, packLabel, packPrice
- createdAt

### Dossier
- id
- clientId (obligatoire)
- title
- status (pipeline)
- packCode, packLabel, packPrice (copie depuis lead/client selon règle)
- champs opérationnels (adresse, puissance, dates, PDL, etc.)
- createdAt

## 4) Principe “Source of Truth”
- Le Master CRM est la vérité métier (création, conversion, décisions).
- Les espaces web.app doivent être considérés comme “portails” :
  - Lecture + mise à jour contrôlée
  - Jamais de création d’entités métier depuis ces portails
  - Jamais de conversion lead → client depuis ces portails
