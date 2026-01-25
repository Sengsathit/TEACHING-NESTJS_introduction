# Authentification avec JWT : sécuriser votre API

## Pourquoi l'authentification est nécessaire

Jusqu'à présent, notre API est complètement ouverte. N'importe qui peut créer, modifier ou supprimer des articles. Dans une application réelle, vous voudrez probablement contrôler qui peut faire quoi. C'est là qu'intervient l'authentification.

L'authentification permet de répondre à la question : "qui es-tu ?". Une fois que vous savez qui est l'utilisateur, vous pouvez décider s'il a le droit d'effectuer certaines actions (c'est l'autorisation, que nous verrons ensuite).

Il existe plusieurs façons d'authentifier les utilisateurs dans une API REST : sessions avec cookies, OAuth, API keys, et JWT (JSON Web Tokens). Nous allons nous concentrer sur JWT car c'est l'approche la plus courante pour les API modernes, et celle que NestJS supporte le mieux.

## Comprendre JWT : les bases

Un JWT est un jeton de sécurité qui contient des informations (appelées "claims") encodées en JSON. Imaginez-le comme un badge d'accès numérique que l'utilisateur reçoit après s'être connecté, et qu'il doit présenter à chaque requête suivante.

Un JWT se compose de trois parties séparées par des points :

```
header.payload.signature
```

Le **header** décrit le type de jeton et l'algorithme de signature utilisé.
Le **payload** contient les informations sur l'utilisateur (son ID, son email, ses rôles, etc.).
La **signature** garantit que le jeton n'a pas été modifié.

La beauté de JWT réside dans son caractère "stateless" : le serveur n'a pas besoin de stocker les sessions en mémoire ou en base de données. Toutes les informations nécessaires sont dans le jeton lui-même. Le serveur peut vérifier la validité du jeton en vérifiant sa signature.

Le flux typique est le suivant :
1. L'utilisateur envoie ses identifiants (email + mot de passe) au serveur
2. Le serveur vérifie les identifiants
3. Si tout est correct, le serveur génère un JWT et le renvoie au client
4. Le client stocke ce JWT (généralement dans le localStorage ou un cookie)
5. Pour chaque requête suivante, le client envoie le JWT dans l'en-tête `Authorization`
6. Le serveur vérifie le JWT et identifie l'utilisateur

## Installer les dépendances nécessaires

Pour gérer l'authentification JWT dans NestJS, nous aurons besoin de plusieurs packages :

```bash
npm install @nestjs/jwt @nestjs/passport passport passport-jwt
npm install -D @types/passport-jwt
```

- `@nestjs/jwt` : module NestJS pour gérer les JWT
- `@nestjs/passport` : intégration de Passport.js avec NestJS
- `passport` : bibliothèque d'authentification très populaire en Node.js
- `passport-jwt` : stratégie Passport pour valider les JWT
- `@types/passport-jwt` : types TypeScript pour passport-jwt

Pour gérer les mots de passe de manière sécurisée, nous aurons aussi besoin de bcrypt :

```bash
npm install bcrypt
npm install -D @types/bcrypt
```

Bcrypt est une fonction de hachage conçue spécifiquement pour sécuriser les mots de passe. Elle est intentionnellement lente pour rendre les attaques par force brute plus difficiles.

## Créer le module Users

Avant de gérer l'authentification, nous avons besoin d'un système d'utilisateurs. Créons un module Users qui gérera nos utilisateurs :

```bash
nest g module users
nest g service users
```

Créons d'abord l'interface utilisateur. Créez le fichier `src/users/interfaces/user.interface.ts` :

```typescript
export interface User {
  id: string;
  email: string;
  password: string;  // Sera toujours haché
  name: string;
  createdAt: Date;
}
```

Maintenant, implémentons le service users dans `src/users/users.service.ts` :

