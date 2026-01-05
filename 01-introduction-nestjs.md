# Introduction à NestJS : Comprendre le contexte et la philosophie

Avant de plonger dans le code et la technique, prenons un moment pour comprendre ce qu'est réellement NestJS et pourquoi ce framework existe. Cette compréhension du contexte vous permettra de mieux saisir les choix de conception que vous rencontrerez tout au long de votre apprentissage.

## Le paysage du développement backend en Node.js

Lorsque Node.js est apparu, il a révolutionné le développement backend en permettant d'utiliser JavaScript côté serveur. Express.js est rapidement devenu le framework de référence pour créer des API. Sa philosophie est simple : fournir le strict minimum pour gérer des routes HTTP, des requêtes et des réponses. Cette approche minimaliste offre une grande liberté, mais elle laisse de nombreuses questions en suspens.

Imaginons que vous démarrez un nouveau projet backend avec Express. Vous créez un fichier, vous définissez quelques routes, et en quelques minutes votre API répond. C'est rapide, c'est efficace, et cela fonctionne parfaitement pour des projets simples. Mais à mesure que votre application grandit, vous commencez à vous poser des questions :
- Comment organiser mon code ? 
- Où placer ma logique métier ? 
- Comment gérer la connexion à la base de données ? 
- Comment structurer mes tests ? 
- Comment faire communiquer différentes parties de mon application de manière propre ?


Express ne répond pas à ces questions. Ce n'est pas un défaut, c'est un choix de conception. Express vous donne les outils de base, et c'est à vous de construire votre architecture. Chaque équipe, chaque développeur peut donc adopter une organisation différente. Dans une petite équipe ou sur un petit projet, ce n'est pas un problème. Mais dans un contexte professionnel avec plusieurs développeurs, ou sur des projets amenés à évoluer pendant des années, cette liberté peut devenir un obstacle.

## L'origine de NestJS : apporter structure et conventions

NestJS est né de cette problématique. Son créateur, Kamil Myśliwiec, a voulu proposer une solution qui combine la puissance de Node.js et d'Express avec une architecture structurée et des conventions claires. L'inspiration vient en partie d'Angular, le framework frontend développé par Google, qui impose une **architecture modulaire stricte**.

L'idée centrale de NestJS peut se résumer ainsi : plutôt que de laisser chaque développeur inventer sa propre architecture, proposons un cadre qui guide vers les bonnes pratiques tout en restant flexible. Ce cadre s'appuie sur des concepts éprouvés dans d'autres langages et frameworks, notamment la programmation orientée objet, l'injection de dépendances et la séparation des responsabilités.

## Ce que NestJS apporte concrètement

Prenons un exemple concret pour comprendre la différence. Avec Express, vous pourriez écrire une route comme ceci :

```javascript
app.get('/users/:id', (req, res) => {
  const userId = req.params.id;
  const user = database.findUserById(userId);
  res.json(user);
});
```

Ce code fonctionne, mais il mélange plusieurs responsabilités : la gestion de la route HTTP, la logique métier (trouver un utilisateur), et l'accès aux données. Si vous avez dix routes qui manipulent des utilisateurs, vous allez probablement dupliquer du code. Si vous devez tester cette logique, vous devez simuler l'objet `req` et `res` d'Express, ce qui n'est pas toujours évident.

NestJS propose une autre approche. La même fonctionnalité serait structurée en plusieurs couches distinctes : un contrôleur qui gère la route HTTP, un service qui contient la logique métier, et éventuellement un repository pour l'accès aux données. Chaque couche a une responsabilité claire, et elles communiquent entre elles de manière explicite grâce à l'injection de dépendances.

Cette structuration n'est pas qu'une question d'élégance du code. Elle a des conséquences pratiques importantes : votre code devient plus facile à tester, car vous pouvez tester chaque couche indépendamment. Il devient plus facile à maintenir, car chaque modification est localisée dans un composant spécifique. Il devient plus facile à comprendre pour un nouveau développeur, car l'architecture suit des conventions reconnues.

## NestJS et TypeScript : un mariage naturel

NestJS est construit sur TypeScript et en tire pleinement parti. Là où Express peut utiliser TypeScript de manière optionnelle, NestJS en fait un pilier de son architecture. Les décorateurs TypeScript, les types, les interfaces, tout cela est intégré au cœur du framework.

