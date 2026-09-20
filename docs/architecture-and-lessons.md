---
name: architecture-and-lessons
---

# Décisions d'architecture et bugs racines

Extrait de l'ancien `CLAUDE.md` monolithique (sessions de juillet-août 2026, avant le passage au journal par session). Ce sont les décisions et bugs fondateurs — ce qu'il ne faut pas re-casser en repartant d'une intuition "plus simple". Un résumé condensé vit dans `../CLAUDE.md`, section "Points d'architecture à ne pas re-casser" ; le détail complet est ici.

## Décisions d'architecture

- **cffi mode API (compilation réelle) plutôt qu'ABI-mode (`dlopen`)** : nécessaire parce que l'API de settings « stable » de FreeRDP3 (`freerdp_settings_set_string/uint32/bool`) prend des constantes d'enum générées automatiquement dont les valeurs numériques ne sont pas garanties stables entre versions. Un petit shim C (`asyncrdp_shim.c`) compilé contre les vrais headers laisse le compilateur résoudre ces constantes.
- **Une seule frontière cffi** : tout passe par `_asyncrdp_cffi` (généré par `build_ffi.py`). Une première version mélangeait ABI-mode (`ffi.dlopen`) et API-mode — corrigé tôt dans le projet, voir historique des messages si besoin des détails.
- **`freerdp_client_context_new()` plutôt que `freerdp_new()`+`freerdp_context_new()` bruts** : c'est LE bug racine qui a bloqué tous les canaux pendant une bonne partie du développement (voir section bugs ci-dessous). Ne jamais revenir à la version « minimale ».
- **Redirections locales via `freerdp_client_load_addins()`** plutôt que réimplémenter les protocoles rdpdr/urbdrc/rdpsnd : ce sont les mêmes plugins de canaux que `xfreerdp` utilise, déjà matures dans `libfreerdp-client3`.
- **Clavier/souris synchrones** (pas de `run_in_executor`) : ce sont des écritures bufferisées sur le canal RDP, pas une négociation réseau bloquante comme `connect()`/`disconnect()`.
- **Frames en `Queue(maxsize=1)`** : on ne garde jamais qu'une frame de retard, jamais de backlog qui prend du retard sur l'affichage temps réel.
- **Clipboard fichiers** : téléchargement par blocs (`FILECONTENTS_RANGE`), parcours récursif des dossiers côté annonce locale (chemins relatifs avec `\` en séparateur, convention MS-RDPECLIP).

## Historique des bugs trouvés en testant réellement

Aucun de ces bugs n'était visible à la relecture — tous trouvés en compilant et en exécutant contre un vrai serveur (xrdp, sandbox Ubuntu 24.04, FreeRDP 3.30.0).

### Bugs de compilation (premier passage au compilateur réel)
- Ordre des déclarations `extern "Python"` : cffi place ses définitions générées APRÈS le `#include` du shim dans le fichier C généré — sans prototype `static` explicite en tête de fichier, déclaration implicite non-static en conflit avec la définition static générée plus loin.
- `CLIPRDR_FORMAT_DATA_RESPONSE.msgFlags`/`.dataLen` : en réalité nichés dans un sous-champ `.common` (`CLIPRDR_HEADER`), pas des champs directs.
- `RDPDR_DTYP_PRINTER` n'existe pas : le vrai nom est `RDPDR_DTYP_PRINT`.
- `freerdp_client_add_dynamic_channel()` : signature réelle `(settings, count, const char* const params[])`, pas `(settings, name, args)`.
- `freerdp_client_add_drive()` n'existe pas comme helper dans FreeRDP3.

### Bug mémoire (trouvé via trace gdb)
`asyncrdp_add_drive/printer/serial_port` construits à la main (`calloc` + remplissage des champs `RDPDR_DRIVE`/`RDPDR_PRINTER`/`RDPDR_SERIAL`, dont les layouts de champs étaient pourtant corrects) crashaient à la libération — `freerdp_device_free()` → `freerdp_addin_argv_free()` suppose une structure interne construite par `freerdp_device_new()`, pas un `calloc` brut même avec les bons champs visibles dans le header. **Leçon : toujours utiliser les constructeurs fournis par la lib (`freerdp_device_new`) plutôt que reconstruire la struct à la main, même quand le layout semble public et stable.**

### Bug d'architecture (LE bug racine)
`asyncrdp_context_new()` utilisait `freerdp_new()`+`freerdp_context_new()` bruts au lieu de `freerdp_client_context_new()`. Conséquence : **aucun canal ne se connectait jamais** — confirmé en loggant tous les événements `ChannelConnected` sur 15s de session active (zéro événement). Une fausse piste avait d'abord été suivie (accusant le packaging Ubuntu de bloquer spécifiquement `rdpdr` via `freerdp_client_load_addins`) — cette piste était fausse, `rdpdr` fonctionne parfaitement une fois le vrai constructeur utilisé. Le correctif a aussi nécessité `freerdp_client_context_free()` comme destructeur symétrique.

### Bug de gestion d'erreurs
`_on_fd_readable` (Python) levait une exception depuis un callback `loop.add_reader` (jamais awaited, donc jamais rattrapée) sans désenregistrer les fds — cause une boucle d'erreurs infinie à toute déconnexion en cours de session, puisque le fd reste « lisible » en continu une fois la connexion cassée. Corrigé : stocker l'erreur dans `client.last_error`, appeler `_unregister_fds()`, ne jamais lever depuis ce callback.

### Bug protocolaire (canal cliprdr)
`self._cliprdr` n'était renseigné que de façon réactive (via les callbacks `ServerFormatList`/`DataRequest`/`DataResponse`). Si le serveur attend passivement que le client parle en premier, `announce_local_*()`/`request_remote_*()` restaient bloqués indéfiniment. Corrigé en ajoutant un callback `asyncrdp_on_cliprdr_ready`, symétrique de celui qui existait déjà pour `disp`, déclenché dès la connexion du canal (pas seulement sur premier échange de données).

### Bug de calcul (clipboard image)
`_dib_to_texture()` (côté `gcm_gtk4_clipboard_bridge.py`) calculait `bfOffBits` (offset des pixels dans un BMP reconstruit) sans tenir compte de la table de couleurs présente pour les images en couleurs indexées (≤8 bits/pixel), intercalée entre le header DIB et les données de pixels. Confirmé par diff octet à octet contre l'original (exactement 1 octet de différence, sur ce champ précis, pixels eux-mêmes intacts — un décodeur BMP strict aurait pu s'en trouver perturbé même si GdkPixbuf/ImageMagick semblent tolérants). Corrigé en calculant la taille de palette réelle (`num_colors * 4`) à partir de `biBitCount`/`biClrUsed`.

