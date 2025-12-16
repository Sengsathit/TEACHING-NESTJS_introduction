# Architecture et concepts fondamentaux de NestJS

## Comprendre l'architecture en couches

Avant de plonger dans le code, construisons ensemble un modèle mental de l'architecture NestJS. Imaginez votre application comme un bâtiment à plusieurs étages, où chaque étage a un rôle spécifique et communique avec les étages adjacents selon des règles précises.

Au sommet de ce bâtiment, nous trouvons les contrôleurs. Ce sont eux qui reçoivent les requêtes HTTP provenant du monde extérieur. Leur rôle est simple : comprendre ce que le client demande, déléguer le travail à la bonne personne, puis renvoyer la réponse. Un contrôleur ne fait pas le travail lui-même, il orchestre.

Juste en dessous, nous avons les services. C'est là que réside la vraie logique métier de votre application. Un service sait comment créer un utilisateur, comment calculer le prix total d'une commande, comment valider des données métier. Les services sont les experts de votre domaine.

Enfin, dans les fondations, on peut trouver les repositories ou les providers qui gèrent l'accès aux données, qu'il s'agisse d'une base de données, d'une API externe ou d'un système de fichiers.

Cette séparation claire des responsabilités n'est pas qu'une question d'organisation du code. Elle facilite la maintenance, les tests et la collaboration. Si vous devez changer la façon dont vous stockez vos données, vous modifiez uniquement la couche d'accès aux données. Si vous devez ajouter une nouvelle route, vous ajoutez une méthode dans un contrôleur. Chaque changement a un impact localisé.

## Les modules : organiser votre application

Dans NestJS, tout commence avec les modules. Un module est un conteneur qui regroupe un ensemble de fonctionnalités liées entre elles. Si votre application gère des utilisateurs, des articles et des commandes, vous aurez probablement un module pour chacune de ces entités.

Prenons un exemple concret avec un module utilisateur :

```typescript
import { Module } from '@nestjs/common';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

@Module({
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

Analysons ce code. La classe `UsersModule` est décorée avec `@Module()`. Ce décorateur accepte un objet de configuration qui décrit le contenu du module. Dans `controllers`, nous listons tous les contrôleurs de ce module. Dans `providers`, nous listons les services et autres fournisseurs de fonctionnalités. Le tableau `exports` indique ce que ce module rend accessible aux autres modules.

Cette notion d'export est importante. Par défaut, tout ce qui est dans un module reste privé à ce module. Si vous voulez qu'un autre module puisse utiliser votre `UsersService`, vous devez l'exporter explicitement. C'est une forme d'encapsulation qui vous force à réfléchir aux interfaces publiques de vos modules.

Chaque application NestJS possède au moins un module racine, conventionnellement appelé `AppModule`. C'est le point d'entrée de votre application, celui qui importe tous les autres modules.

```typescript
import { Module } from '@nestjs/common';
import { UsersModule } from './users/users.module';
import { ArticlesModule } from './articles/articles.module';

@Module({
  imports: [UsersModule, ArticlesModule],
})
export class AppModule {}
```

Cette structure modulaire présente plusieurs avantages. Elle permet de découper une grande application en morceaux plus petits et plus gérables. Elle facilite le travail en équipe, car différents développeurs peuvent travailler sur différents modules sans se marcher sur les pieds.

## Les contrôleurs : l'interface avec le monde extérieur

Un contrôleur dans NestJS est une classe TypeScript décorée qui gère les requêtes HTTP entrantes. Voici un exemple simple :

```typescript
import { Controller, Get, Post, Body, Param } from '@nestjs/common';
import { UsersService } from './users.service';