```typescript
import { Injectable } from '@nestjs/common';
import { User } from './interfaces/user.interface';
import * as bcrypt from 'bcrypt';

@Injectable()
export class UsersService {
  private users: User[] = [];

  async create(email: string, password: string, name: string): Promise<User> {
    // Hacher le mot de passe avant de le stocker
    const hashedPassword = await bcrypt.hash(password, 10);

    const newUser: User = {
      id: String(Date.now()),
      email,
      password: hashedPassword,
      name,
      createdAt: new Date(),
    };

    this.users.push(newUser);
    return newUser;
  }

  async findByEmail(email: string): Promise<User | undefined> {
    return this.users.find(user => user.email === email);
  }

  async findById(id: string): Promise<User | undefined> {
    return this.users.find(user => user.id === id);
  }
}
```

Quelques points importants ici :

Le mot de passe est **toujours haché** avant d'être stocké. Le nombre `10` dans `bcrypt.hash(password, 10)` est le "salt rounds", qui détermine combien de fois l'algorithme de hachage sera appliqué. Plus ce nombre est élevé, plus c'est sécurisé, mais plus c'est lent. 10 est un bon compromis.

Nous ne créons pas de contrôleur pour ce module car nous ne voulons pas exposer directement ces endpoints. La création d'utilisateurs sera gérée via un endpoint d'inscription dans le module auth.

Mettez à jour `src/users/users.module.ts` pour exporter le service :

```typescript
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';

@Module({
  providers: [UsersService],
  exports: [UsersService],  // Important : on exporte le service
})
export class UsersModule {}
```

## Créer le module Auth

Le module Auth sera responsable de l'inscription, de la connexion et de la validation des JWT. Créons-le :

```bash
nest g module auth
nest g service auth
nest g controller auth
```

### Configurer le module JWT

Ouvrez `src/auth/auth.module.ts` et configurons le module JWT :

```typescript
import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { PassportModule } from '@nestjs/passport';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { UsersModule } from '../users/users.module';
import { JwtStrategy } from './strategies/jwt.strategy';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.register({
      secret: 'VOTRE_SECRET_TRES_SECRET',  // ⚠️ À mettre dans les variables d'environnement en production !
      signOptions: { expiresIn: '24h' },   // Le token expire après 24h
    }),
  ],
  providers: [AuthService, JwtStrategy],
  controllers: [AuthController],
})
export class AuthModule {}
```

⚠️ **IMPORTANT** : Dans une vraie application, ne mettez JAMAIS le secret JWT directement dans le code. Utilisez des variables d'environnement avec le package `@nestjs/config`. Nous le faisons ici par simplicité pédagogique.

Le paramètre `expiresIn: '24h'` définit la durée de validité du token. Après 24 heures, le token ne sera plus accepté et l'utilisateur devra se reconnecter.

### Créer les DTOs d'authentification

Créons les DTOs pour l'inscription et la connexion. Créez le fichier `src/auth/dto/register.dto.ts` :

```typescript
import { IsEmail, IsString, MinLength, MaxLength } from 'class-validator';

export class RegisterDto {
  @IsEmail({}, { message: 'Veuillez fournir une adresse email valide' })
  email: string;

  @IsString()
  @MinLength(8, { message: 'Le mot de passe doit contenir au moins 8 caractères' })
  @MaxLength(50)
  password: string;

  @IsString()
  @MinLength(2)
  @MaxLength(100)
  name: string;
}
```

Et `src/auth/dto/login.dto.ts` :

```typescript
import { IsEmail, IsString, MinLength } from 'class-validator';

export class LoginDto {
  @IsEmail({}, { message: 'Veuillez fournir une adresse email valide' })
  email: string;

  @IsString()
  @MinLength(1, { message: 'Le mot de passe est requis' })
  password: string;
}
```

### Implémenter le service Auth

Ouvrez `src/auth/auth.service.ts` :

