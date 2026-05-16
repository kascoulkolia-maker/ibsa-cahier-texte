# 📘 Guide de configuration Supabase pour IBSA v44

Ce guide vous accompagne **étape par étape** pour activer toutes les nouvelles fonctionnalités cloud de la v44 :

- ☁️ Stockage et partage des pièces jointes des séances
- 📊 Synchronisation des progressions téléversées entre enseignants et direction
- 📅 Synchronisation des emplois du temps
- 👁️ Consultation par les directeurs de tous les appels avec totaux agrégés

---

## ⏱️ Temps total estimé : 15 minutes

---

## ✅ Étape 1 — Compléter les colonnes manquantes dans `seances` et `appels`

Ouvrez votre projet Supabase → **SQL Editor** → **New query**, et collez :

```sql
-- 1. Compléter la table seances (champs ajoutés en v44)
ALTER TABLE public.seances
  ADD COLUMN IF NOT EXISTS type TEXT DEFAULT 'Cours',
  ADD COLUMN IF NOT EXISTS objectifs TEXT,
  ADD COLUMN IF NOT EXISTS observations TEXT,
  ADD COLUMN IF NOT EXISTS signales TEXT,
  ADD COLUMN IF NOT EXISTS files JSONB DEFAULT '[]'::jsonb;

-- 2. Compléter la table appels avec les agrégats
ALTER TABLE public.appels
  ADD COLUMN IF NOT EXISTS presents INTEGER DEFAULT 0,
  ADD COLUMN IF NOT EXISTS absents INTEGER DEFAULT 0,
  ADD COLUMN IF NOT EXISTS retards INTEGER DEFAULT 0,
  ADD COLUMN IF NOT EXISTS total INTEGER DEFAULT 0,
  ADD COLUMN IF NOT EXISTS taux_presence INTEGER DEFAULT 0;
```

Cliquez sur **Run**.

---

## ✅ Étape 2 — Créer les tables `prog_files` et `edt_files`

Toujours dans le SQL Editor :

```sql
-- Table des fichiers de progression
CREATE TABLE IF NOT EXISTS public.prog_files (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  teacher_id UUID REFERENCES public.users(id) ON DELETE CASCADE,
  classe TEXT NOT NULL,
  matiere TEXT NOT NULL,
  name TEXT NOT NULL,
  size BIGINT,
  type TEXT,
  path TEXT NOT NULL,
  url TEXT,
  date DATE DEFAULT CURRENT_DATE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_prog_files_teacher ON public.prog_files(teacher_id);
CREATE INDEX IF NOT EXISTS idx_prog_files_classe  ON public.prog_files(classe);

-- Table des emplois du temps
CREATE TABLE IF NOT EXISTS public.edt_files (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  classe TEXT NOT NULL,
  name TEXT NOT NULL,
  size BIGINT,
  type TEXT,
  path TEXT NOT NULL,
  url TEXT,
  date DATE DEFAULT CURRENT_DATE,
  uploaded_by TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_edt_files_classe ON public.edt_files(classe);
```

Cliquez sur **Run**.

---

## ✅ Étape 3 — Activer Row Level Security (RLS) pour ces tables

```sql
-- prog_files : lecture pour tous les utilisateurs authentifiés, écriture/suppression pour l'auteur
ALTER TABLE public.prog_files ENABLE ROW LEVEL SECURITY;

CREATE POLICY "prog_files_select_all" ON public.prog_files
  FOR SELECT USING (auth.role() = 'authenticated');

CREATE POLICY "prog_files_insert_own" ON public.prog_files
  FOR INSERT WITH CHECK (auth.uid() = teacher_id);

CREATE POLICY "prog_files_delete_own_or_admin" ON public.prog_files
  FOR DELETE USING (
    auth.uid() = teacher_id
    OR EXISTS (
      SELECT 1 FROM public.users u
      WHERE u.id = auth.uid()
      AND u.role IN ('admin', 'concepteur', 'directeur_ecole', 'directeur_pedagogique', 'direction_generale')
    )
  );

-- edt_files : lecture pour tous, écriture/suppression admin/direction
ALTER TABLE public.edt_files ENABLE ROW LEVEL SECURITY;

CREATE POLICY "edt_files_select_all" ON public.edt_files
  FOR SELECT USING (auth.role() = 'authenticated');

CREATE POLICY "edt_files_insert_admin" ON public.edt_files
  FOR INSERT WITH CHECK (
    EXISTS (
      SELECT 1 FROM public.users u
      WHERE u.id = auth.uid()
      AND u.role IN ('admin', 'concepteur', 'directeur_ecole', 'directeur_pedagogique', 'direction_generale')
    )
  );

CREATE POLICY "edt_files_delete_admin" ON public.edt_files
  FOR DELETE USING (
    EXISTS (
      SELECT 1 FROM public.users u
      WHERE u.id = auth.uid()
      AND u.role IN ('admin', 'concepteur', 'directeur_ecole', 'directeur_pedagogique', 'direction_generale')
    )
  );
```

