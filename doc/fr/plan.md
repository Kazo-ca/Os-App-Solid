> [English version](../en/plan.md)

# Plan : l'app Kazo Solid pour Kazo OS

Étape 0 du [guide de portage des apps Craft](https://github.com/Kazo-ca/Os/blob/main/doc/fr/wasm/dev/craft-app-port-guide.md)
de Kazo OS, écrite le 2026-11-14. Ce document fixe le but, ce que la lecture et l'essai de
SolidCraft ont montré, les décisions, les questions posées à Benoit et les chantiers. Chaque
chantier est écrit pour être confié tel quel à un fil d'implémentation. Les références les plus
proches sont Kazo Deck
([plan](https://github.com/Kazo-ca/Os-App-Deck/blob/deck/base/doc/fr/plan.md), une fenêtre par
document) et Kazo Fig ([plan](https://github.com/Kazo-ca/Os-App-Fig/blob/main/doc/fr/plan.md),
build amont livré sans modification).

## But

Kazo Solid est **l'app de conception 3D paramétrique (CAO) de Kazo OS** : esquisses contraintes,
extrusions, révolutions, congés, assemblages, tôlerie, sur une ligne de temps modifiable. Il
emballe le build web officiel de [SolidCraft](https://github.com/tangency-labs/solidcraft)
(Tangency Labs, Rust et egui, version 0.41) et le branche sur les fichiers, les dialogues et les
associations de Kazo OS. Aucune app de Kazo OS ne fait de CAO aujourd'hui.

**Ce qui distingue ce portage des autres** :

- **La licence.** Le code de géométrie est `MIT OR Apache-2.0`, mais l'interface et la marque
  sont sous une licence à part (`LICENSE-BRAND`). On ne peut pas distribuer de build modifié
  sans permission écrite. Kazo Solid livre donc **le build officiel, octet pour octet**, et rien
  ne va au Store avant la réponse écrite des mainteneurs (voir « Licence »).
- **Pas de mode hébergé Craft.** SolidCraft n'a pas `craft-web-host`. Il a son propre mode,
  `?embed=editor`, où la page hôte garde les fichiers et les échange par trois méthodes de
  contrôle. Un petit script de notre page fait le pont avec le module commun de Kazo OS.
- **La taille.** Le `.wasm` fait 159,6 Mio : au-dessus de la limite de 100 Mio de GitHub et de
  la limite de 128 Mio par fichier de Kazo OS. Il est livré compressé (49 Mio) et décompressé
  par la page au démarrage.
- **Pas de menu Fichier.** En mode `embed=editor`, SolidCraft n'a ni barre d'application ni
  page d'accueil : Ouvrir et Enregistrer sont à la charge de la fenêtre Kazo.

Le code Kazo commun (iframe, proxy, sélecteur de fichiers, raccourcis, `DocumentSession`,
exports) vient du module `Kazo.Os.AppSdk.Craft` de Kazo OS
([ADR 0259](https://github.com/Kazo-ca/Os/blob/main/doc/fr/wasm/dev/adr/0259-shared-craft-app-host-module.md),
[ADR 0264](https://github.com/Kazo-ca/Os/blob/main/doc/fr/wasm/dev/adr/0264-craft-app-exports-reach-the-window.md),
[guide](https://github.com/Kazo-ca/Os/blob/main/doc/fr/wasm/framework/craft-apps.md)). Version
minimale : **Kazo OS 3.3.4**, déjà publiée.

## Licence

Lue le 2026-11-14 dans `LICENSE-BRAND`, `NOTICE` et l'historique git de l'amont.

| Point | Constat |
| --- | --- |
| Code | `MIT OR Apache-2.0` (`LICENSE-MIT`, `LICENSE-APACHE`). Noyau de géométrie `solidcore` (Apache-2.0), intégré au dépôt. |
| Marque et interface | `LICENSE-BRAND`, ajoutée le 2026-11-10 (commit `312454f89`, première version : `v0.40.0`). Elle couvre `crates/ui-kit`, `crates/look`, `crates/theme`, et le nom, le logo et l'icône SolidCraft. |
| Permis sans demander | Lire, compiler, modifier dans SolidCraft lui-même, y compris des forks pour développer, tester, apprendre ou contribuer (art. 2.a). Montrer SolidCraft dans des articles et des tutoriels (art. 2.c). |
| Demande une permission écrite | Utiliser ces crates dans un autre programme (3.a) ; les distribuer autrement que comme partie de SolidCraft (3.b) ; distribuer un fork ou un build modifié sous le nom, le logo ou l'icône SolidCraft, ou avec une apparence qui peut passer pour celle de SolidCraft (3.c) ; utiliser le nom ou le logo pour nommer autre chose (3.d). |
| Conséquence | **Changer le nom ne suffit pas** : un build modifié renommé est « un autre programme » qui utilise le kit d'interface. |
| Versions d'avant | L'article 5 laisse les versions publiées avant la licence sous `MIT OR Apache-2.0` (dernière : `v0.40.0-rc.3`, commit `7001949c4`, 2026-11-07). Voie écartée par Benoit : elle fige l'interface et va contre l'intention des mainteneurs. |
| Auto-hébergement | L'amont le prévoit : zip `solidcraft-web-<version>.zip` dans chaque release, image `ghcr.io/tangency-labs/solidcraft-web`, et le mode `?embed=editor` documenté pour « une page qui garde les designs elle-même ». |
| Nom « Fusion » | Le `NOTICE` amont rappelle que Fusion est une marque d'Autodesk pour un logiciel de CAO. Le nom « Kazo Fusion » a été écarté pour cette raison. |

**Voie choisie par Benoit le 2026-11-14** : demander la permission écrite aux mainteneurs **et**
bâtir pendant ce temps sur le build officiel non modifié. La demande (texte dans
`solidcraft/upstream/permission-request.md` des fichiers du projet, publiée par Benoit comme
issue chez `tangency-labs/solidcraft`) demande la permission d'offrir le build officiel dans
Kazo OS et laisse aux mainteneurs le choix du nom de la fiche : « Kazo Solid, propulsé par
SolidCraft » avec notre icône, ou « SolidCraft » avec leur nom et leur icône.

**Règles qui en découlent pour toute la série** :

1. Aucun build modifié de SolidCraft n'est distribué. Le `.js` et le `.wasm` livrés ont les
   SHA-256 du `build.json` de la release amont ; un test le vérifie.
2. Le fork `Kazo-ca/solidcraft` sert seulement à lire, essayer et proposer des changements en
   amont. Rien de ce qu'il contient n'entre dans le paquet.
3. Un besoin d'intégration qui demande un changement dans SolidCraft devient une issue ou une
   PR en amont (publiée par Benoit), pas un patch livré.
4. **Rien n'est publié au Store avant la permission écrite.** Le chantier 4 attend la réponse.

## Lecture de SolidCraft (étape 0, 2026-11-14)

Lue sur `main` amont (version 0.42.0 en cours) et à l'étiquette `v0.41.0` (2026-11-12). Le fork
[`Kazo-ca/solidcraft`](https://github.com/Kazo-ca/solidcraft) existe.

| Point | Constat |
| --- | --- |
| Build web | `apps/solidcraft-web` (Trunk 0.21.14, `trunk build --release --public-url ./`), eframe sur wgpu : WebGPU, ou WebGL 2 avec `?webgl`. Rust 1.95 au minimum. Zip officiel fait par `packaging/web/package.sh`. |
| Mode `?embed=editor` | Ruban, navigateur de design, ligne de temps et propriétés ; **pas** de barre d'application ni de page d'accueil ; **rien** n'est gardé dans le navigateur. `Ctrl+N`, `Ctrl+O`, `Ctrl+S`, `Ctrl+W` et `Ctrl+,` n'y font rien (« ils appartiennent à la page ») ; `Ctrl+Z` et `Ctrl+Y` annulent et rétablissent. Un design par cadre, pas d'onglets. |
| API de commandes | `window.solidcraft` dans tous les modes (`request`, `execute`, `commands`, `inspect`, `buttons`). La page hôte **de même origine** l'atteint par `frame.contentWindow.solidcraft`. 642 commandes. Paramètres : 16 Mio de JSON au plus ; 256 requêtes en attente au plus. |
| Échange de designs | `document.export` → `{ name, revision, dirty, format: "json", text }` (lecture seule) ; `document.saved { revision }` → le design compte comme enregistré à cette révision, refusé s'il a changé depuis ; `document.import { text, name? }` → remplace le design actif, sans demander pour les modifications en cours (c'est à l'hôte de décider). |
| Suivre les changements | `document.revision` → `{ document_id, revision }` : ne fait que lire, n'attend pas derrière une commande lente ; `revision` bouge à chaque changement du design, jamais pour une sélection ou une vue. Pas de message poussé vers la page. |
| Format des fichiers | Design `.sc` (`.solidcraft` des versions d'avant s'ouvre encore). Deux formes : **conteneur** (un ZIP : le JSON du design, des charges binaires `blobs/<sha256>`, des caches) ou **JSON** seul. SolidCraft reconnaît un conteneur à ses premiers octets (`PK`), quel que soit le nom. `document.export` rend le JSON. |
| Imports | Ruban › Insert : STEP, IGES, maillages (3MF, STL, OBJ, PLY, glTF), nuages de points, images, DXF, SVG, par le sélecteur de fichiers du navigateur (File System Access, ou `<input type=file>`), ou par dépôt sur la page. Les commandes `file.insert_step` et `file.insert_mesh` prennent un **chemin** du système de fichiers virtuel, pas des octets. |
| Exports | Commandes `file.export` (STL, OBJ, STEP, 3MF, glTF, URDF, JSON), `file.export_step`, `file.export_stl`, `file.export_3mf`, `file.export_obj`, `file.export_view_dxf`, par chemin. Dans le build web, un chemin sous `/downloads/` devient un téléchargement. |
| Thème | `ui.set { dark: true\|false }`. 27 thèmes dans les préférences (`ui.prefs`, clé `theme`). Sombre par défaut. |
| Langue | Préférence `language` (`system` par défaut) par `ui.prefs`. |
| Réseau | Vérification des mises à jour (`check_updates`), fonctions d'IA (`ai_features`, agents), envoi vers Printables (`file.open_in_easyprint`, avec consentement). À couper ou à signaler dans la fiche. |
| Service worker | L'`index.html` officiel enregistre `sw.js` (hors ligne et mises à jour). À ne pas enregistrer dans Kazo OS, qui a le sien. |
| Canevas | `solidcraft_canvas` et `#solidcraft_loading` : une instance par document HTML, donc une iframe par fenêtre. |
| Marque | Icônes de l'app dans `icons/`, logo de chargement `loading-logo.svg` : matériel de marque. Non livrés tant que les mainteneurs n'ont pas répondu. |
| Règles du dépôt amont | Rust seulement, pas de panique hors tests, tout est une commande, ne jamais casser wasm, pas de mention d'agent, une issue avant une PR. |

### Essai (2026-11-14)

Fait avec le zip officiel `solidcraft-web-0.41.0.zip` de la release amont
(SHA-256 `dba9a9389bc479c0c12412493b1bec61d4d4d8434cd4e4cca67ad2a38183fc04`, 56,8 Mo), sans aucun
changement, dans Chromium sans carte graphique (WebGL 2 par SwiftShader, `?webgl`), dans un cadre
de même origine avec `?embed=editor`.

| Point | Résultat |
| --- | --- |
| Contenu | `index.html`, un `.js` de 193 Ko, un `.wasm` de **167 302 762 octets (159,6 Mio)**, polices, icônes, licences, `build.json` (SHA-256 de chaque fichier). 176 Mo décompressé. `gzip -9` du `.wasm` : **51 373 954 octets (49,0 Mio)**. |
| Limites | GitHub : 100 Mio par fichier, **dépassé** par le `.wasm` brut. Kazo OS : 128 Mio par fichier et paquet de 128 Mio décompressé, **dépassés** ; paquet de 64 Mio compressé, tenu avec le `.wasm` compressé (environ 52 Mio au total). |
| Démarrage | `window.solidcraft` présent 2,2 s après le chargement de la page, depuis un serveur local. Design vide « Untitled », thème sombre. |
| Notre page avec `.wasm` compressé | Une page à nous qui charge le `.js` officiel et lui donne le `.wasm` par `DecompressionStream('gzip')` (`init({ module_or_path: Response })`) : **démarre en 2,0 s**, même comportement, **aucun service worker** enregistré. Le `.js` et le `.wasm` restent ceux de l'amont, octet pour octet. |
| Échange | Boîte créée par `solid.box` : `document.export` rend `dirty: true`, `format: "json"`, 539 caractères commençant par `"format": "solidcraft/55"`. `document.saved { revision }` : `dirty: false`. `document.import { text, name: "Bracket" }` : design remplacé, nouveau `document_id`, `errors: 0`. |
| Thème | `ui.set { dark: false }` : interface claire. |
| Stockage | `localStorage`, IndexedDB et OPFS vides après l'essai. |
| `Ctrl+S` | Ne fait rien dans le cadre, comme annoncé. |
| Exports par commande | `file.export { path: "/downloads/part.step" }` et `file.save_as { path: "/downloads/part.sc" }` répondent ok (9 745 et 10 199 octets) mais **aucun téléchargement n'est sorti** en mode `embed=editor` pendant l'essai. Aucune méthode ne rend les octets d'un fichier virtuel. |
| Fenêtre | Le ruban est lisible à 1280 × 820. |

Captures et scripts de l'essai : `solidcraft/trials/` dans les fichiers du projet (hors dépôt).

**Non vérifié à cette étape** : le rendu WebGPU (pas de carte graphique ici) ; la mémoire par
fenêtre ; le ruban › Insert de bout en bout (sélecteur de fichiers remplacé par celui de l'OS) ;
un moyen de sortir un export (STEP, STL) en mode `embed=editor` ; si un JSON de design nommé
`.sc` s'ouvre dans SolidCraft de bureau (lu dans le code : oui, par les premiers octets) ; ce que
`document.export` fait des corps importés (dans le JSON ou dans des charges à part) ; un grand
design proche de la limite de 16 Mio.

## Décisions

| # | Sujet | Décision |
| --- | --- | --- |
| 1 | Nom et identité | « **Kazo Solid** » (Benoit, 2026-11-14). Id `ca.kazo.solid`, éditeur `pub.kazo`, assembly `Kazo.Solid`, catégorie de Kazo Vector (graphisme), plusieurs fenêtres, fenêtre de 1280 × 820, contenu minimum 800 × 520. Le nom de la fiche peut devenir « SolidCraft » si les mainteneurs le demandent (voir « Licence »). |
| 2 | Code commun | Module `Kazo.Os.AppSdk.Craft`. Sous-module `external/KazoOs` sur l'étiquette `v3.3.4`, paquets de nuget.org en Release, `minimumVersion` 3.3.4. |
| 3 | Changement d'écran | `canChangeScreen: false` dans le manifeste dès le chantier 2 (ADR 0263), comme les autres apps Craft. |
| 4 | Provenance du build | **Le build officiel, non modifié** : `tools/fetch-solidcraft.sh` télécharge `solidcraft-web-<version>.zip` depuis la release de `tangency-labs/solidcraft`, vérifie son SHA-256 (épinglé dans le script), puis les SHA-256 du `build.json`. Pas de compilation Rust, pas de sous-module du fork. |
| 5 | Taille | Le `.wasm` est rangé **compressé** (`gzip -9`, un seul fichier d'environ 49 Mio) dans `wwwroot/solidcraft/`, à côté du `.js` officiel et d'un `solidcraft.json` (version, URL et SHA-256 du zip, noms, tailles et SHA-256 avant et après compression). Dans le dépôt : le Store compile l'app au commit de l'étiquette et ne prend aucun binaire de l'auteur. Aucune limite de Kazo OS à changer, donc **pas d'ADR dans `Kazo-ca/Os`**. Soumise à Benoit (Q3). |
| 6 | Page de l'iframe | Une page à nous, `wwwroot/host/index.html` : canevas `solidcraft_canvas`, `#solidcraft_loading`, et un module `boot.js` qui décompresse le `.wasm` et démarre le `.js` officiel. Paramètres `?embed=editor` toujours, `webgl` en repli. Elle n'enregistre pas `sw.js` et ne livre ni le manifeste web ni les icônes de l'amont. |
| 7 | Pont avec le module | Un script de la page, `wwwroot/host/solid-bridge.js` (JavaScript minimal et commenté, pas de script en ligne), traduit l'API `window.solidcraft` vers les messages qu'attend `CraftFrame` : `ready`, `open { name, bytes }` → `document.import`, `save { name, saveAs, bytes }` ← `document.export`, `dirty`. Le module C# est réutilisé sans changement. S'il manque un point d'extension, c'est une demande pour `Kazo-ca/Os`, dans un fil à part. |
| 8 | Format enregistré | Le **JSON du design** (`document.export`), écrit en UTF-8 dans un fichier `.sc`. SolidCraft de bureau le reconnaît à son contenu. À vérifier au chantier 3 avec SolidCraft de bureau si possible. |
| 9 | Ouvrir un conteneur | Un `.sc` ou `.solidcraft` enregistré par SolidCraft de bureau est un ZIP. La fenêtre le reconnaît (`PK`), en lit le JSON du design en C# et le passe à `document.import`. Si le design a besoin de charges binaires que `document.import` ne peut pas recevoir, la fenêtre le dit clairement et n'ouvre rien. Comportement exact relevé au chantier 3. |
| 10 | « Modifié » et enregistré | Le pont lit `document.revision` après chaque geste vrai de l'utilisateur (touche, clic, molette), jamais sur une minuterie libre, et envoie `dirty` quand la révision s'éloigne de la dernière enregistrée. Il n'appelle `document.saved { revision }` qu'**après** que la fenêtre a confirmé l'écriture : un dialogue annulé ou une écriture refusée laisse le design modifié. |
| 11 | Un design par fenêtre | SolidCraft n'a pas d'onglets en mode `embed=editor`. Un double-clic, Ouvrir et `Ctrl+N` ouvrent une **nouvelle fenêtre Kazo Solid**, sauf si la fenêtre est encore vide et intacte : elle prend le fichier. Comme Kazo Deck. |
| 12 | Commandes de fichier | Soumis à Benoit (Q4) : une petite barre de menus Kazo au-dessus de SolidCraft, ou les raccourcis seulement. |
| 13 | Raccourcis | `ReservedKeys.Os`. `Ctrl+N`, `Ctrl+O`, `Ctrl+S`, `Ctrl+Maj+S` vont à la fenêtre (SolidCraft les laisse à la page). Les conflits avec les accords de l'OS sont relevés au chantier 3 et notés dans la fiche. |
| 14 | Garde de fermeture | Oui / Non / Annuler si la révision n'est pas celle enregistrée. Oui enregistre et attend la fin de l'écriture. `document.import` ne demande rien : la fenêtre pose la question avant de remplacer un design modifié. |
| 15 | Thème et langue | `ui.set { dark }` à `ready` et à chaque changement de thème de l'OS. Langue de l'OS par `ui.prefs { language }` si SolidCraft a la langue ; sinon `system`. |
| 16 | Réglages et récupération | Rien n'est gardé dans le navigateur en mode `embed=editor` : pas de `WebStorageFile`, **pas de récupération après plantage dans la première version**, avec une issue `TODO` pour la suite (même décision que Kazo Vector, Photo et Fig). |
| 17 | Réseau | Vérification des mises à jour coupée à `ready` (`ui.prefs { check_updates: false }`) : la version vient du paquet. Les fonctions d'IA et l'envoi vers Printables restent ce que SolidCraft en fait ; leur état en mode `embed=editor` est relevé au chantier 3 et écrit dans la fiche. |
| 18 | Imports et exports | Les imports passent par le ruban › Insert de SolidCraft, avec le sélecteur de fichiers de l'OS à la place de celui du navigateur (`PickRequested`). Les exports : relevés au chantier 3 ; si rien ne sort en mode `embed=editor`, c'est une limite de la première version, écrite dans la fiche, et une demande en amont (déjà posée dans la demande de permission). |
| 19 | Associations | Soumises à Benoit (Q5). |
| 20 | Icône et marque | Icône propre à Kazo Solid, sans le logo SolidCraft, et écran de chargement à nous, tant que les mainteneurs n'ont pas choisi. Fiche : « propulsé par SolidCraft », lien vers `solidcraft.io` et vers le dépôt amont. |
| 21 | Piste amont | Brouillons en anglais, sans mention d'agent, dans `solidcraft/upstream/` des fichiers du projet ; Benoit les publie. |
| 22 | ADR de Kazo OS | Aucune prévue. Si une réponse en demande une (Q3 = B), prochain numéro libre **0272**, à revérifier sur `main`, les PR ouvertes et toutes les branches `*/base` de `Kazo-ca/Os` au moment de l'écrire. |

## Questions posées à Benoit

Déjà répondues le 2026-11-14 : la voie pour la licence (permission demandée **et** build officiel
non modifié) et le nom (Kazo Solid, pas Kazo Fusion).

Posées dans le fil le 2026-11-14, **en attente de réponse** :

**Q3. Comment livrer un `.wasm` de 160 Mio ?** Il dépasse la limite de 128 Mio par fichier de
Kazo OS et celle de 100 Mio de GitHub.

- **A (recommandée)** : le `.wasm` compressé dans le paquet (49 Mio), décompressé par notre page
  au démarrage. Essayé : démarre en 2 s. Rien à changer dans Kazo OS, l'app sort avec la 3.3.4.
  Coût : le paquet fait environ 52 Mio pour une limite de 64 Mio, donc peu de marge si SolidCraft
  grossit ; et une décompression à chaque ouverture de fenêtre.
- B : monter les limites de Kazo OS (fichier et paquet décompressé à 256 Mio) par une ADR dans
  `Kazo-ca/Os`. Sert à d'autres grosses apps, mais demande une nouvelle version de l'OS avant de
  sortir Kazo Solid, et le fichier reste trop gros pour GitHub (il faudrait le télécharger à la
  compilation).
- C : A maintenant, et une issue `TODO` pour B si le paquet approche 64 Mio.

**Q4. Où mettre Ouvrir et Enregistrer ?** En mode `embed=editor`, SolidCraft n'a pas de menu
Fichier.

- **A (recommandée)** : une petite barre de menus Kazo au-dessus de SolidCraft (Fichier :
  Nouveau, Ouvrir, Enregistrer, Enregistrer sous), plus les raccourcis `Ctrl+N`, `Ctrl+O`,
  `Ctrl+S`, `Ctrl+Maj+S`. Tout est visible ; un peu d'interface à nous, hors du cadre de
  SolidCraft, donc sans toucher à leur apparence.
- B : les raccourcis seulement, plus la garde de fermeture. Rien à dessiner, mais un nouvel
  utilisateur ne voit pas comment enregistrer.

**Q5. Quelles associations de fichiers ?**

- **A (recommandée)** : app par défaut pour `.sc` et `.solidcraft` seulement. Les STEP, STL, 3MF
  et autres s'importent par le ruban › Insert dans un design ouvert ; pas d'association dans la
  première version, parce que SolidCraft ne reçoit pas leurs octets par commande. Une issue
  `TODO` pour les ouvrir par double-clic quand l'amont le permettra.
- B : comme A, plus « Ouvrir avec » pour `.json` (un design SolidCraft peut être un `.json`).
  Utile à peu de monde, et l'éditeur de texte reste l'app par défaut des `.json`.
- C : attendre que l'amont offre l'ouverture de STEP et de maillages par commande avant de
  sortir l'app. Fiche plus complète, mais date de sortie inconnue.

## Chantiers

Un dépôt de travail, `Kazo-ca/Os-App-Solid` : branche d'intégration `solid/base`, une branche et
une PR par chantier, un fil d'implémentation par chantier. Benoit fusionne `solid/base` dans
`main`.

| N° | Branche (vers `solid/base`) | Contenu | Dépend de |
| --- | --- | --- | --- |
| 1 | `solid/1-plan` | ce plan (fr et en) | — |
| 2 | `solid/2-solution` | solution, manifeste, build officiel récupéré et vérifié, page d'iframe, fenêtre qui démarre | plan fusionné, Q3 |
| 3 | `solid/3-window` | pont, ouvrir, enregistrer, garde de fermeture, menus, raccourcis, thème, imports et exports | 2, Q4 et Q5 |
| 4 | `solid/4-store` | fiche, icône, captures, README, licences, mesures, parcours installé, catalogue, `features.md` | 3 **et la permission écrite des mainteneurs** |

### 2. Solution et build officiel (`solid/2-solution`)

- Solution renommée (`Kazo.Solid`, `Kazo.Solid.Os`, `Kazo.Solid.Tests`, `tests/e2e`),
  `[AppManifest]` de la décision 1, `canChangeScreen: false`, capacités de Kazo Deck,
  associations selon Q5.
- `tools/fetch-solidcraft.sh` (décisions 4 et 5) : télécharge le zip de la version épinglée
  (`0.41.0`, ou la dernière release au moment du chantier), vérifie les SHA-256, compresse le
  `.wasm`, écrit `wwwroot/solidcraft/` et `solidcraft.json`. Ne copie ni `sw.js`, ni
  `manifest.webmanifest`, ni `icons/`, ni `loading-logo.svg` ; copie les licences
  (`LICENSE-MIT`, `LICENSE-APACHE`, `LICENSE-BRAND` si présent, `NOTICE`,
  `THIRD-PARTY-LICENSES.txt`) et les polices. Workflow `fetch-solidcraft.yml` qui relance le
  script et échoue si le résultat diffère du dépôt.
- Tests xUnit : les SHA-256 du `.js` et du `.wasm` décompressé sont ceux du `build.json`
  amont ; les tailles tiennent sous les limites de Kazo OS.
- Page d'iframe et `boot.js` (décision 6), fenêtre sur `CraftFrame` et `FrameStartup` :
  chargement, app, erreur, « Réessayer » ; thème à `ready`. Modèle de `boot.js` :
  `solidcraft/trials/boot.js` dans les fichiers du projet.
- **Terminé quand** : l'app démarre en WebGL 2 (et en WebGPU sur une machine qui l'a), deux
  fenêtres tournent ensemble, le `.kazoapp` s'installe dans le Kazo OS 3.3.4 publié ; taille du
  paquet, temps de démarrage et mémoire par fenêtre notés ici.

### 3. Fenêtre (`solid/3-window`)

- `solid-bridge.js` (décisions 7 et 10) avec ses tests (`npm run test:unit`, page simulée).
- Ouverture : fichier demandé au lancement, Ouvrir, double-clic ; JSON et conteneur
  (décisions 8 et 9) ; fenêtre ou nouvelle fenêtre (décision 11).
- Enregistrement : Enregistrer, Enregistrer sous, premier enregistrement d'un design sans
  fichier ; un dialogue annulé ou une écriture refusée laisse le design modifié, avec un message.
- Titre `*nom — Kazo Solid`, garde de fermeture (décision 14), menus selon Q4, raccourcis
  (décision 13), thème suivi en direct, langue (décision 15), mises à jour coupées (décision 17).
- Imports par le ruban › Insert avec le dialogue de l'OS ; exports : relever ce qui sort en
  mode `embed=editor` (décision 18) et poser la question à Benoit si plusieurs formes se valent.
- Mesures : un design de plusieurs Mo de JSON (temps d'ouverture et d'enregistrement, mémoire),
  et ce qui arrive près de la limite de 16 Mio.
- **Terminé quand** : aucun cas ne perd de modifications sans que l'utilisateur l'ait choisi,
  aucun dialogue ni téléchargement du navigateur ne sort de l'app, et les raccourcis de l'OS
  marchent le curseur dans le modèle.

### 4. Store (`solid/4-store`)

- **Ne démarre qu'avec la permission écrite des mainteneurs**, et suit leurs conditions (nom,
  icône, mention, version).
- Icône et fiche fr et en (« propulsé par SolidCraft » ; limites connues : pas de récupération
  après plantage, imports par le ruban, exports selon le chantier 3, 16 Mio par design,
  fonctions réseau, fenêtre qui reste sur son écran), captures prises dans le Kazo OS publié,
  README refait sur la fiche (`store:readme`, « Essayer » avec `?autolaunch=ca.kazo.solid`),
  `THIRD-PARTY-NOTICES` fr et en (SolidCraft, `solidcore`, `LICENSE-BRAND` et la permission
  reçue, polices).
- Parcours dans le Kazo OS publié (`tests/installed/`), workflow « Tag a release », PR au
  catalogue `Os-AppMarketPlace` (la signature se fait là, aucune clé ici), ligne de
  `features.md` dans `Kazo-ca/Os` (fr et en).
- **Terminé quand** : Kazo Solid s'installe depuis le Store, ouvre un `.sc` par double-clic, le
  modifie et l'enregistre.

## Piste en amont

Publiée par Benoit, jamais par un fil :

- la demande de permission (`solidcraft/upstream/permission-request.md`), qui pose aussi deux
  questions : la stabilité de `?embed=editor`, et un moyen de passer des octets (STEP, 3MF, STL)
  dans les deux sens en mode `embed=editor` ;
- à écrire après le chantier 3, selon ce qu'il relève : l'ouverture d'un conteneur `.sc` par
  `document.import`, un message poussé quand le design change, les exports en mode
  `embed=editor`.

## Avancement

- [x] Étape 0 : lecture de la licence et de SolidCraft 0.41, essai du build web officiel en mode `embed=editor`.
- [x] Réponses de Benoit sur la licence et le nom (2026-11-14).
- [x] Plan fr et en.
- [ ] Demande de permission publiée par Benoit ; réponse des mainteneurs.
- [ ] Réponses de Benoit à Q3, Q4 et Q5.
- [ ] Chantier 2 : solution, build officiel, page d'iframe, fenêtre.
- [ ] Chantier 3 : fenêtre, ouverture, enregistrement, garde de fermeture.
- [ ] Chantier 4 : Store (après la permission écrite).
- [ ] Sortie : `solid/base` dans `main` (Benoit), étiquette `v0.1.0`, catalogue, `features.md`.
