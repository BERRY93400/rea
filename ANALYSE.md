# Analyse du dépôt REA

Le fichier `docs/architecture.mermaid` existe et il sert bien de base pour comprendre le projet. Il montre un système centré sur des points d’entrée CLI/MCP, puis des couches de composition, d’adaptation, d’application, de fournisseurs techniques, puis des contrats et le domaine métier.

## 1. Ce que fait le projet

REA est un outil pour comprendre un logiciel sans avoir son code source. Il aide à analyser :

- des binaires natifs ;
- des applications JavaScript / Electron ;
- des sites web et leur navigateur ;
- des APK Android ;
- des assemblages .NET ;
- d’autres artefacts techniques.

Le projet relie une interface de commande (CLI) et un serveur MCP (modèle de conversation / outil) à des analyseurs locaux. En pratique, le but est de permettre à un agent ou à un développeur de demander :

- « que fait cette application ? » ;
- « comment fonctionne cette fonctionnalité ? » ;
- « quels modules, routes, appels ou dépendances sont visibles ? » ;
- « quelles preuves peuvent être obtenues sans exécuter un code douteux ? »

Le dépôt donne aussi une partie web et navigateur : observation de pages, capture de scripts, collecte de données dans le navigateur, et analyse statique de bundles JavaScript. C’est bien la même logique : prendre un objet technique, le décrire, le relier à des preuves, puis utiliser ces résultats pour expliquer le comportement.

## 2. L’architecture en couches

Le diagramme `docs/architecture.mermaid` montre une pile claire en couches.

### a. Points d’entrée

Les points d’entrée sont :

- `src/cli.ts` et `src/cli/` : interface ligne de commande ;
- `src/main.ts` et `src/main/` : serveur MCP ;
- `scripts/rea.mjs` : petit routeur qui décide quelle commande lancer.

Responsabilité : démarrer le bon mode d’utilisation, configurer le runtime, et orienter vers le bon flux.

### b. Composition de production

La couche `src/composition/` construit les objets de travail et les fournisseurs. Elle met en place les sessions, les bindings et les services optionnels.

Responsabilité : assembler les composants du système sans coller tout le code dans un seul endroit. C’est la couche qui branche les outils disponibles à l’exécution.

### c. Adaptateurs

Les adaptateurs sont dans :

- `src/server/` : adaptation MCP et HTTP ;
- `bridge/` : ponts vers Hopper, Ghidra, Android, etc. ;
- `src/cli/` et `src/application/Setup*` : installation locale et configuration de l’environnement.

Responsabilité : traduire les appels du projet vers des outils externes ou des protocoles de runtime. On y voit notamment les ponts vers Hopper, Ghidra et d’autres analyseurs de niveau système.

### d. Couche application

La couche `src/application/` est le cœur des workflows partagés. On y trouve :

- `SessionProviderRouter` ;
- `BinarySession` ;
- `AnalysisProviderRegistry` ;
- `InvestigationRecords` ;
- `BinarySessionRecords` ;
- `AnalysisSnapshotCache` ;
- `src/application/javascript/` et `src/artifacts/javascript/` pour la reconstruction de JS.

Responsabilité : orchestrer les analyses, gérer les sessions, garder les traces et les preuves, et relier l’analyse au bon fournisseur.

### e. Couches de fournisseurs

Dans `src/providers/`, `src/browser/`, `src/ghidra/`, `src/hopper/`, `src/ida/`, `src/android/`, `src/firmware/`, `src/wasm/`, `src/evm/`, `src/dotnet/`, `src/objdump/`, `src/rizin/`, `src/gdb/`, `src/inspector/`, `src/javascript/sourceMaps/`, etc., on trouve les composants spécifiques à chaque type d’analyse.

Chaque fournisseur a une responsabilité propre :

