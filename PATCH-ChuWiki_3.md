# Correctifs de compatibilité PHP 8.3 — ChuWiki 3.0

Ce patch corrige tout ce qui empêchait ChuWiki 2.0 de fonctionner sous PHP 8.x.
Testé avec PHP 8.3.6. Tous les fichiers passent `php -l` sans erreur, et les
deux moteurs de rendu (wiki2xhtml et WikiRenderer) ont été testés en conditions
réelles (instanciation + rendu de texte wiki).

## 1. Constructeurs PHP4 jamais appelés (bug silencieux, le plus grave)

Depuis PHP 8.0, une méthode portant le même nom que sa classe n'est plus
reconnue comme constructeur. Sans erreur ni avertissement, ces classes ne
s'initialisaient donc plus du tout. Renommées en `__construct()` :

- `ChuWiki` (sdk/sdk.php) — la classe principale du wiki
- `wiki2xhtmlChu` (sdk/wiki2xhtml/class.wiki2xhtml.chu.php) — moteur de rendu par défaut
- `WikiRenderer`, `WikiTag`, `WikiInlineParser`, `WikiRendererBloc` (sdk/WikiRenderer/WikiRenderer.lib.php)
- `cwrtext_title` (sdk/WikiRenderer/rules/classicwr_to_text.php)
- `CClosestFinder`, `CSmileyReplacer` (sdk/smiley-replacer.php)

Un appel explicite `parent::WikiRendererBloc($wr)` a aussi été mis à jour en
`parent::__construct($wr)`.

## 2. Fonctions supprimées en PHP 8.0 (erreurs fatales)

- `get_magic_quotes_gpc()` — supprimée. Utilisée à 2 endroits dans
  `sdk/sdk.php` pour décider s'il faut appliquer `stripslashes()`. Comme les
  magic quotes n'existent plus depuis PHP 5.4, l'appel est maintenant protégé
  par `function_exists()`.
- `create_function()` — supprimée. Remplacée par des closures anonymes
  équivalentes dans `class.wiki2xhtml.basic.php` et `class.wiki2xhtml.chu.php`
  (moteur de rendu par défaut).
- `ereg()` — supprimée depuis PHP 7, jamais disponible en PHP 8. Remplacée par
  `preg_match()` dans `class.wiki2xhtml.basic.php`.

## 3. Syntaxe supprimée en PHP 8.0/7.0 (erreurs de parsing fatales)

- Accès aux caractères d'une chaîne avec des accolades `$str{0}` — supprimé,
  remplacé par `$str[0]` dans les fichiers de règles de WikiRenderer
  (chu_to_xhtml.php et les fichiers classicwr/wr3, non utilisés par défaut
  mais corrigés par cohérence).
- `$x =& new Classe()` — cette syntaxe de référence sur un nouvel objet a été
  supprimée en PHP 7. Remplacée par une simple assignation dans
  `WikiRenderer.lib.php`.

## 4. Méthode statique appelée sans le mot-clé `static` (erreur fatale)

`ChuWiki::Instance()` (le point d'entrée singleton utilisé par `wiki.php`,
`edit.php`, `history.php`) était déclarée `/* static */ function Instance()`
— un simple commentaire, pas le vrai mot-clé. Jusqu'à PHP 7, appeler une
méthode non-statique de façon statique était toléré (avec avertissement).
Depuis PHP 8.0, c'est une erreur fatale. Corrigé en `static function Instance()`.

**Ce bug rendait le wiki entièrement inutilisable** (page blanche/erreur 500
sur toute requête), il n'était visible qu'en testant une vraie requête HTTP,
pas en analysant le code statiquement.

## 5. `implode()` recevant `false` au lieu d'un tableau (erreur fatale)

Dans `GetLatestDate()`, `file()` retourne `false` quand le fichier de cache
de la dernière date n'existe pas encore (cas normal : toute nouvelle page).
Ce `false` était passé directement à `implode()`, ce qui plantait
systématiquement en PHP 8 (`TypeError`, non intercepté par l'arobase `@`).
Corrigé en vérifiant explicitement le retour de `file()` avant l'`implode()`.

**Ce bug empêchait l'affichage de toute page du wiki**, y compris la page
d'accueil sur une installation neuve.

## 6. Nettoyage des avertissements de dépréciation (PHP 8.2+)

Non bloquants, mais nettoyés par cohérence et pour anticiper PHP 9 :
- Ajout de `#[\AllowDynamicProperties]` sur les classes `ChuWiki` et
  `CClosestFinder`, qui créaient des propriétés à la volée (dépréciable
  depuis PHP 8.2).
- `GetPostedValue()` force une chaîne vide si le champ POST est absent,
  au lieu de laisser passer `null` dans `str_replace()`.
- Ajout d'un `?? ''` sur `$_SERVER['HTTP_REFERER']`, absent quand le
  navigateur n'envoie pas d'en-tête Referer.

## Validation effectuée

Tests réels de bout en bout avec un serveur PHP 8.3.6 (`php -S`) :
- Affichage d'une page existante et d'une page inexistante (`wiki.php`)
- Formulaire d'édition (`edit.php` en GET)
- Sauvegarde d'une page avec accents et wiki-syntaxe (`edit.php` en POST)
- Relecture après sauvegarde pour confirmer la persistance
- Historique des révisions (`history.php`)
- Flux RSS des dernières modifications (`latest-change.php`)
- Redirection de la page d'accueil (`index.php`)

Aucune erreur, aucun warning, aucun message de dépréciation sur l'ensemble
de ces scénarios.



Le reste du code (gestion des pages, historique, recherche, thèmes) ne
présentait aucune fonction supprimée ni syntaxe invalide. Aucune modification
fonctionnelle ou de comportement n'a été apportée en dehors de ces
corrections de compatibilité.
