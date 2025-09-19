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

## � **6. Installation des bundles Symfony essentiels**

### **6.1 - Installation du Maker Bundle (outils de développement)**

#### **Méthodes d'accès aux conteneurs (rappel)**

#### **Option A : Terminal (ligne de commande)**
```bash
# Entrer dans le conteneur PHP
docker compose exec php bash

# Installer le maker bundle (outils de développement)
composer require --dev symfony/maker-bundle

# Vérifier l'installation
php bin/console list make
```

#### **Option B : Extension VS Code Docker**
1. Clic droit sur `symfony-docker-php-1` → "Attach Shell"
2. Exécuter : `composer require --dev symfony/maker-bundle`

**🛠️ Ce que le Maker Bundle apporte :**
- `make:controller` - Créer des contrôleurs
- `make:entity` - Créer des entités Doctrine
- `make:form` - Créer des formulaires
- `make:crud` - Générer un CRUD complet
- `make:migration` - Créer des migrations de base de données
- `make:user` - Créer un système d'utilisateurs
- `make:auth` - Créer un système d'authentification
- Et bien d'autres commandes de génération !

### **6.2 - Installation du WebApp Pack (stack complète)**
```bash
# Toujours dans le conteneur PHP
composer require symfony/webapp-pack

# Sortir du conteneur
exit
```

**🚀 Ce que le WebApp Pack apporte :**
- **Twig** - Moteur de templates pour les vues
- **Doctrine Migrations** - Gestion des migrations de BDD
- **Security Bundle** - Authentification et autorisation
- **Form Component** - Création de formulaires
- **Validator** - Validation des données
- **Asset Component** - Gestion des assets (CSS, JS)
- **Mailer** - Envoi d'emails
- **Messenger** - Système de queues et messages asynchrones
- **WebProfiler** - Outils de debug et profiling
- **Et beaucoup d'autres composants essentiels !**

### **6.3 - Reconstruction après installation**
```bash
# Ces installations peuvent modifier les fichiers Docker et ajouter de nouveaux services
docker compose down
docker compose build --pull --no-cache
docker compose up --wait
```

### **6.4 - Test des nouvelles fonctionnalités**
```bash
# Entrer dans le conteneur PHP
docker compose exec php bash

# Voir toutes les commandes Symfony disponibles
php bin/console list

# Voir spécifiquement les commandes make
php bin/console list make

# Exemple : Créer votre première entité
php bin/console make:entity Product
# Suivre les instructions interactives

# Créer une migration après avoir créé des entités
php bin/console make:migration

# Appliquer les migrations à la base de données
php bin/console doctrine:migrations:migrate --no-interaction

# Générer un contrôleur
php bin/console make:controller ProductController

# Générer un CRUD complet pour une entité
php bin/console make:crud Product

exit
```

### **6.5 - Structure enrichie du projet**
Après installation, votre projet contient maintenant :
```
📁 Votre projet Symfony
├── .env                           ← Config avec nouvelles sections (messenger, mailer, etc.)
├── .env.local                     ← Vos secrets
├── .gitignore                     ← Mis à jour automatiquement
├── assets/                        ← Fichiers CSS/JS (asset-mapper)
├── config/
│   └── packages/                  ← Configurations des nouveaux bundles
├── migrations/                    ← Migrations Doctrine
├── src/
│   ├── Controller/                ← Vos contrôleurs
│   ├── Entity/                    ← Vos entités Doctrine
│   ├── Form/                      ← Vos formulaires
│   └── Repository/                ← Vos repositories
├── templates/                     ← Templates Twig
├── tests/                         ← Tests unitaires et fonctionnels
└── var/
    └── log/                       ← Logs de l'application
```

### **6.6 - Commandes utiles pour le développement**
```bash
# Créer une entité avec relations
php bin/console make:entity User
php bin/console make:entity Article

# Générer les getters/setters manquants
php bin/console make:entity --regenerate

# Créer un formulaire pour une entité
php bin/console make:form ArticleType Article

# Créer un contrôleur avec template
php bin/console make:controller HomeController

# Générer un système d'authentification complet
php bin/console make:user
php bin/console make:auth

# Voir l'état des migrations
php bin/console doctrine:migrations:status

# Créer des fixtures (données de test)
composer require --dev doctrine/doctrine-fixtures-bundle
php bin/console make:fixtures ArticleFixtures
```

---