## Résultats de la campagne de tests par redirection

Serveur de test : `xrdp` + `xrdp-sesman` + `openbox` (session minimale), lancés manuellement via `setsid ... &` (pas de systemd dans le sandbox). Utilisateur de test : `rdptest`/`testpass123`.

| Test | Méthode | Résultat |
|---|---|---|
| Clipboard texte | `xclip` sur le X11 de la session distante, changement APRÈS connexion (le serveur ne renvoie pas l'état déjà présent avant connexion — comportement normal du protocole, pas un bug) | ✅ contenu identique, les deux sens |
| Clipboard image | `xclip -t image/bmp` avec une image 4bpp indexée | ✅ byte-identique après correctif palette |
| Clipboard fichiers | `xclip -t text/uri-list` | ✅ métadonnées + contenu réel identiques |
| Disque redirigé | Écriture de part et d'autre du montage FUSE (`~/thinclient_drives/<nom>`) | ✅ les deux sens, en connexion longue durée manuelle. Automatisé en script (`test_full_suite.py`), s'est montré sensible au cycle de vie de la session X sous-jacente (une session Xorg/chansrv réutilisée après déconnexion peut ne pas remonter proprement) — pas reproduit de façon fiable en une seule commande, mais le mécanisme lui-même est prouvé fonctionnel |
| Multi-écran | Deux `MonitorDef`, vérification de la frame reçue | ✅ résolution bounding-box exacte (2304x800), taille en octets correcte |
| Audio | Backend ALSA `null` côté client pour éviter le crash `snd_mixer_attach failed` | ⚠️ négociation de canal confirmée (formats échangés, round trip mesuré), aucun octet audio réel — sandbox sans PipeWire/ALSA fonctionnel côté session |
| Imprimante | CUPS installé et démarré manuellement (`cupsd -f`) | ❌ le CLIENT annonce correctement le device, mais xrdp-chansrv répond explicitement `Detected remote printer 'PRN1' (not supported)` — limitation du serveur de test, code client non mis en cause |
| Pipeline graphique RDPGFX / H.264 | `enable_graphics_pipeline=True`, câblage réel via `gdi_graphics_pipeline_init()` (frames dans le même `gdi->primary_buffer` que le GDI legacy) | ✅ **validé de bout en bout** après recompilation de FreeRDP avec `-DWITH_OPENH264=ON` (le paquet Ubuntu système est compilé `WITH_OPENH264=OFF`/`WITH_FFMPEG=OFF`, sans décodeur H.264 câblé — cause du crash initial). Un deuxième bug trouvé et corrigé au passage : `update->DesktopResize` non câblé, provoquait un `WINPR_ASSERT` (`gdi_ResetGraphics`) — corrigé dans `asyncrdp_shim.c`. Testé contre `freerdp-shadow-cli` (même codebase, recompilé pareil) avec une animation continue (`xclock`) pour garantir un vrai flux : **5 frames GFX/H.264 reçues en continu**, `client._gfx` non-`None` en permanence, résolution/taille correctes, déconnexion propre. Non-régression du chemin GDI legacy confirmée. Procédure complète et vérifiée : voir `H264_BUILD_GUIDE.md`. Non testé : contre un vrai Windows RDS, et avec NLA active. |
| Série / parallèle | Idem imprimante | ❌ même verdict : `(not supported)` côté xrdp pour les deux |
| USB | — | Jamais testé, aucun périphérique physique disponible |
| GTK4 (affichage + ponts) | — | Jamais exécuté, aucun environnement graphique local dans le sandbox pour lancer une vraie fenêtre GTK4 |
