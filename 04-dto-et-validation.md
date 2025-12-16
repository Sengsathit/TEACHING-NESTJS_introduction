# Les DTO et la validation : structurer vos données d'entrée

## Le problème avec notre code actuel

Dans le chapitre précédent, nous avons créé une API fonctionnelle, mais si vous regardez attentivement le code du contrôleur, vous remarquerez quelque chose d'inconfortable. Nous utilisons `Partial<Article>` ou même `any` pour typer les données entrantes. Cela pose plusieurs problèmes.

- **Premier problème** : la sécurité. Imaginez qu'un utilisateur malveillant envoie une requête de création d'article avec un champ `id` personnalisé, ou pire, avec des champs supplémentaires qui pourraient perturber votre application. Sans validation stricte, ces données arrivent directement dans votre service.

- **Deuxième problème** : la documentation. Quand un autre développeur (ou vous-même dans six mois) regarde votre API, comment savoir exactement quels champs sont requis pour créer un article ? Quels sont les formats attendus ? Les contraintes de longueur ? Rien dans le code actuel ne le documente clairement.

- **Troisième problème** : la validation. Dans notre service, nous avons ajouté une validation manuelle pour vérifier que les champs obligatoires sont présents. Mais qu'en est-il du format de l'email de l'auteur ? De la longueur minimale du titre ? De la structure de données complexes ? Écrire toutes ces validations à la main devient vite fastidieux et source d'erreurs.

C'est exactement pour résoudre ces problèmes que NestJS encourage fortement l'utilisation des DTOs.

## Qu'est-ce qu'un DTO ?

DTO signifie Data Transfer Object, littéralement "objet de transfert de données". C'est un objet qui définit exactement la structure des données qui voyagent entre le client et le serveur. Un DTO n'a pas de logique métier, pas de méthodes complexes. C'est une simple définition de structure avec des règles de validation.

La philosophie derrière les DTOs est simple : au lieu de laisser n'importe quelles données entrer dans votre application et de les valider manuellement partout, vous définissez un contrat strict à l'entrée. Toute donnée qui ne respecte pas ce contrat est rejetée immédiatement, avant même d'atteindre votre logique métier.

Dans NestJS, un DTO est typiquement une classe TypeScript avec des décorateurs de validation. Ces décorateurs décrivent les contraintes que chaque champ doit respecter.

## Créer notre premier DTO

Commençons par créer un DTO pour la création d'articles. Créez un dossier `dto` dans votre dossier `articles`, puis un fichier `create-article.dto.ts` :

```
src/articles/
  ├── articles.module.ts
  ├── articles.controller.ts
  ├── articles.service.ts
  └── dto/
      └── create-article.dto.ts
```

```typescript
export class CreateArticleDto {
  title: string;
  content: string;
  author: string;
  published?: boolean;
}
```

C'est un bon début. Nous avons défini la structure exacte des données attendues pour créer un article. Le champ `published` est optionnel (marqué par `?`), les autres sont obligatoires. C'est déjà plus clair que `Partial<Article>`.

Mettons maintenant à jour notre contrôleur pour utiliser ce DTO :

```typescript
import { CreateArticleDto } from './dto/create-article.dto';

@Controller('articles')
export class ArticlesController {
  // ...

  @Post()
  @HttpCode(HttpStatus.CREATED)
  create(@Body() createArticleDto: CreateArticleDto): Article {
    return this.articlesService.create(createArticleDto);
  }
}
```

Et adaptons le service :

```typescript
create(createArticleDto: CreateArticleDto): Article {
  const newArticle: Article = {
    id: String(Date.now()),
    title: createArticleDto.title,
    content: createArticleDto.content,
    author: createArticleDto.author,
    published: createArticleDto.published ?? false,
    createdAt: new Date(),
    updatedAt: new Date(),
  };
  this.articles.push(newArticle);
  return newArticle;
}
```

Notre code est maintenant plus lisible. Mais nous n'avons pas encore de validation automatique. Si un client envoie un objet vide ou avec des champs manquants, l'application acceptera la requête. Passons à la validation.

## Ajouter la validation automatique