## 🎨 **7. Création et personnalisation des contrôleurs**

### **7.1 - Créer un contrôleur d'accueil**
```bash
# Dans le conteneur PHP
php bin/console make:controller HomeController
```

### **7.2 - Configurer la page d'accueil**

#### **Modifier la route du contrôleur**
Éditer le fichier `src/Controller/HomeController.php` :
```php
<?php

namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Annotation\Route;

class HomeController extends AbstractController
{
    #[Route('/', name: 'app_home')]  // <- Changer de '/home' vers '/'
    public function index(): Response
    {
        return $this->render('home/index.html.twig', [
            'controller_name' => 'HomeController',
        ]);
    }
}
```

### **7.3 - Passer des variables à la vue**

#### **Dans le contrôleur - Préparer les données**
```php
#[Route('/', name: 'app_home')]
public function index(): Response
{
    // Créer vos variables
    $message = "Bienvenue sur votre application Symfony !";
    $nom = "Développeur";
    $date = new \DateTime();
    $technologies = ['Docker', 'Symfony', 'PostgreSQL', 'Twig'];
    
    return $this->render('home/index.html.twig', [
        'controller_name' => 'HomeController',
        'message' => $message,
        'nom' => $nom,
        'date' => $date,
        'technologies' => $technologies,
    ]);
}
```

#### **Dans la vue Twig - Afficher les données**
Éditer le fichier `templates/home/index.html.twig` :
```twig
{% extends 'base.html.twig' %}

{% block title %}Accueil - {{ parent() }}{% endblock %}

{% block body %}
<div class="example-wrapper">
    <h1>{{ message }}</h1>
    
    <div class="welcome-info">
        <h2>Bonjour {{ nom }} !</h2>
        <p>Date : {{ date|date('d/m/Y H:i') }}</p>
        
        <h3>Technologies utilisées :</h3>
        <ul>
        {% for techno in technologies %}
            <li>{{ techno }}</li>
        {% endfor %}
        </ul>
    </div>
    
    <div class="status">
        <h3>État de l'installation :</h3>
        <ul>
            <li>✅ Docker : Opérationnel</li>
            <li>✅ Symfony : Installé</li>
            <li>✅ PostgreSQL : Connecté</li>
            <li>✅ Sécurité : Configurée</li>
        </ul>
    </div>
</div>

<style>
    .example-wrapper {
        margin: 1em auto;
        max-width: 800px;
        width: 95%;
        font: 18px/1.5 sans-serif;
    }
    
    .welcome-info, .status {
        background: #f9f9f9;
        padding: 20px;
        margin: 20px 0;
        border-radius: 8px;
        border-left: 4px solid #007bff;
    }
    
    h1 { color: #333; }
    h2, h3 { color: #007bff; }
    ul { margin: 10px 0; }
    li { margin: 5px 0; }
</style>
{% endblock %}
```

### **7.4 - Syntaxes Twig essentielles**

#### **Afficher des variables**
```twig
{{ ma_variable }}                    <!-- Afficher une variable -->
{{ ma_variable|upper }}              <!-- En majuscules -->
{{ ma_date|date('d/m/Y') }}         <!-- Formater une date -->
{{ ma_chaine|length }}              <!-- Longueur d'une chaîne -->
```

#### **Conditions**
```twig
{% if age >= 18 %}
    <p>Vous êtes majeur</p>
{% elseif age >= 16 %}
    <p>Vous êtes presque majeur</p>
{% else %}
    <p>Vous êtes mineur</p>
{% endif %}
```

#### **Boucles**
```twig
{% for item in items %}
    <p>{{ loop.index }}: {{ item }}</p>
{% else %}
    <p>Aucun élément à afficher</p>
{% endfor %}
```

### **7.5 - Créer d'autres contrôleurs**

#### **Contrôleur "À propos"**
```bash
php bin/console make:controller AboutController
```

#### **Exemple de contrôleur avec paramètres**
```php
// src/Controller/AboutController.php
#[Route('/about', name: 'app_about')]
public function about(): Response
{
    $infos = [
        'version' => '1.0.0',
        'auteur' => 'Votre nom',
        'description' => 'Application Symfony avec Docker',
    ];
    
    return $this->render('about/index.html.twig', [
        'infos' => $infos,
    ]);
}

#[Route('/user/{id}', name: 'app_user_profile')]
public function userProfile(int $id): Response
{
    return $this->render('user/profile.html.twig', [
        'user_id' => $id,
    ]);
}
```