@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get()
  findAll() {
    return this.usersService.findAll();
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.usersService.findOne(id);
  }

  @Post()
  create(@Body() createUserDto: any) {
    return this.usersService.create(createUserDto);
  }
}
```

Décomposons ce code étape par étape. Le décorateur `@Controller('users')` indique que cette classe gère toutes les routes commençant par `/users`. C'est le préfixe de base pour toutes les méthodes de ce contrôleur.

Dans le constructeur, nous voyons quelque chose d'intéressant : `private readonly usersService: UsersService`. Cette syntaxe TypeScript est un raccourci qui crée automatiquement une propriété de classe. Mais le plus important ici, c'est que nous ne créons pas nous-mêmes une instance de `UsersService`. Nous déclarons simplement que nous avons besoin d'un `UsersService`, et NestJS se charge de nous en fournir un. C'est le principe de l'**injection de dépendances**, sur lequel nous reviendrons.

Chaque méthode du contrôleur représente un endpoint.\
Le décorateur `@Get()` indique que la méthode `findAll` répond aux requêtes GET sur `/users`.\
Le décorateur `@Get(':id')` crée une route avec un paramètre dynamique : `/users/123`, `/users/abc`, etc.\
Le décorateur `@Param('id')` extrait ce paramètre et le passe à la méthode.\
Le décorateur `@Body()` extrait le corps de la requête HTTP et le passe comme paramètre. Notez que pour le moment nous utilisons `any` comme type, mais dans une vraie application, nous utiliserions un DTO, que nous verrons plus tard.

L'aspect crucial à comprendre ici est que le contrôleur ne fait presque rien lui-même. Il reçoit la requête, extrait les informations pertinentes, appelle la méthode appropriée du service, et retourne le résultat. C'est un simple chef d'orchestre.

## Les services : le cœur métier

Un service contient la logique métier de votre application. C'est là que se trouvent les algorithmes, les règles de gestion, les transformations de données. Voici un exemple simple :

```typescript
import { Injectable } from '@nestjs/common';

@Injectable()
export class UsersService {
  private users = [
    { id: '1', name: 'Alice', email: 'alice@example.com' },
    { id: '2', name: 'Bob', email: 'bob@example.com' },
  ];

  findAll() {
    return this.users;
  }

  findOne(id: string) {
    const user = this.users.find(user => user.id === id);
    if (!user) {
      throw new Error('User not found');
    }
    return user;
  }

  create(userData: any) {
    const newUser = {
      id: String(this.users.length + 1),
      ...userData,
    };
    this.users.push(newUser);
    return newUser;
  }
}
```

Le décorateur `@Injectable()` est fondamental. Il indique à NestJS que cette classe peut être injectée dans d'autres classes. Sans ce décorateur, NestJS ne pourrait pas gérer automatiquement la création et l'injection de cette classe.

Dans cet exemple, nous stockons les utilisateurs en mémoire dans un simple tableau. Bien sûr, dans une vraie application, vous utiliseriez une base de données, mais le principe reste le même : le service contient la logique de manipulation des données.

Remarquez que le service ne sait rien des requêtes HTTP. Il ne manipule pas d'objets `req` ou `res`. Il travaille avec des données JavaScript pures. Cette indépendance vis-à-vis du protocole HTTP facilite énormément les tests. Vous pouvez tester votre logique métier sans avoir à simuler des requêtes HTTP.

## L'injection de dépendances : la magie de NestJS

L'injection de dépendances est probablement le concept le plus important et le plus déroutant de NestJS pour les débutants. Prenons le temps de vraiment comprendre ce mécanisme.

Traditionnellement, si une classe A a besoin d'utiliser une classe B, on écrirait quelque chose comme :

```typescript
class A {
  private b: B;

  constructor() {
    this.b = new B(); // A crée elle-même son instance de B
  }
}
```

Le problème avec cette approche est que A et B sont fortement couplées. A doit savoir comment créer B, et si B change (par exemple si son constructeur nécessite maintenant des paramètres), vous devez modifier A. De plus, tester A devient compliqué car vous ne pouvez pas facilement remplacer B par une version de test.

L'injection de dépendances inverse ce principe :

```typescript
@Injectable()
class B {
  // B doit être marquée comme injectable
}