NestJS s'intègre parfaitement avec la bibliothèque `class-validator`, qui fournit des décorateurs de validation très puissants. Installons les dépendances nécessaires :

```bash
npm install class-validator class-transformer
```

`class-validator` fournit les décorateurs de validation, et `class-transformer` permet de transformer les objets JavaScript simples en instances de classes, ce qui est nécessaire pour que la validation fonctionne.

Maintenant, activons la validation globale dans votre application. Ouvrez `main.ts` et ajoutez :

```typescript
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Active la validation globale
  app.useGlobalPipes(new ValidationPipe({
    whitelist: true,            // Supprime les propriétés non définies dans le DTO
    forbidNonWhitelisted: true, // Rejette la requête si des propriétés non définies sont présentes
    transform: true,            // Transforme automatiquement les types
  }));

  await app.listen(3000);
}
bootstrap();
```

Ce `ValidationPipe` global active la validation sur tous les endpoints de votre application. Les trois options que nous avons configurées sont importantes :

L'option `whitelist: true` supprime automatiquement toutes les propriétés qui ne sont pas définies dans le DTO. Si un client envoie `{ title: "Test", maliciousField: "hack" }`, seul `title` sera conservé.

L'option `forbidNonWhitelisted: true` va plus loin : au lieu de simplement ignorer les propriétés non définies, elle rejette complètement la requête avec une erreur 400. C'est une protection supplémentaire contre les tentatives d'injection de données non prévues.

L'option `transform: true` convertit automatiquement les types. Par exemple, si vous attendez un nombre et que le client envoie la chaîne "42", elle sera automatiquement convertie en nombre 42. C'est particulièrement utile pour les query parameters qui arrivent toujours comme des chaînes.

## Enrichir notre DTO avec des validations

Maintenant que la validation est activée, enrichissons notre DTO avec des contraintes précises :

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

  @IsString()
  @IsNotEmpty()
  @MinLength(2, { message: "Le nom de l'auteur doit contenir au moins 2 caractères" })
  author: string;

  @IsBoolean()
  @IsOptional()
  published?: boolean;
}
```

Chaque décorateur ajoute une contrainte de validation. Passons en revue les principaux :

`@IsString()` vérifie que la valeur est une chaîne de caractères. Si le client envoie un nombre ou un objet, la validation échoue.

`@IsNotEmpty()` vérifie que la valeur n'est pas vide (pas une chaîne vide, pas `null`, pas `undefined`).

`@MinLength()` et `@MaxLength()` vérifient la longueur des chaînes. Vous pouvez personnaliser le message d'erreur avec l'option `message`.

`@IsBoolean()` vérifie que la valeur est un booléen (`true` ou `false`).

`@IsOptional()` indique que le champ n'est pas obligatoire. Si le client ne l'envoie pas, aucune erreur. Mais s'il l'envoie, les autres validations s'appliquent.

Testez maintenant votre API. Essayez d'envoyer une requête POST avec un titre trop court :

```json
{
  "title": "Test",
  "content": "Contenu suffisamment long pour passer la validation...",
  "author": "Marie"
}
```

Vous recevrez une réponse d'erreur 400 avec un message détaillé :

```json
{
  "statusCode": 400,
  "message": [
    "Le titre doit contenir au moins 5 caractères"
  ],
  "error": "Bad Request"
}
```

NestJS a automatiquement intercepté la requête, exécuté les validations, détecté l'erreur, et généré une réponse claire. Vous n'avez écrit aucun code de gestion d'erreur, aucune condition `if`, aucun bloc try-catch. La validation est déclarative.

## Créer un DTO pour les mises à jour

Pour modifier un article, nous avons besoin d'un autre DTO. La différence avec la création est que tous les champs deviennent optionnels : on peut vouloir modifier uniquement le titre, ou uniquement le contenu.

Créez `update-article.dto.ts` :

```
src/articles/
  ├── articles.module.ts
  ├── articles.controller.ts
  ├── articles.service.ts
  └── dto/
      ├── create-article.dto.ts
      └── update-article.dto.ts
```

```typescript
import { IsString, IsBoolean, IsOptional, MinLength, MaxLength } from 'class-validator';

export class UpdateArticleDto {
  @IsString()
  @IsOptional()
  @MinLength(5)
  @MaxLength(100)
  title?: string;