### **7.6 - Navigation entre les pages**

#### **Créer un menu dans base.html.twig**
```twig
<!-- templates/base.html.twig -->
<!DOCTYPE html>
<html>
    <head>
        <meta charset="UTF-8">
        <title>{% block title %}Mon App Symfony{% endblock %}</title>
        <link rel="icon" href="data:image/svg+xml,<svg xmlns=%22http://www.w3.org/2000/svg%22 viewBox=%220 0 128 128%22><text y=%221.2em%22 font-size=%2296%22>⚫️</text></svg>">
    </head>
    <body>
        <nav style="background: #333; padding: 1rem;">
            <a href="{{ path('app_home') }}" style="color: white; margin-right: 1rem;">Accueil</a>
            <a href="{{ path('app_about') }}" style="color: white; margin-right: 1rem;">À propos</a>
        </nav>
        
        {% block body %}{% endblock %}
    </body>
</html>
```

### **7.7 - Commandes utiles pour les contrôleurs**
```bash
# Lister toutes les routes
php bin/console debug:router

# Voir les détails d'une route spécifique
php bin/console debug:router app_home

# Supprimer un contrôleur (manuel)
rm src/Controller/NomController.php
rm templates/nom/index.html.twig
rmdir templates/nom  # si le dossier est vide

# Vider le cache
php bin/console cache:clear
```

### **7.8 - Structure après création des contrôleurs**
```
📁 Votre projet
├── src/
│   └── Controller/
│       ├── HomeController.php         ← Page d'accueil (/)
│       ├── AboutController.php        ← Page à propos (/about)
│       └── ProductController.php      ← Autres contrôleurs
├── templates/
│   ├── base.html.twig                ← Template principal avec navigation
│   ├── home/
│   │   └── index.html.twig           ← Vue de la page d'accueil
│   ├── about/
│   │   └── index.html.twig           ← Vue de la page à propos
│   └── product/
│       └── index.html.twig           ← Autres vues
```

### **7.9 - Test de votre application**
```bash
# Vérifier que tout fonctionne
# Aller sur https://localhost
# - Page d'accueil avec vos variables
# - Navigation vers les autres pages
# - Affichage correct des données

# Consulter les logs en cas de problème
docker compose logs php
```


---

## 🎨 **8. Gestion des feuilles de style CSS**

### **8.1 - Comprendre les deux approches**

Symfony offre deux méthodes principales pour gérer les fichiers CSS :

#### **Approche A : AssetMapper (Moderne - Recommandée)**
- Fichiers dans `assets/styles/`
- Optimisation automatique (minification, cache)
- Gestion moderne des assets

#### **Approche B : Dossier Public (Classique)**
- Fichiers dans `public/assets/css/`
- Accès direct via URL
- Contrôle manuel des assets

### **8.2 - Approche B : Utilisation du dossier public**

#### **Étape 1 : Créer la structure de dossiers**
```bash
# Créer le dossier pour les CSS dans public
mkdir -p public/assets/css
```

#### **Étape 2 : Créer votre fichier CSS**
```bash
# Créer le fichier CSS spécifique au contrôleur
touch public/assets/css/HomeControlleur.css
```

#### **Étape 3 : Ajouter vos styles**
Éditer `public/assets/css/HomeControlleur.css` :
```css
/* Styles spécifiques au HomeController */
body {
    background-color: #f8f9fa;
    color: rgb(255, 105, 105);
    font-family: Arial, sans-serif;
}

.example-wrapper {
    margin: 1em auto;
    max-width: 800px;
    width: 95%;
    font: 18px/1.5 sans-serif;
    background: white;
    padding: 20px;
    border-radius: 8px;
    box-shadow: 0 2px 10px rgba(0,0,0,0.1);
}

.example-wrapper code {
    background: #06f44dff;
    padding: 2px 6px;
    border-radius: 4px;
    color: white;
}

h1 {
    color: #333;
    text-align: center;
    border-bottom: 2px solid rgb(255, 105, 105);
    padding-bottom: 10px;
}

h2 {
    color: rgb(255, 105, 105);
    font-style: italic;
}

ul {
    list-style-type: none;
    padding-left: 0;
}

li {
    background: #e9ecef;
    margin: 10px 0;
    padding: 10px;
    border-left: 4px solid rgb(255, 105, 105);
}
```

