# Persistance de données avec TypeORM

## Le problème de notre stockage actuel

Jusqu'à présent, nos articles et utilisateurs sont stockés dans de simples tableaux donc stockés en mémoire vive. Cela présente bien entendu des limitations majeures.

Premièrement, les données disparaissent à chaque redémarrage de l'application. Arrêtez votre serveur et relancez-le : tous les articles créés ont disparu. C'est évidemment inacceptable pour une vraie application.

Deuxièmement, les performances se dégradent rapidement. Chercher un article par ID dans un tableau de 10 éléments est instantané. Mais avec 100 000 articles, parcourir tout le tableau à chaque requête devient problématique.

Troisièmement, les requêtes complexes deviennent difficiles. Comment trouver tous les articles publiés par un auteur donné, triés par date de création ? Avec un tableau en mémoire, vous devez écrire toute cette logique manuellement.

La solution est d'utiliser une vraie base de données. Et pour interagir avec cette base de données de manière élégante en TypeScript, nous allons utiliser un ORM.

## Qu'est-ce qu'un ORM ?

ORM signifie **Object-Relational Mapping**. C'est une technique qui permet de manipuler les données d'une base de données relationnelle comme si c'étaient des objets de votre langage de programmation.

Sans ORM, vous écrivez des requêtes SQL directement :

```typescript
const result = await db.query('SELECT * FROM articles WHERE id = $1', [articleId]);
const article = result.rows[0];
```

Avec un ORM, vous manipulez des objets :

```typescript
const article = await articleRepository.findOne({ where: { id: articleId } });
```

L'ORM se charge de traduire vos opérations en requêtes SQL appropriées. Il gère aussi la conversion entre les types de la base de données et les types TypeScript.

Les avantages d'un ORM sont nombreux : 
- Votre code devient plus lisible et plus proche de la logique métier. 
- Vous bénéficiez de l'autocomplétion et du typage TypeScript. 
- Vous êtes protégé contre les injections SQL. 
- Et vous pouvez changer de base de données plus facilement, car l'ORM abstrait les différences entre PostgreSQL, MySQL, SQLite, etc.

## TypeORM : l'ORM de référence pour NestJS

TypeORM est l'ORM le plus utilisé dans l'écosystème NestJS. Il s'intègre parfaitement avec TypeScript et utilise des décorateurs pour définir vos modèles de données, ce qui s'inscrit naturellement dans la philosophie de NestJS.

TypeORM supporte de nombreuses bases de données : PostgreSQL, MySQL, MariaDB, SQLite, Microsoft SQL Server, Oracle, et même MongoDB dans une certaine mesure.

Pour ce cours, nous utiliserons **SQLite**. Ce choix est délibéré : 
- SQLite stocke toute la base de données dans un seul fichier, ne nécessite aucune installation de serveur de base de données, et fonctionne immédiatement. 
- C'est parfait pour l'apprentissage et le développement. 

En production, vous migreriez probablement vers PostgreSQL ou MySQL, mais le code TypeORM resterait quasiment identique.

## Installer les dépendances

Commençons par installer TypeORM et les packages nécessaires :

```bash
npm install @nestjs/typeorm typeorm sqlite3
```

- `@nestjs/typeorm` : intégration officielle de TypeORM avec NestJS
- `typeorm` : la bibliothèque ORM elle-même
- `sqlite3` : le driver SQLite pour Node.js


## Configurer TypeORM dans NestJS

### Configuration de la connexion