  @IsString()
  @IsOptional()
  @MinLength(50)
  content?: string;

  @IsString()
  @IsOptional()
  @MinLength(2)
  author?: string;

  @IsBoolean()
  @IsOptional()
  published?: boolean;
}
```

Tous les champs sont maintenant optionnels, mais s'ils sont présents, ils doivent respecter les mêmes contraintes de validation que lors de la création.

NestJS fournit une fonction utilitaire qui peut générer automatiquement un DTO de mise à jour à partir du DTO de création :

```typescript
import { PartialType } from '@nestjs/mapped-types';
import { CreateArticleDto } from './create-article.dto';

export class UpdateArticleDto extends PartialType(CreateArticleDto) {}
```

Cette approche a un avantage majeur : si vous modifiez les validations de `CreateArticleDto`, `UpdateArticleDto` hérite automatiquement des changements. Vous évitez la duplication de code et les incohérences.

Notez que vous devrez installer `@nestjs/mapped-types` :

```bash
npm install @nestjs/mapped-types
```

Mettez à jour votre contrôleur :

```typescript
import { UpdateArticleDto } from './dto/update-article.dto';

@Put(':id')
update(
  @Param('id') id: string,
  @Body() updateArticleDto: UpdateArticleDto,
): Article {
  return this.articlesService.update(id, updateArticleDto);
}
```

Et le service :

```typescript
update(id: string, updateArticleDto: UpdateArticleDto): Article {
  const articleIndex = this.articles.findIndex(a => a.id === id);
  if (articleIndex === -1) {
    throw new NotFoundException(`Article with ID ${id} not found`);
  }

  const updatedArticle = {
    ...this.articles[articleIndex],
    ...updateArticleDto,
    id: this.articles[articleIndex].id,
    updatedAt: new Date(),
  };

  this.articles[articleIndex] = updatedArticle;
  return updatedArticle;
}
```

## Validations plus avancées

`class-validator` offre des dizaines de décorateurs pour tous les cas d'usage courants. Voici quelques exemples utiles :

Pour valider une adresse email :

```typescript
import { IsEmail } from 'class-validator';

@IsEmail({}, { message: 'Veuillez fournir une adresse email valide' })
email: string;
```

Pour valider une URL :

```typescript
import { IsUrl } from 'class-validator';

@IsUrl()
website: string;
```

Pour valider qu'une valeur fait partie d'un ensemble prédéfini :

```typescript
import { IsIn } from 'class-validator';

@IsIn(['draft', 'published', 'archived'])
status: string;
```

Pour valider un nombre dans une plage :

```typescript
import { IsNumber, Min, Max } from 'class-validator';

@IsNumber()
@Min(0)
@Max(5)
rating: number;
```

Pour valider un tableau d'éléments :

```typescript
import { IsArray, ArrayMinSize, ArrayMaxSize } from 'class-validator';

@IsArray()
@ArrayMinSize(1)
@ArrayMaxSize(10)
@IsString({ each: true })  // Chaque élément du tableau doit être une string
tags: string[];
```

Pour valider des objets imbriqués :

```typescript
import { ValidateNested, IsObject } from 'class-validator';
import { Type } from 'class-transformer';

class AddressDto {
  @IsString()
  street: string;

  @IsString()
  city: string;
}

class UserDto {
  @IsString()
  name: string;

  @ValidateNested()
  @Type(() => AddressDto)
  address: AddressDto;
}
```

## Validations personnalisées

Parfois, les validations standards ne suffisent pas. Vous pouvez créer vos propres validateurs. Imaginons que nous voulions vérifier qu'un article ne contient pas de mots interdits :

```typescript
import { registerDecorator, ValidationOptions, ValidationArguments } from 'class-validator';

function DoesNotContainForbiddenWords(validationOptions?: ValidationOptions) {
  return function (object: Object, propertyName: string) {
    registerDecorator({
      name: 'doesNotContainForbiddenWords',
      target: object.constructor,
      propertyName: propertyName,
      options: validationOptions,
      validator: {
        validate(value: any, args: ValidationArguments) {
          if (typeof value !== 'string') return false;
          const forbiddenWords = ['spam', 'interdit', 'illégal'];
          return !forbiddenWords.some(word =>
            value.toLowerCase().includes(word)
          );
        },
        defaultMessage(args: ValidationArguments) {
          return 'Le texte contient des mots interdits';
        },
      },
    });
  };
}

