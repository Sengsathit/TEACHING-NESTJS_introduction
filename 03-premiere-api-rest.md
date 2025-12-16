# Créer votre première API REST avec NestJS

## Mise en place de l'environnement

Avant de commencer à coder, assurons-nous que votre environnement est prêt. Vous aurez besoin de Node.js (version 16 ou supérieure recommandée) et de npm ou yarn. Pour vérifier que tout est en place, ouvrez un terminal et tapez `node --version` et `npm --version`.

NestJS fournit un CLI (Command Line Interface) qui facilite grandement la création et la gestion des projets. Commençons par l'installer globalement :

```bash
npm install -g @nestjs/cli
```

Une fois le CLI installé, créer un nouveau projet devient extrêmement simple :

```bash
nest new mon-api
```

Le CLI vous posera quelques questions, notamment le gestionnaire de paquets que vous souhaitez utiliser (npm ou yarn). Il créera ensuite la structure complète du projet, installera toutes les dépendances, et initialisera un repository Git.

Naviguez dans le dossier du projet et lancez le serveur de développement :

```bash
cd mon-api
npm run start:dev
```

Le mode `start:dev` active le rechargement automatique. À chaque modification de fichier, l'application redémarre automatiquement, ce qui est très pratique pour le développement.

Ouvrez votre navigateur et allez sur `http://localhost:3000`. Vous devriez voir un message "Hello World!". Si c'est le cas, félicitations, votre environnement NestJS fonctionne parfaitement.

## Notre objectif : une API de gestion d'articles

Pour ce chapitre pratique, nous allons créer une API simple pour gérer des articles de blog. Cette API permettra de créer, lire, modifier et supprimer des articles. C'est un cas d'usage classique qui illustre bien les différents types de requêtes HTTP.

Commençons par utiliser le CLI pour générer la structure de base de notre module :

```bash
nest generate module articles
nest generate controller articles
nest generate service articles
```

Ces commandes créent automatiquement les fichiers nécessaires et les connectent entre eux. Le CLI met à jour `app.module.ts` pour importer le nouveau module, et connecte le contrôleur et le service dans le module articles.

Si vous préférez une commande plus courte, vous pouvez utiliser les alias :

```bash
nest g module articles
nest g controller articles
nest g service articles
```

## Modéliser nos données

Avant d'écrire le code du service, définissons ce qu'est un article. Dans un projet réel avec une base de données, vous utiliseriez probablement un ORM comme TypeORM ou Prisma. Pour ce cours d'introduction, nous allons utiliser un simple tableau en mémoire.

Créons une interface TypeScript pour représenter un article. Créez un fichier `src/articles/interfaces/article.interface.ts` :

```typescript
export interface Article {
  id: string;
  title: string;
  content: string;
  author: string;
  published: boolean;
  createdAt: Date;
  updatedAt: Date;
}
```

Cette interface décrit la structure d'un article. Notez que nous utilisons des types TypeScript précis. Cette rigueur dans le typage nous protégera contre de nombreuses erreurs.

## Implémenter le service

Ouvrez le fichier `src/articles/articles.service.ts` et remplacez son contenu par :

```typescript
import { Injectable, NotFoundException } from '@nestjs/common';
import { Article } from './interfaces/article.interface';

@Injectable()
export class ArticlesService {
  private articles: Article[] = [
    {
      id: '1',
      title: 'Introduction à NestJS',
      content: 'NestJS est un framework Node.js...',
      author: 'Marie Dupont',
      published: true,
      createdAt: new Date('2024-01-15'),
      updatedAt: new Date('2024-01-15'),
    },
    {
      id: '2',
      title: 'TypeScript avancé',
      content: 'Découvrons les fonctionnalités avancées...',
      author: 'Jean Martin',
      published: false,
      createdAt: new Date('2024-01-20'),
      updatedAt: new Date('2024-01-20'),
    },
  ];

  findAll(): Article[] {
    return this.articles;
  }

  findOne(id: string): Article {
    const article = this.articles.find(article => article.id === id);
    if (!article) {
      throw new NotFoundException(`Article with ID ${id} not found`);
    }
    return article;
  }

  create(articleData: Partial<Article>): Article {
    // Validation des champs obligatoires
    if (!articleData.title || !articleData.content || !articleData.author) {
        throw new BadRequestException('Title, content, and author are required');
    }

    const newArticle: Article = {
      id: String(Date.now()), // Simple génération d'ID
      title: articleData.title,
      content: articleData.content,
      author: articleData.author,
      published: articleData.published ?? false,
      createdAt: new Date(),
      updatedAt: new Date(),
    };
    this.articles.push(newArticle);
    return newArticle;
  }

  update(id: string, articleData: Partial<Article>): Article {
    const articleIndex = this.articles.findIndex(a => a.id === id);
    if (articleIndex === -1) {
      throw new NotFoundException(`Article with ID ${id} not found`);
    }

    const updatedArticle = {
      ...this.articles[articleIndex],
      ...articleData,
      id: this.articles[articleIndex].id, // On ne change jamais l'ID
      updatedAt: new Date(),
    };

    this.articles[articleIndex] = updatedArticle;
    return updatedArticle;
  }

  remove(id: string): void {
    const articleIndex = this.articles.findIndex(a => a.id === id);
    if (articleIndex === -1) {
      throw new NotFoundException(`Article with ID ${id} not found`);
    }
    this.articles.splice(articleIndex, 1);
  }
}
```

