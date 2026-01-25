# Cours d'introduction à NestJS

Ce repository contient un cours complet d'introduction à NestJS, destiné à une première journée de découverte.

## Public cible

Ce cours s'adresse à des personnes qui connaissent déjà les bases du développement web, JavaScript et TypeScript, ainsi que les principes généraux d'une API REST, mais qui découvrent NestJS. L'objectif est de permettre de comprendre la philosophie du framework, son architecture et ses concepts fondamentaux afin de devenir rapidement autonomes et capables de lire et comprendre la documentation officielle de NestJS.

## Structure du cours

Le cours est organisé en six chapitres progressifs, chacun dans un fichier markdown dédié. Il est recommandé de les lire dans l'ordre présenté ci-dessous.

### 01 - Introduction à NestJS

**Fichier :** `01-introduction-nestjs.md`

Ce chapitre pose le contexte et explique pourquoi NestJS existe. Il couvre le paysage du développement backend en Node.js, les limites d'Express pour les projets complexes, l'origine de NestJS et sa philosophie, ainsi que sa place dans l'écosystème Node.js. Ce chapitre répond à la question fondamentale : pourquoi utiliser NestJS plutôt qu'Express ?

### 02 - Architecture et concepts fondamentaux

**Fichier :** `02-architecture-et-concepts.md`

Ce chapitre explique l'architecture en couches de NestJS et introduit les concepts fondamentaux du framework. Il couvre en détail les modules, les contrôleurs, les services, l'injection de dépendances, les décorateurs, la structure d'un projet NestJS et le cycle de vie d'une requête. L'accent est mis sur la compréhension du "pourquoi" derrière chaque concept avant d'entrer dans les détails techniques.

### 03 - Créer votre première API REST

**Fichier :** `03-premiere-api-rest.md`

Ce chapitre est orienté pratique. Il guide pas à pas la création d'une API REST complète pour gérer des articles de blog. Il couvre la mise en place de l'environnement, la création d'un module complet, l'implémentation des opérations CRUD, la gestion des paramètres de route et query parameters, la gestion des codes HTTP et des erreurs, ainsi que les bonnes pratiques à adopter dès le départ.

### 04 - Les DTO et la validation

**Fichier :** `04-dto-et-validation.md`

Ce chapitre explique pourquoi NestJS encourage fortement l'usage des DTOs, même dans des projets simples. Il couvre la création de DTOs, l'intégration de class-validator pour la validation automatique, les validations avancées et personnalisées, et l'importance des DTOs pour la sécurité, la documentation et la maintenabilité du code.

### 05 - Authentification avec JWT

**Fichier :** `05-authentification-jwt.md`

Ce chapitre introduit l'authentification dans une application NestJS en utilisant les JSON Web Tokens (JWT). Il explique le fonctionnement des JWT, la mise en place de Passport.js avec NestJS, la création d'un système d'inscription et de connexion, la protection des routes avec des Guards, et la gestion sécurisée des mots de passe avec bcrypt. Le chapitre couvre également les bonnes pratiques de sécurité pour l'authentification.

### 06 - Persistance de données avec TypeORM

**Fichier :** `06-persistance-de-donnees.md`

Ce chapitre explique comment passer d'un stockage en mémoire à une vraie base de données en utilisant TypeORM. Il couvre les concepts d'ORM, la configuration de TypeORM avec NestJS, la création d'entités avec les décorateurs TypeORM, l'utilisation des repositories pour les opérations CRUD, les requêtes avancées (filtrage, tri), les relations entre entités (One-to-Many, Many-to-One), et les bonnes pratiques pour la production (migrations, transactions, indexation).

## Prérequis techniques

Pour suivre ce cours dans les meilleures conditions, assurez-vous d'avoir :

- Node.js version 16 ou supérieure installée
- npm ou yarn comme gestionnaire de paquets
- Un éditeur de code (VS Code recommandé)
- Des connaissances de base en JavaScript et TypeScript
- Une compréhension des concepts d'API REST

## Ressources complémentaires

- Documentation officielle : https://docs.nestjs.com
- Repository GitHub officiel : https://github.com/nestjs/nest
- Exemples officiels : https://github.com/nestjs/nest/tree/master/sample
- Discord officiel de la communauté NestJS