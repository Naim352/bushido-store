# 🥋 BUSHIDO STORE — Guide complet pas à pas

## Vue d'ensemble de l'architecture

```
Visiteur
   ↓
public/index.html  ← ton site (HTML/CSS/JS)
   ↓  appelle
server/index.js    ← ton serveur Node.js
   ├── /api/auth/inscription   → crée le compte en base
   ├── /api/auth/connexion     → vérifie et connecte
   ├── /api/profil             → lit/modifie le profil
   ├── /api/create-payment-intent → Stripe
   └── /api/webhook            → Stripe confirme → sauvegarde commande
         ↓
   PostgreSQL (base de données sur Railway)
   Stripe (paiements réels)
```

---

## ÉTAPE 1 — Créer ton compte Railway (hébergement gratuit pour démarrer)

1. Va sur **https://railway.app**
2. Clique **"Start a New Project"**
3. Connecte ton compte GitHub (crée-en un si tu n'en as pas)

---

## ÉTAPE 2 — Créer la base de données PostgreSQL

1. Dans Railway → **"New"** → **"Database"** → **"Add PostgreSQL"**
2. Attends 30 secondes que ça démarre
3. Clique sur la base → onglet **"Connect"**
4. Copie la ligne **"DATABASE_URL"** (elle commence par `postgresql://...`)

C'est tout ! Pas besoin d'éditeur SQL ni de coller `database.sql` quelque part : le serveur crée lui-même les tables `clients` et `commandes` automatiquement la première fois qu'il démarre (étape 3). Tu verras `✅ Tables vérifiées/créées avec succès` dans les logs Railway si ça a marché.

---

## ÉTAPE 3 — Déployer le serveur

### Option A (recommandé) : Via GitHub

1. Crée un repository GitHub et mets-y tous les fichiers
2. Dans Railway → **"New"** → **"GitHub Repo"** → choisis ton repo
3. Railway détecte automatiquement Node.js et lance `npm start`

### Option B : Via CLI Railway

```bash
# Installer le CLI Railway
npm install -g @railway/cli

# Dans le dossier server/
cd server
railway login
railway link    # lie à ton projet
railway up      # déploie
```

---

## ÉTAPE 4 — Configurer les variables d'environnement

Dans Railway → ton service → onglet **"Variables"** → **"Add Variable"**

Ajoute ces variables une par une :

| Variable | Valeur | Où la trouver |
|----------|--------|---------------|
| `DATABASE_URL` | `postgresql://...` | Railway → ta base → Connect |
| `STRIPE_SECRET_KEY` | `sk_live_...` | dashboard.stripe.com → Développeurs → Clés API |
| `STRIPE_WEBHOOK_SECRET` | `whsec_...` | Stripe → Webhooks (voir étape 5) |
| `JWT_SECRET` | une longue chaîne aléatoire | génère sur https://www.uuidgenerator.net/ (mets 2 UUID collés) |
| `ADMIN_PASSWORD` | un mot de passe de ton choix | choisis quelque chose fort |
| `FRONTEND_URL` | `https://ton-projet.up.railway.app` | l'URL que Railway te donne |
| `ADMIN_EMAIL` | `naimbenazzouz5@gmail.com` | l'adresse qui reçoit TES notifications |
| `GMAIL_USER` | `naimbenazzouz5@gmail.com` | le compte Gmail qui ENVOIE les emails |
| `GMAIL_APP_PASSWORD` | `xxxx xxxx xxxx xxxx` | voir étape 5d ci-dessous |

---

## ÉTAPE 5d — Notifications par email (nouveau compte / nouvel achat)

Le serveur envoie automatiquement :
- **À toi** (`ADMIN_EMAIL`) : un email quand un client crée un compte, et un email quand un paiement est confirmé
- **Au client** : un email de bienvenue avec le lien direct vers son espace personnel

Pour que ça fonctionne, il faut un **mot de passe d'application Gmail** (différent de ton mot de passe normal) :

1. Active la validation en 2 étapes sur ton compte Google : **myaccount.google.com/security**
2. Va sur **myaccount.google.com/apppasswords**
3. Crée un mot de passe d'application (nom libre, ex: "Bushido Store")
4. Google te donne un code à 16 caractères → copie-le dans `GMAIL_APP_PASSWORD` (Railway → Variables)
5. Mets ton adresse Gmail dans `GMAIL_USER` et `ADMIN_EMAIL`

⚠️ Si `GMAIL_USER`/`GMAIL_APP_PASSWORD` ne sont pas configurés, le site fonctionne quand même (paiement, comptes) mais aucun email n'est envoyé — ça s'affiche juste dans les logs Railway.

---

## ÉTAPE 5e — Accéder à ta page "Clients" en secret

Le lien **"Clients"** a été retiré du menu visible : tes visiteurs ne le voient plus et ne peuvent pas tomber dessus par hasard.

Pour y accéder toi-même, va sur :
```
https://TON-SITE.up.railway.app/#admin-bushido
```

Tu peux changer le mot "bushido" par ton propre code secret : ouvre `public/index.html`, cherche la ligne :
```js
const ADMIN_SECRET = 'bushido';
```
et remplace `'bushido'` par ce que tu veux (ex: `'naim2026secret'`). Garde cette URL pour toi seul.

Une fois sur la page, clique **"🔐 Charger les clients"** et entre ton `ADMIN_PASSWORD` — c'est la deuxième protection (même si quelqu'un trouve l'URL secrète, il ne peut pas voir les données sans ce mot de passe).

---

## ÉTAPE 5 — Configurer Stripe

### 5a. Récupérer tes clés

1. Va sur **https://dashboard.stripe.com/apikeys**
2. **Clé secrète** `sk_live_...` → copie dans `STRIPE_SECRET_KEY`
3. **Clé publiable** `pk_live_...` → à mettre dans `public/index.html`

### 5b. Mettre la clé publique dans le HTML

Dans `public/index.html`, cherche cette ligne :
```js
const STRIPE_PUBLIC_KEY = 'pk_live_VOTRE_CLE_PUBLIQUE_ICI';
```
Remplace par ta vraie clé `pk_live_...`

### 5c. Configurer le Webhook

1. Stripe → **Développeurs** → **Webhooks** → **"Ajouter un endpoint"**
2. URL : `https://TON-SITE.up.railway.app/api/webhook`
3. Événement : `payment_intent.succeeded`
4. Copie le **"Signing secret"** `whsec_...` → `STRIPE_WEBHOOK_SECRET`

---

## ÉTAPE 6 — Ton URL finale

Railway te donne une URL du type : `https://bushido-store-production.up.railway.app`

Tu peux aussi connecter un domaine personnalisé :
- Railway → ton service → **"Settings"** → **"Custom Domain"**
- Entre `bushidostore.fr` (ou ce que tu veux)
- Puis configure le DNS chez ton registrar (OVH, Namecheap, etc.)

---

## Ce qui se passe quand un client utilise le site

### Inscription
```
Client remplit le formulaire
        ↓
POST /api/auth/inscription
        ↓
Mot de passe hashé (bcrypt)
        ↓
Compte créé dans PostgreSQL
        ↓
Token JWT renvoyé au navigateur
        ↓
Client connecté, son espace s'affiche
```

### Achat
```
Client clique "Payer"
        ↓
POST /api/create-payment-intent → Stripe crée un paiement
        ↓
Client entre sa carte → Stripe valide
        ↓
Stripe envoie confirmation au webhook → /api/webhook
        ↓
La commande est sauvegardée dans PostgreSQL
        ↓
Le total dépensé + points du client sont mis à jour
        ↓
L'argent arrive sur ton compte Stripe (virement auto tous les 7j)
```

### Toi tu vois tes clients
```
Tu vas sur ton-site.com/#admin-bushido (lien secret, pas dans le menu)
        ↓
Clique "Charger les clients"
        ↓
Entre ton ADMIN_PASSWORD
        ↓
Tous tes vrais clients s'affichent avec leurs commandes
```

### Toi tu reçois une notification (email)
```
Client crée un compte OU paye
        ↓
Le serveur t'envoie un email à ADMIN_EMAIL
        ↓
Tu sais en temps réel qui s'inscrit / qui achète, combien
```

---

## Recevoir l'argent

- Stripe garde l'argent ~7 jours puis vire sur ton compte bancaire
- Configure ton RIB : Stripe Dashboard → **"Paramètres"** → **"Données bancaires"**
- Tu peux voir chaque paiement en temps réel sur **dashboard.stripe.com**

---

## Structure des fichiers à déployer

```
bushido/
├── package.json          ← dépendances (à la racine, important pour Railway)
├── server/
│   ├── index.js          ← le serveur (NE PAS MODIFIER)
│   ├── database.sql      ← référence (les tables se créent toutes seules)
│   └── .env.example      ← modèle des variables
└── public/
    └── index.html        ← ton site (avec ta clé Stripe dedans)
```

---

## Problèmes fréquents

| Problème | Solution |
|----------|----------|
| `Cannot find module` | Lance `npm install` dans le dossier `server/` |
| Stripe erreur 401 | Vérifie `STRIPE_SECRET_KEY` dans Railway Variables |
| DB erreur | Vérifie que tu as bien lancé `database.sql` sur Railway |
| JWT expired | Le token dure 30 jours, l'utilisateur se reconnecte |
| Webhook 400 | `STRIPE_WEBHOOK_SECRET` incorrect ou endpoint mal configuré |