#### **Étape 4 : Inclure le CSS dans le template de base**
Modifier `templates/base.html.twig` :
```twig
<!DOCTYPE html>
<html>
    <head>
        <meta charset="UTF-8">
        <title>{% block title %}Welcome!{% endblock %}</title>
        <link rel="icon" href="data:image/svg+xml,<svg xmlns=%22http://www.w3.org/2000/svg%22 viewBox=%220 0 128 128%22><text y=%221.2em%22 font-size=%2296%22>⚫️</text><text y=%221.3em%22 x=%220.2em%22 font-size=%2276%22 fill=%22%23fff%22>sf</text></svg>">
        
        <!-- CSS via dossier public -->
        <link rel="stylesheet" href="{{ asset('assets/css/HomeControlleur.css') }}">
        
        {% block stylesheets %}
            <!-- CSS spécifiques aux pages individuelles -->
        {% endblock %}

        {% block javascripts %}
            {% block importmap %}{{ importmap('app') }}{% endblock %}
        {% endblock %}
    </head>
    <body>
        {% block body %}{% endblock %}
    </body>
</html>
```

#### **Étape 5 : Nettoyer le template de la page**
Modifier `templates/home/index.html.twig` pour supprimer le CSS inline :
```twig
{% extends 'base.html.twig' %}

{% block title %}Hello HomeController!{% endblock %}

{% block body %}
<div class="example-wrapper">
    <h1>Hello {{ controller_name }}! ✅</h1>
    <h2>{{ ma_variable }}</h2>

    <p>This friendly message is coming from:</p>
    <ul>
        <li>Your controller at <code>/app/src/Controller/HomeController.php</code></li>
        <li>Your template at <code>/app/templates/home/index.html.twig</code></li>
        <li>Your CSS at <code>/app/public/assets/css/HomeControlleur.css</code></li>
    </ul>
</div>
{% endblock %}
```

### **8.3 - Comprendre la fonction asset()**

#### **Importance de la fonction asset() :**
```twig
<!-- ❌ Incorrect - Chemin en dur -->
<link rel="stylesheet" href="/assets/css/HomeControlleur.css">

<!-- ✅ Correct - Utilisation d'asset() -->
<link rel="stylesheet" href="{{ asset('assets/css/HomeControlleur.css') }}">
```

#### **Avantages d'asset() :**
- **URLs absolues** → Fonctionne même si l'app n'est pas à la racine
- **Cache busting** → Évite les problèmes de cache navigateur
- **Environnements** → S'adapte au développement/production
- **CDN Support** → Peut pointer vers un CDN en production

### **8.4 - Approche A : AssetMapper (Alternative moderne)**

#### **Structure AssetMapper :**
```bash
# Créer dans assets/styles/
touch assets/styles/HomeControlleur.css
```

#### **Inclusion avec AssetMapper :**
```twig
<!-- Dans base.html.twig -->
<link rel="stylesheet" href="{{ asset('styles/HomeControlleur.css') }}">
```

#### **Différences importantes :**
```bash
# Dossier public
{{ asset('assets/css/HomeControlleur.css') }}  # Chemin complet

# AssetMapper  
{{ asset('styles/HomeControlleur.css') }}      # assets/ automatiquement ajouté
```

### **8.5 - CSS spécifiques aux pages**

#### **CSS global (base.html.twig) :**
```twig
<!-- CSS pour toute l'application -->
<link rel="stylesheet" href="{{ asset('assets/css/global.css') }}">
```

#### **CSS spécifique à une page :**
```twig
<!-- Dans templates/home/index.html.twig -->
{% block stylesheets %}
    <link rel="stylesheet" href="{{ asset('assets/css/home-specific.css') }}">
{% endblock %}
```

#### **CSS inline pour des cas spéciaux :**
```twig
{% block stylesheets %}
    <style>
        .page-specific-class {
            background: linear-gradient(45deg, #667eea, #764ba2);
        }
    </style>
{% endblock %}
```

### **8.6 - Organisation recommandée des CSS**

#### **Structure pour projets moyens/grands :**
```
📁 public/assets/css/
├── global.css              ← Styles généraux (body, navigation)
├── components.css          ← Composants réutilisables (boutons, cartes)
├── HomeControlleur.css     ← Spécifique à la page d'accueil
├── ProductController.css   ← Spécifique aux produits
└── themes/
    ├── dark.css           ← Thème sombre
    └── light.css          ← Thème clair
```

