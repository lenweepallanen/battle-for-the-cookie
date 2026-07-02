# BATTLE FOR THE COOKIE — Brief de passation final (MVP complet)

> Document de référence pour Claude Code : état complet du jeu, patterns techniques, déploiement Vercel, roadmap.
> **Fichier jouable de référence : `battle-for-the-cookie-jalon7.html`** (~2100 lignes, autonome, zéro asset, zéro build).
> À placer dans le repo. Copier les sections 4 à 7 dans un `CLAUDE.md` à la racine.

---

## 1. Le pitch

Battle royale 2D top-down croisant BR classique et jeu de déduction visuelle (« Qui est-ce ? » létal). Une foule de 120 PNJ aux attributs visuels variés. Un PNJ est « le coupable » (voleur de cookie). À chaque manche, un indice exclut un attribut ; on purge les PNJ exclus (+points), on évite les suspects (malus), on affronte les autres joueurs (down 30 s + drop d'équipement), et on finit par **capturer le coupable VIVANT au fouet**. La zone se resserre à chaque manche. Style cozy-chibi + violence cartoon assumée (contraste comique type Party Animals).

Cible : navigateur (déploiement Vercel), Steam plus tard (wrap Electron/Tauri). Multijoueur prévu en V1 ; le MVP est solo contre 3 bots.

## 2. Structure d'une partie (~2 min 30 – 3 min)

| Manche | Temps | Événement | Zone (rayon px) | Malus erreur de tir |
|---|---|---|---|---|
| Loot | 0–30 s | bannière « PHASE DE LOOT », caisses au sol | 1560 | −10 |
| 1–4 | 30 s chacune | indice + resserrement (rétraction 10 s, ×0.7) | 1092 → 764 → 535 → 374 | −10 / −15 / −20 / −25 |
| Finale | 120 s+ | capture au fouet, zone stable (374) | 374 | −25 |

Spawns : joueur + 3 bots **en anneau équidistant** à ~58-70 % du rayon (angle de base aléatoire) — personne ne démarre sur le loot central.

## 3. Barème complet

| Action | Points |
|---|---|
| Capture du coupable (ligotage complet) | **+500** + fin de partie, victoire |
| Down d'un joueur/bot | +50 (+ il drop son équipement) |
| PNJ exclu tué | +10 |
| Erreur de tir (suspect tué) | −10 → −25 selon la manche (`MALUS_TABLE`) |
| Mauvais ligotage complet | −100 (joueur ET bots) |
| Tuer le coupable par balle | −100 + **transfert du rôle** (voir §4) |
| Tuer le DERNIER suspect | −100 puis −500, fin immédiate, **victoire au score** |

Mêmes règles pour les bots (leurs balles perdues et erreurs leur coûtent). Leaderboard temps réel sous la minimap + classement final sur l'écran de fin.

## 4. Mécaniques (toutes implémentées et playtestées)

### Contrôles
ZQSD + WASD + flèches (les deux mappings actifs, couvre AZERTY/QWERTY) ; souris = visée 360° + tir maintenu ; 1-4 + molette = armes ; R = medkit (canalisation 2 s, vitesse ×0.5, interrompue par dégât, +50 PV) ; E = fouet (phase finale).

### La foule
120 PNJ, 4 axes : cheveux roux/blond/noir × long/court × chapeau/rien × haut rouge/bleu/vert = 36 combos, une texture générée par combo. 20 PV. Errance marche/pause ; **migration vers le centre** hors zone (dispersion angulaire anti-colonne) ; **panique au coup de feu** : tout tir fait fuir les PNJ dans un rayon de 170 px (300 pour une explosion) à l'opposé du tireur ~1 s — tirer disperse ses propres cibles, approcher au fouet en silence est la bonne manœuvre.

### Coupable, indices, transfert
- Coupable = un individu précis tiré au sort. 4 indices (sur 6 exclusions possibles), un par manche → ~8-15 suspects restants en finale.
- Bannière « NOUVEL INDICE ! » + liste permanente des indices à gauche.
- **Transfert de rôle** : coupable tué par balle → −100 au tueur, le rôle passe à un suspect vivant compatible avec les indices révélés, les indices restants sont recalculés (`rebuildCluePool`), bannière « LE COUPABLE EST TOMBÉ ! Un complice a récupéré le cookie ». Aucun accident ne casse la partie ; anti-grief natif pour le multi. Plus de suspects vivants → fin au score (−500 au destructeur).

### La capture au fouet (phase finale, après le 4e indice)
E arme le fouet (tir impossible). Clic = lash directionnel, portée 200 px, cooldown 600 ms ; attrape **le premier PNJ sur la trajectoire, sans filtre** (viser à travers la foule = le skill). Prise sur un exclu → avertissement immédiat « Déjà hors de cause ! ». Ligotage = clic maintenu 10 s, immobile et désarmé, corde + anneaux + barre + décompte ; tout dégât ou relâchement = perdu. Mauvais suspect complet → −100, il fuit, fouet en cooldown 1,2 s.

### Armes & loot (hotbar 5 slots, bas centre)
| Slot | Arme | Dégâts | CD | Spécial |
|---|---|---|---|---|
| 1 | Pistolet | 12 | 400 ms | munitions infinies (filet de sécurité) |
| 2 | Fusil | 12 | 150 ms | pack +30 |
| 3 | Pompe | 8×5 plombs | 900 ms | cône 0.45 rad, portée courte, pack +12 |
| 4 | Roquette | 60 zone r=90 | 1200 ms | explose au contact/expiration, interrompt les ligotages, scoring normal sur chaque victime (transfert possible !), pack +1 |
| R | Medkit | +50 PV | canal 2 s | max 3 |

26 caisses (40 % au centre = hot drop). Première prise d'une arme = équipée auto. À sec = retour pistolet auto. **Drop à la mort** : le joueur down lâche tout (armes + munitions restantes + medkits), repart au pistolet ; les bots downés lâchent une caisse aléatoire. Les bots se soignent sur les medkits croisés s'ils sont blessés.

### Down / respawn
PV joueur 100. À 0 (balles ou zone) : down 30 s, overlay « ÉLIMINÉ par X ! », drop d'équipement, respawn aléatoire dans la zone, full PV. Bots : 60 PV, mêmes règles, +50 pour qui les down. Dégâts de zone : 3 + phase × 2.5 PV/s, vignette rouge pulsante.

### IA des bots (3 : Detective72, CookieHunter, ClueMaster)
Priorités (réévaluées ~250 ms) : **1.** joueur < 380 px → combat (vraies balles physiques, spread 0.22, distance gérée) ; **2.** hors zone → migration ; **3.** phase finale → capture (approche + canal 10 s, barre à leur couleur, interrompu par tout dégât, **victoire du bot = ta défaite**) ; **4.** purge ; **5.** patrouille vers points lointains.

**La purge est résolue aux dés, pas à la physique** (leçon clé : ne pas simuler ce que l'IA doit réussir facilement) : balles cosmétiques (flag `cosmetic` réassigné à chaque tir — pool !), cadence 320 ms, 80 % de touche, impact différé du temps de vol. **Faillibilité calibrée** : 15 % d'erreur d'identification (verrouille un suspect voisin et le tue → malus, cible persistante = il va au bout de sa conviction), 18 % de balles qui touchent le voisin ; au ligotage, 25 % d'erreur de lecture (cible un exclu → −100). Ces taux = les futurs **curseurs de difficulté**.

### Effets & juice
- **Sang** : 14 gouttes par impact (20 explosion) projetées dans l'axe du tir, 55 % tachent le sol définitivement ; flaque de mort persistante — 45 % ronde + satellites, 55 % giclée dans l'axe du dernier coup (`npc.lastHitAngle`, étirée, éclaboussure secondaire) ; nuée de 7-14 gouttelettes persistantes par flaque. **Curseur perf mobile : la proba 0.55 et les comptes de `bloodDroplets`** (~3000 images statiques max en fin de partie, OK desktop).
- **Cadavres persistants** : le PNJ bascule au sol (180 ms), teint grisé `0xbbb3a8`, **attributs encore lisibles** (info de déduction : on voit ce qui a été purgé), depth −4 (au-dessus des flaques −5, sous les vivants).
- **Sons** : 15 événements, 100 % Web Audio procédural (`tone`/`noise`/`sfx`), atténuation par distance (`volAt` — les tirs lointains renseignent sur la position). Déverrouillé par le clic de l'écran titre.
- Traînées de balles, bobbing de marche (scaleY sinusoïdal), bulles « ?! » des témoins (max 3, fuient), screenshake différencié, écran titre (cookie + pitch + contrôles, skippé au « rejouer » via `registry.skipTitle`).

## 5. Technique

### Stack
Phaser 3.60 (cdnjs), **un fichier HTML autonome**, une scène (`MainScene`), Arcade Physics, `Scale.RESIZE`. Toutes textures générées par `Graphics.generateTexture()` au boot. Décor déterministe (PRNG seedé `seed*16807%2147483647`).

### ⚠ Les 3 pièges Phaser durement acquis (NE PAS réintroduire)
1. **Pool de balles + minuteurs = balles fantômes.** Jamais de `delayedCall` d'autodestruction par balle (le minuteur périmé tue la balle recyclée). Pattern : `bullet.born = time` + balayage dans `update()`. Idem pour tout flag sur objet poolé : **toujours réassigner** (`bullet.cosmetic = x` à chaque tir).
2. **Un corps physique à coordonnées NaN corrompt silencieusement TOUT l'arbre de collisions** (symptôme : tout traverse tout, zéro erreur console). Cause type : spawn utilisant une valeur non initialisée (`this.zoneR` avant l'init de la zone). `randomPointInZone` a un repli `|| ZONE_START_R`.
3. **`overlap`/`collider` sprite-vs-groupe : le callback reçoit TOUJOURS (sprite, membreDuGroupe)**, quel que soit l'ordre des arguments. Seul groupe-vs-groupe respecte l'ordre. (Symptôme historique : `killBullet(player)` → joueur au corps désactivé, « je tire mais je ne bouge plus ».)

### Autres patterns
Tri en profondeur `setDepth(obj.y)` chaque frame (foule lisible) ; HUD depth 10000+, overlays 20000+, titre 30000+ ; `npc.nextThink = Number.MAX_SAFE_INTEGER` gèle un PNJ (ligotage) ; toutes les constantes de gameplay en tête de fichier.

### Constantes de tuning principales
`WORLD=2200, PLAYER_SPEED=230, PLAYER_MAX_HP=100, NPC_COUNT=120, NPC_HP=20, CLUE_INTERVAL=30000, MAX_ROUNDS=4, MALUS_TABLE=[-10,-10,-15,-20,-25], SCORE_GOOD=10, SCORE_BOT=50, SCORE_CULPRIT=500, SCORE_WRONG_TIE=-100, SCORE_COOKIE_LOST=-500, ZONE_START_R=1560, ZONE_SHRINK_FACTOR=0.7, ZONE_SHRINK_MS=10000, WHIP_RANGE=200, WHIP_COOLDOWN=600, CAPTURE_TIME=10000, BOT_HP=60, BOT_VIEW=380, BOT_PURGE_HIT=0.8, BOT_PURGE_MISTAKE=0.18, BOT_TARGET_ERROR=0.15, BOT_TIE_ERROR=0.25, BOT_PURGE_CD=320, RESPAWN_MS=30000, WEAPONS{...}, LOOT_COUNTS{...}`

## 6. Déploiement Vercel (tâche immédiate pour Claude Code)

Le jeu est un site statique : c'est le déploiement le plus simple du monde.

**Option A — direct (10 minutes) :**
```bash
mkdir battle-for-the-cookie && cd battle-for-the-cookie
git init
cp <chemin>/battle-for-the-cookie-jalon7.html index.html   # renommage = tout ce qu'il faut
# vercel.json optionnel — aucun réglage requis pour du statique
npx vercel          # login puis déploiement ; `npx vercel --prod` pour la prod
```
Résultat : une URL publique à partager. Phaser est chargé depuis cdnjs (pas de dépendance à builder).

**Option B — propre (recommandée juste après) :** repo Vite (`npm create vite@latest . -- --template vanilla`), refactor du monolithe (voir §7.1), `vercel` détecte Vite automatiquement (build `npm run build`, output `dist/`). Connecter le repo GitHub à Vercel = déploiement continu à chaque push.

Points d'attention : garder `index.html` comme nom d'entrée ; le jeu n'a AUCUNE API/backend (rien à configurer) ; tester l'audio après déploiement (il se déverrouille au clic du titre — comportement navigateur normal).

## 7. Roadmap

### 7.1 Refactor (première tâche de fond dans Claude Code)
Éclater le monolithe **à comportement strictement identique** (le fichier de référence sert de test de non-régression manuel) :
```
src/
├── main.js  config.js
├── scenes/MainScene.js
├── systems/  clues.js zone.js capture.js scoring.js bots.js loot.js audio.js blood.js
└── entities/ npc.js player.js bullets.js
```
Séparer la **logique de simulation pure** (positions, pas de Phaser) du rendu : prérequis du multijoueur ET d'un éventuel rendu 3D alternatif.

### 7.2 Améliorations gameplay priorisées (validées ensemble)
1. **Variété d'indices** (meilleur ratio profondeur/effort) : positifs (« porte un chapeau »), de position (« moitié nord »), comportementaux, flash visuel éphémère du coupable.
2. **Personnalités de bots** via les curseurs existants : Chasseur / Détective / Bourrin.
3. **Équipement tactique** autour du ligotage : fumigène, piège, leurre, longue-vue.
4. **Tell comportemental** subtil du coupable (profiling au-delà de la checklist).
5. Map vivante (bâtiments, points d'intérêt attirant les PNJ), événements de manche (airdrop, brouillard), traînées de sang des blessés en fuite (→ mécanique de pistage).
6. Indispensables : mode daltonisme (attributs = couleurs ! motifs en complément), kill feed, spectateur pendant le down.
7. Long terme : **mode asymétrique** — le coupable est un joueur qui imite les PNJ (genre SpyParty).

### 7.3 V1 multijoueur
Serveur autoritatif Node (Colyseus recommandé). **Le client ne doit JAMAIS connaître l'identité du coupable** (anti-cheat vital pour un jeu de déduction). Étapes : 2 joueurs LAN → interpolation/prédiction → lobby → 4-8 joueurs. Les bots actuels remplissent les lobbies incomplets. La génération déterministe (seed partagée) réduit la bande passante décor.

### 7.4 Option 3D (décision différée, en connaissance de cause)
Si un jour : Three.js, 3e personne caméra haute d'abord (préserve la lecture de foule), personnages en primitives colorées (les 36 combos se transposent tels quels), simulation restant en 2D (map plate), fouet → grappin raycast. **Prérequis absolu : un prototype de lisibilité des attributs à distance.** Coût ×4-6 vs le parcours 2D.

## 8. Philosophie de design (à préserver dans chaque arbitrage)
1. **La déduction avant le carnage** — chaque règle (malus, coupable vivant, fouet sans filtre, panique au tir) pousse vers « observer puis agir ».
2. **Le risque toujours affiché** — le joueur connaît le prix d'une erreur avant de tirer (HUD).
3. **La vulnérabilité comme climax** — le ligotage (10 s immobile et désarmé) est le moment que tout prépare ; le PvP existe pour le protéger ou l'interrompre.
4. **Aucun accident ne casse la partie** (transfert de coupable) ; les edge cases deviennent des choix stratégiques coûteux (fin au score à −500).
5. **L'IA triche intelligemment** — décider ce qu'elle doit réussir facilement, simuler ce qui doit être équitable ; sa faillibilité est calibrée et humaine.
6. **Un jalon = une version jouable** ; tout paramètre = une constante nommée.
