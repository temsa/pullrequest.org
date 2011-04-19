Title: Sortie de RESThub 1.1-rc1
Author: Sébastien Deleuze 
Date: Apr 19 2011 16:30:00 GMT-0200 (CDT)
Categories: resthub, java, javascript 

A l'occasion de la sortie de la première Release Candidate de [RESThub](http://resthub.org) 1.1 , voici une synthèse des principales améliorations par rapport à la version 1.0.

Premier changement majeur, la séparation en 2 projets distincts (mais complémentaires) :

*   Une stack Java basée sur Spring 3 et des webservices REST qui vise à fournir des classes génériques réutilisables et des principes d'architectures pour développer plus rapidement vos applications Java.
*   Une stack Javascript basée sur jQuery, visant à fournir l'outillage et les repères nécessaires au développement d'applications professionnelles (cf. l'excellent article [Building Large-Scale jQuery Applications](http://addyosmani.com/blog/large-scale-jquery/)). Elle peut se câbler sur n'importe quelle technologie serveur qui expose des webservices REST (Java, Python, Ruby, NodeJS).
 Le projet est désormais hébergé sur 2 dépôt différents ([resthub](https://bitbucket.org/ilabs/resthub/) et [resthub-js](https://bitbucket.org/ilabs/resthub-js/)), ce qui permettra de les faire évoluer de façon indépendante, et d'encourager l'utilisation de la stack JS dans d'autres contextes.

## RESThub 1.1 rc1

Le nouveau site [http://resthub.org](http://resthub.org) héberge désormais la documentation du projet que nous allons encore compléter d'ici au lancement de la version finale.

Le développement de cette version 1.1 a été pour nous l'occasion de réaliser un gros travail visant à améliorer la pertinence des tests génériques proposés par le framework. Les 3 points clefs sont :

*   Désactivation du rollback transactionnel à la fin des tests unitaires proposés par défaut par Spring. C'est pratique pour de petits prototypes mais ça limite la pertinence des tests (pas de vérification des contraintes d'intégrités BDD par exemple).
*   Nouvelle classe de test AbstractResthubTransactionAwareTest destinée à tester vos classes transactionnelles comme celles de la couche service. Cette classe gère les transactions des méthodes qui y sont appelées, mais  elle-même n’est pas  transactionnelle. Elle simule en quelques sortes le comportement du filtre OpenSessionInViewFilter de Spring.
*   Une intégration Spring DBUnit permet une gestion simple et puissante des données de tests (jeux de donnée propres à chaque méthode de test).

La sérialisation JSON a été changée. Toujours basée sur l'excellent [Jackson](http://jackson.codehaus.org/) (mis à jour pour l'occasion en version 1.7), les annotations Jackson sont maintenant utilisées par défaut au lieu des annotations JAXB. Ces dernières étaient pratiques (une seule annotation pour les sérialiations XML et JSON) mais pas assez souples pour les besoins liés au développement d'application riches Javascripts.

La stack Javascript a elle été complétement réécrite afin de ne plus dépendre de [Sammy.JS](http://sammyjs.org/) (qui comportait de nombreuses fonctionnalités que nous n'utilisions pas. Nous avons donc réimplémenté une gestion des routes qui garde le même principe, mais qui est beaucoup plus légère. L'autre point qui était gênant avec Sammy.JS est sa notion de contexte, assez problématique quand on essaye de développer une application modulaire utilisant des composants.

La nouvelle stack RESThub JS 1.1 est composés des modules suivants développés sur une base [jQuery](http://jquery.com/) et [jQuery UI](http://jqueryui.com/) :

*   Chargement des scripts : gestion des dépendances transitives et de l'ordre de chargement des fichiers Javascript (basé sur RequireJS)
*   Logging : fontionnalité de gestion des logs conforme à la norme CommonJS
*   Routes : gestion des routes à la Sammy.JS, permettant l'utilisation de marques pages même sur une interface riche. Actuellement basé sur les évènements hashchange, le support du Push State HTML5 est prévu prochainement.
*   Bus d'évènement : bus d'évènement permettant de découpler les fonctionnalités de votre application (principe classique publish/subscribe)
*   Templating : Dynamisation de templates côté client, basé sur [jQuery Tmpl](http://api.jquery.com/jquery.tmpl/)
*   Repositories : permet de regrouper tous les échanges avec le serveur. Très utile pour Mocker votre application JS pour la tester sans serveur. 
*   Controller : implémentation côté client du pattern MVC, librement inspiré le jQuery MVC
*   Class : héritage en Javascript, basé sur la version améliorée par Javascript MVC du code proposé par John Resig dans son article [Simple JavaScript Inheritance](http://ejohn.org/blog/simple-javascript-inheritance/)
*   Stockage : abstrait les différents moyens de stockage disponibles dans le navigateur (cookie, localstorage, sessionstorage ...)
*   Internationalisation : gestion de l'internationalisation (similaire à ce qui se fait en Java)
    
Nous aurons l'occasion de présenter ces nouveautés ce soir au [LyonJUG](http://www.lyonjug.org/evenements/2eme-anniversaire).