Ouvrez `src/app.module.ts` et configurons la connexion à la base de données :

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ArticlesModule } from './articles/articles.module';
import { UsersModule } from './users/users.module';
import { AuthModule } from './auth/auth.module';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'sqlite',
      database: 'database.sqlite',
      entities: [__dirname + '/**/*.entity{.ts,.js}'],
      synchronize: true,
    }),
    ArticlesModule,
    UsersModule,
    AuthModule,
  ],
})
export class AppModule {}
```


### Analysons cette configuration

NestJS utilise une convention pour la configuration des modules dynamiques : `forRoot()` et `forFeature()`. 

La méthode **`forRoot()`** configure le module une seule fois au niveau global de l'application, généralement dans `AppModule`. C'est ici que vous définissez la connexion à la base de données : type de base, credentials, options de synchronisation, etc. Cette configuration est ensuite partagée avec toute l'application. 

À l'inverse, **`forFeature()`** (utilisée plus bas dans ce cours) est une méthode présente dans les modules fonctionnels pour déclarer quelles entités ce module va utiliser. Elle crée les repositories correspondants et les rend disponibles pour l'injection de dépendances dans ce module spécifique. 

Cette séparation suit le principe de responsabilité unique : `forRoot()` gère "comment se connecter à la base", tandis que `forFeature()` gère "quelles tables ce module utilise".

Regardons maintenant les paramètres de configuration :

- Le paramètre `type: 'sqlite'` indique le type de base de données. Pour PostgreSQL, vous utiliseriez `'postgres'`.

- Le paramètre `database: 'database.sqlite'` spécifie le fichier où SQLite stockera les données. Ce fichier sera créé automatiquement à la racine de votre projet.

- Le paramètre `entities` indique à TypeORM où trouver vos entités (vos modèles de données). Le pattern `[__dirname + '/**/*.entity{.ts,.js}']` signifie "tous les fichiers se terminant par `.entity.ts` ou `.entity.js` dans n'importe quel sous-dossier".

- Le paramètre `synchronize: true` est très important à comprendre. Quand il est activé, TypeORM synchronise automatiquement le schéma de la base de données avec vos entités à chaque démarrage de l'application. Si vous ajoutez un champ à une entité, TypeORM ajoutera automatiquement la colonne correspondante dans la table.

⚠️ **ATTENTION** : L'option `synchronize: true` est pratique pour le développement, mais ne doit ❌ **JAMAIS**  être utilisée en production. Elle pourrait supprimer des données ou modifier le schéma de manière inattendue. En production, utilisez des migrations TypeORM pour gérer les évolutions de schéma de manière contrôlée.

## Créer notre première entité

Une entité TypeORM représente une table dans la base de données. Chaque instance de l'entité correspond à une ligne de cette table. Transformons notre interface `Article` en entité TypeORM.

Créez le fichier `src/articles/entities/article.entity.ts` :

```typescript
import {
  Entity,
  Column,
  PrimaryGeneratedColumn,
  CreateDateColumn,
  UpdateDateColumn,
} from 'typeorm';

@Entity('articles')
export class ArticleEntity {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ length: 100 })
  title: string;

  @Column('text')
  content: string;

  @Column({ length: 100 })
  author: string;

  @Column({ default: false })
  published: boolean;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

Décomposons chaque décorateur :

- `@Entity('articles')` marque cette classe comme une entité TypeORM. Le paramètre `'articles'` est le nom de la table dans la base de données. Si vous l'omettez, TypeORM utilisera le nom de la classe en minuscules.

- `@PrimaryGeneratedColumn('uuid')` définit la colonne `id` comme clé primaire auto-générée. L'option `'uuid'` génère des identifiants UUID au lieu de nombres auto-incrémentés. Les UUID sont préférables car ils sont uniques globalement et ne révèlent pas d'informations sur le nombre d'enregistrements.

- `@Column({ length: 100 })` définit une colonne texte avec une longueur maximale de 100 caractères. Pour les textes longs comme le contenu d'un article, on utilise `@Column('text')` qui n'a pas de limite de taille.

- `@Column({ default: false })` définit une valeur par défaut. Si vous créez un article sans spécifier `published`, il sera automatiquement `false`.

- `@CreateDateColumn()` et `@UpdateDateColumn()` sont des décorateurs spéciaux de TypeORM. Ils gèrent automatiquement les timestamps : `createdAt` est défini lors de la création, `updatedAt` est mis à jour à chaque modification.