- `src/browser/` : inspection de pages web et navigateur ;
- `src/inspector/` : observation passive de scripts et contextes V8 ;
- `src/javascript/sourceMaps/` : lecture des sourcemaps ;
- `src/android/` : APK et ressources Android ;
- `src/firmware/` : binwalk / unblob ;
- `src/ghidra/` et `src/hopper/` : analyse native ;
- `src/gdb/` et `src/rizin/` : debug et outils d’analyse ;
- `src/wasm/`, `src/evm/`, `src/dotnet/` : formats spécialisés ;
- `src/reference/` : lecture de sources de référence.

### f. Fondation des processus et du système

La couche `src/process/` et `src/windows/` gère les processus, les limites, les fichiers, les jobs, les runtime privés.

Responsabilité : garder les analyses dans un cadre contrôlé, avec nettoyage, délais, accès limité et sécurité de l’environnement hôte.

### g. Contrats, domaine et configuration

- `src/contracts/` : schémas publics et résultats d’analyse ;
- `src/domain/` : logique pure, sans effets de bord ;
- `src/config/` : configuration validée.

Responsabilité : séparer la définition des données et des règles du reste de l’exécution.

### h. Tests et évaluation

- `tests/` : suite de tests ;
- `src/evaluation/` : évaluation des agents et des résultats.

Responsabilité : vérifier que les analyses restent fiables et cohérentes.

En résumé, le projet suit une logique simple :

- point d’entrée ;
- composition ;
- adaptation technique ;
- workflows applicatifs ;
- fournisseurs spécialisés ;
- contrats et domaine ;
- tests et validation.

## 3. Comment ajouter un nouveau type d’analyse web

Pour ajouter un nouveau type d’analyse web, la méthode la plus claire dans ce dépôt est de suivre le modèle déjà utilisé pour le navigateur et les bundles JavaScript.

### Fichiers concernés

Les fichiers et dossiers utiles sont généralement :

- `src/browser/` : code du navigateur et des captures ;
- `src/inspector/` : observation de scripts et contenus V8 ;
- `src/application/` : workflows de haut niveau ;
- `src/composition/` : branchement du nouveau service ;
- `src/contracts/` : schémas et résultats ;
- `src/server/` : exposition via MCP / HTTP ;
- `docs/browser-observation.md` : documentation de référence ;
- `tests/` : tests de validation.

### Étapes concrètes

1. Définir le besoin.
   - Quel type de donnée on analyse ?
   - Une page, un bundle, un script, une route, un flux réseau, une donnée de DOM, un outil WebMCP ?

2. Définir le contrat de sortie.
   - Ajouter une structure de résultat dans `src/contracts/`.
   - La sortie doit distinguer : données observées, limites, erreurs, couverture, preuves.

3. Implémenter le traitement dans la bonne couche.
   - Soit dans `src/browser/` pour le niveau navigateur/CDP ;
   - soit dans `src/application/` pour une logique plus globale ;
   - soit dans `src/inspector/` si l’analyse porte sur les scripts ou le runtime.

4. Brancher le service dans la composition.
   - Ajouter le provider ou le factory dans `src/composition/`.
   - Lier le nouveau type avec le registre d’analyses.

5. Exposer le workflow.
   - Ajouter la commande ou l’outil dans `src/server/` ou dans la couche CLI.
   - Vérifier que le résultat est bien visible dans le même format que les autres analyses.

6. Tester en conditions réelles.
   - Tests unitaires sur le code ;
   - test d’intégration avec un navigateur réel ;
   - validation de sortie JSON / MCP ;
   - contrôle des limites (taille, temps, mémoire, erreurs).

### Points d’intégration

Le point clé est de ne pas ajouter le nouveau type au hasard. Il faut l’intégrer dans le flux suivant :

- entrée : page, bundle ou script ;
- observation / capture ;
- transformation vers un modèle interne ;
- classification et preuve ;
- sortie normalisée ;
- publication via MCP / CLI.

### Validation recommandée

- vérifier que le nouveau type est bien référencé dans le registry ;
- vérifier qu’il est bien exposé au client ;
- vérifier qu’il retourne des preuves lisibles ;
- vérifier qu’il gère correctement les cas d’échec, les pages non chargées et les données incomplètes ;
- ajouter un test réel sur un navigateur ou un bundle de test.

