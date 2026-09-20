[← index](README.md) — session 03 — 2026-09-02

## Exécution réelle des ponts GTK4 dans un vrai GTK4 + Xvfb (2026-09-02)

Contrairement à l'hypothèse de la session du 2026-09-02 précédente (voir
section revérification `_shim.c` plus bas, écrite plus tôt le même
jour), ce sandbox-ci s'est révélé avoir accès à tout ce qu'il fallait
via `apt` (`archive.ubuntu.com`/`security.ubuntu.com` accessibles) :
`freerdp3-dev`/`libwinpr3-dev` (3.31.0), `gir1.2-gtk-4.0` (GTK 4.14,
déjà avec `python3-gi`/`xvfb` préinstallés), `xclip`, `x11-apps`
(`xwd`), `imagemagick` (`convert`). De quoi enfin faire ce que le stub
du 2026-09-01 documentait comme volontairement hors de portée.

**Binding compilé pour de vrai.** `pip install -e .` (pas
`python3 build_ffi.py` seul, qui compile dans `./asyncrdp/` au lieu de
`src/asyncrdp/` — le module ne s'importe pas depuis là) compile sans
erreur contre FreeRDP 3.31.0, seulement des avertissements de
dépréciation. `import asyncrdp` fonctionne, les 46 tests unitaires
existants passent avec ce binding réel chargé.

**Piège n°1 — Xvfb ne survit pas de façon fiable d'un appel outil à
l'autre.** Démarré en arrière-plan (`setsid Xvfb :99 ... &`), parfois
toujours là à l'appel suivant, parfois disparu sans message d'erreur
(`Fatal server error: Server is already active for display 99` dans le
log au *prochain* essai de démarrage, alors que le premier avait
pourtant réussi). Pattern retenu : toujours démarrer Xvfb ET lancer le
test dans le même appel outil (`setsid Xvfb :99 ... & sleep 1.5 &&
DISPLAY=:99 ... pytest ...`).

**Piège n°2 — `Gtk.Application` segfault au lieu de lever une
exception Python si le display par défaut n'est pas valide.** Le tout
premier essai (Xvfb du coup déjà mort, cf. piège n°1) a produit un
`Segmentation fault` sec après `Gtk-CRITICAL: gtk_icon_theme_get_for_
display: assertion 'GDK_IS_DISPLAY (display)' failed` — aucune
exception Python propre. `Gtk.init_check()` donne un diagnostic correct
(`Gtk couldn't be initialized`) là où laisser `Gtk.Application()`
échouer plus tard segfault. Utile si un futur test doit détecter
proprement l'absence de display plutôt que planter.

**Piège n°3 — deadlock garanti si un test lit son PROPRE presse-papier
via un `subprocess.run()` bloquant.** `xclip -o` interroge notre
application (propriétaire de la sélection X11) ; celle-ci ne peut
répondre que si sa boucle GLib tourne. Un `subprocess.run()` synchrone
appelé depuis un callback GLib bloque cette même boucle → gel complet,
observé concrètement (`timeout` a dû tuer le process). Solution :
`subprocess.Popen` non bloquant + `GLib.timeout_add` qui poll
`proc.poll()` périodiquement, laissant la boucle GLib respirer entre
deux vérifications (voir `_run_async` dans `tests/test_gtk4_live.py`).

**Piège n°4 — `xclip` en mode "pose une valeur" démonise et hérite des
fds.** Quand on lui donne du texte à poser sur le presse-papier, `xclip`
fork un processus qui continue à tourner en arrière-plan pour servir
les futures requêtes de lecture — ce démon hérite de `stdout`/`stderr`.
Les capturer avec `subprocess.PIPE` fait bloquer indéfiniment
`proc.stdout.read()` une fois le parent sorti, puisque le tube reste
ouvert côté démon (jamais d'EOF). Solution : `stdout=DEVNULL,
stderr=DEVNULL` pour cet appel précis ; `PIPE` reste correct pour une
lecture (`xclip -o`, qui ne démonise pas, lui).

**Résultats, une fois ces pièges évités :**

1. *Rendu écran réel* — une frame BGRA32 de couleurs connues (rouge,
   vert, bleu, jaune) poussée via la vraie `RdpView._show_frame()`,
   fenêtre GTK4 réelle présentée sous Xvfb, capture d'écran par `xwd`
   (lecture directe des pixels du serveur X — outil complètement
   indépendant de GDK) puis conversion PNG par `convert` : les quatre
   quadrants affichent les bonnes couleurs à l'écran. Confirme
   concrètement ce que le commentaire du code ne faisait que supposer :
   `Gdk.MemoryFormat.B8G8R8A8` est le bon format pour du BGRA32 natif
   little-endian sur cette plateforme — pas de canaux inversés.
2. *Presse-papier texte, bidirectionnel, via un client X11 externe
   réel* — `bridge._on_remote_text_changed(...)` suivi d'une lecture
   par un `xclip -o` externe (pas depuis l'intérieur du même process) :
   contenu identique. Dans l'autre sens, un `xclip -selection
   clipboard` externe pose du texte, le vrai signal GDK
   `Gdk.Clipboard::changed` se déclenche (4 fois de suite pour un seul
   changement — probablement lié au fork de xclip, sans conséquence
   puisque le pont ne garde que la valeur, idempotente), et le pont
   livre bien le texte à `rdp.clipboard.announce_local_text(...)`.
3. *Décodage BMP indexé réel* — le DIB 4bpp/16 couleurs rejouant
   exactement le bug `bfOffBits` historique (cf. plus haut) est décodé
   avec succès par le **vrai** `GdkPixbuf.PixbufLoader`, pas le stub qui
   se contentait de capturer les octets. Reconfirme le correctif en
   conditions réelles.
4. *Presse-papier image, offert à un lecteur externe* — **ne
   fonctionne pas dans ce sandbox**, et ce n'est pas un bug du pont :
   `Gdk.ContentProvider.new_for_value(texture).ref_formats().
   get_mime_types()` renvoie une liste **vide**, y compris pour
   `image/bmp` alors que `/usr/.../gdk-pixbuf-2.0/2.10.0/loaders.cache`
   liste bien un module BMP avec le flag « writable ». Aucun module de
   chargement PNG/JPEG n'est présent du tout dans ce paquet Ubuntu
   minimal (seuls ani/bmp/gif/icns/ico/pnm/qtif/tga/tiff/xbm/xpm
   existent), ce qui pourrait expliquer que GTK4 ne trouve rien à
   proposer — mais BMP à lui seul aurait dû suffire vu son flag
   « writable », donc la cause exacte n'est pas élucidée (peut-être que
   le registre de sérialisation interne de GTK4 pour `GdkTexture` ne
   passe pas du tout par l'énumération `gdk_pixbuf_get_formats()`).
   Point notable : le décodage RDP → texture (le sens qui avait le bug
   historique) fonctionne très bien avec le vrai `GdkPixbuf`
   (résultat 3 ci-dessus) — c'est seulement l'étape *suivante*, offrir
   cette texture à un lecteur système externe, qui ne trouve aucun
   format. Capturé en `xfail` (pas `skip`) dans
   `tests/test_gtk4_live.py::test_clipboard_image_available_to_external_reader`
   pour qu'un futur passage inattendu à `XPASS` serve de signal.

**Non couvert, volontairement, faute de temps dans cette session :** le
pont fichiers (`_on_remote_files_changed`/dossiers récursifs) en
conditions réelles GTK4 — resté stub seulement ; l'intégration complète
avec une vraie connexion RDP asyncio (ces tests utilisent un objet
`RdpClient` factice `types.SimpleNamespace`, volontairement — seul le
côté GTK4 des ponts était en jeu ici, pas `asyncrdp` lui-même) ; un
vrai gestionnaire de fenêtres (aucun tourné sous Xvfb, sans conséquence
observée pour ces tests précis mais pourrait en avoir pour d'autres
scénarios, ex. focus).

Fichier : `tests/test_gtk4_live.py`, 4 tests, marqueur `gtk4_live`,
activé par `ASYNCRDP_TEST_GTK4=1` (skip propre sinon — voir
`conftest.py`). Mise en place détaillée dans le docstring du fichier.