Remarquez que nous n'avons plus besoin de l'interface `Article` séparée. L'entité elle-même définit la structure de nos articles.

## Configurer le module Articles pour utiliser TypeORM

Maintenant que nous avons notre entité, configurons le module Articles pour l'utiliser. Ouvrez `src/articles/articles.module.ts` :

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ArticlesController } from './articles.controller';
import { ArticlesService } from './articles.service';
import { ArticleEntity } from './entities/article.entity';

@Module({
  imports: [TypeOrmModule.forFeature([ArticleEntity])],
  controllers: [ArticlesController],
  providers: [ArticlesService],
})
export class ArticlesModule {}
```

La ligne `TypeOrmModule.forFeature([ArticleEntity])` est cruciale. Elle indique à NestJS de créer un **repository** pour l'entité `ArticleEntity` et de le rendre disponible pour l'injection de dépendances dans ce module.

Un repository est un objet fourni par TypeORM qui offre des méthodes pour interagir avec la table correspondante : `find`, `findOne`, `save`, `delete`, etc. C'est lui qui traduit vos opérations en requêtes SQL.

## Réécrire le service avec TypeORM

Voici la partie la plus intéressante : réécrivons notre `ArticlesService` pour utiliser TypeORM au lieu du tableau en mémoire. Ouvrez `src/articles/articles.service.ts` :

```typescript
import { Injectable, NotFoundException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { ArticleEntity } from './entities/article.entity';
import { CreateArticleDto } from './dto/create-article.dto';
import { UpdateArticleDto } from './dto/update-article.dto';

@Injectable()
export class ArticlesService {
  constructor(
    @InjectRepository(ArticleEntity)
    private articlesRepository: Repository<ArticleEntity>,
  ) {}

  async findAll(): Promise<ArticleEntity[]> {
    return this.articlesRepository.find();
  }

  async findOne(id: string): Promise<ArticleEntity> {
    const article = await this.articlesRepository.findOne({ where: { id } });
    if (!article) {
      throw new NotFoundException(`Article with ID ${id} not found`);
    }
    return article;
  }

  async create(createArticleDto: CreateArticleDto): Promise<ArticleEntity> {
    const article = this.articlesRepository.create(createArticleDto);
    return this.articlesRepository.save(article);
  }

  async update(id: string, updateArticleDto: UpdateArticleDto): Promise<ArticleEntity> {
    const article = await this.findOne(id);

    Object.assign(article, updateArticleDto);

    return this.articlesRepository.save(article);
  }

  async remove(id: string): Promise<void> {
    const article = await this.findOne(id);
    await this.articlesRepository.remove(article);
  }
}
```

Analysons les changements importants :

### L'injection du repository

```typescript
constructor(
  @InjectRepository(ArticleEntity)
  private articlesRepository: Repository<ArticleEntity>,
) {}
```

Le décorateur `@InjectRepository(ArticleEntity)` injecte le repository de l'entité `ArticleEntity`. C'est ce repository qui nous permet d'interagir avec la base de données. 

Le type `Repository<ArticleEntity>` est générique et typé, ce qui vous donne l'autocomplétion complète sur les méthodes disponibles.

### Les méthodes deviennent asynchrones

Toutes les opérations de base de données sont asynchrones par nature (elles nécessitent des appels réseau ou des lectures disque). C'est pourquoi nos méthodes retournent maintenant des `Promise<ArticleEntity>` ou `Promise<ArticleEntity[]>`, et nous utilisons `async/await`.

### La méthode find

```typescript
async findAll(): Promise<ArticleEntity[]> {
  return this.articlesRepository.find();
}
```

`find()` sans paramètre retourne tous les enregistrements de la table. TypeORM traduit cela en `SELECT * FROM articles`.

### La méthode findOne

```typescript
async findOne(id: string): Promise<ArticleEntity> {
  const articleEntity = await this.articlesRepository.findOne({ where: { id } });
  if (!article) {
    throw new NotFoundException(`Article with ID ${id} not found`);
  }
  return articleEntity;
}
```

`findOne({ where: { id } })` recherche un article par son ID. Si aucun article n'est trouvé, `findOne` retourne `null`, et nous lançons une `NotFoundException`.

### La création avec create et save

```typescript
async create(createArticleDto: CreateArticleDto): Promise<ArticleEntity> {
  const articleEntity = this.articlesRepository.create(createArticleDto);
  return this.articlesRepository.save(articleEntity);
}
```

La distinction entre `create` et `save` est importante :

- `create()` crée une instance de l'entité en mémoire à partir des données fournies, mais ne l'enregistre pas en base de données
- `save()` persiste l'entité en base de données

Pourquoi deux étapes ? Parce que `create()` vous permet de manipuler l'entité avant de la sauvegarder si nécessaire. Par exemple, vous pourriez vouloir ajouter des valeurs calculées ou valider des choses supplémentaires.

### La mise à jour

```typescript
async update(id: string, updateArticleDto: UpdateArticleDto): Promise<ArticleEntity> {
  const articleEntity = await this.findOne(id);

  Object.assign(articleEntity, updateArticleDto);

  return this.articlesRepository.save(articleEntity);
}
```

Pour mettre à jour, nous :
1. Récupérons l'article existant (ce qui vérifie aussi qu'il existe)
2. Fusionnons les nouvelles données avec `Object.assign`
3. Sauvegardons l'article modifié

TypeORM est assez intelligent pour faire un `UPDATE` plutôt qu'un `INSERT` quand l'entité a déjà un ID.

### La suppression

```typescript
async remove(id: string): Promise<void> {
  const articleEntity = await this.findOne(id);
  await this.articlesRepository.remove(articleEntity);
}
```

`remove()` supprime l'entité de la base de données. Nous vérifions d'abord que l'article existe pour retourner une erreur 404 appropriée si ce n'est pas le cas.

## Mettre à jour le contrôleur

Le contrôleur doit être mis à jour pour gérer les méthodes asynchrones. Ouvrez `src/articles/articles.controller.ts` :

```typescript
import {
  Body,
  Controller,
  Delete,
  Get,
  HttpCode,
  HttpStatus,
  Param,
  Post,
  Put,
  UseGuards,
} from '@nestjs/common';
import { ArticlesService } from './articles.service';
import { CreateArticleDto } from './dto/create-article.dto';
import { UpdateArticleDto } from './dto/update-article.dto';
import { ArticleEntity } from './entities/article.entity';
import { JwtAuthGuard } from '../auth/guards/jwt-auth.guard';

@Controller('articles')
export class ArticlesController {
  constructor(private readonly articlesService: ArticlesService) {}

  @Get()
  async findAll(): Promise<ArticleEntity[]> {
    return this.articlesService.findAll();
  }

  @Get(':id')
  async findOne(@Param('id') id: string): Promise<ArticleEntity> {
    return this.articlesService.findOne(id);
  }

  @UseGuards(JwtAuthGuard)
  @Post()
  @HttpCode(HttpStatus.CREATED)
  async create(@Body() createArticleDto: CreateArticleDto): Promise<ArticleEntity> {
    return this.articlesService.create(createArticleDto);
  }

  @UseGuards(JwtAuthGuard)
  @Put(':id')
  async update(
    @Param('id') id: string,
    @Body() updateArticleDto: UpdateArticleDto,
  ): Promise<ArticleEntity> {
    return this.articlesService.update(id, updateArticleDto);
  }

  @UseGuards(JwtAuthGuard)
  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  async remove(@Param('id') id: string): Promise<void> {
    return this.articlesService.remove(id);
  }
}
```

> 💡 **Note importante** : Dans cet exemple, nous retournons directement les entités TypeORM depuis le contrôleur. C'est acceptable pour une introduction, mais en production, il est recommandé d'utiliser des **Response DTOs** (Data Transfer Objects) pour contrôler précisément les données exposées par l'API. Cela évite notamment d'exposer accidentellement des champs sensibles (comme le mot de passe d'un utilisateur) et découple la structure de votre base de données de votre contrat API.
>
> Voici un exemple avec `findOne` :
>
> ```typescript
> // dto/article-response.dto.ts
> export class ArticleResponseDto {
>   id: string;
>   title: string;
>   content: string;
>   published: boolean;
>   createdAt: Date;
> }
> ```
>
> ```typescript
> // Dans le contrôleur
> @Get(':id')
> async findOne(@Param('id') id: string): Promise<ArticleResponseDto> {
>   const entity = await this.articlesService.findOne(id);
>   return {
>     id: entity.id,
>     title: entity.title,
>     content: entity.content,
>     published: entity.published,
>     createdAt: entity.createdAt,
>   };
> }
> ```

## Requêtes avancées avec TypeORM

TypeORM offre des possibilités de requêtes bien plus avancées que ce que nous avons vu. Voici quelques exemples utiles.

### Filtrer les résultats

Pour récupérer uniquement les articles publiés :

```typescript
// Dans le service
async findPublished(): Promise<ArticleEntity[]> {
  return this.articlesRepository.find({
    where: { published: true },
  });
}
```

### Trier les résultats

Pour trier par date de création, du plus récent au plus ancien :

```typescript
// Dans le service
async findAll(): Promise<ArticleEntity[]> {
  return this.articlesRepository.find({
    order: { createdAt: 'DESC' },
  });
}
```

### Combiner plusieurs conditions

Pour des requêtes plus complexes :

```typescript
// Dans le service
async findByAuthorPublished(author: string): Promise<ArticleEntity[]> {
  return this.articlesRepository.find({
    where: {
      author: author,
      published: true,
    },
    order: { createdAt: 'DESC' },
  });
}
```

### Utiliser le QueryBuilder

Pour des requêtes encore plus complexes, TypeORM propose un QueryBuilder qui vous donne un contrôle total :

```typescript
// Dans le service
async searchArticles(searchTerm: string): Promise<ArticleEntity[]> {
  return this.articlesRepository
    .createQueryBuilder('article')
    .where('article.title LIKE :search', { search: `%${searchTerm}%` })
    .orWhere('article.content LIKE :search', { search: `%${searchTerm}%` })
    .andWhere('article.published = :published', { published: true })
    .orderBy('article.createdAt', 'DESC')
    .getMany();
}
```

Le QueryBuilder construit une requête SQL pièce par pièce. C'est plus verbeux mais offre une flexibilité maximale.

## Créer l'entité UserEntity

Appliquons les mêmes principes à notre module Users. Créez `src/users/entities/user.entity.ts` :

```typescript
import {
  Entity,
  Column,
  PrimaryGeneratedColumn,
  CreateDateColumn,
} from 'typeorm';

@Entity('users')
export class UserEntity {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ unique: true })
  email: string;

  @Column()
  password: string;

  @Column({ length: 100 })
  name: string;

  @Column({ default: 'user' })
  role: string;

  @CreateDateColumn()
  createdAt: Date;
}
```

Le décorateur `@Column({ unique: true })` sur l'email garantit qu'aucun doublon ne sera accepté au niveau de la base de données. C'est une sécurité supplémentaire en plus de la vérification dans le service.

Mettez à jour `src/users/users.module.ts` :

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UsersService } from './users.service';
import { UserEntity } from './entities/user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([UserEntity])],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

Et réécrivez `src/users/users.service.ts` :

```typescript
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { UserEntity } from './entities/user.entity';
import * as bcrypt from 'bcrypt';

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(UserEntity)
    private usersRepository: Repository<UserEntity>,
  ) {}

  async create(email: string, password: string, name: string): Promise<UserEntity> {
    const hashedPassword = await bcrypt.hash(password, 10);

    const userEntity = this.usersRepository.create({
      email,
      password: hashedPassword,
      name,
    });

    return this.usersRepository.save(userEntity);
  }

  async findByEmail(email: string): Promise<UserEntity | null> {
    return this.usersRepository.findOne({ where: { email } });
  }

  async findById(id: string): Promise<UserEntity | null> {
    return this.usersRepository.findOne({ where: { id } });
  }
}
```

## Les relations entre entités

Dans une vraie application, les données sont rarement isolées. Les articles ont des auteurs, les utilisateurs ont des articles, etc. TypeORM gère ces relations de manière élégante.

### Relation One-to-Many / Many-to-One

Imaginons que nous voulons lier les articles à leurs auteurs utilisateurs. Modifions nos entités.

Dans `src/users/entities/user.entity.ts`, ajoutez la relation :

```typescript
import {
  Entity,
  Column,
  PrimaryGeneratedColumn,
  CreateDateColumn,
  OneToMany,
} from 'typeorm';
import { ArticleEntity } from '../../articles/entities/article.entity';