## 4. Plan pour adapter la partie web à un usage 100 % mobile

Le projet est déjà pensé pour le navigateur et les outils web, mais il faut faire un vrai travail de simplification pour un usage mobile complet.

### a. Rendu et interface

- utiliser une seule colonne, claire et verticale ;
- rendre la lecture lisible sur petit écran ;
- réduire le nombre de panneaux en même temps ;
- faire des blocs plus compacts : titre, preuve, action, détail ;
- prévoir des points d’arrêt (breakpoints) pour téléphone, tablette, paysage/portrait.

### b. Interaction tactile

- remplacer les interactions de souris par gestes tactiles ;
- utiliser des boutons de grande taille ;
- éviter les menus hover ;
- privilégier les actions simples : glisser, appuyer, basculer, ouvrir un panneau, revenir en arrière ;
- prévoir de bonnes zones de clic pour les écrans de petite taille.

### c. Performance

- charger uniquement les éléments nécessaires ;
- éviter les gros calculs sur le thread principal ;
- décomposer les analyses longues en étapes ;
- afficher immédiatement un résumé puis les détails plus tard ;
- compresser les captures et limiter les données transférées ;
- faire du cache local pour les résultats déjà vus.

### d. Autonomie et batterie

- limiter les captures automatiques et les boucles actives ;
- arrêter les analyses si l’écran est verrouillé ou si l’app est en arrière-plan ;
- réduire les mises à jour visuelles fréquentes ;
- réutiliser des résultats déjà calculés ;
- détecter les conditions faibles de batterie et faire un mode économe.

### e. Navigation et écrans

- gérer les écrans de façon lisible : accueil, analyse, preuves, détails, paramètres ;
- utiliser une navigation simple avec boutons en bas ou en haut selon le contexte ;
- garder un historique des analyses récentes ;
- permettre de reprendre une analyse interrompue sans perdre le contexte.

### f. Gestion des tailles d’écran et des capteurs

- détecter la taille de l’écran et l’orientation ;
- adapter les tableaux, graphiques et cartes à des zones très étroites ;
- gérer les éléments de plein écran en portrait ;
- demander explicitement les permissions nécessaires pour les capteurs ou la caméra ;
- prévoir des modes sans capteur si le mobile ne les supporte pas.

### g. Utilisation des capteurs et des données du navigateur

Pour un usage mobile, il faut rester honnête sur les limites :

- pas tous les capteurs sont disponibles ;
- certains navigateurs limitent l’accès aux APIs ;
- les capacités peuvent varier selon le système et la version ;
- le produit doit donc avoir des modes de secours, des messages clairs et des données partielles propres.

### h. Plan de mise en œuvre réaliste

1. Identifier la partie web réellement utile sur mobile.
2. Supprimer les éléments trop desktop-first.
3. Reprendre le rendu avec les écrans mobiles en priorité.
4. Réduire les charges CPU et mémoire.
5. Vérifier les permissions, la taille d’écran et la navigation.
6. Tester sur plusieurs téléphones et orientations.
7. Garder une version “desktop” pour les cas lourds, et une version “mobile” plus simple et plus robuste.

Le point important est de faire un produit adapté au mobile, pas seulement un site desktop redimensionné. Il faut un vrai design mobile, avec des tâches plus courtes, un contenu plus compact, et une expérience qui fonctionne bien sans clavier, sans souris et avec une autonomie limitée.

Le dépôt montre bien une architecture solide pour l’analyse technique. La partie mobile doit donc s’appuyer sur cette même logique : garder les couches de données, les preuves et les résultats, mais simplifier l’interface, la navigation et les traitements lourds pour les écrans et appareils mobiles.

---

Cette analyse correspond bien aux 4 points demandés :

1. ce que fait le projet ;
2. l’architecture en couches, basée sur `docs/architecture.mermaid` ;
3. comment ajouter un nouveau type d’analyse web ;
4. un plan réaliste pour un usage 100 % mobile.

Le document a été écrit en français simple, avec des responsabilités explicites et sans inventer de modules absents du dépôt.