---

## ✅ Étape 4 — Vérifier les politiques RLS sur `seances` et `appels`

Pour que les directeurs voient TOUTES les séances et tous les appels :

```sql
-- Si pas déjà créées, voici les politiques recommandées :

-- SEANCES : lecture par direction + admin
DROP POLICY IF EXISTS "seances_select_director_or_owner" ON public.seances;
CREATE POLICY "seances_select_director_or_owner" ON public.seances
  FOR SELECT USING (
    auth.uid() = teacher_id
    OR EXISTS (
      SELECT 1 FROM public.users u
      WHERE u.id = auth.uid()
      AND u.role IN ('admin', 'concepteur', 'directeur_ecole', 'directeur_pedagogique', 'direction_generale')
    )
  );

-- APPELS : idem
DROP POLICY IF EXISTS "appels_select_director_or_owner" ON public.appels;
CREATE POLICY "appels_select_director_or_owner" ON public.appels
  FOR SELECT USING (
    auth.uid() = teacher_id
    OR EXISTS (
      SELECT 1 FROM public.users u
      WHERE u.id = auth.uid()
      AND u.role IN ('admin', 'concepteur', 'directeur_ecole', 'directeur_pedagogique', 'direction_generale')
    )
  );

-- USERS : lecture pour tous (nécessaire pour résoudre teacher_id → nom enseignant)
DROP POLICY IF EXISTS "users_select_all" ON public.users;
CREATE POLICY "users_select_all" ON public.users
  FOR SELECT USING (auth.role() = 'authenticated');
```

---

## ✅ Étape 5 — Créer les buckets Storage

Dans le menu Supabase, allez dans **Storage** → **Create bucket** et créez ces 3 buckets :

| Nom du bucket          | Public ?   | Description                          |
|------------------------|------------|--------------------------------------|
| `seances-files`        | ✅ Public  | Pièces jointes des séances           |
| `progressions-files`   | ✅ Public  | Fichiers de progression téléversés   |
| `edt-files`            | ✅ Public  | Emplois du temps                     |

