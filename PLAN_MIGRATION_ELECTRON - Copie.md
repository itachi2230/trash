# 🚀 DataEdge — Plan de Migration vers Electron

> **Document de référence pour la réécriture complète de DataEdge** (WPF .NET Framework 4.7.2 → Electron + TypeScript/React)
> **Objectif : un seul code source produisant un `.dmg` (macOS) et un `.exe` (Windows), installables, fonctionnant hors ligne.**
>
> **Version du document :** 1.0 — créé le 2026-10-31
> **Périmètre :** 100 % du logiciel client (`backtest/`). Le backend Symfony (`fxglobal/`) est **hors périmètre** (inchangé).
> **Statut :** Plan approuvé pour exécution.

---

## 📑 Sommaire

| # | Section |
|---|---|
| 0 | [Contexte, objectifs, périmètre](#0--contexte-objectifs-périmètre) |
| 1 | [Décisions d'architecture (ADR)](#1--décisions-darchitecture-adr) |
| 2 | [Inventaire exhaustif du logiciel actuel](#2--inventaire-exhaustif-du-logiciel-actuel) |
| 3 | [Cartographie de migration](#3--cartographie-de-migration) |
| 4 | [Architecture cible Electron](#4--architecture-cible-electron) |
| 5 | [Spécification du bridge Chart](#5--spécification-du-bridge-chart) |
| 6 | [Spécification du module MT5](#6--spécification-du-module-mt5) |
| 7 | [Spécification de l'agent IA](#7--spécification-de-lagent-ia) |
| 8 | [Spécification du module Space](#8--spécification-du-module-space) |
| 9 | [Migration du format `.etude`](#9--migration-du-format-etude) |
| 10 | [Stockage & chemins par OS](#10--stockage--chemins-par-os) |
| 11 | [Sécurité](#11--sécurité) |
| 12 | [Packaging & distribution](#12--packaging--distribution) |
| 13 | [CI/CD GitHub Actions](#13--cicd-github-actions) |
| 14 | [Plan de test & parité fonctionnelle](#14--plan-de-test--parité-fonctionnelle) |
| 15 | [Plan par phases](#15--plan-par-phases) |
| 16 | [Plan d'apprentissage](#16--plan-dapprentissage) |
| 17 | [IDE & outillage](#17--ide--outillage) |
| 18 | [Risques & mitigations](#18--risques--mitigations) |
| 19 | [Décisions ouvertes](#19--décisions-ouvertes) |
| 20 | [Annexes](#20--annexes) |

---

## 0 — Contexte, objectifs, périmètre

### 0.1 Situation actuelle

**DataEdge** est une application **WPF .NET Framework 4.7.2** de backtesting et journal de trading Forex/CFD.

| Métrique | Valeur |
|---|---|
| Fichiers C# | **89** (~29 500 lignes) |
| Fichiers XAML | **33** (~5 900 lignes, 346 Ko) |
| Fenêtres & vues | 19 fenêtres racine + 14 vues `Views/` |
| Services | 30 fichiers (`services/`) |
| Tools IA | **34** (20 workspace + 7 web/marché + 7 fichiers locaux) |
| Modules MT5 | 14 fichiers (`services/Mt5/`) |
| Modules Space | 12 fichiers (`services/Space/` + `Views/Space/`) |
| Formats de fichiers locaux | 16 |
| Endpoints serveur consommés | 30+ |
| Bibliothèque graphique | WebView2 + TradingView Lightweight Charts (HTML/JS) |

### 0.2 Le problème à résoudre

L'application est **Windows-only** et ne peut pas tourner sur macOS :

1. **WPF + .NET Framework 4.7.2** → Windows exclusivement.
2. **WebView2** (moteur du graphique) → **n'existe pas sur macOS**. Sous Wine/CrossOver, le runtime WebView2 ne s'installe pas → **le graphique reste vide alors que les données sont bien téléchargées**. C'est le symptôme rapporté par les utilisateurs.
3. **DPAPI** (`ProtectedData`) → Windows exclusivement.
4. **P/Invoke `user32.dll`** (`MaximizeHelper.cs`, `Mt5BridgeInstaller.cs`) → Windows exclusivement.
5. **`System.Management` (WMI)**, **`System.Drawing` (GDI+)**, **`System.Windows.Forms`** → Windows exclusivement.
6. **Toutes les données écrites à côté du `.exe`** (`AppDomain.CurrentDomain.BaseDirectory`) → incompatible avec un bundle macOS signé en lecture seule.
7. **Format `.etude` = XamlPackage WPF** → format propriétaire, non lisible hors WPF.
8. **Chaîne de distribution** : installeur Windows + `UpdateService` → aucune équivalent macOS.

### 0.3 Objectifs de la migration

| # | Objectif | Mesure de succès |
|---|---|---|
| **O1** | **Un seul code source** pour macOS et Windows | Un seul dépôt, une seule CI, aucun fork par plateforme |
| **O2** | **Installeurs natifs** | `.dmg` (macOS, signé + notarized) et `.exe` (Windows, NSIS) |
| **O3** | **Fonctionnement hors ligne** | L'app démarre et fonctionne sans aucune connexion réseau |
| **O4** | **Réutilisation maximale du graphique** | `resources/chart/*` réutilisé à ≥ 95 %, **même moteur Chromium sur les 2 OS** |
| **O5** | **Parité fonctionnelle 100 %** | Les ~200 points de la checklist §14 passent sur les deux OS |
| **O6** | **Aucune perte de données utilisateur** | Migration automatique des `.etude`, JSON, captures, caches |
| **O7** | **Maintenir la robustesse existante** | Écritures atomiques, `.bak`, verrous, anti-doublon MT5 conservés |
| **O8** | **Mises à jour simplifiées** | `electron-updater` : badge + téléchargement + installation |

### 0.4 Non-objectifs (hors périmètre)

- ❌ Modification du backend Symfony `fxglobal/` (sauf ajout **optionnel** de `POST /api/mt5/ingest` en phase 6+).
- ❌ Version web / navigateur / mobile (ce plan vise **uniquement** le desktop installable).
- ❌ Version Linux (possible quasi gratuitement plus tard, mais non testée ici).
- ❌ Changement du design visuel (on **reproduit** la charte actuelle, on ne la réinvente pas).
- ❌ Réécriture du MQL5 (`DataEdgeBridge.mq5` / `DataEdgeExport.mq5` restent **inchangés**).

### 0.5 Contraintes

| Contrainte | Impact |
|---|---|
| Le chart (`resources/chart/*`, 280 Ko de JS) est **l'actif le plus précieux et le plus fragile** | Règle : aucune modification du moteur de chart sans test de non-régression complet |
| `.etude` est un format **propriétaire WPF** avec des fichiers utilisateurs réels | Conversion obligatoire + `.bak` + tests d'aller-retour |
| MT5 sur macOS tourne **sous Wine** | Chemin de données à auto-détecter + réglage manuel |
| Codesign/notarization Apple = **99 $/an obligatoires** | Prérequis bloquant pour un `.dmg` utilisable |
| Windows Authenticode (~200-400 €/an) | Recommandé pour éviter SmartScreen |
| Il faut **un Mac et un PC** (ou du CI) pour builder | GitHub Actions avec matrix obligatoire |

---

## 1 — Décisions d'architecture (ADR)

> Chaque décision est figée sauf mention contraire. Les décisions marquées ⚠️ **À CONFIRMER** appliquent la recommandation par défaut en attendant validation.

### ADR-001 — Coquille native : **Electron**

| | |
|---|---|
| **Décision** | Utiliser **Electron** (et non Tauri, Flutter, MAUI, Avalonia, XPF). |
| **Raison 1** | Le chart tourne **déjà sur Chromium** (WebView2). Electron embarque Chromium → **même moteur sur macOS et Windows** → zéro risque de régression visuelle. Tauri utiliserait **WKWebView (Safari)** sur macOS, jamais testé avec `chart_engine.js`. |
| **Raison 2** | Node.js dans le main process → **portage quasi direct** du C# (MT5, Excel/Word/PDF, JSON local) avec des bibliothèques plus mûres. Tauri imposerait Rust. |
| **Raison 3** | **Hors ligne garanti par conception** : Chromium est embarqué, aucun runtime à télécharger. Tauri sur Windows exigerait le mode `offlineInstaller` (+180 Mo) pour rester hors ligne, annulant son avantage de taille. |
| **Raison 4** | L'écosystème Electron pour l'UI dense (DataGrid, éditeur riche) est le plus mature. |
| **Coût accepté** | Installeur ~90-120 Mo / ~250 Mo installés, RAM au repos ~180 Mo. Acceptable pour une application de trading. |
| **Alternative écartée** | Tauri : à reconsidérer **uniquement** si la taille de l'installeur devient un critère commercial et après validation du chart sur WKWebView. |

### ADR-002 — Langage : **TypeScript strict, partout**

| | |
|---|---|
| **Décision** | `strict: true` dans tous les `tsconfig.json`. Aucun fichier `.js` en dehors du chart existant. |
| **Raison** | TypeScript est **très proche de C#** (génériques, interfaces, classes, `async/await`, null-safety) → courbe d'apprentissage minimale. |
| **Exception** | `resources/chart/*.js` reste en JavaScript pur (réutilisé tel quel). Seul `bridge.ts` est nouveau. |

### ADR-003 — Outil de build : **electron-vite**

| | |
|---|---|
| **Décision** | `electron-vite` (Vite + esbuild) comme bundler. |
| **Raison** | HMR instantané sur le renderer, rebuild rapide du main, configuration en 3 cibles (`main` / `preload` / `renderer`), gestion automatique des assets. |

### ADR-004 — Framework UI : **React 19** ⚠️ **À CONFIRMER**

| | |
|---|---|
| **Décision** | **React 19** + TypeScript. |
| **Raison** | Écosystème le plus large pour les besoins de DataEdge : DataGrid denses, éditeur riche (TipTap), graphes, virtualisation de listes longues (feeds Space). |
| **Alternatives** | Svelte (plus simple à apprendre, écosystème plus petit) · Vue 3 (intermédiaire). |
| **Impact si changé** | Faible sur l'architecture (toutes les vues sont dans `renderer/features/`), élevé sur l'apprentissage. |

### ADR-005 — Gestion d'état : **Zustand + TanStack Query**

| | |
|---|---|
| **Décision** | **Zustand** pour l'état UI local (thème, filtres, sélection, onglets) · **TanStack Query** pour tout ce qui vient du serveur (cache, invalidation, retry, optimistic updates). |
| **Raison** | Zéro boilerplate (contrairement à Redux), API minimale, typage parfait. TanStack Query remplace la logique de cache manuelle du C#. |
| **Note** | Remplace `INotifyPropertyChanged` : les modèles deviennent des objets immuables + store réactif. |

### ADR-006 — Sécurité Electron : durcissement strict

| | |
|---|---|
| **Décision** | `contextIsolation: true`, `nodeIntegration: false`, `sandbox: true`, `webSecurity: true`, `preload` unique, **CSP stricte**, validation de **tous** les arguments IPC côté main. |
| **Raison** | Remplacer proprement l'isolation que `AddHostObjectToScript` offrait implicitement. Le renderer ne doit **jamais** avoir accès à `fs`, `child_process` ou `require`. |
| **Détail** | Voir §11 Sécurité. |

### ADR-007 — Bridge chart : **IPC typé** au lieu d'objets COM

| | |
|---|---|
| **Décision** | Remplacer `CoreWebView2.AddHostObjectToScript("chartService", bridge)` par des **canaux IPC** exposés via `contextBridge`, nommés `chart:*`. |
| **Raison** | Les objets COM n'existent que sur WebView2 (Windows). L'IPC Electron est **portable, typé, testable et débogable**. |
| **Conséquence** | 4 des 8 appels JS existants **disparaissent** (remplacés par du `fetch` direct dans le renderer). Voir §5. |

### ADR-008 — Données locales : **fichiers JSON identiques**

| | |
|---|---|
| **Décision** | Conserver **exactement** les mêmes formats de fichiers (`data/{Nom}.json`, `metadata/strategies.txt`, `cacheimage/*.png`, `chart/historical/*.csv`), déplacés dans `app.getPath('userData')`. |
| **Raison** | Compatibilité totale, migration par simple copie de dossier, possibilité de revenir à l'ancien client pendant la transition. |
| **Exception** | `.etude` (XamlPackage) et `config.txt` / `session.bin` (formats liés à l'OS). Voir §9 et §10. |

### ADR-009 — Chiffrement des tokens : **`safeStorage` Electron**

| | |
|---|---|
| **Décision** | Remplacer `ProtectedData.Protect(..., DataProtectionScope.CurrentUser)` (DPAPI) par **`safeStorage.encryptString` / `decryptString`**. |
| **Raison** | `safeStorage` utilise le **Keychain** sur macOS et **DPAPI** sur Windows → équivalent fonctionnel natif, cross-platform, sans dépendance externe. |
| **Migration** | Un `session.bin` chiffré par DPAPI sur Windows n'est **pas portable**. Au premier lancement : détection de l'ancien format → **ré-authentification** (l'utilisateur ressaisit son mot de passe). La session n'est jamais perdue silencieusement : un message explicite est affiché. |

### ADR-010 — MT5 : **pont local conservé**, l'EA ne change pas

| | |
|---|---|
| **Décision** | Conserver le mécanisme de dossier partagé (`Common\Files\DataEdge`) et porter `services/Mt5/*.cs` en Node. **Aucune modification de `DataEdgeBridge.mq5` / `DataEdgeExport.mq5`.** |
| **Raison** | Les utilisateurs ont déjà l'EA installé et compilé. Le changer casserait les installations existantes et imposerait une migration manuelle chez chaque utilisateur. |
| **Spécificité macOS** | MT5 sur Mac tourne **sous Wine** → dossier Common dans `~/Library/Application Support/net.metaquotes.wine.metatrader5/drive_c/...`. **Auto-détection + réglage manuel** obligatoires. Voir §6. |
| **Option future** | `POST /api/mt5/ingest` (l'EA pousse en HTTP) — à envisager en phase 6+, comme **amélioration**, pas comme prérequis. |

### ADR-011 — Éditeur riche : **TipTap** ⚠️ **À CONFIRMER**

| | |
|---|---|
| **Décision** | Remplacer `FlowDocument` + `XamlPackage` par **TipTap** (basé sur ProseMirror), avec un format `.etude` v2 = **ZIP { `content.json` + `assets/` }**. |
| **Raison** | TipTap gère nativement : gras/italique/souligné, couleurs, listes, tableaux, images, blocs de code, liens, undo/redo — soit l'essentiel de l'éditeur WPF actuel. |
| **Alternatives** | Lexical (Meta, plus jeune) · ProseMirror direct (plus bas niveau) · Quill (plus simple, moins extensible). |
| **Voir** | §9 pour la spécification complète du format et du convertisseur. |

### ADR-012 — Graphiques statistiques : **Lightweight Charts + ECharts** ⚠️ **À CONFIRMER**

| | |
|---|---|
| **Décision** | Remplacer `OxyPlot.Wpf` par **Lightweight Charts** (déjà présent dans le repo : courbes de prix/équité) et **ECharts** (histogrammes, camemberts, heatmaps : winrate par jour, sessions, types d'ordre). |
| **Raison** | Zéro nouvelle dépendance lourde pour les courbes (Lightweight Charts est déjà là) ; ECharts couvre tout le reste avec un rendu canvas performant. |
| **Impact** | `StatisticsView`, `ControlStat`, `StatisticsControl` refaits — mais le **calcul** (`strategie.cs`, `ResultatHelper`) est porté à l'identique. |

### ADR-013 — Distribution : **electron-builder + GitHub Actions**

| | |
|---|---|
| **Décision** | `electron-builder` pour produire `.dmg` (macOS, arm64 + x64) et `.exe` (NSIS) + `.msi`. Build via **GitHub Actions matrix** (`macos-latest`, `windows-latest`). |
| **Raison** | Le `.dmg` ne se construit que sur macOS, le `.exe` que sur Windows. **Le CI est obligatoire, pas optionnel.** |
| **Prérequis** | Compte Apple Developer (99 $/an) pour la notarization ; certificat Authenticode Windows recommandé (SmartScreen). |

### ADR-014 — Backend Symfony : **inchangé**

| | |
|---|---|
| **Décision** | **Aucune modification de `fxglobal/`** dans le périmètre de ce plan. |
| **Raison** | Les 30+ endpoints consommés (auth, cloud, public data, ai SSE, market, software, support) sont déjà exactement ce dont un client Electron a besoin : le backend a été conçu pour un client HTTP générique. |
| **Seule exception envisageable** | `POST /api/mt5/ingest` (phase 6+, optionnel) — voir ADR-010. |

### ADR-015 — Ordre de construction : **vertical d'abord, horizontal ensuite**

| | |
|---|---|
| **Décision** | Construire **une tranche verticale complète et fonctionnelle en premier** (chart + données marché + installeurs), avant d'attaquer l'UI large. |
| **Raison** | Cette tranche valide en quelques semaines **les 3 inconnues majeures** : le bridge IPC, le rendu du chart sur macOS, et la chaîne de packaging/signature. Le reste n'est plus que du volume. |
| **Conséquence** | Le Journal, les Stats, les Études et Space viennent **après** validation de la tranche verticale. |

---

## 2 — Inventaire exhaustif du logiciel actuel

> Référence de non-oubli : aucun élément du logiciel actuel ne doit manquer à l'appel.
> Compteurs : **33 XAML**, **89 fichiers C#**, **30 services**, **34 tools IA**, **14 modules MT5**, **12 modules Space**, **16 formats de fichiers**, **7 familles d'intégrations**.

### 2.1 — Fenêtres & vues (33 fichiers XAML)

#### 2.1.1 Fenêtres racine (19)

| # | Fichier | Taille | Rôle | Fonctionnalités détaillées |
|---|---|---|---|---|
| 1 | `App.xaml` | 59 Ko | Ressources globales | Thème sombre complet, `CyberButton`, pilules Space (`SpacePillPrimary/Green/Ghost/Danger`), `SpaceRoundTextBox`, `CircleNavButton`, `KindChip`, converters, templates |
| 2 | `MainWindow.xaml` | 77 Ko | **Hub central** | Dashboard (journal, vignettes stratégies, calendrier économique), barre de titre custom (drag/min/max/close), navigation (Accueil 📋, Études 📂, Investing 📊, Espace 👥, Configuration ⚙️), notifications, badge MAJ, bouton Support, FAB agent IA, prompt import MT5, widget Fear & Greed, 2 barres de progression de synchro |
| 3 | `backtesteur.xaml` | 3,7 Ko | Fenêtre de backtest | Héberge `Chart` + stratégie + bouton **📦 Partager** (Backtest Pack) |
| 4 | `Chart.xaml` | 14,7 Ko | **Hôte du graphique** | WebView2, watchlist, sélecteur de TF, statuts, bouton replay |
| 5 | `AjoutTrade.xaml` | 9,6 Ko | Ajout/édition de trade | Tous les champs, champs de confluence dynamiques, captures HTF/LTF, RR calculé |
| 6 | `Mt5ImportWindow.xaml` | 26,8 Ko | **Import MT5** | Prévisualisation, tout cocher/décocher, « Tout vers… », stratégie par défaut, stratégie par trade, filtre d'affichage (doublons masqués), compteur de lignes masquées, **glisser-déposer d'un rapport**, état du terminal + bouton « ▶ Lancer MT5 », positions ouvertes en direct, solde/équité/marge |
| 7 | `addStrategieWindow.xaml` | 9,0 Ko | Ajout/modification stratégie | Nom, description, champs personnalisés |
| 8 | `newvisu.xaml` | 2,2 Ko | Visualisation détaillée d'un trade | Champs + captures |
| 9 | `opening.xaml` | 1,7 Ko | Splash de démarrage | Logo + animation d'ouverture |
| 10 | `ZoomImageWindow.xaml` | 1,1 Ko | Zoom sur une image | Utilisé par Notes/Études/captures |
| 11 | `StatisticsControl.xaml` | 9,6 Ko | Stats d'une stratégie | DataGrid + graphes + bouton 📤 Partager mes stats |
| 12 | `StatisticsView.xaml` | 15,0 Ko | **Visualisation avancée** | OxyPlot : courbe d'équité, courbe de R, winrate par jour, sessions, paires, types d'ordre, meilleures/pires configs, **audit** |
| 13 | `SettingsView.xaml` | 57,8 Ko | **Paramètres (4 onglets)** | APPLICATION (thème, agent IA, synchro cloud, dossier de données), COMPTE GLOBAL (profil, avatar), CLOUD (stockage, liste fichiers, versions `.bak`, restauration, suppression, dossiers exclus), **JOURNAL MT5** (fichiers distants, sauvegardes, restauration/suppression, réglages journalisation auto) |
| 14 | `ControlStat.xaml` | 8,5 Ko | Vignette de perf d'une stratégie | Badge compact (winrate, R, nb trades) |
| 15 | `EtudesView.xaml` | 17,4 Ko | **Éditeur riche `.etude`** | FlowDocument, barre d'outils (gras/italique/souligné/couleur/surlignage/alignement/listes/tableaux/images), arborescence de dossiers, CRUD |
| 16 | `Demo.xaml` | 21,9 Ko | Onboarding | 4 slides textuels avec animations et style cyber + `Images/slide1-3.png` |
| 17 | `CustomMessageBoxView.xaml` | 3,5 Ko | MessageBox maison | Style DataEdge, remplace `MessageBox.Show` |
| 18 | `InputDialog.xaml` | 5,2 Ko | Saisie texte | Dialog de saisie |
| 19 | `TradeVisualizerControl.xaml` | 1,5 Ko | Captures HTF/LTF | Affichage des 2 captures d'un trade |

#### 2.1.2 Vues `Views/` (14)

| # | Fichier | Taille | Rôle |
|---|---|---|---|
| 20 | `Views/FxAiChatControl.xaml` (+ `.cs` 38 Ko) | 34,2 Ko | **Chat de l'agent IA** : bulles, alignement, couleurs, bandeau de statut/reasoning, quick prompts, streaming, bouton 🗑 (efface l'historique), pièce jointe, copie, FAB |
| 21 | `Views/Space/SpaceView.xaml` (+ `.cs` 31 Ko) | 9,6 Ko | **Racine Space** : rail (avatar, Accueil, DMs, Communautés, Recherche, Missions P2, ⚙️ Paramètres, ＋ groupe), liste conversations/communautés avec badges non-lus, zone contenu, polling 20 s, toast, pile de navigation `‹ Retour` / `⌂ Accueil`, Échap = Accueil |
| 22 | `Views/Space/SpaceHubHomeView.xaml` | 4,3 Ko | Accueil Space : feed global, suggestions, encarts partage, bannière gradient |
| 23 | `Views/Space/SpaceConversationView.xaml` | 3,6 Ko | DM/groupe : thread, composer type WhatsApp (stats/stratégie/week/trade/étude/fichier), chips d'attachments |
| 24 | `Views/Space/SpaceCommunityView.xaml` | 3,1 Ko | Communauté : bannière + onglets Feed / Membres / Missions 🔒 / Signaux 🔒 |
| 25 | `Views/Space/SpaceProfileView.xaml` | 1,0 Ko | Profil (avatar initiales, bio, snapshots, Message/Bloquer/Signaler) |
| 26 | `Views/Space/SpaceSearchView.xaml` | 1,9 Ko | Recherche utilisateurs & communautés (anti-rebond 350 ms) |
| 27 | `Views/Space/SpacePostEditorWindow.xaml` | 4,6 Ko | Création de post (kind + contenu + multi-attachments + communauté) |
| 28 | `Views/Space/SpacePackViewerWindow.xaml` (+ `.cs` 15 Ko) | 2,1 Ko | **Visionneuse Backtest Pack** : vérif SHA-256, stats, DataGrid des trades, captures zoomables, bouton **Rejouer sur le chart** |
| 29 | `Views/Space/SpaceShareDestinationWindow.xaml` | 2,5 Ko | Destination d'un partage (DM/groupe/communauté) — appelée par le backtesteur et StatisticsView |
| 30 | `Views/Space/SpaceGroupCreateWindow.xaml` | 2,8 Ko | Création de groupe (nom + membres, recherche anti-rebond) |
| 31 | `Views/Space/SpaceSettingsWindow.xaml` | 3,7 Ko | Paramètres Space (visibilité recherche, consentement RGPD, blocages, CGU) |
| 32 | `Views/Space/SpaceCguWindow.xaml` | 2,1 Ko | CGU complètes (objet, UGC, données de backtest, modération, RGPD, disclaimer AMF) |
| 33 | `Views/Space/SpacePostCard.cs` + `SpaceUiHelper.cs` (45 Ko) | — | Carte de post réutilisable + helpers UI (avatars initiales, rendu attachments, temps/tailles, `MakePill`, `SoftCard`, `MakeCardStyle`, `MakeRoundTextBox`) |

### 2.2 — Services (30 fichiers)

#### 2.2.1 Services racine `services/`

| Fichier | Taille | Rôle |
|---|---|---|
| `FxCloudService.cs` | 67 Ko | **Cœur cloud** : auth JWT + refresh, `/api/me`, sync complète (`FullSyncAsync` : manifest → scan local md5 → upload par lots de 8 → download → file de suppressions), détection « restauration initiale » (nouveau PC), résolution de conflits via `synccache.json`, profil (cache 5 min), handshake + verrouillage distant + notifications push, crash reporting, support, tokens chiffrés **DPAPI** (`session.bin`), **safe-delete** → `.bak` local, **safe-copy**, file d'attente des suppressions serveur, `config.txt` (server_url, app_id, `sync_cacheimage`, `sync_folder_*`), annulation de sync par `CancellationToken` |
| `FxAiAgentService.cs` | 39 Ko | **Client de l'agent IA** : POST SSE `/api/ai/chat`, lecture du flux en entier, routage des chunks (`text`, `reasoning`, `tool_call`, `ping`, `error`), **boucle agent 8 tours max** + tour de finalisation garanti, collecte des `tool_call` parallèles, exécution locale, renvoi `tool_results` |
| `AgentWorkspaceService.cs` | 87 Ko | **20 tools workspace** + contexte d'identité + `storage_knowledge` + `BuildWorkspaceSnapshotAsync` + coercition tolérante des arguments + confirmations sélectives |
| `AgentMarketService.cs` | 75 Ko | **7 tools web & marché** : `get_server_time`, `get_local_time`, calendrier économique, FedWatch, sentiment retail, cotations, actualités, recherche web, lecture de page (cache partagé `/api/market` + repli direct) |
| `AgentFileService.cs` | 67 Ko | **7 tools fichiers locaux** : lecture txt/md/json/csv/xml/**pdf**/**xlsx**/**docx**, recherche de texte, création txt/json/md/pdf/word, ouverture dossier, lancement de fichier. Contient aussi `AgentMarkdown` (parseur partagé) et `WordDocumentBuilder` (`.docx` via OpenXML). Écriture confinée à `Documents\DataEdge`, `append_document` pour les longs documents |
| `AgentStudiesService.cs` | 51 Ko | **Tools « études »** : catalogue, lecture (XamlPackage → texte sans images), recherche, création, écriture, suppression |
| `AgentWeeksService.cs` | 28 Ko | **Tools « notes hebdo »** : catalogue, lecture, recherche, calendrier mensuel, création/écriture/suppression `Notes_*.etude`, markdown → FlowDocument |
| `PdfExportService.cs` | 45 Ko | **QuestPDF** : rapport PDF moderne et structuré d'une stratégie + `ExportDocumentPdfToFile` (markdown → design DataEdge) |
| `Dataservice.cs` | 10 Ko | Récupération des données marché (`/api/public/data/fetch`), cache CSV local, watchlist (`/api/public/data/pairs`) + cache, tuple `(success, notFound, message, filePath)` |
| `ChartBridge.cs` | 6,5 Ko | **Pont C# ↔ JS** (COM) : 11 méthodes (voir §5) |
| `ReplayPositionStore.cs` | 13 Ko | Persistance JSON de la **dernière position replay** + **SESSION de backtest** (stratégie/paire → TF + position) dans `%LOCALAPPDATA%\DataEdge` |
| `MaximizeHelper.cs` | 4 Ko | **P/Invoke `user32.dll`** : hook `WM_GETMINMAXINFO` pour forcer la taille maximisée à la zone de travail exacte (taskbar visible, multi-écrans) |
| `UpdateService.cs` | 10 Ko | MAJ : lit la version depuis l'Assembly, `GET /software/version`, `Version.Parse`, télécharge `/download/{appId}/latest` avec progression, lance l'installeur, ferme l'app |
| `RichTextService.cs` | 3,5 Ko | Sauvegarde/chargement **XamlPackage** |
| `Mt5ImportService.cs` (racine) | 241 o | Shim de compatibilité |

#### 2.2.2 Services MT5 — `services/Mt5/` (14 fichiers)

| Fichier | Taille | Rôle |
|---|---|---|
| `Mt5Models.cs` | 21,9 Ko | Modèles : deals, positions, lignes d'import, options, rapport d'import, `ImportedSourceKeys` |
| `Mt5Text.cs` | 10,3 Ko | Normalisation de texte (accents, séparateurs décimaux, encodages) |
| `Mt5FileUtil.cs` | 3,9 Ko | Lecture de fichier avec ré-essais (150/400/1000 ms) |
| `Mt5PositionBuilder.cs` | 6,9 Ko | Agrégation des deals en positions (entrées/sorties/partiels, dépôts ignorés) |
| `Mt5TradeMapper.cs` | 9,5 Ko | **Table de décision `TP/TR/SL/BE/PARTIAL`**, R réalisé au SL de clôture |
| `Mt5BridgeSource.cs` | 22,5 Ko | **Pont local** : lecture `%APPDATA%\MetaQuotes\Terminal\Common\Files\DataEdge`, fusion des exports, comptage dédupliqué des positions ouvertes |
| `Mt5ReportSource.cs` | 25,8 Ko | **Rapport exporté** `.html` (UTF-8/UTF-16) et `.xlsx`, colonnes localisées, encodages et séparateurs variables |
| `Mt5Settings.cs` | 7,9 Ko | Réglages de la journalisation automatique |
| `StrategieJournalWriter.cs` | 6,7 Ko | Écriture groupée dans `data\{Nom}.json` (backup `.bak` + écriture atomique) |
| `Mt5ImportService.cs` | 21,9 Ko | Aperçu en mémoire, dédoublonnage par `Trade.SourceKey`, import groupé par stratégie, **verrou nommé `DataEdge_Mt5Import`**, rollback, rapport, journal technique |
| `Mt5BridgeInstaller.cs` | 18,5 Ko | **Installation 1 clic** : détection des terminaux, copie de `DataEdgeBridge.mq5` + `.tpl` dans `MQL5\Experts\DataEdge\`, compilation par `metaeditor64.exe /compile /log`, lecture du journal UTF-16. **P/Invoke `user32.dll`** pour réduire la fenêtre MetaEditor |
| `Mt5BridgeWatcher.cs` | 10,5 Ko | **Veille automatique** : `FileSystemWatcher` + contrôle périodique, comptage des trades importables, import silencieux, arrêt de boucle quand aucune écriture |
| `Mt5BridgeStatus.cs` | 4,7 Ko | Lecture du **battement de cœur** de l'EA |
| `Mt5WatchState.cs` | 2,3 Ko | Événements statiques `Changee` / `ViderDemande` (pilotage pause/reprise depuis les réglages) |
| `Mt5LivePositions.cs` | 8,2 Ko | **Positions ouvertes en direct** : `positions_<login>.live` (ticket, symbole, sens, lot, prix, SL, TP, profit flottant + swap, magic, commentaire) + **état du compte** (Balance, Equity, MarginFree, Leverage), `EstFrais()`, `Trouver(ticket)`, `CompteTexte()`, tolérance 15 s |
| `Mt5ExportArchiver.cs` | 4,2 Ko | **Archivage CIBLÉ** : un export n'est déplacé dans `importes\` que si **TOUS** ses trades sont réglés. `MessageConserves()` |

#### 2.2.3 Services Space — `services/Space/` (12 fichiers)

| Fichier | Taille | Rôle |
|---|---|---|
| `ISpaceService.cs` | 4,2 Ko | **Interface** : événements `MessageReceived` / `BadgesChanged`, `SyncAsync`, `SearchUsersAsync`, `GetProfileAsync`, `GetMyProfileAsync`, conversations, groupes, posts, réactions, blocage, settings |
| `SpaceServiceFactory.cs` | 0,8 Ko | Fabrique unique `Create()` → **seul fichier à modifier au swap coquille → réel** |
| `SpaceDtos.cs` | 6,1 Ko | DTOs figés v1 (ids `int`, timestamps `long` unix, chemins relatifs au storage) |
| `SpaceMockService.cs` | 32,4 Ko | Coquille : simulation d'événements, latence, pagination |
| `SpaceMockStore.cs` | 7,7 Ko | Persistance locale de la coquille (`%LOCALAPPDATA%\DataEdge\space`) |
| `SpaceMockSeed.cs` | 16,1 Ko | Jeu de données de démo |
| `SpaceBacktestPackBuilder.cs` | 6,2 Ko | **Génération du Backtest Pack** : JSON + copies des captures HTF/LTF + **hash SHA-256** |
| `SpacePackVerifier.cs` | 3,6 Ko | Vérification du hash à la réception |
| `SpaceLocalExportBuilder.cs` | 12,9 Ko | Exports locaux (stats / stratégie / week / trade / étude / fichier) pour le composer |
| `SpaceAttachmentFormats.cs` | 2,6 Ko | Formats et kinds d'attachments |

### 2.3 — Modèles & logique métier

| Fichier | Taille | Contenu |
|---|---|---|
| `strategie.cs` | 57 Ko | **Cœur métier** : `Strategie`, `Trade`, `PerformanceStat`, `AdvancedStats`, `utils`, **`ResultatHelper`** (`EstGagnant()`, `RRealise()` — **source de vérité unique**), `CalculateStatistics`, CRUD trades/journal, **champs personnalisés / confluences** (`GetStructure()`, `FusionnerChamps`), `SaveDataAtomic` (écriture `.tmp` GUID + `File.Replace` + vérification + rollback), `VerrouEcriture` (statique, réentrant), `TryReadFileText`, `LoadMetadata`, migration Excel→JSON |
| `CalculStatistics.cs` | 1,8 Ko | Structure `Statistics` (stats globales) |
| `Models/ChatMessage.cs` | 3,0 Ko | Bulle de chat (`INotifyPropertyChanged`, alignement, couleurs) |
| `Models/AiAgentError.cs` | 749 o | Structuration des erreurs agent + `ToDisplayText()` |
| `SpotlightOverlay.cs` | 23,5 Ko | **Tour guidé / Feature Highlight** du dashboard |
| `SpotlightStep.cs` | 1,4 Ko | Modèle d'une étape du tour |
| `FirstLaunchManager.cs` | 1,5 Ko | Détection du premier lancement + flag spotlight |
| `NetworkUtils.cs` | 7,3 Ko | **Fear & Greed** (API CNN + alternative.me) |
| `Properties/AssemblyInfo.cs` | 2,4 Ko | Version, titre, copyright |
| `Properties/Settings.settings` / `Resources.resx` | 834 o / 5,6 Ko | Réglages et ressources |
| `App.xaml.cs` | 3,5 Ko | Entry point, `RenderMode.SoftwareOnly`, **3 gestionnaires d'exceptions globales** (UI, domaine, TPL), nettoyage du stack trace, envoi du crash report |

#### 2.3.1 Code-behind volumineux (le cœur de l'UI à porter)

| Fichier | Taille | Contenu |
|---|---|---|
| `MainWindow.xaml.cs` | 99,3 Ko | **Hub** : navigation, dashboard, journal, `loadStrategies`, notifications, `PerformSystemHandshake`, `CheckAppUpdateAsync`, `LoadInvestingCalendar` (WebView2 → Investing.com), Fear & Greed, badges, timers (positions, terminal, bannière), `ShowDashboard` / `ShowEtude` / `ShowSpace`, prompt import MT5 |
| `SettingsView.xaml.cs` | 67,1 Ko | 4 onglets, `RefreshCloudPanelAsync`, `CloudDisplayHelper` (libellés métier), `CloudFileItem`, cases de dossiers exclus, réglages MT5 |
| `Chart.xaml.cs` | 62,6 Ko | **Logique du chart** : `InitBrowser` (WebView2, hôte virtuel `dataedge.local`, bridge COM), `LoadBacktestData`, `LoadMoreData`, `LoadYearForBacktest`, `LoadPreviousYearForReplay`, `GetFileToRequest` (règles de partitionnement), `ParseCsvToCandles`, `SafeExecuteJs`, `EscapeJs`, `ShowNoticeAsync` / `ShowNoDataNotice`, `ToggleLoader`, `SetStatus`, `PrepareReplayResume`, `ExitReplayAndGoToPresent`, formulaire d'ajout de trade intégré, `_ctsGlobal` (annulation), `_isLoadingMore`, `_isPrefetching` |
| `EtudesView.xaml.cs` | 38,3 Ko | Éditeur riche, `TextRange.Save/Load` **XamlPackage**, arborescence, CRUD, copier/coller, `ColorDialog` |
| `Mt5ImportWindow.xaml.cs` | 32,7 Ko | Import MT5 (voir §2.1.1 #6) |
| `Views/Space/SpaceView.xaml.cs` | 31,1 Ko | Navigation, polling, badges, pile d'historique, toast |
| `StatisticsControl.xaml.cs` | 17,4 Ko | Stats de stratégie + partage |
| `StatisticsView.xaml.cs` | 14,5 Ko | Graphes OxyPlot, courbe de R, projection monétaire, audit |
| `AjoutTrade.xaml.cs` | 11,9 Ko | Formulaire de trade complet |
| `Views/Space/SpacePackViewerWindow.xaml.cs` | 15,0 Ko | Rendu du pack, vérification, Rejouer |
| `Views/Space/SpaceConversationView.xaml.cs` | 15,7 Ko | Thread, composer, attachments |
| `Views/Space/SpaceCommunityView.xaml.cs` | 12,4 Ko | Feed, membres, missions, signaux |
| `Views/Space/SpaceProfileView.xaml.cs` | 9,5 Ko | Profil, actions |
| `Views/Space/SpacePostEditorWindow.xaml.cs` | 8,1 Ko | Création de post |
| `Views/Space/SpaceGroupCreateWindow.xaml.cs` | 8,0 Ko | Création de groupe |
| `Views/Space/SpaceSearchView.xaml.cs` | 7,3 Ko | Recherche anti-rebond |
| `Views/Space/SpaceShareDestinationWindow.xaml.cs` | 6,3 Ko | Destination de partage |
| `Views/Space/SpaceHubHomeView.xaml.cs` | 5,6 Ko | Feed global |
| `Views/Space/SpaceSettingsWindow.xaml.cs` | 5,2 Ko | Paramètres Space |
| `Views/Space/SpaceCguWindow.xaml.cs` | 3,5 Ko | CGU |
| `Views/Space/SpacePostDetailView.xaml.cs` | 1,8 Ko | Détail d'un post |
| `Demo.xaml.cs`, `opening.xaml.cs`, `newvisu.xaml.cs`, `addStrategieWindow.xaml.cs`, `InputDialog.xaml.cs`, `CustomMessageBoxView.xaml.cs`, `ZoomImageWindow.xaml.cs`, `ControlStat.xaml.cs`, `TradeVisualizerControl.xaml.cs`, `backtesteur.xaml.cs` | — | Code-behind des vues restantes |

### 2.4 — Fonctionnalités transverses (les « petits coins »)

| # | Fonctionnalité | Implémentation actuelle | Détail à préserver |
|---|---|---|---|
| T1 | **Thème sombre + accents néon** | `App.xaml` (59 Ko) | Fond `#131722` / `#0D141D`, accents **vert EDGE `#FF00FF03`** et **cyan `#00BFFF`**, palette Space (`#151D27`, `#25333F`, `#0E2A3D`, `#18232E`, `#202B36`), typo Segoe UI |
| T2 | **Barre de titre personnalisée** | `MainWindow` + `MaximizeHelper` (user32) | Drag, minimiser, maximiser, fermer, **taille max = zone de travail exacte** (taskbar visible, multi-écrans) |
| T3 | **Rendu logiciel forcé** | `App.xaml.cs` → `RenderMode.SoftwareOnly` | Évite les crashs GPU (fréquent en trading). En Electron : `--disable-gpu` / `app.disableHardwareAcceleration()` à évaluer |
| T4 | **Onboarding 4 slides** | `Demo.xaml` + `Images/slide1-3.png` | Animations, style cyber, affiché au premier lancement |
| T5 | **Tour guidé Spotlight** | `SpotlightOverlay.cs` (23,5 Ko) + `SpotlightStep.cs` + `FirstLaunchManager.cs` | Overlay de mise en évidence sur les éléments du dashboard, étapes ordonnées, flag de premier lancement |
| T6 | **Notifications / toasts** | `ShowNotification` (MainWindow) | Toast néon, auto-dismiss, styles par type |
| T7 | **Crash reporting** | `App.xaml.cs` (3 handlers) → `POST /software/report-crash` | Nettoyage du stack (ne garder que les frames du code applicatif), rapport envoyé au support, message utilisateur simplifié |
| T8 | **Badge + mise à jour** | `MainWindow` + `UpdateService` | Point néon au démarrage si version distante > locale, popup (versions, ProgressBar, bouton TÉLÉCHARGER), toast, téléchargement avec % puis lancement de l'installeur + fermeture |
| T9 | **Panneau Support** | `BtnSupport` → `POST /support/send` | Formulaire de message au développeur |
| T10 | **Fear & Greed** | `NetworkUtils.cs` | Indices US + Crypto (API CNN + alternative.me), widget dashboard. ⚠️ En Electron : **appel depuis le main process** (pas de CORS) |
| T11 | **Calendrier économique Investing** | `MainWindow.xaml:874` WebView2 → `sslecal2.investing.com` | URL avec paramètres (colonnes, pays, semaine, timezone 55, langue FR) |
| T12 | **Verrouillage à distance** | `POST /software/handshake` → `is_locked` | Blocage du logiciel piloté depuis l'admin |
| T13 | **Notifications push serveur** | handshake → `SoftwareNotification` | Types `info/offer/maintenance/alert` |
| T14 | **Champs dynamiques / confluences** | `GetStructure()`, `ChampsPersonnalises` | Colonnes de DataGrid **générées dynamiquement**, stats par valeur de champ, fusion sans écrasement |
| T15 | **Verrou d'écriture partagé** | `Strategie.VerrouEcriture` (statique, réentrant) | Protège `AddTrade`, `AddJournal`, `UpdateTrade`, `UpdateJournal`, `RemoveTradeById`, `RemoveJournalById`, `AddJournalBatch`, `UpdateJournalBatch` |
| T16 | **Écriture atomique + vérifiée + rollback** | `SaveDataAtomic` | `.tmp` avec GUID unique + `File.Replace` + relecture de vérification + rollback depuis `.bak` ; **ré-essais 0/80/200/500/1200 ms** (antivirus / indexation / cloud qui tient le fichier) |
| T17 | **Safe-delete → `.bak` horodaté** | `SafeDeleteToLocalBackup`, `SafeCopyToLocalBackup` | Pattern `{chemin}.{YYYYMMDD_HHMMSS}.bak`, jamais de `File.Delete` sec |
| T18 | **File d'attente des suppressions serveur** | `metadata/cloud_pending_deletions.json` | Offline-friendly : la suppression distante est appliquée à la prochaine sync réussie |
| T19 | **Synchro annulable** | `CancellationToken` vérifié entre phases et entre fichiers | Bouton ✕ dans la barre de progression, cache de synchro enregistré à l'annulation |
| T20 | **Single instance / verrou global** | `Mutex "DataEdge_Mt5Import"` | Deux instances ne peuvent pas importer 2× les mêmes trades |
| T21 | **Langue** | 100 % français | Aucune internationalisation aujourd'hui ; prévoir i18n FR extensible |
| T22 | **Licence QuestPDF** | `QuestPDF.Settings.License` | Community (< 1 M$ CA/an) |
| T23 | **Diagnostic « données inexistantes »** | `cyber-notify` + `ShowNoDataNotice` | Toast explicite (année sans fichier, début/fin d'historique, erreur réseau) au lieu d'un graphique vide |

### 2.5 — Formats de fichiers locaux (16)

| # | Chemin actuel | Contenu | Portable ? |
|---|---|---|---|
| F1 | `data/{Nom}.json` | Stratégie + trades + journal + champs de confluence | ✅ |
| F2 | `metadata/strategies.txt` | Registre des stratégies (`%`-séparé) | ✅ |
| F3 | `metadata/synccache.json` | Derniers hash serveur vus (résolution de conflits) | ✅ |
| F4 | `metadata/cloud_pending_deletions.json` | File de suppressions serveur | ✅ |
| F5 | `metadata/mt5_import_log.txt` | Journal technique des imports MT5 | ✅ |
| F6 | `etudes/{dossier}/{nom}.etude` | **XamlPackage WPF** | ⚠️ **NON** |
| F7 | `Notes/Notes_yyyyMMdd.etude` | **XamlPackage WPF** (notes hebdo) | ⚠️ **NON** |
| F8 | `cacheimage/{ts}_{HTF\|LTF}.png` | Captures de chart | ✅ |
| F9 | `chart/historical/{SYM}_{TF}_{BLOK}.csv` (+ `.gz`) | Cache des données marché | ✅ |
| F10 | `chart/historical/watchlist_cache.json` | Liste de paires en cache | ✅ |
| F11 | `config.txt` | `server_url`, `app_id`, `sync_cacheimage`, `sync_folder_*` | 🔨 Converti en JSON |
| F12 | `session.bin` | JWT + refresh, **chiffré DPAPI** | ⚠️ **NON** (non déchiffrable hors Windows) |
| F13 | `session_v1.json` | Profil utilisateur | ✅ |
| F14 | `cache/` | Cache du profil | ✅ |
| F15 | `log.txt` | Log applicatif | ✅ |
| F16 | `*.{YYYYMMDD_HHMMSS}.bak` | Sauvegardes horodatées | ✅ |
| F17 | `importes/` | Exports MT5 archivés | ✅ |
| F18 | `firstLaunch1.txt` | Flag premier lancement | ✅ |
| F19 | `%LOCALAPPDATA%\DataEdge\` | `ReplayPositionStore` + `space/` (coquille) | ✅ |

### 2.6 — Intégrations externes (7 familles)

#### 2.6.1 Backend Symfony `fxdataedge.com` — **30+ endpoints**

| Famille | Endpoints |
|---|---|
| **Auth & profil** | `POST /api/register` · `POST /api/login` · `POST /token/refresh` · `GET /api/me` · `POST /api/user/update` |
| **Cloud storage** | `POST /api/cloud/sync-batch` (lots de 8, max 30 fichiers / 25 Mo) · `POST /api/cloud/sync-file` (repli unitaire) · `POST /api/cloud/file-info` · `GET /api/cloud/list` · `GET /api/cloud/download` · `GET /api/cloud/list-backups` · `POST /api/cloud/restore-backup` · `POST /api/cloud/delete-file` · `GET /api/cloud/storage` |
| **Données marché (public)** | `GET /api/public/data/fetch?pair&tf&year` (CSV.gz) · `GET /api/public/data/pairs` |
| **Agent IA** | `POST /api/ai/chat` (**SSE**) · `GET /api/ai/history?limit=` · `DELETE /api/ai/history` |
| **Marché mutualisé** | `GET /api/market/time` · `GET /api/market/{channel}` (calendar / news / quotes / sentiment / fedwatch) · `POST /api/market/{channel}/refresh` |
| **Logiciel** | `POST /software/handshake` · `GET /software/version?app_id=` · `GET /download/{appId}/latest` · `POST /software/report-crash` |
| **Support** | `POST /support/send` |

> **Toutes ces routes restent consommées à l'identique.** Aucune modification serveur (ADR-014).

#### 2.6.2 MT5

| Élément | Détail |
|---|---|
| Pont local | Dossier commun `%APPDATA%\MetaQuotes\Terminal\Common\Files\DataEdge` |
| EA de veille | `DataEdgeBridge.mq5` : `OnTradeTransaction` + `OnTimer` 1 s, export incrémental, rattrapage au démarrage, contrôle historique toutes les 15 s (trades pris depuis le téléphone), écriture `.tmp` + `FileMove` atomique, battement de cœur, publication `positions_<login>.live` toutes les 5 s |
| Script manuel | `DataEdgeExport.mq5` (repli, jamais mis en avant) |
| Modèle de graphique | `DataEdgeBridge.tpl` (applique l'EA en un clic) |
| Documentation | `INSTALL_MT5.txt` (11,8 Ko) |
| Exécutables invoqués | `metaeditor64.exe /compile /log`, `terminal64.exe` |
| Dossier d'archive | `importes/` |

#### 2.6.3 Fournisseurs LLM (via le serveur uniquement)

Chaîne de secours inter-fournisseurs pilotée par `/admin/ai` : **OpenRouter :free → Google Gemini (tous modèles) → Groq → Z.AI GLM gratuit → OpenRouter payant**, avec mémoire de saturation en base (`ai_provider_state`) et bascule immédiate sur 429.
> ⚠️ **Le client ne connaît aucun fournisseur** : il ne parle qu'à `/api/ai/chat`. Aucune clé API côté client. **Rien à porter** hormis le protocole SSE.

#### 2.6.4 Autres

| Intégration | Détail | Action migration |
|---|---|---|
| **Fear & Greed** | API CNN + alternative.me, **appelées directement depuis le client** | 🔨 À déplacer dans le **main process** (CORS) |
| **Investing.com** | `sslecal2.investing.com` dans une WebView2 | 🔨 → `<webview>` ou `WebView` Electron |
| **PdfPig** (Apache 2.0) | Extraction de texte PDF | 🔨 → `pdf-parse` |
| **ExcelDataReader** (MIT) | Lecture `.xlsx` / `.xls` | 🔨 → `exceljs` |
| **DocumentFormat.OpenXml** (MIT) | Création/lecture `.docx` | 🔨 → `docx` + `mammoth` |
| **QuestPDF** (Community) | Génération PDF | 🔨 → `pdfkit` |
| **OxyPlot.Wpf** | Graphes statistiques | 🔨 → Lightweight Charts + ECharts |
| **Newtonsoft.Json / System.Text.Json** | Sérialisation | 🔨 → `JSON` natif / `zod` pour la validation |

#### 2.6.5 Ressources embarquées

| Ressource | Contenu | Action |
|---|---|---|
| `resources/chart/` | 11 fichiers, **280 Ko** : `index.html`, `lightweight-charts.js` (160 Ko), `chart_engine.js` (51 Ko), `drawing_manager.js` (27 Ko), `drawing_plugin.js`, `drawing_configs.js`, `drawing_utils.js`, `style.css`, `toolbar.css`, `replay.css` | ♻️ **Réutilisé à ~95 %** |
| `resources/mt5/` | `DataEdgeBridge.mq5` (19 Ko), `DataEdgeExport.mq5`, `DataEdgeBridge.tpl`, `INSTALL_MT5.txt` | ♻️ **Inchangé** |
| `Images/` | `slide1.png` (376 Ko), `slide2.png`, `slide3.png` | ♻️ Réutilisé (onboarding) |
| `resources/` | `default_user.png`, `default_image.png`, `file.png`, `folder.png` | ♻️ Réutilisé |
| Icônes | `trading.ico` (94 Ko), `cf-flow-*.ico` (34 Ko), `logopng.png` (341 Ko), `logo copie.png` | ♻️ Converti en `.icns` (macOS) + `.ico` (Windows) |
| `Properties/Resources.resx` | Ressources .NET | 🔨 Converti |

---

## 3 — Cartographie de migration

> Légende : ♻️ **Réutilisé** (tel quel ou quasi) · 🔨 **Porté/Réécrit** · 🗑️ **Supprimé** (plus nécessaire)

### 3.1 Synthèse par volume

| Catégorie | Volume actuel | ♻️ Réutilisé | 🔨 Porté | 🗑️ Supprimé |
|---|---|---|---|---|
| **Chart HTML/JS/CSS** | 280 Ko (11 fichiers) | **~95 %** | `bridge.ts` (nouveau) | — |
| **MQL5 / EA MT5** | 4 fichiers | **100 %** | — | — |
| **Backend Symfony** | `fxglobal/` | **100 %** | — | — |
| **Logique métier MT5** | 14 fichiers, 175 Ko | — | **~90 %** | — |
| **Logique métier stats** | `strategie.cs` + `CalculStatistics.cs` | — | **~85 %** | — |
| **Tools IA** | 5 services, 308 Ko | — | **~80 %** | — |
| **Services cloud** | `FxCloudService.cs` (67 Ko) | — | **~90 %** | — |
| **UI (33 XAML + code-behind)** | 346 Ko XAML + ~400 Ko C# | — | **100 % réécrit** | — |
| **Code Windows-only** | `MaximizeHelper`, DPAPI, WMI, GDI+, `explorer.exe` | — | — | **100 %** |
| **Chaîne de distribution** | installeur MSI + `UpdateService` | — | `electron-builder` + `electron-updater` | **100 %** |
| **`backtest.csproj`, `App.config`, `packages.config`** | 3 fichiers | — | — | **100 %** |
| **Dépendances NuGet** | 22 packages | — | 8 remplacées par des équivalents npm | 14 supprimées |

### 3.2 Tableau de correspondance C# → Node/TypeScript

| Source C# | Cible TypeScript | Statut |
|---|---|---|
| `strategie.cs` | `shared/models/strategy.ts` + `main/stats/` | 🔨 |
| `CalculStatistics.cs` | `shared/stats/statistics.ts` | 🔨 |
| `FxCloudService.cs` | `main/cloud/cloudService.ts` (+ `auth.ts`, `sync.ts`, `handshake.ts`, `crash.ts`, `support.ts`) | 🔨 |
| `Dataservice.cs` | `main/market/dataservice.ts` (fetch) + `renderer/chart/csv.ts` (parse) | 🔨 |
| `ChartBridge.cs` | `preload/chartBridge.ts` + `main/ipc/chart.ts` | 🔨 |
| `ReplayPositionStore.cs` | `main/store/replayStore.ts` | 🔨 |
| `UpdateService.cs` | `main/updater.ts` | 🔨 |
| `RichTextService.cs` | `main/store/etudes.ts` | 🔨 |
| `MaximizeHelper.cs` | — | 🗑️ |
| `NetworkUtils.cs` | `main/market/sentiment.ts` | 🔨 |
| `FirstLaunchManager.cs` | `main/store/flags.ts` | 🔨 |
| `SpotlightOverlay.cs` / `SpotlightStep.cs` | `renderer/features/spotlight/` | 🔨 |
| `FxAiAgentService.cs` | `main/agent/agentClient.ts` | 🔨 |
| `AgentWorkspaceService.cs` | `main/agent/tools/workspace.ts` | 🔨 |
| `AgentMarketService.cs` | `main/agent/tools/market.ts` | 🔨 |
| `AgentFileService.cs` | `main/agent/tools/files.ts` + `main/doc/word.ts` + `main/doc/markdown.ts` | 🔨 |
| `AgentStudiesService.cs` | `main/agent/tools/studies.ts` | 🔨 |
| `AgentWeeksService.cs` | `main/agent/tools/weeks.ts` | 🔨 |
| `PdfExportService.cs` | `main/doc/pdfExport.ts` | 🔨 |
| `services/Mt5/*.cs` (14) | `main/mt5/*.ts` (14) | 🔨 |
| `services/Space/*.cs` (12) | `main/space/*.ts` (12) | 🔨 |
| `Views/Space/SpacePostCard.cs`, `SpaceUiHelper.cs` | `renderer/features/space/components/` | 🔨 |
| `App.xaml.cs` | `main/index.ts` + `main/crashReporter.ts` | 🔨 |
| `App.xaml` | `renderer/theme/` (CSS variables) | 🔨 |
| `Properties/AssemblyInfo.cs` | `package.json` (`version`) | 🔨 |

### 3.3 Correspondances techniques C# → Electron/Node

| Concept C# / WPF | Équivalent Electron / Node | Notes |
|---|---|---|
| `AppDomain.CurrentDomain.BaseDirectory` | `app.getPath('userData')` | **Remplace les ~15 usages** |
| `ProtectedData.Protect` (DPAPI) | `safeStorage.encryptString` | Keychain macOS / DPAPI Windows |
| `CoreWebView2.AddHostObjectToScript` | `contextBridge.exposeInMainWorld` + IPC | Voir §5 |
| `CoreWebView2.SetVirtualHostNameToFolderMapping` | `protocol.handle('app', ...)` | Hôte virtuel `dataedge.local` → `app://local/` |
| `CoreWebView2.ExecuteScriptAsync` | `webContents.executeJavaScript` / `send` | |
| `FileSystemWatcher` | `fs.watch` (+ `chokidar` si besoin) | |
| `Process.Start` / `UseShellExecute` | `shell.openPath` / `shell.openExternal` | |
| `explorer.exe /select` | `shell.showItemInFolder` | |
| `Mutex` | `app.requestSingleInstanceLock()` + `proper-lockfile` | |
| `CancellationToken` | `AbortController` / `AbortSignal` | |
| `HttpClient` | `fetch` (Node 18+ natif) | |
| `INotifyPropertyChanged` | Zustand / `useState` | |
| `DataGrid` | TanStack Table / AG Grid | |
| `FlowDocument` + `XamlPackage` | TipTap (ProseMirror JSON) | Voir §9 |
| `OxyPlot` | Lightweight Charts + ECharts | |
| `MessageBox.Show` | `dialog.showMessageBox` + `Dialog.tsx` maison | |
| `OpenFileDialog` / `SaveFileDialog` | `dialog.showOpenDialog` / `showSaveDialog` | |
| `System.Windows.Forms.ColorDialog` | `<input type="color">` / ColorPicker | |
| `Clipboard` (WPF) | `navigator.clipboard` | |
| `RenderMode.SoftwareOnly` | `app.disableHardwareAcceleration()` | À évaluer |
| `QuestPDF` | `pdfkit` | |
| `PdfPig` / `ExcelDataReader` / `OpenXml` | `pdf-parse` / `exceljs` / `docx`+`mammoth` | |
| `Newtonsoft.Json` | `JSON` natif + `zod` (validation) | |
| `Task.Delay` / `DispatcherTimer` | `setTimeout` / `setInterval` | |
| `Application.Current.Dispatcher.Invoke` | `ipcRenderer.invoke` (asynchrone) | ⚠️ Plus de modèle « UI thread » : tout est asynchrone |

---

## 4 — Architecture cible Electron

### 4.1 Arborescence du dépôt

```
dataedge-electron/
├── package.json · electron.vite.config.ts · electron-builder.yml
├── tsconfig.json (+ tsconfig.node.json, tsconfig.web.json)
├── .github/workflows/build.yml
├── resources/                        ♻️ inchangé
│   ├── chart/                        ♻️ index.html, *.js, *.css (280 Ko)
│   ├── mt5/                          ♻️ *.mq5, *.tpl, INSTALL_MT5.txt
│   ├── icon.icns · icon.ico · icon.png
│   └── onboarding/slide1-3.png       ♻️
├── src/
│   ├── main/                         ← Processus principal (Node) = tout le C# non-UI
│   │   ├── index.ts                  bootstrap, fenêtres, menu, tray, single-instance
│   │   ├── window.ts                 création/gestion des BrowserWindow
│   │   ├── paths.ts                  app.getPath('userData') + arborescence de dossiers
│   │   ├── crashReporter.ts          uncaughtException / unhandledRejection
│   │   ├── updater.ts                electron-updater
│   │   ├── security/
│   │   │   ├── session.ts            safeStorage (tokens), CSP, permissions
│   │   │   └── ipcGuard.ts           validation de tous les arguments IPC (zod)
│   │   ├── ipc/                      routeurs typés par domaine
│   │   │   ├── index.ts              enregistrement central
│   │   │   └── chart.ts · store.ts · cloud.ts · mt5.ts
│   │   │       agent.ts · space.ts · market.ts · doc.ts · app.ts
│   │   ├── store/                    persistance locale (JSON, atomique, verrouillée)
│   │   │   ├── atomic.ts             SaveDataAtomic (tmp GUID + replace + vérif + rollback)
│   │   │   ├── lock.ts               verrous réentrants + proper-lockfile
│   │   │   ├── strategyStore.ts      data/{Nom}.json, metadata/strategies.txt
│   │   │   ├── etudesStore.ts        etudes/ + Notes/ (.etude v2)
│   │   │   ├── replayStore.ts        ReplayPositionStore
│   │   │   ├── configStore.ts        config.txt → config.json
│   │   │   └── flags.ts              firstLaunch, spotlight
│   │   ├── stats/
│   │   │   ├── resultatHelper.ts     ⚠️ SOURCE DE VÉRITÉ UNIQUE
│   │   │   ├── calculateStatistics.ts
│   │   │   └── advancedStats.ts      jour, paire, session, type d'ordre, confluence
│   │   ├── market/
│   │   │   ├── dataservice.ts        /api/public/data/fetch + cache CSV
│   │   │   ├── watchlist.ts          /api/public/data/pairs + cache
│   │   │   └── sentiment.ts          Fear & Greed (CNN + alternative.me)
│   │   ├── cloud/
│   │   │   ├── auth.ts               login, refresh, /api/me, tokens (safeStorage)
│   │   │   ├── sync.ts               FullSyncAsync (manifest, scan md5, upload, download)
│   │   │   ├── conflicts.ts          synccache.json, résolution de conflits
│   │   │   ├── deletions.ts          file d'attente des suppressions serveur
│   │   │   ├── safeDelete.ts         safe-delete / safe-copy → .bak horodaté
│   │   │   ├── handshake.ts          /software/handshake (verrouillage, notifs, version)
│   │   │   ├── crash.ts              /software/report-crash
│   │   │   └── support.ts            /support/send
│   │   ├── mt5/                      ← 14 modules (voir §6)
│   │   ├── agent/
│   │   │   ├── agentClient.ts        SSE + boucle agent (8 tours)
│   │   │   ├── context.ts            identité + storage_knowledge
│   │   │   ├── confirmations.ts      requires_confirmation
│   │   │   └── tools/
│   │   │       ├── index.ts          registre des 34 tools
│   │   │       ├── workspace.ts (20) · market.ts (7) · files.ts (7)
│   │   │       └── studies.ts · weeks.ts
│   │   ├── space/                    ← 12 modules (voir §8)
│   │   └── doc/
│   │       ├── pdfExport.ts          pdfkit
│   │       ├── word.ts               docx + mammoth
│   │       ├── excel.ts              exceljs
│   │       ├── pdfRead.ts            pdf-parse
│   │       └── markdown.ts           AgentMarkdown
│   ├── preload/
│   │   ├── index.ts                  contextBridge → window.dataedge.*
│   │   └── api.d.ts                  contrat typé main ↔ renderer
│   ├── renderer/                     ← L'UI (React + TS)
│   │   ├── main.tsx · App.tsx · router.tsx
│   │   ├── theme/                    CSS variables, Tailwind config, polices
│   │   ├── components/               Dialog, InputDialog, Toast, PerfBadge, ImageViewer,
│   │   │                             TradeImages, DataGrid, VirtualList, ColorPicker
│   │   ├── layout/                   Shell, TitleBar, SideRail, ProgressBars
│   │   ├── features/
│   │   │   ├── splash/ · onboarding/ · spotlight/
│   │   │   ├── dashboard/ · journal/ · trade/ · strategy/
│   │   │   ├── stats/ · settings/ (4 onglets)
│   │   │   ├── chart/                ChartHost.tsx
│   │   │   ├── backtester/ · mt5-import/ · etudes/ · notes/
│   │   │   ├── space/ (12 vues) · agent/ (chat) · support/
│   │   │   └── investing/            calendrier économique (webview)
│   │   ├── chart/                    ♻️ copie de resources/chart
│   │   │   ├── index.html · chart_engine.js · drawing_*.js · *.css
│   │   │   ├── lightweight-charts.js
│   │   │   ├── bridge.ts             ⭐ REMPLACE window.chrome.webview.hostObjects
│   │   │   └── csv.ts                fetch + DecompressionStream + parse CSV
│   │   ├── store/                    Zustand stores
│   │   ├── hooks/                    useIpc, useCloudQuery, useStrategies…
│   │   └── i18n/                     FR (extensible)
│   └── shared/
│       ├── models/                   Trade, Strategie, Statistics, DTOs Space, ChatMessage
│       ├── constants/                routes API, TF supportées, statuts, kinds
│       └── schemas/                  zod (validation IPC + API)
├── tests/
│   ├── unit/                         Vitest (stats, MT5 mapper, pack verifier…)
│   └── e2e/                          Playwright (chart, import, replay, sync)
└── build/                            icônes générées, entitlements macOS, certificats
```

### 4.2 Les 3 processus et leur responsabilité

| Processus | Rôle | Remplace |
|---|---|---|
| **Main** (Node) | Accès disque, réseau, MT5, chiffrement, fenêtres, menu, tray, updater, cloud, agent IA, stats | Tout le C# de `services/` + `strategie.cs` + `FxCloudService` |
| **Preload** | Passerelle sécurisée : expose **uniquement** une API typée via `contextBridge` | `ChartBridge.cs` (pont COM) |
| **Renderer** (Chromium) | L'UI React + le chart | WPF + WebView2 |

**Règle d'or :** le renderer ne fait **jamais** d'accès disque ni de requête privilégiée directement. Tout passe par `window.dataedge.*` → IPC → main.

### 4.3 Fenêtre principale (sécurité non négociable)

```ts
// src/main/window.ts
new BrowserWindow({
  width: 1400, height: 900, minWidth: 1100, minHeight: 700,
  frame: false,                                     // barre de titre custom (T2)
  titleBarStyle: process.platform === 'darwin' ? 'hiddenInset' : 'hidden',
  backgroundColor: '#131722',
  show: false,                                      // affichée après 'ready-to-show'
  webPreferences: {
    preload: path.join(__dirname, '../preload/index.js'),
    contextIsolation: true,     // ⚠️ NON NÉGOCIABLE
    nodeIntegration: false,     // ⚠️ NON NÉGOCIABLE
    sandbox: true,              // ⚠️ NON NÉGOCIABLE
    webSecurity: true,
    spellcheck: false,
  },
});
```

### 4.4 Le contrat `window.dataedge` (extrait)

```ts
// src/preload/index.ts
contextBridge.exposeInMainWorld('dataedge', {
  // ── App ──
  getVersion: () => ipcRenderer.invoke('app:getVersion'),
  getPaths:   () => ipcRenderer.invoke('app:getPaths'),
  minimize:   () => ipcRenderer.send('app:minimize'),
  maximize:   () => ipcRenderer.send('app:maximize'),
  close:      () => ipcRenderer.send('app:close'),

  // ── Chart (remplace ChartBridge.cs) ──
  chart: {
    saveReplayPosition:      (sym, tf, ts) => ipcRenderer.invoke('chart:saveReplayPosition', sym, tf, ts),
    prepareReplayResume:     (sym, tf)     => ipcRenderer.invoke('chart:prepareReplayResume', sym, tf),
    getReplayPosition:       (sym, tf)     => ipcRenderer.invoke('chart:getReplayPosition', sym, tf),
    saveScreenshot:          (type, b64)   => ipcRenderer.invoke('chart:saveScreenshot', type, b64),
    onTradeSetupCompleted:   (trade)       => ipcRenderer.invoke('chart:onTradeSetupCompleted', trade),
    loadCandles:             (sym, tf, year) => ipcRenderer.invoke('chart:loadCandles', sym, tf, year),
  },

  // ── Store ──
  strategies: { list, get, create, update, remove, addTrade, updateTrade, removeTrade },
  etudes:     { catalog, read, write, create, remove },
  replay:     { save, load },

  // ── Cloud ──
  cloud: { login, logout, me, sync, cancelSync, listFiles, listBackups, restore, deleteFile, storage },

  // ── MT5 ──
  mt5: { detectTerminals, installBridge, launchTerminal, preview, import, watchState, livePositions },

  // ── Agent IA ──
  agent: { chat: (payload, onChunk) => { /* EventEmitter → onChunk */ }, history, clearHistory, confirm },

  // ── Space ──
  space: { sync, searchUsers, getProfile, conversations, sendMessage, posts, react, block, settings },

  // ── Docs ──
  doc: { exportPdf, exportWord, readExcel, readPdf, openFolder, openFile, saveImportedFile },
});
```

> ⚠️ **Chaque handler IPC valide ses arguments avec `zod`** avant toute opération disque/réseau (`main/security/ipcGuard.ts`). Un renderer compromis ne peut pas écrire hors de `userData`.

---

## 5 — Spécification du bridge Chart

### 5.1 État actuel (`ChartBridge.cs` — 11 méthodes, `AddHostObjectToScript`)

| Méthode C# | Type | Appelée depuis `chart_engine.js` |
|---|---|---|
| `SaveReplayPosition(string, string, long)` | void | ligne 25 |
| `GetReplayPositionTimestamp(string, string)` | long | (via `PrepareReplayResume`) |
| `PrepareReplayResume(string, string)` | string (JSON) | ligne 419 |
| `OnSetupCreated(string)` | void | — (logique vide) |
| `LoadYearForBacktest(int)` | void | lignes 664, 682 |
| `loadPreviousYear(long)` | void | ligne 360 |
| `loadNextYear(long)` | void | ligne 934 |
| `LoadPreviousYearForReplay(long)` | void | ligne 244 |
| `ExitReplayMode()` | void | ligne 399 |
| `OnTradeSetupCompleted(string)` | void | ligne 1013 |
| `SaveChartScreenshot(string, string)` | void | ligne 1037 |

### 5.2 État cible (9 canaux IPC)

| Canal IPC | Direction | Remplace | Note |
|---|---|---|---|
| `chart:saveReplayPosition` | R→M | `SaveReplayPosition` | — |
| `chart:prepareReplayResume` | R→M→R | `PrepareReplayResume` | — |
| `chart:getReplayPosition` | R→M→R | `GetReplayPositionTimestamp` | — |
| `chart:loadCandles` | R→M→R | *(nouveau)* | Renvoie le CSV brut ou l'URL de fetch |
| `chart:saveScreenshot` | R→M | `SaveChartScreenshot` | → `cacheimage/{ts}_{HTF\|LTF}.png` |
| `chart:onTradeSetupCompleted` | R→M | `OnTradeSetupCompleted` | → `store:addBacktestTrade` |
| ~~`LoadYearForBacktest`~~ | — | **SUPPRIMÉ** | Le renderer fait le `fetch` lui-même |
| ~~`loadPreviousYear`~~ | — | **SUPPRIMÉ** | idem |
| ~~`loadNextYear`~~ | — | **SUPPRIMÉ** | idem |
| ~~`LoadPreviousYearForReplay`~~ | — | **SUPPRIMÉ** | idem |
| ~~`ExitReplayMode`~~ | — | **SUPPRIMÉ** | Plus de C# à prévenir |
| `OnSetupCreated` | — | — | Vide aujourd'hui, non porté |

**➡️ 4 appels sur 8 disparaissent. Toute la couche C# de téléchargement / GZip / parsing CSV (`Dataservice.cs` + ~800 lignes de `Chart.xaml.cs`) est supprimée.**

### 5.3 `renderer/chart/bridge.ts` — le remplacement exact

```ts
// ⭐ LE SEUL NOUVEAU FICHIER DU CHART
// Remplace : window.chrome.webview.hostObjects.chartService
type ChartBridgeApi = Window['dataedge']['chart'];

const api: ChartBridgeApi = window.dataedge.chart;

/** Compat : les 8 appels JS existants continuent de fonctionner sans modification. */
export function installChartBridge() {
  (window as any).chartService = {
    // ── Conservés ──
    SaveReplayPosition: (sym: string, tf: string, ts: number) =>
      void api.saveReplayPosition(sym, tf, ts),

    PrepareReplayResume: (sym: string, tf: string) =>
      api.prepareReplayResume(sym, tf),          // Promise<JSON string>

    SaveChartScreenshot: (type: string, b64: string) =>
      void api.saveScreenshot(type, b64),

    OnTradeSetupCompleted: (json: string) =>
      void api.onTradeSetupCompleted(JSON.parse(json)),

    // ── Supprimés côté C# → implémentés ici ──
    LoadYearForBacktest: (year: number) => void loadYear(year),
    loadPreviousYear:    (ts: number) => void loadAdjacentYear(ts, 'prev'),
    loadNextYear:        (ts: number) => void loadAdjacentYear(ts, 'next'),
    LoadPreviousYearForReplay: (ts: number) => void loadPrecedingContext(ts),
    ExitReplayMode:      () => { /* no-op : plus de C# à prévenir */ },
  };
}
```

> **Conséquence :** dans `chart_engine.js`, **aucune ligne des 8 appels n'a besoin de changer** si on installe `window.chartService`. La migration est **transparente** pour le moteur de chart.
> *(Variante plus propre, optionnelle : remplacer `window.chrome.webview.hostObjects.chartService` par `window.chartService` aux 8 endroits.)*

### 5.4 `renderer/chart/csv.ts` — le remplacement de `Dataservice.cs`

```ts
// Remplace : Dataservice.GetMarketDataAsync + ParseCsvToCandles (~800 lignes de C#)
const API = 'https://fxdataedge.com/api/public/data/fetch';

export type Candle = { time: number; open: number; high: number; low: number; close: number };

export async function fetchCandles(pair: string, tf: string, year: string): Promise<Candle[]> {
  const url = `${API}?pair=${pair}&tf=${tf}&year=${year}`;
  const res = await fetch(url);
  if (res.status === 404) throw new NoDataError(`${pair} ${tf} ${year}`);   // → toast T23
  if (!res.ok) throw new Error(`SERVER_ERROR: ${res.status}`);

  // Décompression gzip NATIVE (Chromium) — remplace le GZipStream du C#
  const stream = res.body!.pipeThrough(new DecompressionStream('gzip'));
  const text = await new Response(stream).text();

  return parseCsv(text);   // dans un Web Worker si le fichier dépasse ~50 000 lignes
}
```

**Règles de partitionnement à conserver (`GetFileToRequest`)** :

| Timeframe | Bloc demandé |
|---|---|
| `1m`, `5m`, `15m`, `30m`, `1h` | Année exacte |
| `4h`, `D` | `2006-2016` → `"2016"` · `2017-2026` → `"2026"` |
| `W`, `M` | Toujours `"2026"` |

**Gains :** suppression de `Dataservice.cs`, du cache CSV intermédiaire côté C#, du parsing CSV C#, de la sérialisation JSON + `ExecuteScriptAsync` (limité en taille de payload). **Le rendu sera plus rapide qu'aujourd'hui.**

### 5.5 Ce qui reste **strictement identique** dans le chart

| Élément | Fichier | Statut |
|---|---|---|
| 12 outils de dessin | `drawing_configs.js` | ♻️ Inchangé |
| Contrôleur de dessin (mode, points, sélection, drag, redim., édition texte/Fibo) | `drawing_manager.js` (27 Ko) | ♻️ Inchangé |
| Plugin de rendu canvas + `_resolveX` | `drawing_plugin.js` | ♻️ Inchangé |
| Utilitaires de dessin (hit test, distance, preview SVG) | `drawing_utils.js` | ♻️ Inchangé |
| `initChart`, `updateChartData`, `mergePrecedingData`, `prependChartData` | `chart_engine.js` | ♻️ Inchangé |
| Machine à états replay (play/pause/step/jump, vitesse, `endReached`) | `chart_engine.js` | ♻️ Inchangé |
| Lazy loading + préchargement du contexte gauche | `chart_engine.js` | ♻️ Inchangé |
| `getExtendedTimeline` (200 fantômes avant, 2000 après) | `chart_engine.js` | ♻️ Inchangé |
| Thème, grille, échelles P/L/A, couleurs up/down | `chart_engine.js` | ♻️ Inchangé |
| `cyber-notify`, `cyberLog`, loader | `chart_engine.js` + CSS | ♻️ Inchangé |
| `captureChart(type)` | `chart_engine.js` | ♻️ Inchangé (le bridge change, pas la fonction) |
| Persistance des dessins | `localStorage` (`Drawings_{symbol}`) | ♻️ Inchangé *(option : migrer vers le serveur pour la synchro multi-appareils)* |
| HTML (toolbar, sidebar, container, notify, loader) | `index.html` | ♻️ Inchangé |
| Styles | `style.css`, `toolbar.css`, `replay.css` | ♻️ Inchangé |
| `lightweight-charts.js` (160 Ko) | — | ♻️ Inchangé |

**➡️ Sur 11 fichiers du chart, seul un nouveau `bridge.ts` est ajouté. Le reste est copié tel quel.**

---

## 6 — Spécification du module MT5

### 6.1 Les 14 modules C# → 14 modules Node

| # | Source C# | Cible TS | Points d'attention du portage |
|---|---|---|---|
| 1 | `Mt5Models.cs` | `mt5/models.ts` | Types stricts (deals, positions, lignes d'import, options, rapport) |
| 2 | `Mt5Text.cs` | `mt5/text.ts` | Normalisation accents, **séparateurs décimaux** (`,` / `.`), encodages |
| 3 | `Mt5FileUtil.cs` | `mt5/fileUtil.ts` | Lecture avec ré-essais (150/400/1000 ms) |
| 4 | `Mt5PositionBuilder.cs` | `mt5/positionBuilder.ts` | Agrégation deals → positions (entrées/sorties/partiels, **dépôts ignorés**) |
| 5 | `Mt5TradeMapper.cs` | `mt5/tradeMapper.ts` | **Table TP/TR/SL/BE/PARTIAL** + R au SL de clôture |
| 6 | `Mt5BridgeSource.cs` | `mt5/bridgeSource.ts` | ⚠️ **Chemins par OS** (voir 6.2), fusion des exports, comptage dédupliqué |
| 7 | `Mt5ReportSource.cs` | `mt5/reportSource.ts` | Parsing `.html` (UTF-8 **et** UTF-16) + `.xlsx` (→ `exceljs`), colonnes localisées |
| 8 | `Mt5Settings.cs` | `mt5/settings.ts` | Réglages de la journalisation auto |
| 9 | `StrategieJournalWriter.cs` | `mt5/journalWriter.ts` | Écriture groupée + `.bak` + atomique (réutilise `store/atomic.ts`) |
| 10 | `Mt5ImportService.cs` | `mt5/importService.ts` | ⚠️ **Verrou nommé** → `proper-lockfile` ; anti-doublon **sous verrou** ; rollback ; rapport |
| 11 | `Mt5BridgeInstaller.cs` | `mt5/bridgeInstaller.ts` | ⚠️ **P/Invoke `user32` supprimé** ; `metaeditor64.exe /compile /log` via `child_process.spawn` ; lecture journal UTF-16 |
| 12 | `Mt5BridgeWatcher.cs` | `mt5/bridgeWatcher.ts` | `FileSystemWatcher` → **`fs.watch`** + contrôle périodique de secours |
| 13 | `Mt5BridgeStatus.cs` | `mt5/bridgeStatus.ts` | Battement de cœur EA |
| 14 | `Mt5LivePositions.cs` | `mt5/livePositions.ts` | `positions_<login>.live`, fraîcheur (horloge locale), solde/équité/marge, tolérance 15 s |
| 15 | `Mt5WatchState.cs` | `mt5/watchState.ts` | → **`EventEmitter`** Node au lieu d'événements statiques C# |
| 16 | `Mt5ExportArchiver.cs` | `mt5/exportArchiver.ts` | Archivage **ciblé** (tous les trades réglés) + `MessageConserves()` |

### 6.2 ⚠️ Le point critique : les chemins par OS

| OS | Dossier Common de MT5 | Détection |
|---|---|---|
| **Windows** | `%APPDATA%\MetaQuotes\Terminal\Common\Files\DataEdge` | Automatique |
| **macOS** (MT5 sous Wine) | `~/Library/Application Support/net.metaquotes.wine.metatrader5/drive_c/users/<user>/AppData/Roaming/MetaQuotes/Terminal/Common/Files/DataEdge` | **Auto-détection + repli manuel** |
| **Varia** | Autre préfixe Wine / autre emplacement | Réglage manuel dans **Settings > JOURNAL MT5** |

**Stratégie d'implémentation :**
1. Auto-détection au démarrage : tester une liste ordonnée de chemins candidats par OS.
2. Si aucun trouvé → **ne pas bloquer** : afficher dans Settings > JOURNAL MT5 un sélecteur de dossier (`dialog.showOpenDialog`) avec explication.
3. Persister le chemin retenu dans `config.json` (`mt5_data_folder`).
4. Revalider à chaque démarrage (un utilisateur peut réinstaller MT5).
5. **Bouton « Détecter automatiquement »** dans les réglages.

**Détection des terminaux (Windows)** : parcourir `%APPDATA%\MetaQuotes\Terminal\*\`, retenir ceux contenant `MQL5\`, nom lisible via `origin.txt`.
**Détection des terminaux (macOS)** : même logique dans le préfixe Wine détecté.

### 6.3 Ce qui NE change PAS

| Élément | Statut |
|---|---|
| `resources/mt5/DataEdgeBridge.mq5` | ♻️ **Inchangé** |
| `resources/mt5/DataEdgeExport.mq5` | ♻️ **Inchangé** |
| `resources/mt5/DataEdgeBridge.tpl` | ♻️ **Inchangé** |
| `resources/mt5/INSTALL_MT5.txt` | ♻️ Inchangé *(option : ajouter une section macOS)* |
| Format des fichiers d'export MT5 | ♻️ **Inchangé** |
| Format de `positions_<login>.live` | ♻️ **Inchangé** |
| Table de décision `TP/TR/SL/BE/PARTIAL` | ♻️ **Comportement identique** |
| Anti-doublon par `Trade.SourceKey` | ♻️ Comportement identique (mécanisme différent) |

### 6.4 Robustesse à conserver (exigence non négociable)

| Garantie | Implémentation actuelle | Cible Node |
|---|---|---|
| Aperçu 100 % en mémoire (rien écrit avant validation) | `Mt5ImportService.BuildPreview` | Identique |
| Backup `.bak` + écriture **atomique** (`.tmp` GUID + `File.Replace`) | `Strategie.SaveDataAtomic` | `store/atomic.ts` (tmp + `fs.rename` + vérif + rollback) |
| Remplacement avec **ré-essais** 0/80/200/500/1200 ms | `SaveDataAtomic` | Identique |
| Vérification post-écriture tolérante (3 tentatives) | `SaveDataAtomic` | Identique |
| **Verrou d'écriture partagé** (réentrant) | `Strategie.VerrouEcriture` | `store/lock.ts` (mutex async réentrant + `proper-lockfile` inter-processus) |
| Anti-doublon **sous verrou** | `Mutex "DataEdge_Mt5Import"` | `proper-lockfile` sur un fichier sentinelle |
| Vérification post-import (clés relues) | `Mt5ImportService.Import` | Identique |
| Refus d'écrire si fichier illisible/verrouillé | `TryReadFileText` | Identique |
| Archivage ciblé (pas de perte de trades) | `Mt5ExportArchiver` | Identique |

---

## 7 — Spécification de l'agent IA

### 7.1 Architecture (inchangée)

```
Renderer (ChatPanel.tsx)
    │ window.dataedge.agent.chat(payload, onChunk)
    ▼
Main (agentClient.ts)
    │ POST /api/ai/chat  (SSE, JWT)
    ▼
Backend Symfony (fxglobal/)  ──►  Chaîne de fournisseurs LLM
    │  chunks: {text} | {reasoning} | {tool_call} | {ping} | {error} | {status}
    ▼
Main exécute les tools LOCALEMENT → renvoie tool_results → tour N+1
```

**Le backend ne change pas.** Le protocole SSE ne change pas. Ce qui change : le portage des 5 services de tools en TypeScript.

### 7.2 Les 34 tools

| Famille | Nb | Fichier cible | Statut |
|---|---|---|---|
| **Workspace** | 20 | `main/agent/tools/workspace.ts` | 🔨 Porté |
| **Web & Marché** | 7 | `main/agent/tools/market.ts` | 🔨 Porté |
| **Fichiers locaux** | 7 | `main/agent/tools/files.ts` | 🔨 Porté |
| **Études** (inclus dans workspace) | — | `main/agent/tools/studies.ts` | 🔨 Porté |
| **Weeks / notes hebdo** (7 dédiés) | — | `main/agent/tools/weeks.ts` | 🔨 Porté |

### 7.3 Tools workspace (20)

| Tool | Rôle | Confirmation |
|---|---|---|
| `get_workspace_snapshot` | Résumé workspace (stratégies + stats + structure des champs de confluence + 25 derniers trades + chemins des études) | — |
| `get_strategy_details` | Détail d'une stratégie | — |
| `search_trades` | Recherche de trades | — |
| `create_strategy` | Création de stratégie | — |
| `delete_strategy` | Suppression de stratégie | ✅ |
| `add_journal_trade` | Ajout d'un trade au journal | — |
| `add_backtest_trade` | Ajout d'un trade au backtest | — |
| `update_journal_trade` | Modification d'**UN SEUL** trade par `trade_id` | ✅ |
| `update_backtest_trade` | idem backtest | ✅ |
| `delete_journal_trade` | Suppression d'**UN SEUL** trade par `trade_id` | ✅ |
| `delete_backtest_trade` | idem backtest | ✅ |
| `reload_application` | Rechargement de l'application | ✅ |
| `list_local_folder` | Lister un dossier | — |
| `read_local_file` | Lire un fichier | — |
| + tools études | `get_catalog`, `read`, `search`, `create`, `write`, `delete` | ✅ pour `write`/`delete` |

### 7.4 Tools web & marché (7)

| Tool | Source |
|---|---|
| `get_server_time` | `GET /api/market/time` |
| `get_local_time` | Horloge locale du poste |
| Calendrier économique | `GET /api/market/calendar` (cache partagé serveur) |
| FedWatch | `GET /api/market/fedwatch` |
| Sentiment retail | `GET /api/market/sentiment` |
| Cotations | `GET /api/market/quotes` |
| Actualités | `GET /api/market/news` |
| Recherche web / lecture de page | Étape Gemini `google_search` + lecture directe |

### 7.5 Tools fichiers locaux (7)

| Tool | Rôle | Cible Node | ⚠️ |
|---|---|---|---|
| `read_local_file` | Lire txt/md/json/csv/xml/**pdf**/**xlsx**/**docx** | `pdf-parse`, `exceljs`, `mammoth` | — |
| `search_in_files` | Recherche de texte | Identique | — |
| `create_text_file` | Créer txt/json/md | Identique | — |
| `create_pdf_file` | PDF A4 design DataEdge + side-cars `.src.md` / `.title.txt` | `pdfkit` | — |
| `create_word_file` | `.docx` A4 (styles Normal/Heading1-4, listes, citations) | `docx` | — |
| `append_document` | Ajouter une section en fin de document (`.pdf`, `.docx`, `.md`, `.txt`) | Identique | ⚠️ verrou de fichier macOS ≠ Windows |
| `open_folder` / `open_file` | Ouvrir un dossier / lancer un fichier | `shell.openPath`, `shell.showItemInFolder` | — |

**Sécurité des chemins (à conserver) :**
- **Lecture** : tout chemin valide (`C:\...`, `~`, `%USERPROFILE%`… ou relatif) — l'agent peut lire **tous** les dossiers, dans la limite des permissions OS.
- **Écriture** : **uniquement** sous `Documents/DataEdge` (`IsInsideAgentRoot`).
- `open_file` : `shell.openPath` (équivalent cross-platform de `Process.Start` + `UseShellExecute`).
- `save_imported_file` : copie dans `DataEdge/Imports` horodaté, `.doc` refusé.

### 7.6 Points d'attention du portage

| Point | Détail |
|---|---|
| **Boucle agent** | 8 tours max + **tour de finalisation garanti** (uniquement si le modèle n'a pas répondu en texte — flag `answeredTextually`) |
| **Confirmations sélectives** | `requires_confirmation` : seules les actions qui **écrasent/suppriment** (delete_*, update_*_trade, write_study/week, reload_application). Les créations et ouvertures passent directement. Boîte à 3 choix : Autoriser / Tout autoriser / Refuser |
| **Coercition des arguments** | `GetString`/`GetBool`/`GetNumber` tolèrent tous les ValueKind JSON ; `GetStringList` accepte un objet ; `GetCustomFieldValues` + `NormalizeFieldName` + `BuildConfluences` remplissent `ChampsPersonnalises` **exactement comme la fenêtre AjoutTrade** |
| **Contexte d'identité** | Sérialisé en JSON (profil cloud + `storage_knowledge`), envoyé **au premier tour seulement**, persisté `role=context` côté serveur |
| **`storage_knowledge`** | Constante décrivant le stockage interne (`data/<Nom>.json`, `metadata/strategies.txt`, `.bak`, onglet CLOUD) — le modèle l'utilise **en interne** pour diagnostiquer, **sans jamais l'exposer** à l'utilisateur |
| **Reasoning** | Les chunks `{reasoning}` vont au **bandeau de statut**, jamais dans le texte final |
| **Statuts des tools** | `🔍 …` / `✓ …` émis au callback de statut |
| **Erreurs de tool** | `ExecuteSafelyAsync` intercepte et renvoie `is_error: true` (le modèle se corrige) |
| **Historique** | `GET /api/ai/history` pour l'affichage ; `DELETE` efface tout (serveur + mémoire modèle) |
| **Bouton 🗑** | Supprime l'historique serveur + la session de prompt caching |
| **Payload** | Le client n'envoie **que l'identité** au premier tour ; le workspace est lu **à la demande** par les tools |

---

## 8 — Spécification du module Space

### 8.1 État actuel

- **Phase 1 — Frontend coquille COMPLET ✅** (build OK). Le **backend `/api/space/**` n'existe pas encore** : les services sont des mocks contract-first persistés localement.
- Le contrat (`ISpaceService`, DTOs, format du Backtest Pack) est **figé v1** → il doit être respecté à l'identique pour que le swap coquille → réel fonctionne.

### 8.2 Décisions structurantes à préserver

| # | Décision | Choix |
|---|---|---|
| **D1** | Temps réel | **Polling intelligent** (`GET /api/space/sync?since=`, ~20 s, **seulement quand la vue Space est ouverte**). Pas de WebSocket ni SSE (PHP-FPM : 1 worker = 1 requête bloquée). Badge non-lus piggyback sur le `HandshakeResponse` |
| **D2** | Ordre de construction | **Frontend d'abord**, backend ensuite |
| **D3** | Le différenciateur | **La donnée vérifiable** : Backtest Pack exporté du backtesteur, vérifié par SHA-256, ouvrable et **rejouable** par le mentor |

### 8.3 Les 12 modules

| Source C# | Cible TS | Rôle |
|---|---|---|
| `ISpaceService.cs` | `main/space/ISpaceService.ts` (+ `spaceContract.ts`) | Interface : événements `MessageReceived` / `BadgesChanged`, `sync`, `searchUsers`, `getProfile`, `getMyProfile`, conversations, groupes, posts, réactions, blocage, settings |
| `SpaceServiceFactory.cs` | `main/space/factory.ts` | **Seul fichier à modifier au swap** |
| `SpaceDtos.cs` | `shared/models/space.ts` | DTOs figés v1 (ids `int`, timestamps `long` unix, chemins relatifs) |
| `SpaceMockService.cs` | `main/space/mockService.ts` | Coquille (simulation d'événements, latence, pagination) |
| `SpaceMockStore.cs` | `main/space/mockStore.ts` | Persistance locale (`userData/space/`) |
| `SpaceMockSeed.cs` | `main/space/mockSeed.ts` | Jeu de démo |
| `SpaceBacktestPackBuilder.cs` | `main/space/packBuilder.ts` | **Pack** : JSON + copies des captures HTF/LTF + **hash SHA-256** |
| `SpacePackVerifier.cs` | `main/space/packVerifier.ts` | Vérification du hash |
| `SpaceLocalExportBuilder.cs` | `main/space/localExport.ts` | Exports stats / stratégie / week / trade / étude / fichier |
| `SpaceAttachmentFormats.cs` | `main/space/attachmentFormats.ts` | Formats et kinds |
| `SpacePostCard.cs` | `renderer/features/space/components/PostCard.tsx` | Carte de post réutilisable |
| `SpaceUiHelper.cs` | `renderer/features/space/components/ui.ts` | Avatars initiales (8 tons + ✦ mentor + présence), rendu attachments, temps/tailles, `Pill`, `SoftCard`, `RoundInput` |

### 8.4 Les 12 vues

| Vue | Cible | Contenu à reproduire |
|---|---|---|
| `SpaceView` | `SpaceRoot.tsx` | Rail gauche (avatar, Accueil, DMs, Communautés, Recherche, Missions P2, ⚙️, ＋ groupe) + liste avec badges + zone contenu + polling 20 s + toast + pile `‹ Retour`/`⌂ Accueil` + **Échap = Accueil** |
| `SpaceHubHomeView` | `Hub.tsx` | Feed global, suggestions, encarts partage, bannière gradient |
| `SpaceConversationView` | `Conversation.tsx` | Thread, composer WhatsApp-like (stats/stratégie/week/trade/étude/fichier), chips, zone safe 46 px pour le FAB, composer réservé 64 px à droite |
| `SpaceCommunityView` | `Community.tsx` | Bannière + onglets Feed / Membres / Missions 🔒 / Signaux 🔒 |
| `SpaceProfileView` | `Profile.tsx` | Avatar initiales, bio, snapshots, Message/Bloquer/Signaler |
| `SpaceSearchView` | `Search.tsx` | Anti-rebond 350 ms |
| `SpacePostEditorWindow` | `PostEditor.tsx` | kind + contenu + multi-attachments + communauté |
| `SpacePackViewerWindow` | `PackViewer.tsx` | Bandeau de vérification SHA-256, en-tête (stratégie/auteur/total R), grille de stats, DataGrid des trades, détail (champs perso + captures zoomables), **Rejouer sur le chart** |
| `SpaceShareDestinationWindow` | `ShareDialog.tsx` | Message optionnel + conversations + mes communautés |
| `SpaceGroupCreateWindow` | `GroupCreate.tsx` | Nom + membres pré-remplis + recherche |
| `SpaceSettingsWindow` | `Settings.tsx` | Visibilité recherche, consentement RGPD, blocages, CGU |
| `SpaceCguWindow` | `Cgu.tsx` | CGU complètes (objet, UGC, données de backtest, modération, RGPD, disclaimer AMF, formations) |

### 8.5 Design System à préserver

- Thème sombre ; accents **vert EDGE `#FF00FF03`** et **cyan `#00BFFF`**.
- **Boutons pilules** (radius 18), cartes radius 12 avec hover, **chips colorés par kind** (📦 pack / 📊 snapshot / 📄 étude / 🖼️ image / 📎 fichier), champs arrondis, avatars à anneau, bannières gradient statique, **fondu d'opacité 0,15 s**.
- Fenêtres Space arrondies (radius 14), surfaces `#151D27` / `#25333F`.
- Cartes/bulles : `#0D141D` fond, `#131C26` cartes, `#0A0A0A` champs, bordures `#202B36`, hover `#18232E`.
- Typo Segoe UI ; labels secondaires gris `#5E6A76`–`#8A94A0` (9,5-10 px).
- **FAB du copilote masqué dans Space** (réglage global `ShowAiFabOutsideDashboard`).
- **Listes virtualisées** dès le premier jet (les feeds peuvent être longs).
- **Images décodées en miniature** dans les feeds (`DecodePixelWidth`), plein format seulement dans la visionneuse.
- ⚠️ **Contrainte de rendu historique** (`SoftwareOnly`) : plus d'actualité en Electron (rendu GPU Chromium) → **les animations lourdes redeviennent possibles**, mais rester sobre par cohérence.

### 8.6 ⚠️ Le point d'attention

> **`SpacePackViewerWindow` doit pouvoir « Rejouer sur le chart »** : le pack injecte une stratégie + ses trades dans le backtesteur. Ce chemin traverse **Space → store → backtester → chart**. C'est l'intégration la plus profonde du logiciel : **à tester en priorité** (voir §14).

---

## 9 — Migration du format `.etude`

### 9.1 Le problème

`.etude` = `TextRange.Save(stream, DataFormats.XamlPackage)` → **format propriétaire WPF** (XAML + images dans un conteneur ZIP OPC). **Aucun autre framework ne sait le lire.**

Fichiers concernés :
- `etudes/{dossier}/{nom}.etude` — études libres
- `Notes/Notes_yyyyMMdd.etude` — notes hebdomadaires du dashboard

⚠️ **Ces fichiers contiennent des données réelles d'utilisateurs.** C'est le chantier le plus risqué du projet.

### 9.2 Le nouveau format `.etude` v2

**Conteneur ZIP** (extension conservée `.etude` pour la compatibilité) :

```
mon_etude.etude (ZIP)
├── manifest.json      { "version": 2, "format": "dataedge-etude",
│                        "createdAt": "...", "updatedAt": "...",
│                        "title": "...", "author": "..." }
├── content.json       Document ProseMirror/TipTap (JSON)
├── content.html       Rendu HTML (repli + export)
└── assets/
    ├── img_001.png
    └── img_002.jpg
```

**Avantages :** lisible (ZIP standard), extensible (ajouter un champ = ajouter une clé), versionné, images séparées, compatible export HTML/PDF/Markdown.

### 9.3 Le convertisseur XamlPackage → v2

**Où l'exécuter ?** Les deux options sont à implémenter :

| Option | Où | Quand | Avantage |
|---|---|---|---|
| **A. Côté serveur (recommandé)** | Symfony, sur `fx_global_storage/user_{id}/FX_DATAEDGE/etudes/` | Une fois, en batch, à la bascule | Aucune dépendance WPF, traite **tous** les utilisateurs d'un coup |
| **B. Côté ancien client WPF** | Dans le client C# (qui lit le XamlPackage nativement) | Avant la bascule, pendant la synchro | Fidélité maximale de conversion |

**Mapping de conversion :**

| WPF (FlowDocument) | TipTap / HTML |
|---|---|
| `Run` / `Span` | nœud `text` |
| `Bold` / `Italic` / `Underline` | `bold` / `italic` / `underline` |
| `Foreground` (couleur) | `textStyle` → `color` |
| `Background` (surlignage) | `highlight` |
| `FontSize` | `textStyle` → `fontSize` |
| `Paragraph` / `TextAlignment` | `paragraph` / `textAlign` |
| `List` (Ordered / Unordered) | `orderedList` / `bulletList` |
| `Table` / `TableRow` / `TableCell` | `table` / `tableRow` / `tableCell` |
| `Image` (+ `BitmapSource`) | `image` + fichier dans `assets/` |
| `LineBreak` | `hardBreak` |
| `Hyperlink` | `link` |
| `Section` | (aplati) |

### 9.4 Procédure de migration (sécurisée)

1. **Ne jamais modifier l'original.** Chaque `.etude` converti est écrit à côté, l'original conservé.
2. **Backup `.bak`** avant toute écriture (mécanisme `safeDelete` / `safeCopy` existant).
3. **Idempotence** : si `manifest.json` a `version: 2`, on ne reconvertit pas.
4. **Rollback** : bouton « Revenir à la version précédente » via l'onglet CLOUD (les `.bak` serveur sont déjà listés et restaurables).
5. **Journal** : chaque conversion tracée (`metadata/etude_migration_log.txt`) avec le nombre d'éléments convertis et les avertissements.
6. **Test d'aller-retour obligatoire** : v1 → v2 → rendu → comparaison visuelle avec l'original WPF, sur un jeu de fichiers réels (gras, italique, souligné, couleurs, surlignage, listes, tableaux, images, alignements).

### 9.5 Points de vigilance

| Risque | Mitigation |
|---|---|
| Perte de mise en forme exotique (WPF a des capacités que TipTap n'a pas : floaters, pagination, colonnes) | **Inventorier** ce qui est réellement utilisé dans les fichiers utilisateurs **avant** de convertir ; documenter les pertes connues |
| Images mal extraites | Vérifier que le nombre d'images v1 == nombre de fichiers dans `assets/` |
| Encodage / accents | Lire en UTF-8 strict, tester avec des fichiers accentués |
| Fichiers corrompus | Ne pas échouer le batch : marquer en erreur, continuer, rapporter |
| Régressions silencieuses | **Test d'égalité de texte extrait** : `AgentStudiesService` sait extraire le texte d'un `.etude` → le texte v1 vs v2 doit être **identique** |

> 💡 **Le test d'égalité de texte extrait est la meilleure garantie automatique de non-perte de contenu.** À exécuter sur **100 %** des fichiers utilisateurs avant bascule.

---

## 10 — Stockage & chemins par OS

### 10.1 Matrice des chemins

| Donnée | Windows (actuel) | Windows (Electron) | macOS (Electron) |
|---|---|---|---|
| **Racine** | `AppDomain.BaseDirectory` ⚠️ (à côté du `.exe`) | `%APPDATA%\DataEdge\` | `~/Library/Application Support/DataEdge/` |
| `data/` | `.exe/data/` | `<userData>/data/` | `<userData>/data/` |
| `metadata/` | `.exe/metadata/` | `<userData>/metadata/` | `<userData>/metadata/` |
| `etudes/` | `.exe/etudes/` | `<userData>/etudes/` | `<userData>/etudes/` |
| `Notes/` | `.exe/Notes/` | `<userData>/Notes/` | `<userData>/Notes/` |
| `cacheimage/` | `.exe/cacheimage/` | `<userData>/cacheimage/` | `<userData>/cacheimage/` |
| `chart/historical/` | `.exe/chart/historical/` | `<userData>/chart/historical/` | `<userData>/chart/historical/` |
| `importes/` | `.exe/importes/` | `<userData>/importes/` | `<userData>/importes/` |
| `config` | `.exe/config.txt` | `<userData>/config.json` | `<userData>/config.json` |
| `session` | `.exe/session.bin` (DPAPI) | `<userData>/session.bin` (**safeStorage**) | `<userData>/session.bin` (**safeStorage**) |
| `log` | `.exe/log.txt` | `<userData>/logs/main.log` | `<userData>/logs/main.log` |
| Replay / Space | `%LOCALAPPDATA%\DataEdge\` | `<userData>/` | `<userData>/` |
| **Dossier agent IA** | `Documents\DataEdge\` | `Documents/DataEdge/` | `~/Documents/DataEdge/` |
| **Imports agent** | `Documents\DataEdge\Imports` | idem | idem |
| **Rapports PDF** | `Documents\DataEdge\Rapports` | idem | idem |
| **Dossier MT5** | `%APPDATA%\MetaQuotes\...` | idem | préfixe Wine (voir §6.2) |

> `app.getPath('userData')` retourne automatiquement le bon chemin par OS. **Aucun `if (platform)` nécessaire** pour le stockage applicatif.

### 10.2 Migration des données

**Scénario 1 — Nouvel utilisateur (macOS ou Windows)** : rien à migrer, dossiers créés à vide.

**Scénario 2 — Utilisateur Windows existant passant à la version Electron** :
1. Au premier lancement, détecter un dossier `DataEdge` existant (ancien emplacement à côté du `.exe`).
2. Proposer : **« Importer mes données existantes »** (recommandé) ou **« Repartir de zéro »**.
3. Copier `data/`, `metadata/`, `etudes/`, `Notes/`, `cacheimage/`, `chart/historical/`, `importes/` — les originaux sont **conservés**, pas déplacés.
4. Convertir `config.txt` → `config.json`.
5. Convertir les `.etude` (§9).
6. **Ne pas tenter de déchiffrer `session.bin`** (DPAPI Windows) → demander une **ré-authentification** (ADR-009).
7. Journaliser dans `logs/migration.log`.

**Scénario 3 — Utilisateur macOS** : le dossier `userData` est vierge → récupérer les données via **la synchronisation cloud** (elles sont déjà sur `fxdataedge.com` depuis le client WPF). C'est le chemin normal.

> 💡 **Le cloud est le vrai outil de migration Mac.** Un utilisateur qui a déjà utilisé DataEdge sur Windows retrouvera tout (stratégies, trades, notes, captures) en se connectant depuis le Mac — **à condition que les `.etude` aient été convertis** (§9.3 option A : conversion côté serveur).

### 10.3 Robustesse du stockage (à porter à l'identique)

| Mécanisme | Implémentation C# | Cible Node |
|---|---|---|
| Écriture atomique | `.tmp` GUID + `File.Replace` | `fs.writeFile(tmp)` + `fs.rename(tmp, final)` |
| Vérification après écriture | relecture + comparaison | identique |
| Rollback | restauration depuis `.bak` | identique |
| Ré-essais | 0/80/200/500/1200 ms | identique |
| Verrou d'écriture | `VerrouEcriture` statique réentrant | mutex async réentrant + `proper-lockfile` |
| Backup horodaté | `{chemin}.{YYYYMMDD_HHMMSS}.bak` | identique |
| Refus d'écrire si illisible | `TryReadFileText` | identique |
| Lecture tolérante | ré-essais 150/400/1000 ms | identique |

---

## 11 — Sécurité

### 11.1 Modèle de menace

| Menace | Mitigation |
|---|---|
| Renderer compromis (contenu web, dépendance npm) | `contextIsolation`, `sandbox`, `nodeIntegration: false`, **CSP stricte**, validation `zod` de tous les arguments IPC |
| Fuite de tokens | `safeStorage` (Keychain / DPAPI), **jamais** de token en clair, jamais dans le renderer |
| Écriture hors du dossier applicatif | `ipcGuard` : tout chemin d'écriture est résolu puis vérifié comme **sous** `userData` (ou `Documents/DataEdge` pour l'agent) |
| Navigation malveillante | `will-navigate` + `setWindowOpenHandler` → tout lien externe part dans le navigateur système via `shell.openExternal` |
| Injection dans le chart | Le chart ne reçoit **que** des données JSON typées via IPC ; `EscapeJs` n'est plus nécessaire (plus d'`ExecuteScriptAsync` avec concaténation) |
| Traversée de chemin (backend) | **Hors périmètre** — backend inchangé, déjà protégé par `resolveUserPath()` / `resolveUserDir()` |

### 11.2 Configuration de sécurité (obligatoire)

```ts
// src/main/security/session.ts
app.on('web-contents-created', (_e, contents) => {
  // Aucune navigation hors de l'app
  contents.on('will-navigate', (e, url) => {
    if (!url.startsWith('app://')) { e.preventDefault(); shell.openExternal(url); }
  });
  // Aucune nouvelle fenêtre (le calendrier Investing est une webview déclarée)
  contents.setWindowOpenHandler(({ url }) => {
    shell.openExternal(url);
    return { action: 'deny' };
  });
  // Permissions : refuser tout par défaut
  contents.session.setPermissionRequestHandler((_wc, permission, cb) => {
    cb(['clipboard-read', 'clipboard-sanitized-write', 'notifications'].includes(permission));
  });
});
```

**CSP** (via `session.defaultSession.webRequest.onHeadersReceived`) :

```
default-src 'self' app:;
script-src  'self' app:;
style-src   'self' app: 'unsafe-inline';
img-src     'self' app: data: blob:;
font-src    'self' app: data:;
connect-src 'self' app: https://fxdataedge.com https://*.fxdataedge.com;
frame-src   'self' app: https://sslecal2.investing.com;
object-src  'none';
base-uri    'self';
form-action 'none';
```

### 11.3 Chiffrement des tokens

```ts
// src/main/security/session.ts
import { safeStorage } from 'electron';

export function saveTokens(tokens: { jwt: string; refresh: string }) {
  if (!safeStorage.isEncryptionAvailable()) throw new Error('Chiffrement indisponible');
  fs.writeFileSync(sessionPath, safeStorage.encryptString(JSON.stringify(tokens)));
}
export function loadTokens(): Tokens | null {
  if (!fs.existsSync(sessionPath)) return null;
  try { return JSON.parse(safeStorage.decryptString(fs.readFileSync(sessionPath))); }
  catch { return null; }   // ⚠️ ancien format DPAPI Windows → ré-authentification
}
```

### 11.4 Règles absolues

1. ❌ **Jamais** `nodeIntegration: true`.
2. ❌ **Jamais** `contextIsolation: false`.
3. ❌ **Jamais** `@electron/remote`.
4. ❌ **Jamais** exposer `fs`, `child_process` ou `ipcRenderer` brut via `contextBridge`.
5. ❌ **Jamais** concaténer une chaîne dans `executeJavaScript`.
6. ✅ **Toujours** valider les arguments IPC avec `zod` avant usage.
7. ✅ **Toujours** vérifier qu'un chemin d'écriture est sous `userData` (ou `Documents/DataEdge`).
8. ✅ **Toujours** tenir Electron à jour (une CVE Electron = un `.exe`/`.dmg` à republier).

---

## 12 — Packaging & distribution

### 12.1 `electron-builder.yml` (extrait)

```yaml
appId: com.dataedge.app
productName: DataEdge
directories: { output: dist, buildResources: build }
files: ['out/**/*', 'resources/**/*', 'package.json']

mac:
  target:
    - { target: dmg, arch: [arm64, x64] }   # Apple Silicon + Intel
  category: public.app-category.finance
  icon: build/icon.icns
  hardenedRuntime: true
  gatekeeperAssess: false
  entitlements: build/entitlements.mac.plist
  entitlementsInherit: build/entitlements.mac.plist
  notarize: true        # ⚠️ nécessite APPLE_ID / APPLE_APP_SPECIFIC_PASSWORD / APPLE_TEAM_ID

dmg:
  title: DataEdge ${version}
  contents:
    - { x: 130, y: 220, type: file }
    - { x: 410, y: 220, type: link, path: /Applications }

win:
  target:
    - { target: nsis, arch: [x64, arm64] }
  icon: build/icon.ico
  certificateFile: build/cert.pfx              # Authenticode (recommandé)
  certificatePassword: ${env.CSC_KEY_PASSWORD}

nsis:
  oneClick: false
  perMachine: false
  allowToChangeInstallationDirectory: true
  createDesktopShortcut: true
  createStartMenuShortcut: true
  shortcutName: DataEdge
  artifactName: DataEdge-Setup-${version}-${arch}.${ext}

publish:
  provider: generic
  url: https://fxdataedge.com/download/updates
```

### 12.2 Entitlements macOS (`build/entitlements.mac.plist`)

```xml
<key>com.apple.security.cs.allow-jit</key><true/>                      <!-- V8 -->
<key>com.apple.security.cs.allow-unsigned-executable-memory</key><true/>
<key>com.apple.security.cs.disable-library-validation</key><true/>
<key>com.apple.security.files.user-selected.read-write</key><true/>    <!-- MT5, rapports -->
<key>com.apple.security.network.client</key><true/>
```

> ⚠️ **Sans `hardenedRuntime` + notarization**, macOS refuse le `.dmg` (« application endommagée »).
> ⚠️ **Sans `allow-unsigned-executable-memory`**, Chromium (donc **le chart**) ne démarre pas.

### 12.3 Prérequis bloquants

| Prérequis | Coût | Bloquant ? |
|---|---|---|
| **Apple Developer Program** | **99 $/an** | ✅ **OUI** — sans notarization, pas de `.dmg` utilisable |
| Certificat **Authenticode** Windows | ~200-400 €/an | ⚠️ Non bloquant, mais avertissement SmartScreen sinon |
| **Un Mac** (ou runner macOS CI) | — | ✅ **OUI** — le `.dmg` ne se construit que sur macOS |
| **Un PC Windows** (ou runner Windows CI) | — | ✅ **OUI** — le `.exe` ne se construit que sur Windows |

### 12.4 Auto-update

`electron-updater` avec un flux **generic** (ou GitHub Releases) :
1. `autoUpdater.checkForUpdates()` au démarrage.
2. Badge néon dans la barre de titre si une version est disponible (**T8** conservé).
3. Popup avec versions locale/distante + barre de progression (**T8** conservé).
4. Téléchargement → `autoUpdater.quitAndInstall()`.
5. **Différence avec l'ancien système** : plus de `.exe` à télécharger depuis `/download/{appId}/latest` → les binaires sont hébergés par le flux de mise à jour (`latest.yml` / `latest-mac.yml` générés par `electron-builder`). `/software/version` peut rester comme source de vérité du numéro de version.

### 12.5 Icônes à produire

| Plateforme | Fichier | Source |
|---|---|---|
| macOS | `build/icon.icns` | `logopng.png` (341 Ko) ou `logo copie.png` → `iconutil` |
| Windows | `build/icon.ico` | `trading.ico` / `cf-flow-*.ico` (déjà présents) |
| Linux (futur) | `build/icon.png` 512×512 | idem |

### 12.6 Taille attendue

| Livrable | Taille estimée |
|---|---|
| `DataEdge-Setup-x.y.z-arm64.dmg` | ~95-115 Mo |
| `DataEdge-Setup-x.y.z-x64.dmg` | ~100-120 Mo |
| `DataEdge-Setup-x.y.z-x64.exe` | ~85-105 Mo |
| Installé (macOS) | ~260-300 Mo |
| Installé (Windows) | ~240-280 Mo |

*(Contient Chromium (~150 Mo), Node, React, les 280 Ko du chart, les ressources MT5.)*

---

## 13 — CI/CD GitHub Actions

### 13.1 Workflow `build.yml`

```yaml
name: Build DataEdge
on:
  push: { tags: ['v*'] }
  workflow_dispatch:

jobs:
  build:
    strategy:
      matrix:
        include:
          - os: macos-latest      # .dmg (arm64 + x64)
            script: build:mac
          - os: windows-latest    # .exe NSIS (x64 + arm64)
            script: build:win
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - uses: pnpm/action-setup@v4
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint && pnpm typecheck
      - run: pnpm test                       # Vitest
      - run: pnpm ${{ matrix.script }}
        env:
          # macOS
          APPLE_ID:                 ${{ secrets.APPLE_ID }}
          APPLE_APP_SPECIFIC_PASSWORD: ${{ secrets.APPLE_APP_SPECIFIC_PASSWORD }}
          APPLE_TEAM_ID:            ${{ secrets.APPLE_TEAM_ID }}
          CSC_LINK:                 ${{ secrets.MAC_CERT_P12 }}
          CSC_KEY_PASSWORD:         ${{ secrets.MAC_CERT_PASSWORD }}
          # Windows
          WIN_CSC_LINK:             ${{ secrets.WIN_CERT_P12 }}
          WIN_CSC_KEY_PASSWORD:     ${{ secrets.WIN_CERT_PASSWORD }}
          # Auto-update
          GH_TOKEN:                 ${{ secrets.GITHUB_TOKEN }}
      - uses: actions/upload-artifact@v4
        with:
          name: DataEdge-${{ matrix.os }}
          path: dist/*.dmg
          # Windows : dist/*.exe
```

### 13.2 Scripts `package.json`

```json
{
  "scripts": {
    "dev":        "electron-vite dev",
    "build":      "electron-vite build",
    "typecheck":  "tsc --noEmit -p tsconfig.node.json && tsc --noEmit -p tsconfig.web.json",
    "lint":       "eslint . --ext .ts,.tsx",
    "test":       "vitest run",
    "test:e2e":   "playwright test",
    "build:mac":  "pnpm build && electron-builder --mac",
    "build:win":  "pnpm build && electron-builder --win",
    "build:all":  "pnpm build && electron-builder --mac --win"
  }
}
```

### 13.3 Stratégie de branches

| Branche | Rôle | Déclencheur |
|---|---|---|
| `main` | Stable | Tag `v*` → build + release |
| `develop` | Intégration | Build de vérification (sans release) |
| `feature/*` | Développement | Lint + typecheck + tests |

### 13.4 Environnements de test nécessaires

| Environnement | Pourquoi |
|---|---|
| **macOS Apple Silicon** (M1/M2/M3/M4) | Tester le chart, le rendu, MT5 sous Wine |
| **macOS Intel** | Tester la compatibilité x64 |
| **Windows 10** | Cible utilisateur la plus ancienne |
| **Windows 11** | Cible majoritaire |
| **VM Windows sur Mac** | Pour tester MT5 + DataEdge ensemble sur la même machine |

---

## 14 — Plan de test & parité fonctionnelle

> **Checklist de recette** : chaque point doit passer **sur macOS ET sur Windows**, hors ligne **et** en ligne.
> C'est la définition de « fini » du projet. ~200 points.

### 14.1 Démarrage & cycle de vie (12)

- [ ] 1. Splash d'ouverture affiché puis disparaît
- [ ] 2. Onboarding 4 slides au **premier** lancement uniquement
- [ ] 3. Tour guidé Spotlight sur le dashboard (étapes, mise en évidence, fermeture)
- [ ] 4. Single instance : un 2ᵉ lancement ramène la fenêtre existante au premier plan
- [ ] 5. Fermeture par le bouton ✕ (barre de titre custom)
- [ ] 6. Minimiser / maximiser / restaurer
- [ ] 7. Maximisation = **zone de travail exacte** (taskbar / Dock visibles, multi-écrans)
- [ ] 8. Drag de la fenêtre par la barre de titre
- [ ] 9. Redimensionnement, taille minimale respectée
- [ ] 10. Position/taille de la fenêtre mémorisée entre deux lancements
- [ ] 11. Menu applicatif natif (macOS : menu bar ; Windows : menu de fenêtre)
- [ ] 12. Quitter proprement (aucune écriture en cours perdue)

### 14.2 Authentification & session (10)

- [ ] 13. Inscription (`/api/register`) avec avatar
- [ ] 14. Connexion (`/api/login`) → JWT + refresh
- [ ] 15. Tokens stockés chiffrés (`safeStorage`), **jamais en clair sur le disque**
- [ ] 16. Session restaurée au redémarrage (pas de re-login)
- [ ] 17. Refresh automatique du JWT expiré
- [ ] 18. Déconnexion efface les tokens
- [ ] 19. Profil affiché (`/api/me`)
- [ ] 20. Mise à jour du profil + avatar (`/api/user/update`)
- [ ] 21. Migration d'un ancien `session.bin` DPAPI → ré-authentification proposée clairement
- [ ] 22. Mode hors ligne : l'app démarre et fonctionne sans réseau

### 14.3 Handshake, verrouillage, notifications, MAJ (10)

- [ ] 23. Handshake au démarrage (`/software/handshake`)
- [ ] 24. **Verrouillage à distance** (`is_locked`) → app bloquée avec message
- [ ] 25. Message système (`msg_title` / `msg_body` / `msg_type`)
- [ ] 26. Notifications push serveur (4 types) affichées et marquables comme lues
- [ ] 27. Badge MAJ détecté au démarrage
- [ ] 28. Popup MAJ : version locale/distante + barre de progression
- [ ] 29. Téléchargement de la MAJ avec %
- [ ] 30. Installation de la MAJ (`.dmg` / `.exe`) et redémarrage
- [ ] 31. Panneau Support → envoi d'un message (`/support/send`)
- [ ] 32. Crash report envoyé (`/software/report-crash`) après une exception volontaire

### 14.4 Chart — le module critique (30)

- [ ] 33. Le chart s'affiche sur macOS **et** Windows (le bug d'origine)
- [ ] 34. Données chargées pour la paire par défaut
- [ ] 35. Watchlist remplie depuis `/api/public/data/pairs`
- [ ] 36. Cache local des CSV créé dans `userData/chart/historical/`
- [ ] 37. Chargement **depuis le cache** si hors ligne
- [ ] 38. Changement de timeframe (7 TF : 1m, 5m, 15m, 30m, 1h, 4h, D)
- [ ] 39. Changement de paire
- [ ] 40. Règle de partitionnement respectée (1m–1h par année, 4h/D par bloc, W/M fichier unique)
- [ ] 41. Lazy loading vers la gauche (année précédente)
- [ ] 42. Lazy loading vers la droite
- [ ] 43. Fin d'historique détectée → `endReached`, plus de rechargement en boucle
- [ ] 44. Toast « aucune donnée pour {année} sur le serveur » (404)
- [ ] 45. Toast « début de l'historique »
- [ ] 46. Toast « fin des données »
- [ ] 47. Toast « impossible de récupérer les données » (erreur réseau)
- [ ] 48. Timeline étendue (200 fantômes avant, 2000 après) → scroll sans bordure
- [ ] 49. Thème sombre / clair
- [ ] 50. Grille affichée / masquée
- [ ] 51. Échelles P / L / A
- [ ] 52. Couleurs up/down personnalisables
- [ ] 53. Reset de la vue (`fitContent`)
- [ ] 54. Loader affiché pendant les chargements
- [ ] 55. Console de debug masquée (pas de fuite visuelle)
- [ ] 56. Redimensionnement de la fenêtre → le chart s'adapte
- [ ] 57. Performance sur un gros fichier (1m, ~370 000 bougies) : pas de gel > 1 s
- [ ] 58. Mémoire stable après 10 changements de TF/paire (pas de fuite)
- [ ] 59. Le chart reste fonctionnel après un cycle minimiser/restaurer
- [ ] 60. `localStorage` des dessins isolé par symbole

### 14.5 Chart — dessins (16)

- [ ] 61. `trendline` (2 clics)
- [ ] 62. `rectangle` (2 clics)
- [ ] 63. `long_pos` (1 clic, TP/SL auto)
- [ ] 64. `short_pos` (1 clic)
- [ ] 65. `curve` (3 clics)
- [ ] 66. `text` (1 clic, éditable)
- [ ] 67. `horz_ray` / `horz_line` / `vert_line`
- [ ] 68. `path` (polyligne, double-clic pour finir)
- [ ] 69. `arrow`
- [ ] 70. `fibo` (niveaux configurables, éditeur)
- [ ] 71. Prévisualisation SVG pendant le dessin
- [ ] 72. Sélection d'un dessin, déplacement, redimensionnement par ancres
- [ ] 73. Suppression d'un dessin / tout supprimer
- [ ] 74. Persistance après rechargement
- [ ] 75. Pas de conflit double-clic vs dessin
- [ ] 76. Annulation (Échap / clic droit)

### 14.6 Chart — replay / backtest (18)

- [ ] 77. Entrée en mode replay
- [ ] 78. Play / pause / step avant / step arrière
- [ ] 79. Vitesse de replay
- [ ] 80. Saut à une date (`jumpToReplayDate`)
- [ ] 81. Saut vers une année (`LoadYearForBacktest`)
- [ ] 82. Rechargement de l'année cible si la date est hors buffer
- [ ] 83. Centrage correct après un saut
- [ ] 84. Préchargement du contexte gauche (année − 1) sans saut visuel
- [ ] 85. Position du replay **préservée** pendant le préchargement
- [ ] 86. `endReached` stoppe proprement le play
- [ ] 87. Sauvegarde de la position (throttle 2 s + force)
- [ ] 88. **Reprise de session** au clic sur le mode replay (TF + position + switch auto de TF)
- [ ] 89. La reprise **ne se déclenche pas** à l'ouverture du chart
- [ ] 90. Sortie du replay → retour au présent
- [ ] 91. Détection automatique de clôture TP/SL d'un setup
- [ ] 92. Envoi du trade détecté vers le formulaire
- [ ] 93. Capture HTF → `cacheimage/{ts}_HTF.png`
- [ ] 94. Capture LTF → `cacheimage/{ts}_LTF.png`

### 14.7 Journal, stratégies, trades (20)

- [ ] 95. Création de stratégie
- [ ] 96. Modification / suppression de stratégie
- [ ] 97. Registre `metadata/strategies.txt` cohérent (`%`-séparé)
- [ ] 98. Chargement des stratégies au démarrage (registre vide → rien ne « disparaît » silencieusement)
- [ ] 99. Ajout d'un trade (tous les champs)
- [ ] 100. Édition d'un trade
- [ ] 101. Suppression d'un trade (→ `.bak`)
- [ ] 102. RR calculé automatiquement
- [ ] 103. **Champs de confluence dynamiques** : colonnes générées, saisie, persistance
- [ ] 104. Fusion des confluences lors d'une mise à jour (jamais d'écrasement)
- [ ] 105. Captures HTF/LTF attachées à un trade
- [ ] 106. Visualisation détaillée d'un trade (`newvisu`)
- [ ] 107. Zoom sur une capture
- [ ] 108. Filtres / tri du journal
- [ ] 109. Stats recalculées après chaque modification
- [ ] 110. Écriture atomique : couper l'app pendant une écriture → aucune perte
- [ ] 111. `.bak` créé avant chaque modification destructive
- [ ] 112. Verrou d'écriture : deux écritures concurrentes ne se perdent pas
- [ ] 113. Refus d'écrire si le fichier est illisible (aucun écrasement)
- [ ] 114. Stratégie avec beaucoup de trades (1000+) : DataGrid fluide

### 14.8 Statistiques (18)

- [ ] 115. Winrate (TP **et** TR comptés comme gagnants — `ResultatHelper`)
- [ ] 116. Expectancy
- [ ] 117. Total R / R moyen
- [ ] 118. Drawdown
- [ ] 119. `BE` et `PARTIAL` comptés à 0 R (pas −1)
- [ ] 120. Performance par jour (lundi-vendredi)
- [ ] 121. Performance par paire
- [ ] 122. Performance par session (Tokyo / Londres / New York)
- [ ] 123. Performance par type d'ordre (BUY / SELL)
- [ ] 124. Performance par champ dynamique (RSI, ZONE…)
- [ ] 125. Best / Worst configs (top 5)
- [ ] 126. Courbe d'équité
- [ ] 127. Courbe de R
- [ ] 128. Projection monétaire
- [ ] 129. Audit
- [ ] 130. Vignette `ControlStat` (winrate, R, nb trades)
- [ ] 131. **Parité stricte** : les mêmes données produisent **exactement** les mêmes chiffres qu'en WPF
- [ ] 132. Performance : stats d'une stratégie de 1000 trades calculées < 500 ms

### 14.9 Études & Notes (14)

- [ ] 133. Création d'une étude
- [ ] 134. Édition : gras, italique, souligné
- [ ] 135. Couleurs de texte + surlignage
- [ ] 136. Tailles de police
- [ ] 137. Alignements
- [ ] 138. Listes à puces et numérotées
- [ ] 139. Tableaux
- [ ] 140. Insertion d'images
- [ ] 141. Liens hypertexte
- [ ] 142. Annuler / rétablir
- [ ] 143. Arborescence de dossiers, création / renommage / suppression
- [ ] 144. **Conversion `.etude` v1 → v2** : contenu identique (test d'égalité de texte extrait sur **100 %** des fichiers)
- [ ] 145. Notes hebdomadaires du dashboard (`Notes_yyyyMMdd.etude`)
- [ ] 146. Rollback d'une étude convertie depuis l'onglet CLOUD

### 14.10 MT5 (24)

- [ ] 147. Détection automatique des terminaux MT5 (Windows)
- [ ] 148. Détection automatique du préfixe Wine (macOS)
- [ ] 149. Réglage manuel du dossier MT5 (repli)
- [ ] 150. Installation 1 clic de la veille (copie `.mq5` + `.tpl`)
- [ ] 151. Compilation par `metaeditor64.exe /compile /log`
- [ ] 152. Lecture du journal UTF-16 en cas d'échec de compilation
- [ ] 153. Bouton « ▶ Lancer MT5 »
- [ ] 154. Veille automatique : détection d'un nouvel export pendant que DataEdge est ouvert
- [ ] 155. Import silencieux (sans fenêtre)
- [ ] 156. Battement de cœur de l'EA lu et affiché
- [ ] 157. **Positions ouvertes en direct** (`positions_<login>.live`)
- [ ] 158. **Solde / équité / marge libre** affichés (`CompteTexte()`)
- [ ] 159. Fraîcheur de la photo `.live` (tolérance 15 s)
- [ ] 160. **Positions fantômes** écartées (photo fraîche fait autorité)
- [ ] 161. Prévisualisation de l'import (tout cocher / décocher)
- [ ] 162. « Tout vers… » (stratégie pour toutes les lignes)
- [ ] 163. Stratégie par défaut + stratégie par trade
- [ ] 164. **Filtre d'affichage** : doublons masqués par défaut
- [ ] 165. Case « Afficher les trades déjà importés »
- [ ] 166. Compteur de lignes masquées dans le résumé
- [ ] 167. **Anti-doublon** : 2 imports simultanés écrivent 12 trades, jamais 24
- [ ] 168. **Archivage ciblé** : un export partiellement importé **reste en attente**
- [ ] 169. Message de conservation expliqué à l'utilisateur
- [ ] 170. Import d'un rapport `.html` (UTF-8 **et** UTF-16) par glisser-déposer

### 14.11 MT5 — mapping des résultats (10)

- [ ] 171. `TP` : raison `DEAL_REASON_TP` (même si net négatif après swap)
- [ ] 172. `TR` : profit > 0 sans raison TP
- [ ] 173. `SL` : profit net < 0
- [ ] 174. `BE` : |profit net| ≤ tolérance
- [ ] 175. `PARTIAL` : plusieurs deals de sortie
- [ ] 176. Repli configurable pour profit > 0 sans info de raison (TP ou TR)
- [ ] 177. `RR` = distance / risque initial avec le **SL au moment de la clôture**
- [ ] 178. Repli sans SL : 1 R gagnant / 0
- [ ] 179. Dépôts ignorés
- [ ] 180. Import `.xlsx` de rapport MT5

### 14.12 Agent IA (20)

- [ ] 181. Envoi d'un message simple → réponse streamée
- [ ] 182. Bandeau de statut affiche les fragments `reasoning`
- [ ] 183. Chunks `{ping}` ignorés silencieusement
- [ ] 184. Chunks `{status}` (bascules de fournisseur) gérés
- [ ] 185. **Boucle agent** : un `tool_call` est exécuté puis un nouveau tour est envoyé
- [ ] 186. Plafond de 8 tours respecté
- [ ] 187. **Tour de finalisation garanti** si le modèle n'a pas répondu en texte
- [ ] 188. `get_workspace_snapshot` renvoie le résumé attendu
- [ ] 189. `add_journal_trade` ajoute un trade au journal
- [ ] 190. `add_backtest_trade` ajoute un trade au backtest
- [ ] 191. `update_journal_trade` modifie **UN SEUL** trade par id
- [ ] 192. `delete_journal_trade` supprime **UN SEUL** trade par id
- [ ] 193. Confirmation demandée pour les mutations destructives (3 choix)
- [ ] 194. « Tout autoriser » fonctionne pour la session
- [ ] 195. `create_strategy` / `delete_strategy`
- [ ] 196. Tools études : catalogue, lecture, recherche, création, écriture, suppression
- [ ] 197. Tools weeks : notes hebdo, calendrier mensuel
- [ ] 198. Tools marché : heure serveur, heure locale, calendrier, FedWatch, sentiment, cotations, news
- [ ] 199. Tools fichiers : lecture **PDF / XLSX / DOCX**, recherche, création txt/json/md/**pdf**/**docx**
- [ ] 200. `append_document` ajoute une section (`.pdf`, `.docx`, `.md`, `.txt`)

### 14.13 Space (16)

- [ ] 201. Navigation : rail, Accueil, DMs, Communautés, Recherche, ⚙️, ＋ groupe
- [ ] 202. Badges non-lus (rail + bouton 👥)
- [ ] 203. Pile `‹ Retour` (max 20) + `⌂ Accueil` + **Échap = Accueil**
- [ ] 204. Feed global (hub) avec bannières gradient
- [ ] 205. Conversation : thread, composer WhatsApp-like, chips d'attachments
- [ ] 206. Attachments : stats / stratégie complète / week / trade / étude / fichier
- [ ] 207. Communauté : onglets Feed / Membres / Missions 🔒 / Signaux 🔒
- [ ] 208. Profil : avatar initiales, bio, snapshots, Message / Bloquer / Signaler
- [ ] 209. Recherche anti-rebond 350 ms
- [ ] 210. Création de post (kind + contenu + multi-attachments + communauté)
- [ ] 211. Création de groupe (membres pré-remplis + recherche)
- [ ] 212. Paramètres Space (visibilité, consentement RGPD, blocages)
- [ ] 213. CGU affichées et acceptées
- [ ] 214. **Backtest Pack** : génération avec hash SHA-256, vérification à la réception
- [ ] 215. **PackViewer → « Rejouer sur le chart »** : stratégie + trades injectés dans le backtesteur
- [ ] 216. Polling 20 s uniquement quand la vue Space est ouverte

### 14.14 Cloud & synchronisation (18)

- [ ] 217. Sync complète : manifest → scan local md5 → upload par lots de 8 → download
- [ ] 218. Détection « restauration initiale » (nouveau PC, aucun fichier local)
- [ ] 219. Résolution de conflits via `synccache.json` (3 cas)
- [ ] 220. Fichiers absents localement **jamais écrasés** (téléchargés)
- [ ] 221. Onglet CLOUD : 4 cartes de stats (stockage, éléments, versions, dernière synchro)
- [ ] 222. Libellés métier (`CloudDisplayHelper`) : STRATÉGIE / ÉTUDE / NOTE / CAPTURE / SYSTÈME
- [ ] 223. Case « Afficher les fichiers techniques »
- [ ] 224. Recherche / filtre dans la liste des fichiers
- [ ] 225. Tri par catégorie puis date
- [ ] 226. Onglet VERSIONS : `.bak` filtrés par sélection
- [ ] 227. Restaurer en local
- [ ] 228. Restaurer une version sur le serveur
- [ ] 229. Supprimer du cloud (fichier + ses `.bak`)
- [ ] 230. **Suppression locale → `.bak` horodaté** (jamais de suppression sèche)
- [ ] 231. **File d'attente des suppressions** traitée à la prochaine sync réussie
- [ ] 232. Annulation de la sync (✕) → rapport « annulée », cache enregistré
- [ ] 233. Barres de progression (Settings + Dashboard) avec phase traduite + %
- [ ] 234. Toggle `sync_cacheimage` et cases de dossiers exclus

### 14.15 Paramètres, thème, dashboard (20)

- [ ] 235. Onglet APPLICATION : thème, agent IA, synchro cloud, dossier de données
- [ ] 236. Réglage `ShowAiFabOutsideDashboard` (FAB masqué hors dashboard, jamais dans Space)
- [ ] 237. Onglet COMPTE GLOBAL : profil + avatar
- [ ] 238. Onglet CLOUD (voir 14.14)
- [ ] 239. Onglet JOURNAL MT5 : fichiers distants, sauvegardes, restauration/suppression, réglages
- [ ] 240. Scroll du panneau de paramètres (aucune coupure du bas)
- [ ] 241. Dashboard : journal des trades
- [ ] 242. Dashboard : vignettes de stratégies (`ControlStat`)
- [ ] 243. Dashboard : calendrier économique Investing (webview)
- [ ] 244. Dashboard : widget **Fear & Greed** US + Crypto
- [ ] 245. Dashboard : notes hebdomadaires
- [ ] 246. Navigation : Accueil 📋 / Études 📂 / Investing 📊 / Espace 👥 / Configuration ⚙️
- [ ] 247. Barre de navigation latérale avec dépliage au survol
- [ ] 248. Notifications / toasts (styles par type, auto-dismiss)
- [ ] 249. MessageBox maison (3 boutons, 4 types)
- [ ] 250. InputDialog
- [ ] 251. Thème : toutes les couleurs et styles reproduits (`App.xaml` → CSS)
- [ ] 252. FAB de l'agent IA : ouverture/fermeture du panneau (430 → 1000 px)
- [ ] 253. Rendu : plus de `SoftwareOnly` → vérifier l'absence de crash GPU sur les 2 OS
- [ ] 254. Localisation FR intégrale (aucun texte anglais résiduel)

### 14.16 Robustesse transverse (12)

- [ ] 255. App fermée brutalement (kill) pendant une écriture → aucune corruption
- [ ] 256. Fichier `data/*.json` corrompu à la main → l'app démarre et signale, sans écraser
- [ ] 257. Disque plein → erreur explicite, aucune perte
- [ ] 258. Dossier `userData` en lecture seule → erreur explicite au démarrage
- [ ] 259. Antivirus / indexation qui verrouille un fichier → les ré-essais absorbent
- [ ] 260. Réseau coupé pendant une sync → reprise propre
- [ ] 261. Réseau coupé pendant un chat IA → erreur neutre (« serveur IA momentanément indisponible »)
- [ ] 262. Token expiré pendant un appel → refresh transparent + retry
- [ ] 263. Serveur 500 → message clair, pas de crash
- [ ] 264. `metaeditor64.exe` absent → message actionnable, pas de crash
- [ ] 265. Aucun processus orphelin après fermeture (vérifier le gestionnaire de tâches)
- [ ] 266. Consommation mémoire stable sur 8 h d'utilisation

---

## 15 — Plan par phases

### Phase 0 — Apprentissage (3-4 semaines, en parallèle du travail préparatoire)

| Livrable | Critère de sortie |
|---|---|
| JS moderne, TypeScript, Node, React, Electron | Le chart s'affiche dans une fenêtre Electron avec les vraies données |

### Phase 1 — Tranche verticale (3 semaines) 🎯 **dé-risque tout**

| Contenu | Détail |
|---|---|
| Squelette Electron + TS + electron-vite | `pnpm dev` démarre |
| `electron-builder` + **GitHub Actions matrix** | Un `.dmg` **et** un `.exe` installables |
| Import de `resources/chart/` | Le chart fonctionne dans le renderer |
| `bridge.ts` (remplacement du pont COM) | Les 8 appels JS fonctionnent |
| `csv.ts` (fetch + DecompressionStream + parse) | Les données s'affichent |
| Réglages `sandbox` / `contextIsolation` / CSP | Aucune erreur de sécurité |
| Codesign + notarization + Authenticode | Le `.dmg` s'ouvre **sans** avertissement |

**✅ Critère de sortie : `DataEdge.dmg` et `DataEdge-Setup.exe` installés, le chart affiche les vraies données, hors ligne, sur macOS et Windows.**
**➡️ À ce stade, le problème d'origine est résolu et les 3 inconnues majeures sont levées.**

### Phase 2 — Noyau données (4 semaines)

| Contenu | Détail |
|---|---|
| `paths.ts` + arborescence `userData` | Tous les dossiers créés |
| `store/atomic.ts` + `store/lock.ts` | Écriture atomique + verrous testés |
| `store/strategyStore.ts` | CRUD stratégies/trades/journal + registre |
| `store/etudesStore.ts` | CRUD études (format v2) |
| `store/replayStore.ts`, `configStore.ts`, `flags.ts` | Persistance complète |
| `stats/` (ResultatHelper, CalculateStatistics, AdvancedStats) | **Tests unitaires de parité avec le C#** |
| `cloud/` (auth, sync, conflits, suppressions, safeDelete, handshake, crash, support) | Sync complète fonctionnelle |
| `market/dataservice.ts` + `watchlist.ts` + `sentiment.ts` | Données + watchlist + Fear & Greed |
| Migration des données (§10.2) | Un utilisateur Windows retrouve tout |

**✅ Critère de sortie : toutes les données de l'utilisateur sont là et fonctionnent hors ligne ; la sync cloud fonctionne ; les stats sont identiques au WPF.**

### Phase 3 — UI cœur du produit (6 semaines)

| Contenu | Détail |
|---|---|
| Shell : `TitleBar`, `SideRail`, layout, thème | Charte reproduite |
| Splash + Onboarding + Spotlight | T1-T5 |
| Dashboard (journal, vignettes, notes hebdo, Fear & Greed, Investing) | |
| Journal + TradeForm + StrategyForm + TradeDetail + TradeImages | |
| Statistiques (StrategyStats, AdvancedStats, PerfBadge) | Lightweight Charts + ECharts |
| Settings (4 onglets) | |
| Backtester (chart + stratégie + 📦 Partager) | |
| Composants transverses (Dialog, InputDialog, Toast, DataGrid, ImageViewer) | |

**✅ Critère de sortie : le cœur du produit est utilisable — gérer ses stratégies, ses trades, voir ses stats.**

### Phase 4 — Études & Notes (3 semaines) ⚠️ **phase sensible**

| Contenu | Détail |
|---|---|
| Éditeur TipTap complet | Points 133-143 de §14.9 |
| Format `.etude` v2 + lecture/écriture | §9.2 |
| **Convertisseur XamlPackage → v2** (client **et** serveur) | §9.3 |
| Test d'égalité de texte extrait sur 100 % des fichiers | **Bloquant** |
| Arborescence de dossiers, CRUD | |
| Rollback via l'onglet CLOUD | |

**✅ Critère de sortie : 100 % des `.etude` utilisateurs convertis sans perte de contenu vérifiée automatiquement.**

### Phase 5 — MT5 (3 semaines)

| Contenu | Détail |
|---|---|
| 14 modules Node | §6.1 |
| Chemins par OS + auto-détection + réglage manuel | §6.2 |
| Fenêtre d'import complète | §14.10 |
| Mapping TP/TR/SL/BE/PARTIAL + tests unitaires | §14.11 |
| Veille automatique + import silencieux | |
| Positions live + état du compte | |
| Installateur de veille 1 clic | |

**✅ Critère de sortie : la journalisation automatique MT5 fonctionne sur Windows **et** macOS.**

### Phase 6 — Agent IA (3 semaines)

| Contenu | Détail |
|---|---|
| `agentClient.ts` (SSE + boucle 8 tours + finalisation) | |
| Les 34 tools | §7 |
| Confirmations sélectives | |
| Chat panel (bulles, streaming, reasoning, statuts, 🗑) | |
| Tools documentaires (PDF/Word/Excel en lecture et écriture) | |

**✅ Critère de sortie : le copilote est pleinement fonctionnel (§14.12).**

### Phase 7 — Space (4 semaines)

| Contenu | Détail |
|---|---|
| 12 modules + 12 vues | §8 |
| Design system Space | §8.5 |
| Backtest Pack (génération + vérification + visionneuse + **Rejouer**) | §8.6 |

**✅ Critère de sortie : Space est fonctionnel en coquille, prêt pour le swap backend.**

### Phase 8 — Finitions & parité (3 semaines)

Support, MAJ, crash reporting, notifications push, verrouillage distant, i18n FR, polissage UI, performances.

### Phase 9 — Distribution & recette (3 semaines)

Codesign, notarization, Authenticode, auto-update, **exécution complète des 266 points de §14 sur les 2 OS**, corrections.

**✅ Critère de sortie : version 1.0 publiée, checklist §14 à 100 %.**

### Phase 10 — Transition (4-8 semaines)

Double distribution WPF + Electron, communication utilisateurs, migration guidée, support, retrait progressif de l'ancien client.

### Récapitulatif

| Phase | Durée | Cumul |
|---|---|---|
| 0 — Apprentissage | 3-4 sem. | 3-4 sem. |
| 1 — Tranche verticale 🎯 | 3 sem. | 6-7 sem. |
| 2 — Noyau données | 4 sem. | 10-11 sem. |
| 3 — UI cœur | 6 sem. | 16-17 sem. |
| 4 — Études (sensible) | 3 sem. | 19-20 sem. |
| 5 — MT5 | 3 sem. | 22-23 sem. |
| 6 — Agent IA | 3 sem. | 25-26 sem. |
| 7 — Space | 4 sem. | 29-30 sem. |
| 8 — Finitions | 3 sem. | 32-33 sem. |
| 9 — Distribution & recette | 3 sem. | 35-36 sem. |
| 10 — Transition | 4-8 sem. | 39-44 sem. |

**Total : ~9-11 mois à temps plein, 14-18 mois à temps partiel.**

---

## 16 — Plan d'apprentissage

### 16.1 La bonne nouvelle : TypeScript est **beaucoup plus proche de C# que de JavaScript**

| C# | TypeScript | Verdict |
|---|---|---|
| `async Task<T>` / `await` | `async Promise<T>` / `await` | **Identique** |
| `interface` | `interface` | **Identique** |
| Génériques `<T>` | Génériques `<T>` | **Identique** |
| `string?` + nullable | `strictNullChecks` | **Identique** |
| `class`, `record` | `class`, `type`, `interface` | Quasi identique |
| `try / catch / finally` | idem | Identique |
| `enum` | `enum` (ou union de littéraux) | Quasi identique |
| LINQ (`.Where().Select()`) | `array.filter().map()` | Syntaxe différente, logique identique |
| `Dictionary<K,V>` | `Map<K,V>` / `Record<K,V>` | Proche |
| `CancellationToken` | `AbortSignal` | Proche |
| `HttpClient` | `fetch` | Proche |
| `INotifyPropertyChanged` | Zustand / `useState` | Concept différent |
| XAML + binding | JSX (React) | Concept différent |
| `IDisposable` / `using` | `try/finally` / cleanup | Différent |
| `Task.WhenAll` | `Promise.all` | Identique |
| Attributs `[JsonProperty]` | `zod` / décoration de type | Différent |

**➡️ Ton expérience C# est un atout majeur. Le système de types de TypeScript est même plus expressif que celui de C# (union types, narrowing).**

### 16.2 Étape 0 — Socle JavaScript moderne (1 semaine)

**Objectif :** lire et écrire du JS sans hésiter.

| Notion | Détail |
|---|---|
| Déclarations | `let` / `const` (plus de `var`), portée |
| Chaînes | Template literals `` `Bonjour ${nom}` `` |
| Destructuring | `const { a, b } = obj`, `const [x, y] = arr` |
| Spread / rest | `{...obj}`, `[...arr]`, `(...args)` |
| Modules | `import` / `export` (ESM) |
| Tableaux (remplace LINQ) | `map`, `filter`, `reduce`, `find`, `findIndex`, `some`, `every`, `flat`, `sort` |
| Objets | `Object.keys/values/entries`, `structuredClone` |
| Asynchrone | `Promise`, `async/await`, `Promise.all`, `allSettled`, `race` |
| Optionnels | `?.`, `??`, `??=` |
| Collections | `Map`, `Set`, `WeakMap` |
| Fonctions | closures, `this`, fonctions fléchées, valeur par défaut, arguments nommés (objet) |
| Réseau | `fetch`, `Response`, `AbortController`, `Request` |
| Erreurs | `throw`, `Error`, classes d'erreur personnalisées |

📖 **Ressources :**
- **[javascript.info](https://javascript.info)** — la référence (existe en français), gratuit. Chapitres 1-11 + la section « Asynchrone, promesses, async/await ».
- **[MDN Web Docs](https://developer.mozilla.org/fr/docs/Web/JavaScript)** — pour la référence au quotidien.
- **« You Don't Know JS Yet »** (Kyle Simpson, gratuit sur GitHub) — pour approfondir les mécanismes.

⏱️ **Exercice pratique :** écrire un script Node qui lit un `data/*.json` réel de DataEdge, calcule le winrate / l'expectancy / le total R, et affiche le résultat. **Ça t'entraîne sur du vrai code.**

### 16.3 Étape 1 — TypeScript (1 semaine) ⭐ **la plus rentable pour toi**

| Notion | Détail |
|---|---|
| Types de base | `string`, `number`, `boolean`, `null`, `undefined`, `unknown`, `never`, `any` (à éviter) |
| `interface` vs `type` | Quand utiliser lequel |
| **Unions discriminées** | `type Result = { ok: true; data: T } \| { ok: false; error: string }` |
| Génériques | `<T>`, contraintes `<T extends X>` |
| Utility types | `Partial`, `Pick`, `Omit`, `Record`, `Readonly`, `ReturnType`, `Awaited` |
| `keyof` / `typeof` | Accès aux clés et types |
| Narrowing | `typeof`, `in`, `instanceof`, prédicats de type |
| `strict: true` | Non-négociable (ADR-002) |
| `tsconfig.json` | Chemins (`paths`), cibles, `moduleResolution` |
| Typage React | `FC`, props, `useState<T>`, `useRef<T>` |
| Typage IPC | Le contrat `window.dataedge` entièrement typé |
| Validation runtime | **`zod`** — le pont entre les types statiques et les données réelles |

📖 **Ressources :**
- **[TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)** (officiel, gratuit)
- **[Total TypeScript](https://www.totaltypescript.com)** (Matt Pocock) — la meilleure ressource pratique, une grande partie est gratuite
- **Type Challenges** (GitHub) — exercices progressifs

⏱️ **Exercice pratique :** typer **entièrement** le script de l'étape 0 : `Trade`, `Strategie`, `Statistics`, `ResultatHelper`. **C'est directement du travail utile pour le projet.**

### 16.4 Étape 2 — Node.js (1 semaine)

| Notion | Détail |
|---|---|
| Fichiers | `node:fs/promises` (`readFile`, `writeFile`, `rename`, `mkdir`, `readdir`, `stat`), `node:fs` (`watch`) |
| Chemins | `node:path` (`join`, `resolve`, `dirname`, `basename`, `extname`) |
| Processus | `node:child_process` (`spawn`, `execFile`) — pour MetaEditor / MT5 |
| Flux | `node:stream`, `pipeline` |
| Événements | `node:events` (`EventEmitter`) |
| Crypto | `node:crypto` (`createHash` → md5 / SHA-256), `randomUUID` |
| Réseau | `fetch` natif, `node:http` |
| Concurrence | `worker_threads` (parsing CSV lourd) |
| Paquets | `npm` / `pnpm`, `package.json`, `scripts`, `dependencies` vs `devDependencies` |

📖 **Ressources :** [nodejs.org/learn](https://nodejs.org/en/learn) + la doc `node:fs` + [Node.js Best Practices](https://github.com/goldbergyoni/nodebestpractices).

⏱️ **Exercice pratique :** écrire la lecture/écriture atomique (`atomic.ts`) + le calcul md5 d'un dossier (remplace `ScanLocalFiles` de `FxCloudService`).

### 16.5 Étape 3 — HTML / CSS moderne (3-4 jours)

| Notion | Détail |
|---|---|
| Flexbox / Grid | Mise en page |
| Variables CSS | `--color-bg: #131722` (remplace les ressources XAML) |
| Sélecteurs, spécificité | |
| Pseudo-classes / pseudo-éléments | `:hover`, `:focus-visible`, `::before` |
| Transitions / animations | `transition`, `@keyframes`, `transform` |
| `-webkit-app-region` | Barre de titre draggable (remplace `MaximizeHelper`) |
| Responsive / `clamp()` | Tailles fluides |

📖 **Ressources :** [CSS-Tricks Flexbox](https://css-tricks.com/snippets/css/a-guide-to-flexbox/) + [web.dev/learn/css](https://web.dev/learn/css).
💡 **Tu as déjà un excellent exemple dans ton propre repo :** `resources/chart/style.css`, `toolbar.css`, `replay.css`.

### 16.6 Étape 4 — React + TypeScript (2 semaines)

| Notion | Détail |
|---|---|
| Composants | fonction, props typées, `children` |
| Hooks | `useState`, `useEffect`, `useRef`, `useMemo`, `useCallback`, `useContext` |
| Listes & clés | `key`, réconciliation |
| Formulaires | composants contrôlés, validation |
| Performance | mémoïsation, virtualisation |
| **Zustand** | Store global typé (remplace `INotifyPropertyChanged`) |
| **TanStack Query** | Cache serveur, `useQuery` / `useMutation`, invalidation, retry |
| **TanStack Table** ou **AG Grid** | Les DataGrid denses du Journal |
| Styling | Tailwind CSS (rapide) ou CSS Modules |

📖 **Ressources :** [react.dev/learn](https://react.dev/learn) (tutoriel officiel, excellent) + [TanStack](https://tanstack.com) + Tailwind CSS docs.

⏱️ **Exercice pratique :** le **Journal** (DataGrid + formulaire d'ajout de trade) branché sur le store local. **C'est un vrai morceau du produit.**

### 16.7 Étape 5 — Electron (1,5 semaine) ⭐ **le cœur**

| Concept Electron | À maîtriser | Remplace en C# / WPF |
|---|---|---|
| **Main process** | `app`, `BrowserWindow`, cycle de vie, `app.whenReady()` | `App.xaml.cs` |
| **Renderer process** | Ta page web (Chromium) | WPF |
| **Preload + `contextBridge`** | `exposeInMainWorld` — exposer une API typée | `ChartBridge` (COM) |
| **IPC** | `ipcMain.handle` / `ipcRenderer.invoke` (requête-réponse), `send` / `on` (événements) | Appels directs |
| **`app.getPath()`** | Dossiers par OS (`userData`, `documents`, `logs`) | `AppDomain.BaseDirectory` |
| **`safeStorage`** | Chiffrement des tokens | **DPAPI** |
| **`dialog`** | `showOpenDialog`, `showSaveDialog`, `showMessageBox` | `OpenFileDialog`, `MessageBox` |
| **`shell`** | `openPath`, `openExternal`, `showItemInFolder` | `Process.Start`, `explorer.exe /select` |
| **`Menu`, `Tray`, `globalShortcut`** | Menus natifs, icône de barre système | Menus WPF |
| **`protocol`** | Servir `app://` depuis un dossier | `SetVirtualHostNameToFolderMapping` |
| **`webContents`** | `executeJavaScript`, `send`, `did-finish-load` | `ExecuteScriptAsync` |
| **`session`** | Cookies, permissions, `onHeadersReceived` (CSP) | — |
| **`webview` / `BrowserView`** | Le calendrier Investing | WebView2 |
| **`autoUpdater`** | Mises à jour | `UpdateService` |
| **Sécurité** | `contextIsolation`, `sandbox`, CSP, validation IPC | — |
| **`electron-vite`** | Dev (HMR), build (main/preload/renderer) | MSBuild |
| **`electron-builder`** | `.dmg`, `.exe`, signature, notarization | Installeur MSI |

📖 **Ressources :**
- **[electronjs.org/docs](https://www.electronjs.org/docs/latest/)** — lire la page **Security** en entier, c'est indispensable
- **[electron-vite.org](https://electron-vite.org)** — le boilerplate recommandé
- **[Electron Fiddle](https://www.electronjs.org/fiddle)** — bac à sable
- **[electron-builder docs](https://www.electron.build)**

⏱️ **Exercice pratique :** la **Phase 1** complète (le chart dans Electron + les deux installeurs).

### 16.8 Étape 6 — Build & distribution (1 semaine)

electron-vite, electron-builder (targets `dmg` / `nsis` / `msi`), codesign + notarization macOS, Authenticode Windows, GitHub Actions matrix, `electron-updater`.

📖 **Ressources :** [electron-builder docs](https://www.electron.build) + [Electron Code Signing](https://www.electronjs.org/docs/latest/tutorial/code-signing) + [Electron Notarization](https://www.electronjs.org/docs/latest/tutorial/notarization).

### 16.9 Étape 7 — Tests (3-4 jours)

| Outil | Usage |
|---|---|
| **Vitest** | Tests unitaires (stats, MT5 mapper, pack verifier, atomic) |
| **Playwright** | Tests E2E Electron (chart, import MT5, replay, sync) |
| **ESLint + Prettier** | Qualité et formatage |

⏱️ **Exercice pratique :** porter les tests de `tools/mt5test/` et `tools/spacetest/` en Vitest.

### 16.10 Étape 8 — Éditeur riche & documents (1 semaine)

| Outil | Remplace |
|---|---|
| **TipTap** (+ extensions) | `FlowDocument` |
| **JSZip** | `XamlPackage` (conteneur) |
| **pdfkit** | QuestPDF |
| **docx** + **mammoth** | OpenXML |
| **exceljs** | ExcelDataReader |
| **pdf-parse** | PdfPig |

### 16.11 Calendrier d'apprentissage recommandé

| Semaine | Apprentissage | Livrable concret |
|---|---|---|
| 1 | JS moderne | Script Node qui lit un `data/*.json` et calcule les stats |
| 2 | TypeScript | Le même script **entièrement typé** (`Trade`, `Strategie`, `Statistics`) |
| 3 | Node + Electron (base) | **Fenêtre Electron qui affiche `index.html`** |
| 4 | Electron (IPC, preload, sécurité) | **Le chart s'affiche avec les vraies données** 🎯 |
| 5-6 | React + TS + Zustand + TanStack | Le **Journal** (DataGrid + formulaire trade) |
| 7 | electron-builder + GitHub Actions | **Un `.dmg` ET un `.exe` installés sur les deux OS** 🎯 |
| 8 | TipTap + pdfkit + docx | L'éditeur Études + l'export PDF |

**➡️ Semaine 4 : le problème d'origine est résolu. Semaine 7 : les deux installeurs existent. Le reste, c'est du volume, pas de l'inconnu.**

### 16.12 Estimation de l'effort d'apprentissage

| Profil | Temps pour être productif | Temps pour être à l'aise |
|---|---|---|
| Développeur C# expérimenté (ton cas) | **3-4 semaines** | 2-3 mois |
| Sans expérience C# | 8-10 semaines | 5-6 mois |

**Ton profil C# + Symfony est un vrai avantage** : tu comprends déjà les types, l'asynchrone, le HTTP/JWT, les architectures en couches, le JSON. Le reste est de la syntaxe et de l'outillage.

---

## 17 — IDE & outillage

### 17.1 Recommandation principale : **VS Code** (gratuit)

C'est **l'IDE de référence pour Electron / JS / TS** — c'est littéralement celui dans lequel Electron est développé. **C'est déjà ton IDE actuel.**

**Extensions indispensables :**

| Extension | Rôle |
|---|---|
| **ESLint** (+ `@typescript-eslint`) | Lint |
| **Prettier** | Formatage automatique |
| **Error Lens** | Erreurs inline (l'équivalent de ReSharper en C#) |
| **Pretty TypeScript Errors** | Messages TS lisibles |
| **GitLens** | Historique git |
| **Tailwind CSS IntelliSense** | Autocomplétion CSS |
| **Vitest** | Lancer les tests depuis l'IDE |
| **Playwright Test** | Tests E2E |
| **REST Client** / **Thunder Client** | Tester l'API Symfony depuis l'IDE |
| **Path Intellisense** | Chemins de fichiers |
| **Todo Tree** | Suivi des TODO du plan |
| **SQLite Viewer** | Si tu ajoutes du SQLite plus tard |

**Débogage :** VS Code debugge **nativement** le main process (Node) et le renderer (Chromium) avec points d'arrêt, watch, pile d'appels — **aussi confortable qu'en C#**. Tu ne perds rien de ton confort actuel.

### 17.2 Alternatives

| IDE | Prix | Pour qui | Verdict |
|---|---|---|---|
| **VS Code** | Gratuit | Tout le monde | ✅ **Recommandé** — ton IDE actuel |
| **Cursor** / **Windsurf** | ~20 $/mois | Ceux qui veulent l'IA intégrée | ✅ Excellent si tu veux l'IA dans l'IDE (tu utilises déjà Cline) |
| **WebStorm** | ~7 €/mois | Ceux qui viennent de JetBrains | ✅ Le meilleur **refactoring** TS et la meilleure complétion |
| **Zed** | Gratuit | Ceux qui veulent de la vitesse | ⚠️ Jeune, écosystème plus limité |
| **Visual Studio** | Gratuit | C# | ❌ **Inadapté** à JS/TS/Electron |

**Conseil :** **VS Code** (tu l'as déjà) + **WebStorm en essai gratuit** si le refactoring TS te manque. Et garde **Visual Studio ouvert** pendant la transition — tu vas devoir lire le C# existant en parallèle.

### 17.3 Outils à installer

| Outil | Rôle | Installation |
|---|---|---|
| **Node.js LTS** (≥ 22) | Runtime | via **fnm** ou **nvm** (plusieurs versions possibles) |
| **pnpm** | Gestionnaire de paquets plus rapide que npm | `npm i -g pnpm` |
| **Git** | Déjà en place | — |
| **GitHub Actions** | **Obligatoire** — produit le `.dmg` (runner macOS) et le `.exe` (runner Windows) | — |
| **Un Mac** (Apple Silicon) | Tests + signature locale | — |
| **Un PC Windows** | Tests + signature locale | — |
| **Compte Apple Developer** | **99 $/an** — notarization | [developer.apple.com](https://developer.apple.com) |
| **Certificat Authenticode Windows** | ~200-400 €/an — éviter SmartScreen | Fournisseur de certificats |
| **Figma** (optionnel) | Reprendre les maquettes de l'UI | — |

### 17.4 Configuration VS Code recommandée

```json
// .vscode/settings.json
{
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": { "source.fixAll.eslint": "explicit" },
  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.preferences.importModuleSpecifier": "non-relative",
  "files.exclude": { "**/out": true, "**/dist": true, "**/node_modules": true },
  "search.exclude": { "**/out": true, "**/dist": true, "**/node_modules": true },
  "editor.rulers": [100]
}
```

### 17.5 Extensions C# à garder pendant la transition

**Ne désinstalle pas** l'extension **C# Dev Kit** / **OmniSharp** : tu vas devoir lire le code WPF existant pour porter la logique métier, et la navigation de symboles (aller à la définition) sera précieuse.

---

## 18 — Risques & mitigations

| # | Risque | Prob. | Impact | Mitigation |
|---|---|---|---|---|
| **R1** | **Perte de données utilisateurs lors de la conversion `.etude`** | Moyenne | 🔴 **Critique** | Ne jamais modifier l'original · `.bak` · test d'égalité de texte extrait sur **100 %** des fichiers · rollback via CLOUD · conversion côté serveur |
| **R2** | **Régression du chart** (l'actif le plus précieux) | Moyenne | 🔴 **Critique** | Le chart reste en JS pur · `bridge.ts` isole le changement · **checklist §14.4 à §14.6 exécutée intégralement** avant chaque release |
| **R3** | **MT5 macOS** : chemin Wine introuvable ou instable | Élevée | 🟠 Élevé | Auto-détection multi-chemins + **réglage manuel obligatoire** + bouton « Détecter automatiquement » + documentation dédiée |
| **R4** | **Notarization Apple refusée / blocante** | Moyenne | 🟠 Élevé | Faire un build signé+notarisé **dès la Phase 1**, pas à la fin · entitlements testés tôt |
| **R5** | **Écart de chiffres entre stats WPF et Electron** | Moyenne | 🟠 Élevé | `resultatHelper.ts` porté **1:1** · **tests unitaires de parité** sur des jeux de données réels (point 131) |
| **R6** | **Dépassement du calendrier** (sous-estimation de l'UI) | Élevée | 🟡 Moyen | Tranche verticale d'abord · livrer par phases utilisables · ne pas tout construire avant de tester |
| **R7** | **Régression de la robustesse d'écriture** (atomic, verrous) | Moyenne | 🟠 Élevé | `atomic.ts` + `lock.ts` portés **en premier** (Phase 2) avec tests dédiés · points 110-113 |
| **R8** | **Écosystème npm** : dépendance abandonnée / vulnérable | Moyenne | 🟡 Moyen | `pnpm audit` en CI · privilégier les libs maintenues · limiter le nombre de dépendances · `electron-builder` + `electron` à jour |
| **R9** | **Performances du chart** sur gros fichiers (1m) | Moyenne | 🟡 Moyen | Parsing en **Web Worker** · transfert binaire (`ArrayBuffer`) · point 57 (pas de gel > 1 s) |
| **R10** | **Migration d'un `session.bin` DPAPI impossible** | Certaine | 🟢 Faible | Ré-authentification explicite et bien expliquée (ADR-009, point 21) |
| **R11** | **Perte de la contrainte `SoftwareOnly`** → crashs GPU | Faible | 🟡 Moyen | Tester sur GPU variés · `app.disableHardwareAcceleration()` en repli si nécessaire · point 253 |
| **R12** | **Perte de fonctionnalité « Rejouer » depuis Space** | Moyenne | 🟠 Élevé | Chemin `Space → store → backtester → chart` testé **en priorité** (point 215) |
| **R13** | **Taille de l'installeur** (250 Mo) → friction au téléchargement | Certaine | 🟢 Faible | Acceptable · `electron-builder` supporte la mise à jour différentielle |
| **R14** | **Apprentissage insuffisant** → code de mauvaise qualité | Moyenne | 🟡 Moyen | Phase 0 dédiée · `strict: true` · ESLint en CI · revue de code |
| **R15** | **Double maintenance WPF + Electron** pendant la transition | Certaine | 🟡 Moyen | Limiter la transition dans le temps · ne plus ajouter de fonctionnalité au WPF · corriger seulement les bugs critiques |

### 18.1 Les 3 risques à surveiller en priorité

1. **R1 — `.etude`** : c'est le seul risque qui peut faire perdre la confiance des utilisateurs. **À traiter tôt, avec des tests automatiques.**
2. **R2 — Chart** : c'est ton actif. **Ne jamais modifier le moteur sans la checklist.**
3. **R3 — MT5 macOS** : c'est l'inconnue technique la plus probable. **À tester dès la Phase 1 sur un vrai Mac.**

---

## 19 — Décisions ouvertes

| # | Décision | Recommandation appliquée | Alternatives | Impact si changé |
|---|---|---|---|---|
| **D1** | **Framework UI** | **React 19** ⚠️ à confirmer | Svelte · Vue 3 | Faible sur l'architecture, élevé sur l'apprentissage |
| **D2** | **Éditeur riche** | **TipTap** ⚠️ à confirmer | Lexical · ProseMirror · Quill | Moyen (impacte le format `.etude`) |
| **D3** | **DataGrid** | **TanStack Table** ⚠️ à confirmer | AG Grid · Handsontable | Moyen (Journal, Stats, Import MT5, PackViewer) |
| **D4** | **Graphes statistiques** | **Lightweight Charts + ECharts** ⚠️ à confirmer | Chart.js · Recharts · Plotly | Faible |
| **D5** | **Styling** | **Tailwind CSS** ⚠️ à confirmer | CSS Modules · styled-components | Faible |
| **D6** | **Tests** | **Vitest + Playwright** ⚠️ à confirmer | Jest · Cucumber | Faible |
| **D7** | **Sort de l'ancien client WPF** | Maintenu pendant la transition (Phase 10), puis retiré ⚠️ | Maintenu indéfiniment | Élevé sur la maintenance |
| **D8** | **Durée de la double distribution** | 4-8 semaines ⚠️ | 3 mois · 6 mois | Moyen |
| **D9** | **Où exécuter la conversion `.etude`** | **Côté serveur (batch) + côté client WPF (repli)** | Un seul des deux | Élevé (risque R1) |
| **D10** | **Migration des données au 1er lancement** | Proposée à l'utilisateur (import ou zéro) ⚠️ | Automatique silencieuse | Moyen |
| **D11** | **`/software/version` conservé ou remplacé** | Conservé pour le numéro + flux `latest.yml` pour les binaires | Tout via flux | Faible |
| **D12** | **Persistance des dessins du chart** | `localStorage` conservé ⚠️ | Migrer vers le serveur (synchro multi-appareils) | Moyen |
| **D13** | **Versions Node / Electron** | Node 22 LTS · Electron dernière stable ⚠️ | Versions figées | Faible |
| **D14** | **Gestionnaire de paquets** | **pnpm** ⚠️ | npm · yarn · bun | Faible |
| **D15** | **Ajouter Linux ?** | Hors périmètre v1 (quasi gratuit plus tard) | — | Faible |

> ⚠️ **Toutes les décisions marquées « à confirmer » appliquent la recommandation ci-dessus par défaut.**
> **Changer une décision = modifier ce tableau et la section concernée.** L'architecture du plan ne dépend d'aucune d'elles de manière bloquante.

---

## 20 — Annexes

### 20.1 Correspondances C# → npm (bibliothèques)

| Bibliothèque NuGet | Rôle | Remplacement npm |
|---|---|---|
| `Microsoft.Web.WebView2` | Navigateur intégré | **Electron (Chromium embarqué)** |
| `OxyPlot.Core` / `OxyPlot.Wpf` | Graphes | **lightweight-charts** (déjà présent) · **echarts** |
| `QuestPDF` | Génération PDF | **pdfkit** |
| `PdfPig` | Lecture PDF | **pdf-parse** |
| `ExcelDataReader` | Lecture Excel | **exceljs** |
| `DocumentFormat.OpenXml` | Word `.docx` | **docx** (écriture) · **mammoth** (lecture) |
| `Newtonsoft.Json` / `System.Text.Json` | JSON | **JSON natif** · **zod** (validation) |
| `Microsoft.Bcl.AsyncInterfaces`, `System.Memory`, `System.Buffers`, `System.Numerics.Vectors`, `System.Runtime.CompilerServices.Unsafe`, `System.ValueTuple`, `System.Threading.Tasks.Extensions`, `System.IO.Pipelines`, `Microsoft.IO.RecyclableMemoryStream`, `Google.Protobuf`, `System.Security.Cryptography.Xml`, `System.ComponentModel.Annotations`, `Microsoft.Bcl.HashCode`, `System.Text.Encodings.Web` | Support .NET Framework | 🗑️ **Inutiles** (le runtime moderne les intègre) |

**Nouvelles bibliothèques npm à ajouter :**

| Paquet | Rôle |
|---|---|
| `electron` · `electron-vite` · `electron-builder` · `electron-updater` | Coquille + build + MAJ |
| `react` · `react-dom` | UI |
| `zustand` · `@tanstack/react-query` · `@tanstack/react-table` | État + cache + tableaux |
| `zod` | Validation runtime (IPC + API) |
| `tailwindcss` | Styling |
| `@tiptap/react` + extensions | Éditeur riche |
| `jszip` | Conteneur `.etude` |
| `pdfkit` · `docx` · `mammoth` · `exceljs` · `pdf-parse` | Documents |
| `proper-lockfile` | Verrou inter-processus |
| `chokidar` (optionnel) | Surveillance de fichiers plus robuste que `fs.watch` |
| `echarts` | Graphes statistiques |
| `date-fns` | Dates (remplace `DateTime`) |
| `vitest` · `@playwright/test` | Tests |
| `eslint` · `prettier` · `typescript` | Qualité |

### 20.2 Glossaire

| Terme | Définition |
|---|---|
| **Main process** | Le processus Node d'Electron (accès disque / réseau / OS) |
| **Renderer process** | Le processus Chromium qui affiche l'UI |
| **Preload** | Script chargé avant le renderer, seul pont autorisé vers Node |
| **`contextBridge`** | API Electron pour exposer des fonctions du preload au renderer |
| **IPC** | Communication inter-processus (main ↔ renderer) |
| **`.etude`** | Format des études/notes (v1 = XamlPackage WPF, v2 = ZIP + ProseMirror) |
| **XamlPackage** | Format WPF contenant XAML + images (propriétaire, non portable) |
| **Backtest Pack** | Archive signée (SHA-256) partagée dans Space : stratégie + trades + captures |
| **Confluence** | Champ personnalisé d'une stratégie (RSI, ZONE…), génère colonnes et stats |
| **`ResultatHelper`** | Source de vérité unique pour `EstGagnant()` / `RRealise()` |
| **TR** | Take Result — sortie en profit sans atteindre le TP |
| **RR / R réalisé** | Distance parcourue / risque initial (SL au moment de la clôture) |
| **Safe-delete** | Suppression qui renomme en `.bak` horodaté au lieu d'effacer |
| **Handshake** | Appel de démarrage au serveur (verrouillage, notifications, version) |
| **Prompt caching** | Réutilisation du préfixe de prompt côté fournisseur LLM (~92 % de hit mesuré) |
| **Polling intelligent** | Rafraîchissement périodique de Space (20 s) uniquement quand la vue est ouverte |

### 20.3 Checklist de démarrage (les 10 premières actions)

1. [ ] Créer le dépôt `dataedge-electron` (ou une branche dédiée)
2. [ ] `pnpm create @quick-start/electron` → squelette Electron + React + TS
3. [ ] Configurer `electron.vite.config.ts` (3 cibles) + `tsconfig` strict
4. [ ] Copier `resources/` (chart, mt5, images, icônes)
5. [ ] Écrire `main/security/ipcGuard.ts` + la fenêtre sécurisée (§4.3)
6. [ ] Écrire `renderer/chart/bridge.ts` (§5.3) + installer `window.chartService`
7. [ ] Écrire `renderer/chart/csv.ts` (§5.4) et brancher le chargement
8. [ ] **Vérifier que le chart s'affiche avec les vraies données**
9. [ ] Configurer `electron-builder.yml` + le workflow GitHub Actions
10. [ ] **Obtenir un `.dmg` et un `.exe` installables** → Phase 1 terminée 🎯

### 20.4 Règle de mise à jour de ce document

Ce document est **la référence de la migration**. Il doit être mis à jour :

- ✅ À la fin de chaque phase (cocher les points de §14, ajuster les durées)
- ✅ À chaque décision tranchée (§19 : retirer la mention « à confirmer »)
- ✅ À chaque écart constaté par rapport au plan (créer une section « Écarts » si nécessaire)
- ✅ À chaque nouveau risque identifié (§18)
- ❌ Pas pour les détails d'implémentation (ils vivent dans le code et les README)

### 20.5 Documents liés

| Document | Contenu |
|---|---|
| `mapprojet.md` | Carte du projet WPF actuel (vue globale) |
| `mapchart.md` | Carte détaillée du module Chart |
| `mapia.md` | Carte de l'agent IA (front + back) |
| `mapspace.md` | Carte de la fonctionnalité Space |
| `fxglobal/mapbackend.md` | Carte du backend Symfony |
| **`PLAN_MIGRATION_ELECTRON.md`** | **Ce document** — plan de migration |

> ⚠️ **Après la migration**, les cartes `map*.md` devront être remplacées par leurs équivalents Electron. Note : **`mapchart.md` restera largement valable**, puisque le chart ne change pas (seul le pont est remplacé).

---

## ✅ Fin du plan — Résumé exécutif

| Élément | Valeur |
|---|---|
| **Cible** | Electron + TypeScript + React — **un seul code source** |
| **Livrables** | `DataEdge-<version>-arm64.dmg`, `DataEdge-<version>-x64.dmg`, `DataEdge-Setup-<version>-x64.exe` |
| **Réutilisation du chart** | **~95 %** (un seul fichier nouveau : `bridge.ts`) |
| **Réutilisation du MQL5** | **100 %** (l'EA ne change pas) |
| **Réutilisation du backend** | **100 %** (aucune modification Symfony) |
| **Réutilisation de la logique métier** | ~80-90 % (portage mécanique C# → TS) |
| **UI réécrite** | 33 XAML + ~400 Ko de code-behind |
| **Points de recette** | **266** (macOS **et** Windows, en ligne **et** hors ligne) |
| **Durée totale** | **9-11 mois** à temps plein · **14-18 mois** à temps partiel |
| **🎯 Premier jalon utile** | **Semaine 4** — le chart fonctionne dans Electron avec les vraies données (le bug d'origine est résolu) |
| **🎯 Deuxième jalon** | **Semaine 7** — `.dmg` + `.exe` installables et signés |
| **Risque n°1** | Conversion du format `.etude` (données utilisateurs) |
| **Risque n°2** | Régression du chart |
| **Risque n°3** | MT5 sous Wine sur macOS |
| **Prérequis bloquants** | Apple Developer (99 $/an) · un Mac · un PC Windows (ou CI) |

**➡️ Prochaine action : la checklist §20.3 (les 10 premières actions), puis la Phase 1.**