Cette intégration forte avec TypeScript signifie que votre éditeur de code peut vous aider beaucoup plus efficacement. L'autocomplétion fonctionne naturellement, les erreurs sont détectées avant même d'exécuter le code, et la documentation est souvent accessible directement via les définitions de types.

## À qui s'adresse NestJS ?

NestJS n'est pas nécessairement le meilleur choix pour tous les projets. Si vous devez créer un petit script backend pour un prototype rapide, Express ou même un simple serveur HTTP suffira largement. En revanche, si vous travaillez sur une application destinée à évoluer, à être maintenue par une équipe, à grandir en fonctionnalités, alors NestJS devient très pertinent.

Le framework est particulièrement adapté aux contextes professionnels où plusieurs développeurs collaborent. Les conventions qu'il impose facilitent la communication et réduisent les discussions interminables sur l'organisation du code. Un développeur qui rejoint un projet NestJS retrouvera immédiatement des structures familières, même s'il n'a jamais travaillé sur ce projet spécifique.

## La courbe d'apprentissage

NestJS demande un investissement initial plus important qu'Express. Il y a des concepts à comprendre, des conventions à assimiler, une façon de penser à adopter. C'est normal, car le framework vous apporte plus de structure, et cette structure a un coût en termes d'apprentissage.

Cependant, cette courbe d'apprentissage est en réalité un investissement. Une fois que vous aurez compris les principes fondamentaux de NestJS, vous serez capable de créer des applications complexes avec une base solide. Vous n'aurez plus à vous demander comment organiser votre code, car le framework vous guide naturellement vers des solutions éprouvées.

## La philosophie "convention over configuration"

NestJS adopte le principe de "convention over configuration", que l'on pourrait traduire par "privilégier les conventions aux configurations". Cela signifie que si vous suivez les conventions du framework, vous n'avez pas besoin de configurer grand-chose. Le framework devine ce que vous voulez faire en fonction de la manière dont vous structurez votre code.

Par exemple, si vous créez un fichier nommé `users.controller.ts` contenant une classe décorée avec `@Controller('users')`, NestJS comprendra automatiquement que cette classe gère les routes commençant par `/users`. Vous n'avez pas besoin de déclarer explicitement cette route quelque part dans un fichier de configuration central.

Cette approche réduit considérablement la quantité de code "plomberie" que vous devez écrire. Vous vous concentrez sur la logique métier, et le framework s'occupe de connecter les différentes pièces entre elles.

## NestJS dans l'écosystème Node.js

Il est important de comprendre que NestJS ne remplace pas Express, il s'appuie dessus. Par défaut, NestJS utilise Express en interne pour gérer les requêtes HTTP. Vous pouvez d'ailleurs configurer NestJS pour utiliser Fastify à la place si vous le souhaitez. Cela signifie que toutes les bibliothèques et middlewares Express restent utilisables dans un projet NestJS.

Cette compatibilité est un atout majeur. Vous n'êtes pas enfermé dans un écosystème fermé. Si vous avez besoin d'une fonctionnalité spécifique qui n'existe pas directement dans NestJS, vous pouvez probablement utiliser une bibliothèque Express existante.

## Vers une architecture professionnelle

En résumé, NestJS représente une évolution vers des architectures backend plus matures et professionnelles. Il ne s'agit pas de dire qu'Express est obsolète ou inadapté, mais plutôt de reconnaître que pour certains types de projets, une structure plus rigide et des conventions partagées apportent une réelle valeur.

Tout au long de ce cours, nous allons découvrir concrètement comment NestJS traduit cette philosophie en code. Nous verrons comment les différents concepts s'articulent, comment créer une API REST structurée, et comment adopter les bonnes pratiques dès le départ.

L'objectif n'est pas de vous transformer en expert NestJS en une journée, mais de vous donner les fondations solides qui vous permettront de progresser de manière autonome. Une fois que vous aurez compris les principes de base et la logique derrière les choix du framework, la documentation officielle deviendra votre meilleure alliée.

Maintenant que nous avons posé le contexte, entrons dans le vif du sujet et découvrons l'architecture d'une application NestJS.