// Utilisation dans votre DTO
export class CreateArticleDto {
  @IsString()
  @IsNotEmpty()
  @DoesNotContainForbiddenWords()
  title: string;

  // ... autres champs
}
```

Cette approche est puissante mais doit être utilisée avec parcimonie. Pour la plupart des cas, les validateurs standards suffisent.

## Les DTOs comme documentation vivante

Au-delà de la validation, les DTOs servent de documentation vivante pour votre API. Un développeur qui regarde `CreateArticleDto` comprend immédiatement ce qu'il faut envoyer pour créer un article. Plus besoin de chercher dans la documentation ou de deviner.

Cette documentation est même exploitable par des outils automatiques. Si vous utilisez Swagger pour générer une documentation d'API (ce que NestJS supporte très bien), les informations des DTOs sont automatiquement transformées en spécifications OpenAPI complètes.

## Séparer les DTOs des entités métier

⚠️ POINT IMPORTANT : **un DTO n'est pas la même chose qu'une entité métier**. Dans notre exemple, nous avons une interface `Article` qui représente l'entité métier, et des DTOs `CreateArticleDto` et `UpdateArticleDto` qui représentent les données d'entrée.

Cette séparation est volontaire. L'interface `Article` inclut des champs générés automatiquement comme `id`, `createdAt`, et `updatedAt`. Ces champs ne doivent jamais être acceptés en entrée depuis le client. En utilisant des DTOs distincts, vous contrôlez précisément ce qui peut entrer dans votre système.

De même, quand vous renvoyez des données au client, vous pourriez vouloir créer des DTOs de sortie qui excluent certains champs sensibles ou ajoutent des champs calculés. Bien que ce soit moins courant pour des API REST simples, c'est une pratique recommandée pour des applications complexes.

## Pourquoi NestJS insiste sur les DTOs

Vous pourriez vous demander pourquoi NestJS insiste autant sur les DTOs, même dans la documentation officielle. La raison est simple : c'est une bonne pratique professionnelle qui évite de nombreux problèmes.

Sans DTOs, votre validation est dispersée dans votre code métier. Chaque service doit vérifier les données qu'il reçoit. Cela crée de la duplication, des incohérences, et des failles de sécurité potentielles.

Avec DTOs, vous avez un seul endroit centralisé où définir exactement ce qui est acceptable. La validation se fait automatiquement à l'entrée, avant même que votre logique métier ne s'exécute. Votre code métier peut donc partir du principe que les données sont valides, ce qui le rend plus simple et plus lisible.

## Le coût des DTOs

Soyons honnêtes : les DTOs ajoutent du code. Pour chaque endpoint qui accepte des données, vous devez créer un DTO correspondant. Dans de très petits projets ou des prototypes rapides, cela peut sembler excessif.

Mais ce coût est largement compensé dès que votre projet grandit. Les bugs liés à des données mal validées disparaissent. Les discussions sur "quels champs accepte cet endpoint" deviennent inutiles, car le DTO répond à la question. Les tests deviennent plus simples car vous pouvez tester la validation indépendamment de la logique métier.

Dans un contexte d'apprentissage et de développement professionnel, prendre l'habitude d'utiliser des DTOs systématiquement est un excellent investissement. Cela structure votre pensée et vous force à réfléchir explicitement à vos interfaces.

## Récapitulation

Les DTOs sont des objets qui définissent la structure et les contraintes des données échangées entre le client et le serveur. En combinant TypeScript et `class-validator`, vous créez une validation déclarative, claire et automatique.

Le `ValidationPipe` global de NestJS intercepte toutes les requêtes, applique les validations définies dans les DTOs, et rejette automatiquement les requêtes invalides avec des messages d'erreur clairs.

Cette approche apporte sécurité, clarté et maintenabilité. Elle demande un petit effort initial pour créer les DTOs, mais cet effort est rapidement rentabilisé à mesure que le projet évolue.