Analysons ce service en détail. Nous stockons nos articles dans un tableau privé, initialisé avec deux articles d'exemple. Dans une vraie application, ces données viendraient d'une base de données.

La méthode `findAll` retourne simplement tous les articles. C'est la plus simple.

La méthode `findOne` recherche un article par son ID. Si elle ne le trouve pas, elle lance une exception `NotFoundException`. C'est une classe fournie par NestJS qui génère automatiquement une réponse HTTP 404. Remarquez que nous ne gérons pas directement la réponse HTTP dans le service, nous lançons simplement une exception métier, et NestJS s'occupe de la transformer en réponse HTTP appropriée.

La méthode `create` construit un nouvel article à partir des données fournies. Nous générons un ID simple basé sur le timestamp (dans un vrai projet, la base de données s'en chargerait). Nous définissons `published` à `false` par défaut si cette valeur n'est pas fournie, grâce à l'opérateur de coalescence nulle `??`.

La méthode `update` trouve l'article existant, fusionne les nouvelles données avec l'ancien article en utilisant le spread operator, et met à jour le timestamp `updatedAt`. Notez que nous forçons l'ID à rester celui de l'article original pour éviter toute tentative de modification de l'ID via la requête.

La méthode `remove` supprime simplement l'article du tableau après avoir vérifié qu'il existe.

## Implémenter le contrôleur

Maintenant que notre service contient toute la logique métier, créons le contrôleur qui exposera ces fonctionnalités via des endpoints HTTP. Ouvrez `src/articles/articles.controller.ts` :

```typescript
import { Body, Controller, Delete, Get, HttpCode, HttpStatus, Param, Post, Put } from '@nestjs/common';
import * as articleInterface from './interfaces/article.interface';
import { ArticlesService } from './articles.service';
import type { Article } from './interfaces/article.interface';

@Controller('articles')
export class ArticlesController {
    constructor(private readonly articlesService: ArticlesService) { }

    @Get()
    findAll(): articleInterface.Article[] {
        return this.articlesService.findAll();
    }

    @Get(':id')
    findOne(@Param('id') id: string): Article {
        return this.articlesService.findOne(id);
    }

    @Post()
    @HttpCode(HttpStatus.CREATED)
    create(@Body() articleData: Partial<Article>): Article {
        return this.articlesService.create(articleData);
    }

    @Put(':id')
    update(
        @Param('id') id: string,
        @Body() articleData: Partial<Article>,
    ): Article {
        return this.articlesService.update(id, articleData);
    }

    @Delete(':id')
    @HttpCode(HttpStatus.NO_CONTENT)
    remove(@Param('id') id: string): void {
        this.articlesService.remove(id);
    }
}
```

Ce contrôleur est l'interface publique de notre API. Chaque méthode correspond à un endpoint HTTP spécifique.

L'endpoint `GET /articles` retourne la liste complète des articles. Simple et direct.

L'endpoint `GET /articles/:id` retourne un article spécifique. Le paramètre `:id` dans la route est extrait avec `@Param('id')`. Si l'article n'existe pas, le service lancera une `NotFoundException`, qui sera automatiquement convertie en réponse HTTP 404 par NestJS.

L'endpoint `POST /articles` crée un nouveau article. Le décorateur `@Body()` extrait le corps de la requête HTTP et le passe à la méthode. Nous avons ajouté `@HttpCode(HttpStatus.CREATED)` pour indiquer que la réponse doit avoir le code 201 (Created) au lieu du 200 par défaut. C'est une bonne pratique REST : un POST qui crée une ressource devrait retourner 201.

L'endpoint `PUT /articles/:id` met à jour un article existant. Il combine `@Param()` pour l'ID et `@Body()` pour les données à modifier.

L'endpoint `DELETE /articles/:id` supprime un article. Nous retournons le code 204 (No Content), qui est la convention REST pour une suppression réussie. Ce code indique que l'opération a réussi mais qu'il n'y a pas de contenu à retourner.

## Tester votre API

Maintenant que notre API est complète, testons-la. Vous pouvez utiliser plusieurs outils pour cela : Postman, Insomnia, curl, ou même l'extension REST Client de VS Code.

Pour lister tous les articles, envoyez une requête GET à `http://localhost:3000/articles`. Vous devriez recevoir les deux articles d'exemple.

Pour récupérer un article spécifique, essayez `GET http://localhost:3000/articles/1`. Vous devriez recevoir l'article avec l'ID 1.

Pour créer un nouvel article, envoyez une requête POST à `http://localhost:3000/articles` avec ce corps JSON :

```json
{
  "title": "Mon nouvel article",
  "content": "Contenu de l'article...",
  "author": "Votre nom",
  "published": true
}
```

Vous devriez recevoir l'article créé avec son ID généré et les timestamps.

Pour modifier un article, envoyez une requête PUT à `http://localhost:3000/articles/1` avec les champs à modifier :

```json
{
  "title": "Titre modifié"
}
```

Enfin, pour supprimer un article, envoyez une requête DELETE à `http://localhost:3000/articles/1`. Vous devriez recevoir un code 204 sans contenu.

## Gérer les query parameters

Jusqu'ici, nous avons utilisé des paramètres de route (comme `:id`) et le corps de la requête. Il existe un troisième moyen de passer des informations : les query parameters (les paramètres dans l'URL après le `?`).