#### **Inclusion multiple :**
```twig
<!-- base.html.twig -->
<link rel="stylesheet" href="{{ asset('assets/css/global.css') }}">
<link rel="stylesheet" href="{{ asset('assets/css/components.css') }}">

{% block stylesheets %}
    <!-- CSS spécifiques aux pages -->
{% endblock %}
```

### **8.7 - Bonnes pratiques CSS**

#### **Nommage des classes (BEM) :**
```css
/* Block Element Modifier */
.home-wrapper { }              /* Block */
.home-wrapper__title { }       /* Element */
.home-wrapper__title--large { } /* Modifier */
```

#### **Variables CSS :**
```css
:root {
    --primary-color: rgb(255, 105, 105);
    --secondary-color: #007bff;
    --font-family: Arial, sans-serif;
}

body {
    color: var(--primary-color);
    font-family: var(--font-family);
}
```

#### **Responsive Design :**
```css
/* Mobile first */
.example-wrapper {
    width: 95%;
    padding: 10px;
}

/* Tablet */
@media (min-width: 768px) {
    .example-wrapper {
        width: 80%;
        padding: 20px;
    }
}

/* Desktop */
@media (min-width: 1024px) {
    .example-wrapper {
        max-width: 800px;
        padding: 30px;
    }
}
```

### **8.8 - Debugging CSS**

#### **Vérifier le chargement des CSS :**
```bash
# Voir les assets générés
ls -la public/assets/

# Vérifier dans le navigateur (F12)
# → Onglet Network → Voir si le CSS se charge
# → Onglet Elements → Vérifier que les styles s'appliquent
```

#### **Problèmes courants :**
```bash
# CSS ne se charge pas
→ Vérifier le chemin dans asset()
→ Vérifier que le fichier existe
→ Vider le cache : php bin/console cache:clear

# Styles ne s'appliquent pas
→ Vérifier la spécificité CSS
→ Utiliser !important temporairement pour débugger
→ Vérifier l'ordre d'inclusion des CSS
```

### **8.9 - Performance et optimisation**

#### **Minification en production :**
```bash
# En production, Symfony peut minifier automatiquement
# Voir config/packages/prod/framework.yaml
```

#### **Ordre de chargement optimal :**
```twig
<head>
    <!-- 1. CSS externes (CDN) -->
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/css/bootstrap.min.css">
    
    <!-- 2. CSS de base de l'application -->
    <link rel="stylesheet" href="{{ asset('assets/css/global.css') }}">
    
    <!-- 3. CSS spécifiques aux pages -->
    {% block stylesheets %}{% endblock %}
    
    <!-- 4. CSS inline critique -->
    <style>
        /* CSS critique pour éviter le FOUC */
    </style>
</head>
```

### **8.10 - Test de votre CSS**

```bash
# Vérifier que tout fonctionne
# 1. Aller sur https://localhost
# 2. Ouvrir les outils de développement (F12)
# 3. Vérifier dans l'onglet Network que HomeControlleur.css se charge
# 4. Vérifier dans l'onglet Elements que les styles s'appliquent
# 5. Modifier le CSS et rafraîchir pour voir les changements
```

---
---

## 🗄️ **9. Ajout d'Adminer - Interface graphique pour PostgreSQL**

### **9.1 - Qu'est-ce qu'Adminer ?**

Adminer est une interface web légère pour gérer vos bases de données directement depuis votre navigateur, similaire à phpMyAdmin mais pour plusieurs types de bases.

### **9.2 - Ajout du service Adminer au compose.yaml**

Ajouter ce service dans votre [`compose.yaml`](compose.yaml ) :
```yaml
services:
  adminer:
    image: adminer
    restart: always
    ports:
      - 8080:8080
```

### **9.3 - Redémarrage de la stack avec Adminer**
```bash
# Arrêter les services actuels
docker compose down

# Redémarrer avec Adminer inclus
docker compose up --wait

# Vérifier que tous les services fonctionnent
docker compose ps
```

### **9.4 - Accès à Adminer**
- **URL** : `http://localhost:8080`
- **Interface** : Web accessible depuis votre navigateur

### **9.5 - Connexion à PostgreSQL via Adminer**

Dans l'interface Adminer, utiliser ces paramètres :
- **Système** : PostgreSQL
- **Serveur** : database
- **Utilisateur** : symfony_user
- **Mot de passe** : symfony_secure_pwd_123
- **Base de données** : app