```typescript
import { Injectable, UnauthorizedException, ConflictException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { UsersService } from '../users/users.service';
import * as bcrypt from 'bcrypt';
import { RegisterDto } from './dto/register.dto';
import { LoginDto } from './dto/login.dto';

@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService,
  ) {}

  async register(registerDto: RegisterDto) {
    // Vérifier si l'utilisateur existe déjà
    const existingUser = await this.usersService.findByEmail(registerDto.email);
    if (existingUser) {
      throw new ConflictException('Un utilisateur avec cet email existe déjà');
    }

    // Créer l'utilisateur
    const user = await this.usersService.create(
      registerDto.email,
      registerDto.password,
      registerDto.name,
    );

    // Générer le token JWT
    const payload = { sub: user.id, email: user.email };
    const access_token = this.jwtService.sign(payload);

    return {
      access_token,
      user: {
        id: user.id,
        email: user.email,
        name: user.name,
      },
    };
  }

  async login(loginDto: LoginDto) {
    // Trouver l'utilisateur
    const user = await this.usersService.findByEmail(loginDto.email);
    if (!user) {
      throw new UnauthorizedException('Email ou mot de passe incorrect');
    }

    // Vérifier le mot de passe
    const isPasswordValid = await bcrypt.compare(loginDto.password, user.password);
    if (!isPasswordValid) {
      throw new UnauthorizedException('Email ou mot de passe incorrect');
    }

    // Générer le token JWT
    const payload = { sub: user.id, email: user.email };
    const access_token = this.jwtService.sign(payload);

    return {
      access_token,
      user: {
        id: user.id,
        email: user.email,
        name: user.name,
      },
    };
  }

  async validateUser(userId: string) {
    return this.usersService.findById(userId);
  }
}
```

Analysons les points importants :

Dans `register`, nous vérifions d'abord si un utilisateur avec le même email existe déjà. Si oui, nous lançons une `ConflictException` (code HTTP 409). C'est une bonne pratique REST pour indiquer qu'il y a un conflit avec l'état actuel des ressources.

Le payload du JWT contient `sub` (subject, l'ID de l'utilisateur) et `email`. Vous pouvez ajouter d'autres informations, mais gardez le payload léger. Ne mettez JAMAIS le mot de passe dans le payload JWT, même haché.

Dans `login`, nous utilisons un message d'erreur volontairement vague : "Email ou mot de passe incorrect". Ne dites jamais à un attaquant si c'est l'email ou le mot de passe qui est incorrect, car cela l'aiderait à deviner les comptes existants.

Nous utilisons `bcrypt.compare()` pour vérifier le mot de passe. Cette fonction compare le mot de passe en clair fourni avec le hash stocké en base de données.

### Créer la stratégie JWT

La stratégie JWT est un composant Passport qui dit à NestJS comment valider un JWT et extraire l'utilisateur correspondant. Créez le fichier `src/auth/strategies/jwt.strategy.ts` :

```typescript
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { AuthService } from '../auth.service';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(private authService: AuthService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: 'VOTRE_SECRET_TRES_SECRET',  // ⚠️ Même secret que dans auth.module.ts
    });
  }

  async validate(payload: any) {
    // Le payload ici est le contenu décodé du JWT
    // Il contient { sub: userId, email: userEmail }
    const user = await this.authService.validateUser(payload.sub);

    if (!user) {
      throw new UnauthorizedException();
    }

    // Ce que vous retournez ici sera disponible dans req.user
    return {
      userId: user.id,
      email: user.email,
      name: user.name,
    };
  }
}
```

La méthode `jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken()` indique à Passport de chercher le JWT dans l'en-tête `Authorization` avec le format `Bearer <token>`.

Le paramètre `ignoreExpiration: false` dit à Passport de rejeter automatiquement les tokens expirés.

La méthode `validate` est appelée automatiquement par Passport après avoir vérifié la signature du JWT. C'est ici que vous pouvez charger les informations complètes de l'utilisateur depuis la base de données si nécessaire.

### Implémenter le contrôleur Auth

Ouvrez `src/auth/auth.controller.ts` :

```typescript
import { Controller, Post, Body, HttpCode, HttpStatus } from '@nestjs/common';
import { AuthService } from './auth.service';
import { RegisterDto } from './dto/register.dto';
import { LoginDto } from './dto/login.dto';

@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @Post('register')
  async register(@Body() registerDto: RegisterDto) {
    return this.authService.register(registerDto);
  }

  @Post('login')
  @HttpCode(HttpStatus.OK)  // Par défaut POST retourne 201, mais login devrait retourner 200
  async login(@Body() loginDto: LoginDto) {
    return this.authService.login(loginDto);
  }
}
```

Simple et direct. Le contrôleur se contente de recevoir les requêtes et de déléguer au service.

## Protéger vos routes avec des Guards