> ℹ️ **Pourquoi public ?** Pour permettre aux parents (qui n'ont pas de compte) de télécharger les documents via un lien direct envoyé par l'enseignant.
>
> ⚠️ Les chemins sont aléatoires (timestamp + UUID), donc les fichiers ne sont pas listables ; il faut le lien exact.

### Politiques de Storage (à coller dans le SQL Editor)

```sql
-- Permettre aux utilisateurs authentifiés d'uploader
CREATE POLICY "Authenticated users can upload to seances-files"
  ON storage.objects FOR INSERT
  WITH CHECK (bucket_id = 'seances-files' AND auth.role() = 'authenticated');

CREATE POLICY "Authenticated users can upload to progressions-files"
  ON storage.objects FOR INSERT
  WITH CHECK (bucket_id = 'progressions-files' AND auth.role() = 'authenticated');

CREATE POLICY "Authenticated users can upload to edt-files"
  ON storage.objects FOR INSERT
  WITH CHECK (bucket_id = 'edt-files' AND auth.role() = 'authenticated');

-- Permettre la lecture publique (pour le partage parents)
CREATE POLICY "Public read on seances-files"
  ON storage.objects FOR SELECT
  USING (bucket_id = 'seances-files');

CREATE POLICY "Public read on progressions-files"
  ON storage.objects FOR SELECT
  USING (bucket_id = 'progressions-files');

CREATE POLICY "Public read on edt-files"
  ON storage.objects FOR SELECT
  USING (bucket_id = 'edt-files');

-- Permettre la suppression aux uploaders et admins
CREATE POLICY "Owners and admins can delete seances files"
  ON storage.objects FOR DELETE
  USING (
    bucket_id = 'seances-files'
    AND (
      auth.uid() = owner
      OR EXISTS (
        SELECT 1 FROM public.users u
        WHERE u.id = auth.uid()
        AND u.role IN ('admin', 'concepteur', 'directeur_ecole', 'directeur_pedagogique', 'direction_generale')
      )
    )
  );

CREATE POLICY "Owners and admins can delete progressions files"
  ON storage.objects FOR DELETE
  USING (
    bucket_id = 'progressions-files'
    AND (
      auth.uid() = owner
      OR EXISTS (
        SELECT 1 FROM public.users u
        WHERE u.id = auth.uid()
        AND u.role IN ('admin', 'concepteur', 'directeur_ecole', 'directeur_pedagogique', 'direction_generale')
      )
    )
  );

CREATE POLICY "Admins can delete edt files"
  ON storage.objects FOR DELETE
  USING (
    bucket_id = 'edt-files'
    AND EXISTS (
      SELECT 1 FROM public.users u
      WHERE u.id = auth.uid()
      AND u.role IN ('admin', 'concepteur', 'directeur_ecole', 'directeur_pedagogique', 'direction_generale')
    )
  );
```

---

## ✅ Étape 6 — Déploiement sur Netlify

1. Connectez-vous à [netlify.com](https://netlify.com)
2. **Sites** → **Add new site** → **Deploy manually**
3. Glissez-déposez le dossier `ibsa-v44-netlify/` complet
4. Netlify détecte automatiquement `netlify.toml` et `_redirects`
5. Vérifiez que `config.js` contient bien votre URL Supabase et la clé anon

> 💡 **Astuce** : pour mettre à jour la configuration sans redéployer, modifiez `config.js` directement dans Netlify via **Deploys → [dernier deploy] → Files**.

---

## 🎯 Vérifications après déploiement

Connectez-vous avec un compte enseignant et testez :

- [ ] Créer une séance avec une pièce jointe → vérifier qu'elle apparaît avec un badge **☁️ Cloud**
- [ ] Se connecter avec un compte direction → vérifier que la séance apparaît dans **Suivi pédagogique**
- [ ] Cliquer sur **Télécharger** depuis le compte direction → le fichier doit se télécharger
- [ ] Cliquer sur **Partager** → la fenêtre s'ouvre avec un lien public, un bouton WhatsApp et un bouton Email
- [ ] Téléverser une progression → vérifier qu'elle apparaît chez la direction
- [ ] Téléverser un emploi du temps (admin) → vérifier qu'il apparaît chez les enseignants

---

## 🆘 Dépannage

### Les séances ne se synchronisent pas
- Ouvrez la console (F12) et cherchez les erreurs `[Supabase]`
- Vérifiez que le compte connecté a bien un UUID dans la table `users`
- Vérifiez les politiques RLS

### Les fichiers ne s'uploadent pas
- Vérifiez que les 3 buckets Storage existent et sont bien marqués **Public**
- Vérifiez les politiques d'upload (Étape 5)

### Les directeurs ne voient pas les séances des enseignants
- Vérifiez la politique `seances_select_director_or_owner` (Étape 4)
- Vérifiez que le rôle Supabase du directeur est bien `admin`, `directeur_ecole`, `directeur_pedagogique`, ou `direction_generale`

### "Erreur 403" lors d'un téléchargement
- Le bucket n'est pas marqué Public, OU
- La politique de SELECT sur storage.objects manque

---

## 📞 Support

Pour toute question ou anomalie, prenez une capture d'écran de la console (F12 → onglet Console) et de l'erreur précise.

---

**Version 44 — Mai 2026**
**Compatible avec Supabase v2 (2025+) et navigateurs modernes (Chrome, Firefox, Safari, Edge).**