@Entity('users')
export class UserEntity {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ unique: true })
  email: string;

  @Column()
  password: string;

  @Column({ length: 100 })
  name: string;

  @Column({ default: 'user' })
  role: string;

  @CreateDateColumn()
  createdAt: Date;

  @OneToMany(() => ArticleEntity, (article) => article.user)
  articles: ArticleEntity[];
}
```

Et dans `src/articles/entities/article.entity.ts` :

```typescript
import {
  Entity,
  Column,
  PrimaryGeneratedColumn,
  CreateDateColumn,
  UpdateDateColumn,
  ManyToOne,
  JoinColumn,
} from 'typeorm';
import { UserEntity } from '../../users/entities/user.entity';

@Entity('articles')
export class ArticleEntity {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ length: 100 })
  title: string;

  @Column('text')
  content: string;

  @Column({ default: false })
  published: boolean;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;

  @ManyToOne(() => UserEntity, (user) => user.articles)
  @JoinColumn({ name: 'userId' })
  user: UserEntity;

  @Column()
  userId: string;
}
```

Quelques explications :

- `@OneToMany(() => ArticleEntity, (article) => article.user)` définit qu'un utilisateur peut avoir plusieurs articles. Le premier argument est une fonction qui retourne l'entité liée (pour éviter les problèmes de dépendances circulaires). Le second argument définit le côté inverse de la relation.

- `@ManyToOne(() => UserEntity, (user) => user.articles)` définit que plusieurs articles peuvent appartenir à un utilisateur.

- `@JoinColumn({ name: 'userId' })` spécifie le nom de la colonne qui stockera la clé étrangère.

Nous ajoutons aussi `@Column() userId: string;` pour pouvoir accéder directement à l'ID de l'utilisateur sans charger toute la relation.

### Charger les relations

Par défaut, TypeORM ne charge pas automatiquement les relations. Pour les charger, utilisez l'option `relations` :

```typescript
// Dans le service
async findOneWithAuthor(id: string): Promise<ArticleEntity> {
  const articleEntity = await this.articlesRepository.findOne({
    where: { id },
    relations: ['user'],
  });

  if (!articleEntity) {
    throw new NotFoundException(`Article with ID ${id} not found`);
  }

  return articleEntity;
}
```

Cela générera une requête avec un `JOIN` pour récupérer l'article et son auteur en une seule requête.

### Créer un article avec un auteur

Modifions la création d'article pour associer l'utilisateur connecté :

```typescript
// Dans le service
async create(createArticleDto: CreateArticleDto, userId: string): Promise<ArticleEntity> {
  const articleEntity = this.articlesRepository.create({
    ...createArticleDto,
    userId: userId,
  });

  return this.articlesRepository.save(articleEntity);
}
```

Et dans le contrôleur :

```typescript
// Dans le contrôleur
@UseGuards(JwtAuthGuard)
@Post()
@HttpCode(HttpStatus.CREATED)
async create(
  @Body() createArticleDto: CreateArticleDto,
  @Request() req,
): Promise<ArticleEntity> {
  return this.articlesService.create(createArticleDto, req.user.userId);
}
```

## Supprimer le champ author des DTOs

Maintenant que l'auteur est lié via une relation, nous pouvons simplifier notre DTO de création. Modifiez `src/articles/dto/create-article.dto.ts` :

```typescript
import { IsString, IsBoolean, IsOptional, MinLength, MaxLength, IsNotEmpty } from 'class-validator';