class A {
  constructor(private readonly b: B) {
    // A reçoit B déjà créée, elle ne la crée pas elle-même
  }
}
```

Maintenant, A déclare simplement qu'elle a besoin de B, mais c'est quelqu'un d'autre qui est responsable de créer B et de la fournir à A. Ce "quelqu'un d'autre", c'est le conteneur d'injection de dépendances de NestJS.

Quand NestJS démarre, il analyse tous vos modules et construit un graphe de dépendances. Il voit que `UsersController` a besoin de `UsersService`. Il voit que `UsersService` est marqué comme `@Injectable()` et est listé dans les `providers` du module. Il crée donc automatiquement une instance de `UsersService` et l'injecte dans `UsersController`.

Ce mécanisme a plusieurs conséquences importantes. D'abord, par défaut, NestJS crée une seule instance de chaque service (**singleton**) et la réutilise partout où elle est nécessaire. Cela garantit que tous vos contrôleurs partagent la même instance du service, ce qui est généralement ce que vous voulez.

Cela rend votre code plus flexible. Si vous décidez un jour que `UsersController` a besoin d'un autre service comme `LoggerService`, vous ajoutez simplement cette dépendance dans le constructeur de `UsersController`, et NestJS se charge automatiquement de l'injecter.

## Les décorateurs : comprendre la magie

Les décorateurs sont partout dans NestJS : `@Module()`, `@Controller()`, `@Injectable()`, `@Get()`, `@Post()`, etc. Mais que font-ils réellement ?

Un décorateur est une fonctionnalité de TypeScript qui permet d'ajouter des métadonnées à une classe, une méthode, une propriété ou un paramètre. Les métadonnées sont des informations supplémentaires qui ne changent pas le comportement immédiat du code, mais que d'autres parties du programme peuvent lire et utiliser.

Quand vous écrivez `@Controller('users')`, vous ne faites qu'attacher une étiquette à votre classe qui dit "ceci est un contrôleur pour les routes /users". Le code de la classe elle-même reste inchangé. Mais au démarrage de l'application, NestJS parcourt toutes vos classes, lit ces métadonnées, et configure le routage en conséquence.

C'est pour cela qu'on dit que NestJS utilise une approche déclarative. Vous déclarez ce que vous voulez (via les décorateurs), et le framework se charge de mettre tout cela en place. Vous n'avez pas besoin d'appeler manuellement une fonction `app.registerController()` ou quelque chose du genre pour enregistrer vos contrôleurs et configurer les routes.

Cette approche présente un avantage majeur : votre code reste propre et lisible. Au lieu d'avoir un énorme fichier de configuration qui liste toutes les routes, tous les services, etc., chaque élément porte ses propres métadonnées. Le code devient auto-documenté.

Il est important de comprendre que les décorateurs ne font rien d'eux-mêmes au moment de l'exécution de votre code métier. Ils sont lus une seule fois au démarrage de l'application. C'est à ce moment que NestJS construit toute sa configuration interne, son système de routage, son conteneur d'injection de dépendances, etc.

## La structure d'un projet NestJS

Quand vous créez un nouveau projet NestJS avec le CLI (via `nest new mon-projet`), vous obtenez une structure de fichiers prédéfinie. Comprendre cette structure est essentiel pour naviguer confortablement dans un projet.

À la racine, vous trouverez les fichiers de configuration classiques : `package.json`, `tsconfig.json`, etc. Le dossier `src` contient tout votre code source.

Dans `src`, le fichier `main.ts` est le point d'entrée de votre application :

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();
```

Ce fichier est remarquablement simple. Il crée une application NestJS en partant du module racine `AppModule`, puis démarre le serveur sur le port 3000. C'est ici que vous pourriez ajouter des configurations globales, comme l'activation de **CORS** ou l'ajout de **middlewares globaux**.

Le fichier `app.module.ts` est votre module racine :

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';

