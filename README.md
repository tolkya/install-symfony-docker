# 📚 Documentation complète : Installation et sécurisation Docker Symfony

## � **Prérequis**

Avant de commencer, assurez-vous d'avoir installé :
- **Docker Desktop** (version 20.10+)
- **Docker Compose** (version 2.10+)
- **Git**
- **Un éditeur de code** (VS Code recommandé avec l'extension Docker)

### **Vérification des prérequis**
```bash
# Vérifier Docker
docker --version
docker compose version

# Vérifier Git
git --version
```

## �🚀 **1. Installation Docker Symfony (Dunglas)**

### **1.1 - Clonage du repository**
```bash
# Naviguer dans votre dossier de travail souhaité
cd /chemin/vers/votre/dossier

# Cloner le template Symfony Docker de Dunglas directement dans le dossier courant
git clone https://github.com/dunglas/symfony-docker.git .
```

### **1.2 - Construction et démarrage initial**
```bash
# Construire les images Docker (premier build)
docker compose build --pull --no-cache

# Alternative pour builds suivants (plus rapide)
# docker compose build

# Démarrer les conteneurs
docker compose up --wait

# Vérifier l'état des conteneurs
docker compose ps
```

**📝 Note sur les options de build :**
- `--pull` : Force le téléchargement des dernières images (recommandé pour la sécurité)
- `--no-cache` : Reconstruit tout depuis zéro (indispensable au premier build)
- Ces options peuvent être omises pour les builds suivants (plus rapide)

### **1.3 - Accès à l'application**
- URL : `https://localhost`
- ⚠️ Accepter le certificat SSL auto-signé dans le navigateur

### **1.4 - Installation de la base de données**

#### **Méthodes d'accès aux conteneurs**

#### **Option A : Terminal (ligne de commande)**
```bash
# Entrer dans le conteneur PHP
docker compose exec php bash

# Installer le pack ORM (PostgreSQL + Doctrine)
composer require symfony/orm-pack

# Sortir du conteneur
exit
```

#### **Option B : Extension VS Code Docker**
1. Installer l'extension "Docker" par Microsoft
2. Dans la sidebar Docker (icône baleine) :
   - Clic droit sur `symfony-docker-php-1`
   - Sélectionner "Attach Shell"
3. Un terminal s'ouvre directement dans VS Code
4. Exécuter : `composer require symfony/orm-pack`

#### **Option C : Palette de commandes VS Code**
- `Ctrl/Cmd + Shift + P`
- Taper "Docker: Attach Shell"
- Sélectionner `symfony-docker-php-1` dans la liste

#### **Finalisation**
```bash
# Reconstruire après modification des fichiers Docker
docker compose down
docker compose build --pull --no-cache
docker compose up --wait
```

---

## 🔍 **2. Vérification de la connexion PHP ↔ Database**

### **2.1 - Vérifier l'état des conteneurs**
```bash
# Voir les conteneurs actifs
docker compose ps

# Voir les logs en cas de problème
docker compose logs database
docker compose logs php
```

### **2.2 - Test de connexion via Doctrine**
```bash
# Entrer dans le conteneur PHP
docker compose exec php bash

# Tester la connexion avec une requête SQL
php bin/console doctrine:query:sql "SELECT version();"

# Résultat attendu : Version de PostgreSQL affichée
```

### **2.3 - Création de la base de données**
```bash
# Dans le conteneur PHP
php bin/console doctrine:database:create

# Si erreur "database already exists" = Connexion OK !
```

### **2.4 - Test direct PostgreSQL**
```bash
# Se connecter directement à PostgreSQL
docker compose exec database psql -U app -d app

# Vérifier les utilisateurs
\du

# Sortir
\q
```

---

## 🔐 **3. Création d'un utilisateur avec privilèges limités**

### **3.1 - Création du script d'initialisation**
```bash
# Créer le dossier pour les scripts
mkdir -p docker/postgres/init

# Créer le fichier de script
cat > docker/postgres/init/01-create-symfony-user.sql << 'EOF'
-- Script de sécurisation PostgreSQL pour Symfony
-- Objectif: Créer un utilisateur avec permissions limitées pour l'application

-- 1. Créer un utilisateur Symfony avec permissions limitées
CREATE USER symfony_user WITH PASSWORD 'symfony_secure_pwd_123';

-- 2. Accorder seulement les permissions nécessaires sur la base de données
GRANT CONNECT ON DATABASE app TO symfony_user;

-- 3. Accorder les permissions sur le schéma public
GRANT USAGE ON SCHEMA public TO symfony_user;

-- 4. Accorder les permissions CRUD sur toutes les tables existantes
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO symfony_user;

-- 5. Accorder les permissions sur les futures tables (pour les migrations)
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO symfony_user;

-- 6. Accorder les permissions sur les séquences (pour les auto-increment)
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO symfony_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT USAGE, SELECT ON SEQUENCES TO symfony_user;

-- 7. IMPORTANT: symfony_user ne peut PAS:
-- - DROP DATABASE
-- - CREATE DATABASE  
-- - DROP TABLE
-- - ALTER TABLE (structure)
-- - Créer d'autres utilisateurs

-- L'utilisateur 'app' reste propriétaire pour les migrations Doctrine
-- L'utilisateur 'symfony_user' sert uniquement pour l'application

\echo 'Utilisateur symfony_user créé avec permissions limitées'
\echo 'SÉCURITÉ: Cet utilisateur ne peut PAS détruire la base de données'
EOF
```

### **3.2 - Modification du compose.yaml**
Ajouter cette ligne dans la section `volumes` du service `database` :
```yaml
volumes:
  - database_data:/var/lib/postgresql/data:rw
  - ./docker/postgres/init:/docker-entrypoint-initdb.d:ro  # <- Ajouter cette ligne
```

### **3.3 - Modification de la configuration d'utilisateur**
Dans le `compose.yaml`, modifier la ligne `DATABASE_URL` :
```yaml
# Remplacer
DATABASE_URL: postgresql://${POSTGRES_USER:-app}:${POSTGRES_PASSWORD:-!ChangeMe!}@database:5432/${POSTGRES_DB:-app}

# Par
DATABASE_URL: postgresql://${POSTGRES_USER:-symfony_user}:${POSTGRES_PASSWORD:-symfony_secure_pwd_123}@database:5432/${POSTGRES_DB:-app}
```

### **3.4 - Configuration des fichiers d'environnement**
```bash
# Créer .env.local (non versionné) avec les vrais mots de passe
cat > .env.local << 'EOF'
# Fichier .env.local - NE PAS VERSIONNER
# Ce fichier contient les secrets locaux

# Mot de passe de l'utilisateur PostgreSQL sécurisé
SYMFONY_DB_PASSWORD=symfony_secure_pwd_123

# DATABASE_URL pour l'utilisateur sécurisé
DATABASE_URL="postgresql://symfony_user:symfony_secure_pwd_123@database:5432/app?serverVersion=16&charset=utf8"
EOF

# Créer .env.dev.local (non versionné) avec les secrets Symfony
cat > .env.dev.local << 'EOF'
###> symfony/framework-bundle ###
APP_SECRET=$(openssl rand -hex 32)
###< symfony/framework-bundle ###
EOF
```

Modifier le `.env` principal pour avoir des valeurs d'exemple :
```env
DATABASE_URL="postgresql://symfony_user:CHANGE_ME_PASSWORD@database:5432/app?serverVersion=16&charset=utf8"
```

Modifier le `.env.dev` pour avoir une structure d'exemple :
```env
###> symfony/framework-bundle ###
APP_SECRET=
###< symfony/framework-bundle ###
```

### **3.5 - Application des modifications**
```bash
# Arrêter et supprimer les volumes (pour réinitialiser PostgreSQL)
docker compose down -v

# Relancer avec la nouvelle configuration
docker compose up --wait
```

---

## 🧪 **4. Tests de restriction des privilèges**

### **4.1 - Test de connexion avec l'utilisateur limité**
```bash
# Se connecter avec symfony_user
docker compose exec database psql -U symfony_user -d app

# Vérifier l'utilisateur actuel
SELECT current_user;

# Vérifier les rôles et privilèges
\du

# Résultat attendu :
# - app : Superuser, Create role, Create DB, etc.
# - symfony_user : (aucun attribut spécial)
```

### **4.2 - Test des restrictions via psql**
```sql
-- Dans psql connecté avec symfony_user

-- Test 1: Tentative de destruction de la base (DOIT ÉCHOUER)
DROP DATABASE app;
-- Résultat attendu: ERROR: must be owner of database app

-- Test 2: Tentative de création de base (DOIT ÉCHOUER)
CREATE DATABASE test_malware;
-- Résultat attendu: ERROR: permission denied to create database

-- Test 3: Tentative de suppression d'utilisateur (DOIT ÉCHOUER)
DROP USER app;
-- Résultat attendu: ERROR: must be superuser to drop roles

-- Test 4: Tentative de création d'utilisateur (DOIT ÉCHOUER)
CREATE USER hacker WITH PASSWORD 'hack';
-- Résultat attendu: ERROR: permission denied to create role

-- Sortir de psql
\q
```

### **4.3 - Test des restrictions via Symfony**
```bash
# Entrer dans le conteneur PHP
docker compose exec php bash

# Test 1: Tentative de destruction via Doctrine (DOIT ÉCHOUER)
php bin/console doctrine:database:drop --force
# Résultat attendu: SQLSTATE[42501]: Insufficient privilege: 7 ERROR: must be owner of database app

# Test 2: Vérifier que les opérations normales fonctionnent
php bin/console doctrine:query:sql "SELECT version();"
# Résultat attendu: Version PostgreSQL affichée (SUCCÈS)

# Test 3: Tentative de création d'une nouvelle base (DOIT ÉCHOUER)
php bin/console doctrine:query:sql "CREATE DATABASE test_hack;"
# Résultat attendu: ERROR: permission denied to create database
```

### **4.4 - Test des opérations autorisées**
```bash
# Dans le conteneur PHP, tester les opérations CRUD normales

# Créer une table test (cela devrait fonctionner)
php bin/console doctrine:query:sql "
CREATE TABLE test_table (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);"

# Insérer des données (autorisé)
php bin/console doctrine:query:sql "INSERT INTO test_table (name) VALUES ('Test Data');"

# Lire les données (autorisé)
php bin/console doctrine:query:sql "SELECT * FROM test_table;"

# Modifier les données (autorisé)
php bin/console doctrine:query:sql "UPDATE test_table SET name = 'Modified' WHERE id = 1;"

# Supprimer des données (autorisé)
php bin/console doctrine:query:sql "DELETE FROM test_table WHERE id = 1;"

# Nettoyer
php bin/console doctrine:query:sql "DROP TABLE test_table;"
```

---

## ✅ **5. Résumé de la sécurisation**

### **Utilisateurs créés :**
- **`app`** : Superuser (pour migrations et administration)
- **`symfony_user`** : Utilisateur limité (pour l'application)

### **Permissions de `symfony_user` :**
- ✅ **Autorisé :** SELECT, INSERT, UPDATE, DELETE
- ✅ **Autorisé :** Connexion à la base `app`
- ❌ **Interdit :** DROP DATABASE
- ❌ **Interdit :** CREATE DATABASE
- ❌ **Interdit :** DROP TABLE
- ❌ **Interdit :** Création/suppression d'utilisateurs

### **Sécurité GitHub :**
- ✅ `.env.local` ignoré par `.gitignore`
- ✅ `.env` contient des valeurs d'exemple
- ✅ Mots de passe réels non versionnés

### **Tests de validation :**
- ✅ Connexion PHP ↔ PostgreSQL fonctionnelle
- ✅ CRUD normal possible
- ❌ DROP DATABASE bloqué (sécurité OK)
- ❌ Actions dangereuses bloquées

### **Structure finale du projet :**
```
📁 Votre projet
├── .env                           ← Exemples (versionné)
├── .env.local                     ← Secrets BDD (NON versionné)
├── .env.dev                       ← Structure Symfony (versionné)
├── .env.dev.local                 ← Secrets Symfony (NON versionné)
├── .gitignore                     ← Protège les secrets
├── compose.yaml                   ← Configuration Docker (modifié)
├── docker/postgres/init/          ← Scripts d'initialisation
│   └── 01-create-symfony-user.sql
└── INSTALL_SECURE_GUIDE.md        ← Cette documentation
```

---

## 🔧 **Commandes utiles**

### **Gestion des conteneurs**
```bash
# Voir l'état des conteneurs
docker compose ps

# Voir les logs
docker compose logs database
docker compose logs php

# Redémarrer complètement
docker compose down -v
docker compose up --wait
```

### **Méthodes d'accès aux conteneurs**

#### **Option A : Terminal (ligne de commande)**
```bash
# Accès au conteneur PHP
docker compose exec php bash

# Accès direct à PostgreSQL
docker compose exec database psql -U symfony_user -d app
```

#### **Option B : Extension VS Code Docker**
1. Installer l'extension "Docker" par Microsoft
2. Dans la sidebar Docker (icône baleine) :
   - Clic droit sur le conteneur souhaité
   - Sélectionner "Attach Shell"
3. Un terminal s'ouvre directement dans VS Code

#### **Option C : Palette de commandes VS Code**
- `Ctrl/Cmd + Shift + P`
- Taper "Docker: Attach Shell"
- Sélectionner le conteneur dans la liste

### **Connexions à la base**
```bash
# Connexion avec l'utilisateur application (limité)
docker compose exec database psql -U symfony_user -d app

# Connexion avec l'utilisateur admin (superuser)
docker compose exec database psql -U app -d app
```

**🎯 Votre installation Docker Symfony est maintenant complète et sécurisée !**

---

## 🚨 **Dépannage**

### **Problème : "Database connection failed"**
```bash
# Vérifier que les conteneurs sont démarrés
docker compose ps

# Vérifier les logs
docker compose logs database

# Vérifier votre .env.local
cat .env.local
```

### **Problème : "Permission denied for user symfony_user"**
```bash
# Recréer complètement la base avec le script d'initialisation
docker compose down -v
docker compose up --wait
```

### **Problème : "Port already in use"**
```bash
# Arrêter les autres services sur les ports 80/443
sudo lsof -i :80
sudo lsof -i :443

# Ou modifier les ports dans compose.yaml
```

### **Problème : Les modifications du script SQL ne s'appliquent pas**
```bash
# Supprimer les volumes pour forcer la réinitialisation
docker compose down -v
docker volume prune
docker compose up --wait
```