Ajoutons une fonctionnalité de filtrage des articles par statut de publication. Modifions la méthode `findAll` dans le service :

```typescript
findAll(published?: boolean): Article[] {
  if (published === undefined) {
    return this.articles;
  }
  return this.articles.filter(article => article.published === published);
}
```

Et mettons à jour le contrôleur pour accepter ce paramètre :

```typescript
import { Controller, Get, Query, /* ... autres imports */ } from '@nestjs/common';

// ...

@Get()
findAll(@Query('published') published?: string): Article[] {
  // Les query params arrivent toujours comme des strings
  const publishedBoolean = published === 'true' ? true :
                          published === 'false' ? false :
                          undefined;
  return this.articlesService.findAll(publishedBoolean);
}
```

Maintenant, vous pouvez filtrer les articles avec `GET /articles?published=true` pour obtenir uniquement les articles publiés.

⚠️ REMARQUE IMPORTANTE : **les query parameters arrivent toujours comme des chaînes de caractères**. C'est à vous de les convertir dans le type approprié. NestJS fournit des outils pour automatiser cela (les Pipes), mais nous les verrons dans un cours plus avancé.

## Gérer les erreurs correctement

Dans notre code actuel, nous utilisons `NotFoundException` pour les ressources inexistantes. NestJS fournit plusieurs exceptions HTTP prêtes à l'emploi : `BadRequestException`, `UnauthorizedException`, `ForbiddenException`, etc.

C'est dans ce sens que nous avons ajouté une validation simple dans la méthode `create` du service :

```typescript
create(articleData: Partial<Article>): Article {
  // Validation des champs obligatoires
  if (!articleData.title || !articleData.content || !articleData.author) {
      throw new BadRequestException('Title, content, and author are required');
  }


  // ... reste du code
}
```

### Comprendre Partial<T>

Vous avez remarqué que nous utilisons `Partial<Article>` au lieu de `Article` pour le paramètre `articleData`. `Partial<T>` est un **type utilitaire** fourni par TypeScript qui rend tous les champs d'un type optionnels.

Dans notre interface `Article`, tous les champs sont requis :

```typescript
interface Article {
  id: string;
  title: string;
  content: string;
  // ... autres champs requis
}
```

Mais lors de la création ou de la modification d'un article, nous ne voulons pas que le client fournisse l'`id`, les `createdAt` ou `updatedAt` (car nous les générons nous-mêmes). De plus, lors d'une mise à jour, le client peut ne fournir que les champs qu'il souhaite modifier.

En utilisant `Partial<Article>`, TypeScript transforme l'interface en :

```typescript
// Équivalent de Partial<Article>
{
  id?: string;
  title?: string;
  content?: string;
  author?: string;
  published?: boolean;
  createdAt?: Date;
  updatedAt?: Date;
}
```

Tous les champs deviennent optionnels (notez le `?`). C'est pour cette raison que nous devons vérifier manuellement la présence des champs requis avec notre validation `if (!articleData.title || !articleData.content || !articleData.author)`.