Maintenant que nous avons un système d'authentification, nous voulons protéger certaines routes. Par exemple, seuls les utilisateurs connectés devraient pouvoir créer des articles.

NestJS utilise des **Guards** pour cela. Un Guard est une classe qui décide si une requête peut continuer ou non. Créons un Guard JWT.

En fait, NestJS et Passport fournissent déjà un Guard pour JWT : `AuthGuard('jwt')`. Créons un Guard personnalisé qui l'encapsule pour plus de clarté. Créez `src/auth/guards/jwt-auth.guard.ts` :

```typescript
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

Ce Guard hérite de `AuthGuard('jwt')` qui utilise automatiquement notre `JwtStrategy`.

### Utiliser le Guard sur des routes

Pour protéger une route, il suffit d'ajouter le décorateur `@UseGuards()`. Modifions notre contrôleur Articles pour protéger certaines routes. Ouvrez `src/articles/articles.controller.ts` :

```typescript
import { Controller, Get, Post, Put, Delete, Body, Param, UseGuards, Request, HttpCode, HttpStatus } from '@nestjs/common';
import { ArticlesService } from './articles.service';
import { CreateArticleDto } from './dto/create-article.dto';
import { UpdateArticleDto } from './dto/update-article.dto';
import { JwtAuthGuard } from '../auth/guards/jwt-auth.guard';
import { Article } from './interfaces/article.interface';

@Controller('articles')
export class ArticlesController {
  constructor(private readonly articlesService: ArticlesService) {}

  @Get()
  findAll(): Article[] {
    return this.articlesService.findAll();
  }

  @Get(':id')
  findOne(@Param('id') id: string): Article {
    return this.articlesService.findOne(id);
  }

  @UseGuards(JwtAuthGuard)  // Route protégée
  @Post()
  @HttpCode(HttpStatus.CREATED)
  create(@Body() createArticleDto: CreateArticleDto, @Request() req): Article {
    // req.user contient les informations de l'utilisateur connecté
    console.log('Utilisateur connecté :', req.user);
    return this.articlesService.create(createArticleDto);
  }

  @UseGuards(JwtAuthGuard)  // Route protégée
  @Put(':id')
  update(
    @Param('id') id: string,
    @Body() updateArticleDto: UpdateArticleDto,
  ): Article {
    return this.articlesService.update(id, updateArticleDto);
  }

  @UseGuards(JwtAuthGuard)  // Route protégée
  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  remove(@Param('id') id: string): void {
    this.articlesService.remove(id);
  }
}
```

Maintenant, si vous essayez d'accéder à `POST /articles`, `PUT /articles/:id` ou `DELETE /articles/:id` sans JWT valide, vous recevrez une erreur 401 Unauthorized.

L'objet `req.user` contient les informations retournées par la méthode `validate()` de votre `JwtStrategy`. Vous pouvez l'utiliser pour savoir qui fait la requête et adapter la logique en conséquence (par exemple, pour vérifier que l'utilisateur modifie uniquement ses propres articles).

## Tester l'authentification

Testons notre système d'authentification complet.

### 1. S'inscrire

Envoyez une requête POST à `http://localhost:3000/auth/register` :

```json
{
  "email": "marie@example.com",
  "password": "password123",
  "name": "Marie Dupont"
}
```

Vous devriez recevoir :

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "1234567890",
    "email": "marie@example.com",
    "name": "Marie Dupont"
  }
}
```

### 2. Se connecter

Envoyez une requête POST à `http://localhost:3000/auth/login` :

```json
{
  "email": "marie@example.com",
  "password": "password123"
}
```

Vous recevrez le même format de réponse avec un nouveau token.

### 3. Utiliser le token pour créer un article

Copiez le `access_token` reçu. Maintenant, envoyez une requête POST à `http://localhost:3000/articles` avec cet en-tête :

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

Et le corps :

```json
{
  "title": "Mon article protégé",
  "content": "Ce contenu a été créé par un utilisateur authentifié !",
  "author": "Marie Dupont"
}
```

La requête devrait réussir. Si vous essayez sans le header `Authorization`, vous recevrez une erreur 401.

## Gérer le refresh des tokens