### **9.6 - Fonctionnalités disponibles**
- 📊 **Visualisation des tables** - Structure et données
- ✏️ **Édition directe** - Modifier les enregistrements
- 🔍 **Exécution de requêtes SQL** - Interface de requêtage
- 📈 **Schéma de base** - Vue d'ensemble des relations
- 📤 **Import/Export** - Sauvegarde et restauration

---

## 🔐 **10. Extension des privilèges utilisateur**

### **10.1 - Problème rencontré**

L'utilisateur `symfony_user` avait des privilèges trop limités causant des erreurs lors des migrations Doctrine et des opérations sur la structure de la base.

### **10.2 - Solution appliquée**

#### **Connexion en tant qu'administrateur :**
```bash
# Se connecter avec l'utilisateur admin PostgreSQL
docker compose exec database psql -U app -d app
```

#### **Extension des privilèges :**
```sql
-- Accorder tous les privilèges sur la base de données
GRANT ALL PRIVILEGES ON DATABASE app TO symfony_user;

-- Vérifier les nouveaux privilèges
\du

-- Quitter
\q
```

### **10.3 - Modification permanente du script d'initialisation**

Mise à jour du fichier `docker/postgres/init/01-create-symfony-user.sql` :
```sql
-- Script de sécurisation PostgreSQL pour Symfony (Version étendue)
-- Objectif: Créer un utilisateur avec permissions étendues pour l'application

DO $$
BEGIN
    -- Vérifier si l'utilisateur existe déjà
    IF NOT EXISTS (SELECT FROM pg_catalog.pg_roles WHERE rolname = 'symfony_user') THEN
        -- 1. Créer un utilisateur Symfony avec permissions étendues
        CREATE USER symfony_user WITH PASSWORD 'symfony_secure_pwd_123';
        
        -- 2. Accorder tous les privilèges sur la base de données
        GRANT ALL PRIVILEGES ON DATABASE app TO symfony_user;
        
        -- 3. Accorder les permissions sur le schéma public
        GRANT ALL ON SCHEMA public TO symfony_user;
        
        -- 4. Accorder toutes les permissions sur les tables existantes
        GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO symfony_user;
        
        -- 5. Accorder les permissions sur les futures tables
        ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO symfony_user;
        
        -- 6. Accorder les permissions sur les séquences
        GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO symfony_user;
        ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO symfony_user;
        
        -- 7. Permissions pour les fonctions et procédures
        GRANT ALL PRIVILEGES ON ALL FUNCTIONS IN SCHEMA public TO symfony_user;
        ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON FUNCTIONS TO symfony_user;
        
        RAISE NOTICE 'Utilisateur symfony_user créé avec privilèges étendus';
        RAISE NOTICE 'SÉCURITÉ: Cet utilisateur peut maintenant gérer la structure de la base';
    ELSE
        RAISE NOTICE 'Utilisateur symfony_user existe déjà';
    END IF;
END
$$;
```

### **10.4 - Correction du mot de passe par défaut dans compose.yaml**

**Problème identifié** : Le mot de passe par défaut était exposé dans le compose.yaml

**Solution appliquée** :
Dans [`compose.yaml`](compose.yaml ), la ligne `DATABASE_URL` a été modifiée :
```yaml
# Configuration sécurisée avec placeholder
DATABASE_URL: postgresql://${POSTGRES_USER:-symfony_user}:${POSTGRES_PASSWORD:-symfony_secure_pwd_123}@database:5432/${POSTGRES_DB:-app}?serverVersion=${POSTGRES_VERSION:-15}&charset=${POSTGRES_CHARSET:-utf8}
```

**Variables dans .env.local** (non versionné) :
```bash
# Fichier .env.local - NE PAS VERSIONNER
POSTGRES_PASSWORD=symfony_secure_pwd_123
POSTGRES_USER=symfony_user
POSTGRES_DB=app
POSTGRES_VERSION=16
```

### **10.5 - Test des nouveaux privilèges**
```bash
# Connexion avec l'utilisateur étendu
docker compose exec database psql -U symfony_user -d app

# Test de création de table (DOIT FONCTIONNER)
CREATE TABLE test_privileges (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL
);

# Test d'ajout de contrainte (DOIT FONCTIONNER)
ALTER TABLE test_privileges ADD CONSTRAINT unique_nom UNIQUE (nom);

# Test de suppression de table (DOIT FONCTIONNER)
DROP TABLE test_privileges;

# Test de destruction de base (DOIT TOUJOURS ÉCHOUER)
DROP DATABASE app;
-- Résultat attendu: ERROR: must be owner of database

# Sortir
\q
```

