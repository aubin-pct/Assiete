# CLAUDE.md — Assiette

Application web personnelle de suivi nutritionnel. L'utilisateur photographie son repas, un modèle de vision (DeepSeek V4.1 Flash par défaut) estime les aliments, leurs grammes et leurs macros, puis l'utilisateur corrige avant d'enregistrer. Usage mono-utilisateur, sur téléphone, installée via « Ajouter à l'écran d'accueil ».

## Structure

```
index.html            # toute l'appli : HTML, CSS et JS inline
manifest.webmanifest  # PWA : nom, couleurs, icônes, display standalone
icon-192.png          # icône (aussi apple-touch-icon et favicon)
icon-512.png          # icône maskable
```

Pas de build, pas de dépendances, pas de framework. Vanilla JS en `"use strict"`. Ressources externes :
- les polices Google Fonts (Sora pour les titres et chiffres, Figtree pour le texte), avec repli sur les polices système ;
- `barcode-detector@3.2.2` (jsDelivr, IIFE `ponyfill.js`, contrôle d'intégrité `BD_SRI`), chargé à la demande au premier scan seulement si le navigateur n'a pas de `BarcodeDetector` natif (cas de Safari iOS). Il télécharge lui-même `zxing-wasm@3.1.3` (~1 Mo) depuis `fastly.jsdelivr.net`. Pour changer de version : mettre à jour `BD_URL` et recalculer `BD_SRI` (`openssl dgst -sha384 -binary ponyfill.js | openssl base64 -A`) ;
- l'API Open Food Facts (`world.openfoodfacts.org/api/v2/product/{code}.json`), sans clé, CORS ouvert. Limite : 15 fiches produit par minute et par adresse IP (réponse 429, message dédié dans `fetchOFF`) ; ne jamais y faire de recherche à la frappe (10 recherches par minute).

Garder ce principe : un seul fichier HTML autonome, déployable tel quel sur un hébergement statique. N'ajouter une bibliothèque que si elle fait un vrai travail, et alors en `<script>` depuis un CDN avec version figée.

## Déploiement

GitHub Pages depuis la branche `main`, à la racine du dépôt. Aucune étape de build : pousser les fichiers suffit. Tous les chemins sont relatifs (`start_url: "./"`), l'appli fonctionne donc dans un sous-dossier `tonpseudo.github.io/depot/`.

## Architecture du JS

- État global : `S` (données persistées), `view` (onglet courant), `curDate` (jour affiché, `YYYY-MM-DD` en heure locale), `draft` (repas en cours de saisie), `armedDelete` (suppression en attente de confirmation).
- États d'interface : `openMeal` (repas déplié dans le journal), `histSel` (jour sélectionné dans le graphique d'historique). `arm(id)` arme une suppression et la désarme seule après 4 s.
- Le journal se navigue aussi au glissement horizontal (écouteurs `touchstart`/`touchend` globaux, hors `bind()`), et un tap sur le libellé du jour ramène à aujourd'hui.
- Photos : un brouillon peut avoir jusqu'à `MAXP` (6) photos dans `draft.photos = [{ full, thumb }]`, toutes envoyées au modèle dans le même message. Seule la miniature de la première est enregistrée avec le repas (le schéma garde un seul `thumb`). Deux `<input type="file">` (`camInput` et `galInput`, insérés par `captureHTML()` ou `photosHTML()`) : `#photocam` porte `capture="environment"` (ouvre l'appareil photo directement), `#photo` ouvre la photothèque avec `multiple`. Les deux partagent le gestionnaire `onPhoto`. Ne pas ajouter `capture` à `#photo`.
- Suppression de photo : `[data-rmphoto]` retire une photo du brouillon ; `[data-delphoto]` retire la miniature d'un repas enregistré (confirmation en deux temps, `armedDelete = "ph:" + id`).
- Scanner de code-barres (`openScanner`) : superposition plein écran ajoutée à `body`, hors de `#app` et de `render()`, état dans `SC`. `getDetector()` prend le `BarcodeDetector` natif s'il lit l'EAN-13 (Chrome Android), sinon charge la bibliothèque ZXing (iOS). Lecture continue toutes les 150 ms (`tick`). Un produit déjà dans `draft.products` n'est jamais ajouté deux fois ; les pastilles du scanner (`scList`, `[data-sccode]`) permettent de retirer un produit sans fermer. Secours : saisie du code à la main et « Photo du code » (lecture sur une image). La caméra est coupée quand la page passe en arrière-plan et relancée au retour ; le bouton retour d'Android ferme le scanner (`history.pushState` + `popstate`). `getUserMedia` exige HTTPS (ou `localhost`).
- Produits scannés : un scan n'ajoute pas un aliment mais une référence dans `draft.products = [{ code, nom, per, portion, used }]` (`per` = valeurs pour 100 g tel que vendu, `lookupOFF` mis en cache dans `offCache`). Ils s'affichent dans la carte `#prods` (`productsHTML`, retrait par `[data-rmprod]`) et sont envoyés à l'IA avec les photos et les « Quantités et précisions » ; l'analyse marche aussi sans photo (« 300 g de riz »). L'IA renvoie `code` sur l'aliment correspondant et choisit les grammes (convertis en poids tel que vendu si l'utilisateur donne un poids cuit) ; `analyse()` recalcule toujours les macros de ces aliments depuis `per` (`productItem`) et rajoute avec la portion un produit oublié par l'IA. `used` passe à `true` après une analyse ; `saveMeal()` refuse d'enregistrer tant qu'un produit scanné n'a pas été analysé. Sans clé API, `[data-direct]` (« Ajouter tel quel ») transforme le produit en aliment `scan: true`, conservé aux analyses suivantes et signalé au modèle comme déjà compté. `saveMeal()` n'enregistre que `{ nom, g, kcal, p, c, f }`.
- Design : cartes sans bordure avec `--shadow`, rayons `--radius` (26 px) et `--r-md`, dégradé `--grad`, barre du bas flottante (`nav.tabs`). La barre d'enregistrement `.savebar` est en `position:sticky` avec un `bottom` calé sur la hauteur de la barre du bas (~102 px) : si on change la hauteur de `nav.tabs`, ajuster aussi `.savebar`, `.toast` et le `padding-bottom` du `body`. L'animation d'entrée (`#app.enter`) ne se joue que dans `go()` via `animateIn()`, jamais dans `render()`.
- Repas rapides (carte `#quick` de l'écran d'ajout, `quickHTML`, onglet dans `quickTab`) : Favoris (`S.favs`), Récents (`recentMeals()`, repas distincts selon `sig()`, hors favoris) et Produits (`S.scanned`). « + » enregistre tout de suite une copie (`logCopies`, avec « Annuler » dans le toast) en gardant le type d'origine, sauf si l'utilisateur a choisi un type (`draft.typeSet`) ; toucher le nom charge le repas dans le brouillon (`loadIntoDraft`) pour l'ajuster. Un produit s'ajoute à `draft.products`. `lookupOFF` lit d'abord `S.scanned` : un produit connu ne coûte aucun appel réseau.
- Journal : un repas déplié propose Modifier (`editMeal`), Refaire (copie aujourd'hui, type selon l'heure) et Favori (`toggleFav`). Un jour passé propose « Copier ces repas à aujourd'hui » ; aujourd'hui vide propose « Copier les repas d'hier ». Les copies gardent les heures d'origine.
- Modification : `draft.editId` passe l'écran d'ajout en mode modification (`editHTML`) ; `saveMeal()` remplace alors le repas (même `id`). Quitter l'onglet abandonne la modification (`go()`). Date et heure (`#mdate`, `#mtime`) sont modifiables pour tout repas ; par défaut, l'heure actuelle aujourd'hui, sinon l'heure habituelle du type (`TYPICAL`). Pas de date future.
- `toast(msg, { label, fn })` affiche un bouton d'action (« Annuler ») pendant 5 s.
- Poids (carte `#weight` de l'historique) : `trendSeries()` calcule une tendance lissée (moyenne mobile exponentielle à 10 % par jour, tenant compte des jours sans pesée) ; `weightStats()` donne le rythme en kg par semaine par moindres carrés (`slope`) sur les pesées des 28 derniers jours, dès 3 pesées sur au moins 10 jours, et la projection à 4 semaines. Graphique SVG `weightChartHTML` (pesées en points, tendance en ligne), période `wPeriod` (30, 90 ou 0 = tout). Une pesée peut être saisie pour une date passée (remplace celle du jour) et supprimée avec « Annuler ».
- Dépense et objectif adaptatif (carte `#energy`, `energyStats`) : sur les 28 derniers jours sans aujourd'hui, dépense = apports moyens des jours complets (≥ `MIN_DAY_KCAL`, 800 kcal) − pente du poids × `KCAL_KG` (7 700 kcal/kg). Il faut au moins 10 jours complets, 4 pesées et 14 jours entre la première et la dernière ; hors de 1 200 à 6 000 kcal, l'estimation est jugée incohérente. Objectif conseillé = dépense + `settings.rate` × 7 700 / 7, arrondi à 50 kcal ; « Appliquer » (si l'écart dépasse 100 kcal) change `goals.kcal` et reporte l'écart sur les glucides (protéines et lipides inchangés), avec « Annuler ».
- Rendu : `render()` remplace entièrement `#app.innerHTML` avec la vue courante (`journalHTML`, `addHTML`, `historyHTML`, `settingsHTML`), puis appelle `bind()`, qui rattache tous les écouteurs. Tout nouvel élément interactif doit être branché dans `bind()`.
- Exception au re-rendu complet : dans la liste d'aliments du brouillon, la saisie met à jour les champs voisins et `#totals` directement, sans `render()`, pour ne pas perdre le focus du clavier. Conserver ce comportement.
- Navigation : barre du bas `nav.tabs` avec `data-view`, et `go(view)`.

## Données (`localStorage`, clé `assiette.v1`)

```js
{
  settings: {
    apiKey: "", baseUrl: "https://api.deepseek.com", model: "deepseek-flash", deep: false,
    rate: 0,   // objectif de poids en kg par semaine : < 0 perte, 0 maintien, > 0 prise (±0,25 ou ±0,5)
    goals: { kcal: 3000, p: 160, c: 380, f: 90 }
  },
  meals: [{
    id, date: "YYYY-MM-DD", time: "HH:MM",
    type: "Petit-déjeuner" | "Déjeuner" | "Collation" | "Dîner",
    name, items: [{ nom, g, kcal, p, c, f }],
    thumb   // data URL JPEG 160×160, ou null
  }],
  weights: [{ date: "YYYY-MM-DD", kg }],   // une pesée max par jour
  favs: [{ id, name, type, items, thumb }], // repas favoris (copies, indépendantes des repas du journal)
  scanned: [{ code, nom, per: { kcal, p, c, f }, portion }]   // 30 derniers produits scannés, valeurs pour 100 g
}
```

`favs` et `scanned` ont été ajoutés sans changer de clé : ils valent `[]` s'ils manquent au chargement.

- Au chargement, les réglages sont fusionnés avec `DEFAULTS` : on peut ajouter un champ de réglage sans migration.
- Si le schéma change de façon incompatible, passer à `assiette.v2` et écrire une migration depuis `v1`. Ne jamais perdre les données existantes.
- `save()` renvoie `false` si le quota est dépassé (environ 5 Mo). Les miniatures sont le poste le plus lourd. La photo pleine taille n'est jamais stockée.
- L'export JSON exclut volontairement `apiKey` (et `scanned`, simple cache). L'import fusionne (dédoublonnage par `id` pour les repas, par `date` pour les pesées, par signature `sig()` pour les favoris) et ne remplace pas.

## Appel au modèle (`callModel`)

- `POST {baseUrl}/chat/completions`, au format OpenAI (compatible DeepSeek et OpenRouter).
- Message `system` : constante `SYSTEM`, en français. Il gère plusieurs photos (ne pas compter deux fois un aliment) et les étiquettes nutritionnelles photographiées (leurs valeurs priment sur les tables). Message `user` : texte, plus chaque photo en `image_url` sous forme de data URL. DeepSeek refuse les images hors des messages `user`.
- Image envoyée : JPEG, côté le plus long 1280 px, qualité 0,85, redimensionnée côté client par `toJpeg`.
- `response_format: { type: "json_object" }`. Le mot « JSON » doit rester dans le prompt.
- Paramètres propres à DeepSeek, ajoutés seulement si `baseUrl` contient `deepseek.com` : `thinking: { type: "enabled" | "disabled" }` (réflexion activée par défaut côté API, d'où le `disabled` explicite), et `reasoning_effort: "high"` en mode approfondi.
- En cas d'erreur 400, un seul nouvel essai sans `response_format`, `thinking` ni `reasoning_effort`.
- La réponse est analysée entre le premier `{` et le dernier `}`, pour tolérer du texte parasite autour du JSON.

JSON attendu du modèle :

```json
{ "plat": "…", "items": [{ "nom": "…", "grammes": 0, "kcal": 0, "proteines": 0, "glucides": 0, "lipides": 0 }],
  "hypotheses": "…", "confiance": "faible|moyenne|haute" }
```

`normItem` convertit vers le format interne `{ nom, g, kcal, p, c, f, ratio }`. `ratio` contient les valeurs par gramme : quand on change les grammes, les macros suivent proportionnellement. Quand on modifie une macro à la main, `ratio` est recalculé. `ratio` est retiré avant l'enregistrement.

## Conventions

- Langue : toute l'interface est en français, tutoiement, phrases courtes. Les messages d'erreur disent ce qui ne va pas et comment le corriger.
- Sécurité : tout texte venant de l'utilisateur ou du modèle passe par `esc()` avant d'entrer dans le HTML.
- Nombres : `num()` accepte la virgule décimale et renvoie 0 pour les valeurs invalides ou négatives. Affichage avec `fmt()` (format `fr-FR`).
- Dates : toujours en heure locale via `iso()`, `parseISO()` et `shift()`. Ne jamais utiliser `toISOString()` pour une date de journal, car elle décale d'un jour autour de minuit.
- Pas de `alert`, `confirm` ni `prompt` : les confirmations se font dans la page (bouton qui passe en « Confirmer » via `armedDelete`).
- Style : couleurs uniquement via les variables CSS de `:root`. Le thème sombre est redéfini dans `@media (prefers-color-scheme: dark)`. Couleurs des macros : `--prot`, `--carb`, `--fat`. Espacements en `gap`, gouttière de 16 px, zones sûres gérées avec `env(safe-area-inset-*)`.
- Mise en page : conçue pour 360 à 430 px de large, colonne max 560 px. Les champs font 16 px de police pour éviter le zoom automatique d'iOS.

## Tester

1. Vérifier la syntaxe du script :

```bash
sed -n '/<script>/,/<\/script>/p' index.html | sed '1d;$d' > /tmp/app.js && node --check /tmp/app.js
```

2. Faire une capture à 390×844 avec Playwright, après avoir injecté des données de test dans `localStorage` (clé `assiette.v1`). Vérifier le journal et l'écran d'ajout.
3. L'analyse photo ne se teste qu'avec une vraie clé API, dans un navigateur, sur l'URL déployée.
4. Scanner sans caméra : saisir un code à la main (`3017620422003` = Nutella, `3760341070472` = riz basmati, `3500330321013` = valide mais introuvable). La saisie manuelle vérifie le chiffre de contrôle (`gtinOk`), ou injecter une image d'EAN-13 dans `#scphoto`. Pour tester le chemin iOS sur Chrome, neutraliser `window.BarcodeDetector` avant le premier scan. La vraie lecture caméra ne se teste que sur téléphone, en HTTPS.

## Points connus

- CORS non vérifié : l'appel direct à `api.deepseek.com` depuis le navigateur n'a pas encore été confirmé. En cas d'échec réseau, l'appli suggère OpenRouter (`https://openrouter.ai/api/v1`). Si DeepSeek bloque, la solution propre est un petit proxy (Cloudflare Worker, par exemple) qui garde la clé côté serveur.
- Clé API : elle est stockée en clair dans le `localStorage` du téléphone. Acceptable pour un usage personnel. Ne jamais la mettre dans le code ni dans le dépôt.
- Pas de service worker : pas de mode hors ligne. L'analyse a de toute façon besoin du réseau.
- Données sur un seul appareil : la seule sauvegarde est l'export manuel.

## Pistes d'évolution

- Rappel d'export automatique, sauvegarde hors de l'appareil.
- Vérification 4/4/9 des aliments renvoyés par l'IA, table Ciqual intégrée, recherche d'aliment par nom.
- « Annuler » à la place de la double confirmation pour la suppression d'un repas.