@Module({
  imports: [],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

Dans un projet fraîchement créé, ce module contient juste un contrôleur et un service d'exemple. Au fur et à mesure que vous développez votre application, vous ajouterez d'autres modules dans le tableau `imports`.

Les fichiers `app.controller.ts` et `app.service.ts` sont des exemples de base que vous pouvez étudier ou supprimer. Ils démontrent simplement comment créer un contrôleur et un service simple.

Quand vous ajoutez de nouvelles fonctionnalités, la convention est de créer un dossier par module. Par exemple, si vous ajoutez la gestion des utilisateurs, vous créerez un dossier `users` contenant :

```
src/users/
  ├── users.module.ts
  ├── users.controller.ts
  ├── users.service.ts
  └── dto/
      ├── create-user.dto.ts
      └── update-user.dto.ts
```

Cette organisation modulaire rend le projet facile à comprendre d'un coup d'œil. Chaque module est auto-contenu dans son propre dossier avec tous les fichiers liés.

## Le cycle de vie d'une requête

Maintenant que nous avons vu les différents acteurs, suivons le parcours d'une requête HTTP dans une application NestJS.

Une requête arrive, par exemple `GET /users/123`. NestJS consulte son système de routage interne et détermine que cette requête correspond au contrôleur `UsersController`, méthode `findOne`, avec le paramètre `id` valant `'123'`.

NestJS invoque cette méthode. Le décorateur `@Param('id')` a fait son travail au démarrage de l'application en indiquant à NestJS comment extraire ce paramètre, donc NestJS passe directement `'123'` comme argument.

À l'intérieur de la méthode `findOne`, le contrôleur appelle `this.usersService.findOne('123')`. L'instance de `UsersService` est déjà là, injectée par NestJS lors de la création du contrôleur.

Le service exécute sa logique métier et retourne un objet utilisateur. Le contrôleur retourne simplement cet objet.

NestJS intercepte ce retour, le sérialise automatiquement en JSON (par défaut), et l'envoie comme réponse HTTP avec le code de statut 200 (par défaut pour les GET).

Tout ce processus se passe de manière transparente. Vous n'avez pas besoin de gérer manuellement la sérialisation JSON, de définir le Content-Type, ou d'appeler une fonction pour envoyer la réponse. Le framework s'occupe de tout cela.

## Providers : un concept plus large

Jusqu'ici, nous avons parlé principalement des services, mais NestJS utilise un concept plus général : les providers. Un provider est tout simplement quelque chose qui peut être injecté.

Les services sont des providers, mais vous pouvez avoir d'autres types de providers. Par exemple, vous pourriez avoir un provider qui représente une configuration, ou une connexion à une base de données, ou même une simple valeur.

```typescript
@Module({
  providers: [
    UsersService,                     // Provider de type classe (notation raccourcie)
    {
      provide: 'API_KEY',               
      useValue: 'ma-cle-api-secrete', // Provider de type valeur
    },
  ],
})
export class UsersModule {}
```

Dans cet exemple, nous avons deux types de providers :
  - `UsersService` : un provider de type classe (notation raccourcie). Comme nous l'avons vu dans la section sur les contrôleurs, on l'injecte simplement en le déclarant dans le constructeur.
  - Un provider de type valeur constante : le token `'API_KEY'` dit "quand quelqu'un demande 'API_KEY', fournis cette valeur". Contrairement aux classes, ce type de provider nécessite le décorateur `@Inject()` pour l'injection.

Une fois déclaré dans les providers d'un module, ce token `'API_KEY'` devient disponible pour l'injection dans n'importe quel service de votre application. Voici comment l'utiliser (cet exemple est indépendant du module ci-dessus, il montre simplement la syntaxe d'injection) :

```typescript
@Injectable()
export class UsersService {
  constructor(@Inject('API_KEY') private apiKey: string) {
    // this.apiKey vaut 'ma-cle-api-secrete'
  }
}
```

Cette flexibilité dans le système d'injection de dépendances est très puissante. Elle permet de gérer des configurations, des dépendances conditionnelles, du mocking pour les tests, et bien plus encore.

## Récapitulation des concepts clés

Une application NestJS est organisée en **modules** qui regroupent des fonctionnalités liées. Chaque module peut contenir des **contrôleurs** qui gèrent les routes HTTP et des **services** qui contiennent la logique métier.

Les contrôleurs et les services communiquent via l'**injection de dépendances**, un mécanisme géré automatiquement par NestJS qui crée et fournit les instances nécessaires.\
Les **décorateurs** servent à déclarer les intentions : cette classe est un contrôleur, cette méthode répond aux requêtes GET, ce paramètre doit être extrait du corps de la requête, etc.

Cette **architecture en couches** avec injection de dépendances favorise la **séparation des responsabilités**, facilite les tests et rend le code plus maintenable. Chaque élément a un rôle clair et communique avec les autres de manière explicite et contrôlée.
