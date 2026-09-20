[← index](README.md) — session 01 — 2026-09-01

## Piste pour tester les ponts GTK4 sans environnement graphique (2026-09-01)

`gcm_gtk4_display_bridge.py`/`gcm_gtk4_clipboard_bridge.py` importent `gi.repository`
au niveau module, donc aucun test ne peut les importer sans GTK4 installé —
exactement le même problème que rencontré par `pluginvnc2` (paquet séparé,
plugin VNC/GTK4 pour ce même GCM) avec son fichier unique `vnc_tab.py`. Leur
solution, déjà vérifiée fonctionnelle chez eux (17 tests passent) : un stub
léger sous `tests/gtk_stub/` fournissant un faux `gi.repository` (classes
dynamiques acceptant n'importe quel appel de méthode) et un faux module
protocole (chez eux `asyncvnc2` ; ici ce serait `asyncrdp` si le stub doit
aussi éviter de dépendre du binding cffi compilé). Suffisant pour exécuter
la logique pure sans vrai display (Xvfb) ni vraie lib compilée — pas pour
tester du rendu réel. Reprendre `tests/gtk_stub/` de `pluginvnc2.zip` comme
point de départ plutôt que d'en écrire un de zéro.

## Stub GTK4 réalisé : `tests/gtk_stub/` + `tests/test_gtk4_bridges.py` (2026-09-01)

Fait, sans avoir eu `pluginvnc2.zip` sous la main (pas fourni dans cette
session) — le stub a donc été réécrit à partir de la description
ci-dessus plutôt que copié tel quel. Composition :

- `tests/gtk_stub/gi/repository.py` : faux `Gdk`/`GLib`/`Gtk`/`GdkPixbuf`/`Gio`.
  Un `_AnyCall` passe-partout (accepte n'importe quel attribut/appel) pour
  tout ce qui n'a pas de logique à tester (`Gtk.Picture`, les contrôleurs
  d'event, `Gdk.ContentProvider`...). Le reste est réellement implémenté
  quand un test en dépend : `Gdk.KEY_*` (vrais keysyms X11), `Gdk.keyval_to_unicode`
  (approximation Latin-1 imprimable, suffisante — GDK fait pareil sur cette
  plage), `Gdk.MemoryTexture.new` (capture les paramètres au lieu de créer
  une vraie texture), `GdkPixbuf.PixbufLoader` (ne décode rien — capture
  juste les octets écrits dans `last_written`, pour relire le
  `BITMAPFILEHEADER` produit et vérifier `bfOffBits` sans dépendre de
  gdk-pixbuf réel).
- `tests/gtk_stub/asyncrdp.py` : faux module protocole (juste `Client` et
  `RemoteFileInfo`, les deux seuls symboles importés par les ponts).
- `tests/test_gtk4_bridges.py` : au lieu de la commande
  `PYTHONPATH=tests/gtk_stub pytest tests/ --ignore=tests/gtk_stub`
  décrite ci-dessus (qui aurait pollué `sys.modules['asyncrdp']` pour
  TOUTE la suite, cassant potentiellement `test_rdp_options.py`/
  `test_file_group_descriptor.py` qui ont besoin du vrai binding
  compilé), le fichier substitue `sys.modules['gi']`/`['gi.repository']`/
  `['asyncrdp']` juste le temps d'importer les deux ponts via
  `importlib.util.spec_from_file_location`, puis restaure l'état
  précédent. Les modules de pont déjà importés gardent leurs références
  directes vers les objets du stub — rien ne casse pour le reste de la
  suite. Vérifié : `pytest tests/` (sans le binding compilé, comme dans ce
  sandbox) échoue à la collecte de `test_rdp_options.py`/
  `test_integration_live.py` avec un `ModuleNotFoundError: No module
  named 'asyncrdp'` propre — pas de faux positif dû à une fuite du stub.

Résultat : **26 tests, tous passent** (`python3 -m pytest
tests/test_gtk4_bridges.py`). Couverture : mise à l'échelle souris
(`_to_remote_coords`, clamping, widget non dimensionné), mapping
boutons/molette (inversion de signe), mapping clavier (`_CONTROL_KEYS`,
caractères imprimables, `ValueError` du shim avalé proprement), et la
régression directe du bug de palette `bfOffBits` (`_dib_to_texture` pour
24bpp/8bpp/4bpp/1bpp, y compris le cas `biClrUsed == 0` = palette
maximale) + `_pixbuf_to_dib` (ordre BGR, bottom-up, padding de fin de
ligne sur 4 octets).

Limite assumée (documentée dans le docstring du fichier de test) : ceci
ne teste PAS le rendu réel à l'écran, ni le vrai décodage BMP par
gdk-pixbuf, ni le vrai presse-papier système — seulement la logique
Python pure. L'exécution réelle dans une appli GTK4 reste à faire (cf.
`features.md`, désormais priorité haute restante sur ce sujet).