---

## 🏗️ **11. Création d'entités avec Doctrine**

### **11.1 - Création de l'entité Moto**

```bash
# Entrer dans le conteneur PHP
docker compose exec php bash

# Créer l'entité Moto avec le maker
php bin/console make:entity Moto
```

#### **Configuration interactive appliquée :**
```
New property name: marque
Field type: string
Field length: 100
Can this field be null: false

New property name: modele  
Field type: string
Field length: 100
Can this field be null: false

New property name: (Entrée pour finir)
```

### **11.2 - Création de l'entité Garage avec relation**

```bash
# Créer l'entité Garage
php bin/console make:entity Garage
```

#### **Configuration avec relation OneToMany :**
```
New property name: nom
Field type: string
Field length: 150
Can this field be null: false

New property name: adresse
Field type: text
Can this field be null: true

New property name: motos
Field type: relation
Related to: Moto
Relation type: OneToMany
New field name inside Moto: garage
Moto.garage nullable: true
```

### **11.3 - Fichiers générés automatiquement**

#### **Structure créée :**
```
src/
├── Entity/
│   ├── Moto.php              ← Entité avec propriétés et relations
│   └── Garage.php            ← Entité avec collection de motos
└── Repository/
    ├── MotoRepository.php     ← Méthodes de requête personnalisées
    └── GarageRepository.php   ← Méthodes de requête personnalisées
```

---

## 🔄 **12. Gestion des migrations Doctrine**

### **12.1 - Génération des migrations**

```bash
# Après création des entités, générer la migration
php bin/console make:migration

# Résultat : Fichier migration dans migrations/Version[timestamp].php
```

### **12.2 - Application des migrations**

```bash
# Voir l'état actuel des migrations
php bin/console doctrine:migrations:status

# Appliquer toutes les migrations en attente
php bin/console doctrine:migrations:migrate --no-interaction

# Vérifier que les tables sont créées via Adminer
# URL: http://localhost:8080
```

### **12.3 - Migrations générées dans votre projet**

D'après votre structure, ces migrations ont été créées :
- `migrations/Version20250912124208.php` - Création tables Moto/Garage
- `migrations/Version20250912135931.php` - Ajustements structure
- `migrations/Version20250912140406.php` - Contraintes et index

---

## 🎮 **13. Création du TesttableController avec CRUD**

### **13.1 - Génération du contrôleur**

```bash
# Créer le contrôleur de test
php bin/console make:controller TesttableController
```

### **13.2 - Implémentation du contrôleur avec EntityManager**

**Fichier généré** : `src/Controller/TesttableController.php`

**Modifications appliquées** :
```php
<?php

namespace App\Controller;

use App\Entity\Moto;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

final class TesttableController extends AbstractController
{
    #[Route('/testtable', name: 'app_testtable')]
    public function index(): Response
    {
        return $this->render('testtable/index.html.twig', [
            'controller_name' => 'TesttableController',
        ]);
    }

    #[Route('/addmoto', name: 'addmoto')]
    public function addmoto(EntityManagerInterface $entityManager): Response
    {
        // Créer une nouvelle moto
        $moto = new Moto();
        $moto->setMarque('Honda');
        $moto->setModele('CBR600RR');

        // Persister et sauvegarder en base
        $entityManager->persist($moto);
        $entityManager->flush();

        return new Response('Moto enregistrée avec l\'id '.$moto->getId());
    }
}
```

### **13.3 - Résolution de l'erreur d'import**

**Problème rencontré** :
```
Controller requires the "$entityManager" argument that could not be resolved.
Cannot determine controller argument: the $entityManager argument is type-hinted 
with the non-existent class: "App\Controller\EntityManagerInterface"
```

**Solution appliquée** :
Correction des imports dans le contrôleur :
```php
// ❌ Import incorrect (causait l'erreur)
use App\Controller\EntityManagerInterface;

// ✅ Import correct (résout le problème)
use Doctrine\ORM\EntityManagerInterface;
```

### **13.4 - Tests des routes implémentées**

```bash
# Route de test principale
# https://localhost/testtable

# Route d'ajout de moto
# https://localhost/addmoto
# Résultat attendu: "Moto enregistrée avec l'id X"
```