Un problème avec les JWT est qu'une fois émis, ils restent valides jusqu'à expiration. Si un token est volé, l'attaquant peut l'utiliser jusqu'à ce qu'il expire. C'est pourquoi on utilise des durées d'expiration courtes.

Mais cela signifie que l'utilisateur doit se reconnecter fréquemment, ce qui est gênant. La solution classique est d'utiliser des **refresh tokens**.

L'idée est la suivante :
- Le serveur génère deux tokens : un access token (courte durée, ex: 15 minutes) et un refresh token (longue durée, ex: 7 jours)
- Le client utilise l'access token pour les requêtes normales
- Quand l'access token expire, le client utilise le refresh token pour en obtenir un nouveau
- Si le refresh token est compromis, vous pouvez le révoquer côté serveur

L'implémentation complète des refresh tokens est plus avancée et sort du cadre de cette introduction, mais gardez ce concept en tête pour vos projets futurs.

## Bonnes pratiques de sécurité

Quelques règles importantes à retenir :

**Stockage des secrets** : Ne stockez JAMAIS vos secrets JWT dans le code. Utilisez des variables d'environnement avec le package `@nestjs/config`.

**Durée d'expiration** : Adaptez la durée de vie des tokens à votre contexte. Pour une application sensible (banque, santé), utilisez des durées courtes (15-30 minutes). Pour une application moins critique, 24h peut être acceptable.

**HTTPS** : En production, utilisez TOUJOURS HTTPS. Les JWT envoyés via HTTP peuvent être interceptés.

**Validation** : Ne faites jamais confiance au contenu d'un JWT sans vérifier sa signature. C'est ce que fait automatiquement `JwtStrategy`, mais si vous décodez manuellement un JWT, vérifiez toujours la signature.

**Mots de passe** : Utilisez toujours bcrypt (ou une alternative moderne comme argon2) pour hacher les mots de passe. N'inventez jamais votre propre système de hachage.

**Rate limiting** : Implémentez du rate limiting sur les endpoints de login pour prévenir les attaques par force brute.

## Aller plus loin avec les rôles et permissions

Une fois l'authentification en place, vous voudrez probablement implémenter l'autorisation : certains utilisateurs peuvent faire certaines actions, d'autres non.

Vous pouvez ajouter un champ `role` dans votre modèle User :

```typescript
export interface User {
  id: string;
  email: string;
  password: string;
  name: string;
  role: 'user' | 'admin';  // Ou utilisez un enum
  createdAt: Date;
}
```

Puis créer un Guard qui vérifie le rôle :

```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.get<string[]>('roles', context.getHandler());
    if (!requiredRoles) {
      return true;
    }

    const request = context.switchToHttp().getRequest();
    const user = request.user;

    return requiredRoles.some((role) => user.role === role);
  }
}
```

Et un décorateur pour spécifier les rôles requis :

```typescript
import { SetMetadata } from '@nestjs/common';

export const Roles = (...roles: string[]) => SetMetadata('roles', roles);
```

Utilisation :

```typescript
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles('admin')
@Delete(':id')
remove(@Param('id') id: string): void {
  return this.articlesService.remove(id);
}
```

## Récapitulation

L'authentification JWT dans NestJS repose sur plusieurs concepts qui fonctionnent ensemble :

Le **module JWT** génère et vérifie les tokens. Le **module Passport** fournit le système de stratégies d'authentification. La **JwtStrategy** définit comment valider un JWT et extraire l'utilisateur. Les **Guards** protègent les routes en vérifiant l'authentification.

Le flux complet est :
1. L'utilisateur s'inscrit ou se connecte avec ses identifiants
2. Le serveur vérifie les identifiants et génère un JWT
3. Le client stocke le JWT et l'envoie dans l'en-tête `Authorization` pour chaque requête
4. Le Guard vérifie le JWT via la stratégie Passport
5. Si le JWT est valide, la requête continue et `req.user` contient les infos de l'utilisateur

Cette architecture peut sembler complexe au début, mais elle est très puissante et extensible. Une fois en place, ajouter de nouvelles routes protégées devient trivial : il suffit d'ajouter `@UseGuards(JwtAuthGuard)`.

Vous avez maintenant tous les éléments fondamentaux pour construire une API NestJS sécurisée avec authentification JWT.