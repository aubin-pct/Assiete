# CLAUDE.md — Assiette

Application web personnelle de suivi nutritionnel. L'utilisateur photographie son repas, un modèle de vision (DeepSeek V4.1 Flash par défaut) estime les aliments, leurs grammes et leurs macros, puis l'utilisateur corrige avant d'enregistrer. Usage mono-utilisateur, sur téléphone, installée via « Ajouter à l'écran d'accueil ».

## Structure

```
index.html            # toute l'appli : HTML, CSS et JS inline
manifest.webmanifest  # PWA : nom, couleurs, icônes, display standalone
icon-192.png          # icône (aussi apple-touch-icon et favicon)
icon-512.png          # icône maskable
```

Pas de build, pas de dépendances, pas de framework. Vanilla JS en `"use strict"`. Seule ressource externe : les polices Google Fonts (Sora pour les titres et chiffres, Figtree pour le texte), avec repli sur les polices système.

Garder ce principe : un seul fichier HTML autonome, déployable tel quel sur un hébergement statique. N'ajouter une bibliothèque que si elle fait un vrai travail, et alors en `<script>` depuis un CDN avec version figée.

## Déploiement

GitHub Pages depuis la branche `main`, à la racine du dépôt. Aucune étape de build : pousser les fichiers suffit. Tous les chemins sont relatifs (`start_url: "./"`), l'appli fonctionne donc dans un sous-dossier `tonpseudo.github.io/depot/`.

## Architecture du JS

- État global : `S` (données persistées), `view` (onglet courant), `curDate` (jour affiché, `YYYY-MM-DD` en heure locale), `draft` (repas en cours de saisie), `armedDelete` (suppression en attente de confirmation).
- Rendu : `render()` remplace entièrement `#app.innerHTML` avec la vue courante (`journalHTML`, `addHTML`, `historyHTML`, `settingsHTML`), puis appelle `bind()`, qui rattache tous les écouteurs. Tout nouvel élément interactif doit être branché dans `bind()`.
- Exception au re-rendu complet : dans la liste d'aliments du brouillon, la saisie met à jour les champs voisins et `#totals` directement, sans `render()`, pour ne pas perdre le focus du clavier. Conserver ce comportement.
- Navigation : barre du bas `nav.tabs` avec `data-view`, et `go(view)`.

## Données (`localStorage`, clé `assiette.v1`)

```js
{
  settings: {
    apiKey: "", baseUrl: "https://api.deepseek.com", model: "deepseek-flash", deep: false,
    goals: { kcal: 3000, p: 160, c: 380, f: 90 }
  },
  meals: [{
    id, date: "YYYY-MM-DD", time: "HH:MM",
    type: "Petit-déjeuner" | "Déjeuner" | "Collation" | "Dîner",
    name, items: [{ nom, g, kcal, p, c, f }],
    thumb   // data URL JPEG 160×160, ou null
  }],
  weights: [{ date: "YYYY-MM-DD", kg }]   // une pesée max par jour
}
```

- Au chargement, les réglages sont fusionnés avec `DEFAULTS` : on peut ajouter un champ de réglage sans migration.
- Si le schéma change de façon incompatible, passer à `assiette.v2` et écrire une migration depuis `v1`. Ne jamais perdre les données existantes.
- `save()` renvoie `false` si le quota est dépassé (environ 5 Mo). Les miniatures sont le poste le plus lourd. La photo pleine taille n'est jamais stockée.
- L'export JSON exclut volontairement `apiKey`. L'import fusionne (dédoublonnage par `id` pour les repas, par `date` pour les pesées) et ne remplace pas.

## Appel au modèle (`callModel`)

- `POST {baseUrl}/chat/completions`, au format OpenAI (compatible DeepSeek et OpenRouter).
- Message `system` : constante `SYSTEM`, en français. Message `user` : texte, plus l'image en `image_url` sous forme de data URL. DeepSeek refuse les images hors des messages `user`.
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

## Points connus

- CORS non vérifié : l'appel direct à `api.deepseek.com` depuis le navigateur n'a pas encore été confirmé. En cas d'échec réseau, l'appli suggère OpenRouter (`https://openrouter.ai/api/v1`). Si DeepSeek bloque, la solution propre est un petit proxy (Cloudflare Worker, par exemple) qui garde la clé côté serveur.
- Clé API : elle est stockée en clair dans le `localStorage` du téléphone. Acceptable pour un usage personnel. Ne jamais la mettre dans le code ni dans le dépôt.
- Pas de service worker : pas de mode hors ligne. L'analyse a de toute façon besoin du réseau.
- Données sur un seul appareil : la seule sauvegarde est l'export manuel.

## Pistes d'évolution

- Bouton « refaire ce repas » et repas favoris, pour ne pas ré-analyser les plats habituels.
- Modifier un repas déjà enregistré (aujourd'hui, on ne peut que le supprimer).
- Courbe de poids avec moyenne mobile sur 7 jours.
- Scan de code-barres via l'API Open Food Facts.