---

## 📊 **14. Structure finale du projet**

### **14.1 - Services Docker opérationnels**

D'après votre [`compose.yaml`](compose.yaml ) :
- **Application Symfony** : `https://localhost` (ports 80/443)
- **Adminer** : `http://localhost:8080`
- **PostgreSQL** : Interne (port 5432)

### **14.2 - Configuration de sécurité appliquée**

#### **Variables d'environnement sécurisées :**
```yaml
# Dans compose.yaml (valeurs par défaut sécurisées)
POSTGRES_USER: ${POSTGRES_USER:-symfony_user}
POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-symfony_secure_pwd_123}
POSTGRES_DB: ${POSTGRES_DB:-app}
POSTGRES_VERSION: ${POSTGRES_VERSION:-16}
```

#### **Fichiers de secrets (non versionnés) :**
- `.env.local` - Variables de production locales
- `.env.dev.local` - Variables de développement

### **14.3 - Architecture complète actuelle**

```
📁 Votre projet Symfony complet
├── compose.yaml                   ← Configuration Docker avec Adminer
├── docker/postgres/init/          ← Scripts d'initialisation sécurisés
├── migrations/                    ← 3 migrations Doctrine appliquées
├── src/
│   ├── Controller/
│   │   ├── HomeController.php     ← Page d'accueil (route: /)
│   │   └── TesttableController.php ← CRUD motos (routes: /testtable, /addmoto)
│   ├── Entity/
│   │   ├── Moto.php              ← Entité avec marque/modèle
│   │   └── Garage.php            ← Entité avec relation OneToMany
│   └── Repository/               ← Classes de requête générées
├── templates/                    ← Vues Twig
├── public/assets/css/            ← CSS personnalisés
└── .env.local                    ← Secrets (non versionné)
```

### **14.4 - Fonctionnalités validées**

#### **✅ Base de données :**
- PostgreSQL avec utilisateur sécurisé mais étendu
- Interface Adminer opérationnelle
- Migrations Doctrine appliquées
- Tables Moto et Garage créées avec relations

#### **✅ Application :**
- Contrôleurs fonctionnels avec injection de dépendances
- CRUD de base opérationnel (insertion de motos)
- Templates Twig avec CSS personnalisé
- Navigation entre les pages

#### **✅ Sécurité :**
- Scripts d'initialisation automatisés
- Variables sensibles non versionnées
- Privilèges PostgreSQL équilibrés (fonctionnels mais contrôlés)

---

## 🔧 **15. Commandes de validation finale**

### **15.1 - Tests de l'environnement complet**
```bash
# Vérifier tous les conteneurs
docker compose ps

# Tester la connexion BDD
docker compose exec php bash
php bin/console doctrine:query:sql "SELECT version();"

# Tester l'insertion via URL
# https://localhost/addmoto

# Vérifier via Adminer
# http://localhost:8080 (connexion avec symfony_user)
```

### **15.2 - État actuel des migrations**
```bash
# Voir l'état des migrations
php bin/console doctrine:migrations:status

# Résultat attendu : 3 migrations exécutées
# Version20250912124208, Version20250912135931, Version20250912140406
```

### **15.3 - Routes disponibles**
```bash
# Lister toutes les routes actives
php bin/console debug:router

# Routes attendues :
# app_home (/)
# app_testtable (/testtable)  
# addmoto (/addmoto)
```

**🚀 Configuration complète validée : Docker + PostgreSQL + Adminer + Entités Doctrine + CRUD fonctionnel !**


## �� **Commandes utiles**

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

psql -U app

app=# GRANT ALL PRIVILEGES ON DATABASE app TO symfony_user;

app=# GRANT ALL PRIVILEGES IN SCHEMA PUBLIC TO symfony_user;

Dans le dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    acl \
    file \
    gettext \
    git \
    curl \
    && rm -rf /var/lib/apt/lists/*
# Installer Node.js et npm
RUN curl -fsSL https://deb.nodesource.com/setup_lts.x | bash - \
    && apt-get install -y nodejs \
    && npm install -g @dbml/cli
    
Utiliser dbdiagram.io 
db2dbml postgres 'postgresql://user_symfony:secret@database:5432/app' -o database1.dbml

db2dbml postgres 'postgresql://user:password@localhost:5432/dbname?schemas=schema1,schema2,schema3' -o database.dbml
