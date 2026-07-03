# 🍪 Battle for the Cookie

Battle royale 2D top-down croisant BR classique et déduction visuelle (« Qui est-ce ? » létal) : 120 PNJ, un coupable voleur de cookie, des indices à chaque manche, une zone qui se resserre — et une capture finale au fouet, **vivant**.

## Jouer

Deux modes, zéro build (tout en HTML autonome + CDN) : ouvrir `index.html` et choisir, ou visiter le déploiement Vercel. En local, servir le dossier (`python3 -m http.server`) — le mode 3D utilise un module ES.

- **Mode 2D** (`2d.html`) — le jeu complet vu du dessus : bots, zone, loot, capture au fouet.
- **Mode 3D FPS** (`3d.html`) — le même jeu complet à la première personne, style low-poly blocky : bots, zone, arsenal, loot et capture au fouet.

## Contrôles

**2D** : ZQSD/WASD/flèches • souris = visée + tir • 1-4/molette = armes • R = medkit • E = fouet (finale)

**3D** : ZQSD/WASD • souris = visée + tir • 1-4/molette = armes • Shift = sprint • Espace = saut • C = glissade • R = medkit • E = fouet (finale), clic maintenu = ligotage • Échap = pause

## Structure

- `index.html` — accueil, choix du mode
- `2d.html` — jeu 2D complet (Phaser 3, copie déployée de `battle-for-the-cookie-jalon7.html`)
- `3d.html` — jeu complet en FPS 3D (Three.js)
- `BATTLE-FOR-THE-COOKIE-brief-final.md` — brief de conception complet
- `CLAUDE.md` — guide projet pour Claude Code (mécaniques, pièges techniques, roadmap)
