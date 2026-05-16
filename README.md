# 🎓 IBSA — Cahier de texte numérique v45

**International Bilingual School of Africa — Ouagadougou**

Version 45 — Mai 2026

---

## 🆕 Nouveautés v45 (correctifs critiques v44)

Cette version corrige trois lacunes critiques de la v44 signalées par l'usage :

### ✅ Problème 1 corrigé — Séances visibles par la direction

**Cause** : la fonction `sbSaveSeance` envoyait à Supabase des champs (`chapitre`, `objectifs`, `observations`, `signales`, `files`) qui n'existaient pas dans la table `seances` si l'Étape 1 du SETUP-SUPABASE.md n'avait pas été exécutée. L'erreur SQL était silencieusement avalée par le `try/catch`, donc les enseignants pensaient avoir sauvegardé leur séance alors qu'elle ne quittait jamais leur appareil.

**Correctifs** :
- Si l'INSERT complet échoue, l'app retente automatiquement avec un payload minimal (colonnes garanties d'exister)
- Toute erreur Supabase est désormais **affichée à l'enseignant** via un toast et loggée dans la console (F12)
- Les directeurs voient maintenant les séances de TOUS les enseignants, y compris du secondaire (sciences, etc.)

### ✅ Problème 2 corrigé — Statistiques d'appel synchronisées

**Cause** : double bug dans `sbSaveAppel` :
1. Mêmes échecs silencieux que pour les séances quand les colonnes agrégats n'existaient pas
2. La contrainte `onConflict: 'classe,date,heure,matiere'` faisait qu'**un appel d'un enseignant pouvait écraser celui d'un autre** (cas typique : deux enseignants secondaire qui partagent la même classe)

**Correctifs** :
- Recherche explicite par `teacher_id + classe + date + heure + matiere` avant insertion (pas d'écrasement entre enseignants)
- Mode dégradé automatique si colonnes agrégats manquantes
- Feedback visible des erreurs

### ✅ Problème 3 corrigé — Partage WhatsApp/Email amélioré

**Avant** : le lien `wa.me/?text=...` ouvrait WhatsApp sans destinataire et obligeait l'enseignant à chercher le contact manuellement.

**Maintenant** : la fenêtre de partage propose 3 champs :
- 👤 **Nom du parent** (personnalise le message « Bonjour Mme Diallo,... »)
- 📱 **Numéro WhatsApp** (avec indicatif pays — ex : 22670XXXXXX) qui ouvre **directement la conversation avec ce parent**
- 📧 **Email du parent** (prérempli dans le destinataire du mail)

Le message envoyé contient automatiquement :
- Classe, matière, date, enseignant
- 📚 Chapitre étudié
- 🎯 Objectifs
- 📝 Résumé du cours
- ✏️ Travail à faire
- 🔗 Lien direct de téléchargement du document
- Signature de l'enseignant

L'enseignant peut éditer le message avant envoi, et les derniers contacts utilisés sont mémorisés (localStorage local).

### 🔄 Auto-rafraîchissement direction (nouveauté)

Pour les comptes direction (admin, directeur d'école, directeur pédagogique, direction générale, maintenance), l'app **rafraîchit automatiquement les données toutes les 60 secondes** depuis Supabase. Plus besoin de se déconnecter/reconnecter pour voir les nouvelles séances et appels saisis par les enseignants.

---

## 📁 Structure des fichiers

```
ibsa-v45-netlify/
├── index.html           ← Application complète (corrigée)
├── config.js            ← URL et clé Supabase
├── netlify.toml         ← Configuration Netlify
├── _redirects           ← Routes (SPA-style)
├── logo.png             ← Logo IBSA
├── SETUP-SUPABASE.md    ← Identique à v44 — à exécuter si pas déjà fait
└── README.md            ← Ce fichier
```

---

## 🚀 Migration depuis v44

**Bonne nouvelle** : aucune migration de base de données n'est nécessaire si vous aviez déjà exécuté correctement le `SETUP-SUPABASE.md` v44.

Si vous n'aviez **pas** exécuté l'Étape 1 du SETUP-SUPABASE.md (ALTER TABLE ajoutant `chapitre`, `objectifs`, `observations`, `signales`, `files`, et les agrégats `presents`, `absents`, etc.), c'est la cause des problèmes. Dans ce cas :

1. **Soit** vous exécutez maintenant l'Étape 1 du SETUP pour récupérer toutes les fonctionnalités
2. **Soit** vous laissez tel quel — la v45 fonctionnera en mode dégradé (les champs étendus ne seront pas partagés mais les autres oui)

### Diagnostic rapide

Ouvrez la console du navigateur (F12 → Console) après avoir saisi une séance. Vous verrez :
- ✅ `[Supabase v45] ✓ Séance sauvegardée : 6e A SVT 2026-05-09` → Tout fonctionne
- ⚠️ `[Supabase v45] ⚠️ Séance sauvegardée en mode minimal` → Exécutez l'Étape 1 du SETUP-SUPABASE.md
- ❌ `[Supabase v45] ✗✗ Échec total` → Vérifiez les politiques RLS (Étape 4 du SETUP)

---

## 🚀 Déploiement Netlify

**Méthode 1 (Drag & Drop)** :
1. Allez sur [app.netlify.com](https://app.netlify.com)
2. Glissez-déposez le dossier `ibsa-v45-netlify/` complet sur votre site existant
3. Netlify met à jour automatiquement

**Méthode 2 (CLI)** :
```bash
cd ibsa-v45-netlify
netlify deploy --prod
```

---

## 🛠️ Modes de fonctionnement

| Indicateur dashboard | Signification |
|----------------------|---------------|
| ☁️ **Supabase Sync LIVE** | Tout fonctionne, données partagées entre tous les appareils |
| 💾 **Mode Local seulement** | Supabase indisponible. Les données restent sur l'appareil. |

---

## 🆘 Support

- Bug ou question : prenez une capture d'écran de la console (F12 → Console) et de l'erreur exacte
- Documentation Supabase complète : `SETUP-SUPABASE.md`

---

**Conçu pour IBSA Ouagadougou • Burkina Faso**
**Mai 2026 • v45**