export class CreateArticleDto {
  @IsString()
  @IsNotEmpty()
  @MinLength(5, { message: 'Le titre doit contenir au moins 5 caractères' })
  @MaxLength(100, { message: 'Le titre ne peut pas dépasser 100 caractères' })
  title: string;

  @IsString()
  @IsNotEmpty()
  @MinLength(50, { message: 'Le contenu doit contenir au moins 50 caractères' })
  content: string;

  @IsBoolean()
  @IsOptional()
  published?: boolean;
}
```

Le champ `author` a disparu : l'auteur sera automatiquement l'utilisateur connecté.

## Tester votre API avec persistance

Redémarrez votre application. Un fichier `database.sqlite` devrait apparaître à la racine de votre projet.

Créez d'abord un utilisateur via `POST /auth/register`, puis utilisez le token obtenu pour créer des articles. Cette fois, arrêtez et redémarrez votre serveur : vos données sont toujours là. C'est la magie de la persistance.

Vous pouvez examiner le contenu de votre base SQLite avec des outils comme DB Browser for SQLite ou l'extension VSCode SQLite Viewer.

## Bonnes pratiques avec TypeORM

### Évitez synchronize en production

Nous l'avons déjà mentionné, mais c'est assez important pour le répéter : n'utilisez jamais `synchronize: true` en production. Utilisez les migrations TypeORM pour contrôler les évolutions de schéma :

```bash
# Générer une migration basée sur les changements d'entités
npm run typeorm migration:generate -- -n NomDeLaMigration