Dans le prochain chapitre sur les DTOs, nous verrons une approche plus robuste qui nous permettra de définir exactement quels champs sont requis lors de la création ou de la modification.

N'oubliez pas d'importer `BadRequestException` :

```typescript
import { Injectable, NotFoundException, BadRequestException } from '@nestjs/common';
```

Maintenant, si vous essayez de créer un article sans tous les champs requis, vous recevrez une réponse HTTP 400 (Bad Request) avec un message d'erreur clair.

## Comprendre le flux de données

Faisons une pause pour bien comprendre le flux complet d'une requête de création d'article.

Le client envoie une requête `POST /articles` avec un corps JSON. NestJS reçoit cette requête et la route vers la méthode `create` du `ArticlesController`. Le décorateur `@Body()` indique à NestJS d'extraire le corps de la requête et de le passer comme paramètre `articleData`.

Le contrôleur appelle immédiatement `this.articlesService.create(articleData)`. Le service effectue les validations, construit le nouvel article, l'ajoute au tableau, et retourne l'article créé.

Le contrôleur retourne cet article. NestJS intercepte ce retour, le sérialise en JSON, ajoute les en-têtes HTTP appropriés (Content-Type: application/json), utilise le code de statut 201 grâce au décorateur `@HttpCode()`, et envoie la réponse au client.

Si une erreur se produit (par exemple une `BadRequestException`), NestJS l'intercepte, génère une réponse d'erreur standardisée avec le code HTTP approprié, et l'envoie au client. Vous n'avez pas besoin de gérer manuellement les blocs try-catch dans vos contrôleurs pour les erreurs métier.

## Les bonnes pratiques à retenir

Quelques principes importants émergent de ce que nous venons de construire.

Premièrement, gardez vos contrôleurs minces. Ils ne doivent contenir que la logique de routage et de transformation des données HTTP en paramètres métier. Toute la logique métier doit résider dans les services.

Deuxièmement, utilisez les exceptions HTTP de NestJS pour gérer les erreurs. Ne retournez pas manuellement des objets d'erreur avec des codes de statut. Lancez des exceptions, et laissez NestJS les convertir en réponses HTTP.

Troisièmement, soyez explicite avec les codes de statut HTTP quand c'est pertinent. Par défaut, GET, PUT, PATCH retournent 200, POST retourne 201, et DELETE retourne 200. Si vous voulez un comportement différent, utilisez `@HttpCode()`.

Quatrièmement, typez tout. Même si nous utilisons `Partial<Article>` pour les créations et modifications (car tous les champs ne sont pas forcément fournis), nous avons une structure claire. Dans le prochain chapitre, nous verrons comment améliorer cela avec les DTOs.

## Ajouter une couche de logs

Pour mieux comprendre ce qui se passe dans votre application, ajoutons quelques logs. NestJS intègre un logger que vous pouvez utiliser facilement :

```typescript
import { Injectable, NotFoundException, Logger } from '@nestjs/common';

@Injectable()
export class ArticlesService {
  private readonly logger = new Logger(ArticlesService.name);
  private articles: Article[] = [ /* ... */ ];

  findAll(published?: boolean): Article[] {
    this.logger.log(`Finding all articles. Filter: published=${published}`);
    // ... reste du code
  }

  create(articleData: Partial<Article>): Article {
    this.logger.log(`Creating new article: ${articleData.title}`);
    // ... reste du code
    this.logger.log(`Article created with ID: ${newArticle.id}`);
    return newArticle;
  }

  // ... autres méthodes avec logs
}
```

Relancez votre application et observez la console. Chaque fois qu'une opération est effectuée, vous verrez un log détaillé. C'est extrêmement utile pour le débogage et le monitoring en production.

## Récapitulation pratique

Nous avons construit une API REST complète qui gère des articles. Cette API respecte les conventions REST, gère correctement les codes HTTP, et structure le code selon les principes de NestJS.

Nous avons vu comment créer un module, un contrôleur et un service. Nous avons appris à gérer différents types de paramètres HTTP : route parameters, query parameters, et body. Nous avons compris comment les exceptions métier se transforment automatiquement en réponses HTTP.

Cette base est solide, mais il manque encore un élément crucial : les DTOs (Data Transfer Objects). Jusqu'ici, nous avons utilisé `Partial<Article>` ou `any` pour les données entrantes. Dans le prochain chapitre, nous verrons pourquoi c'est problématique et comment les DTOs résolvent ces problèmes de manière élégante.