# Exécuter les migrations
npm run typeorm migration:run
```

### Utilisez les transactions pour les opérations complexes

Si vous devez effectuer plusieurs opérations qui doivent réussir ou échouer ensemble, utilisez les transactions :

```typescript
// Dans le service
async createArticleWithTags(data: CreateArticleWithTagsDto): Promise<ArticleEntity> {
  return this.articlesRepository.manager.transaction(async (manager) => {
    const articleEntity = manager.create(ArticleEntity, data.article);
    await manager.save(articleEntity);

    // Autres opérations qui doivent être atomiques

    return articleEntity;
  });
}
```

### Ne chargez que ce dont vous avez besoin

Les relations peuvent être coûteuses en performance. Ne les chargez que si vous en avez vraiment besoin. Utilisez `select` pour ne récupérer que les champs nécessaires.

### Indexez vos colonnes de recherche

Si vous recherchez souvent par un champ particulier, ajoutez un index :

```typescript
// Dans l'entité
@Column()
@Index()
author: string;
```

Cela accélère considérablement les recherches sur ce champ.

## Récapitulation

La persistance de données avec TypeORM transforme votre application d'un prototype éphémère en une vraie application prête pour la production.

Les **entités** définissent la structure de vos tables grâce aux décorateurs TypeORM. Les **repositories** fournissent les méthodes CRUD de base pour interagir avec la base de données. Les **relations** permettent de modéliser les liens entre vos données.

Le flux de données devient : **Requête HTTP** -> **Contrôleur** -> **Service** -> **Repository** -> **Base de données**, puis le chemin inverse pour la réponse.

TypeORM s'intègre parfaitement avec NestJS grâce au module `@nestjs/typeorm`. L'injection de dépendances fonctionne naturellement : vous déclarez avoir besoin d'un repository dans le constructeur de votre service, et NestJS vous le fournit.

Cette architecture en couches, combinée à la persistance, vous donne une base solide pour construire des applications robustes et évolutives.
