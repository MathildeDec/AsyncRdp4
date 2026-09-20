[← index](README.md) — session 05 — 2026-09-02

## Diagnostic affiné du bug mime-type presse-papier image GTK4 (2026-09-02, nouvelle session)

Sandbox repartie de zéro pour cette session (comme d'habitude — non
persistante). Tâche choisie dans le backlog moyenne-priorité de
`features.md` : approfondir le seul point encore ouvert issu de la
campagne GTK4 live du 2026-09-02 précédent, à savoir pourquoi
`Gdk.ContentProvider.ref_formats().get_mime_types()` est vide pour une
image, empêchant un lecteur presse-papier externe de récupérer l'image
collée depuis RDP (`test_clipboard_image_available_to_external_reader`,
`xfail`).

**Mise en place** : `apt-get install -y gir1.2-gtk-4.0 python3-gi xvfb
xclip x11-apps imagemagick` puis `pip install --break-system-packages
pytest loguru` (numpy déjà présent). Volontairement **pas** de
`freerdp3-dev`/`libwinpr3-dev`/`pip install -e .` cette fois — inutile
pour cette investigation précise, voir plus bas.

**Piège retrouvé, nouvelle variante** : réutiliser tel quel
`PYTHONPATH=tests/gtk_stub` (comme le fait `test_gtk4_bridges.py` en
interne, via substitution `sys.modules`) casse `test_gtk4_live.py`,
parce que `tests/gtk_stub/gi/` est un vrai package sur le disque — le
mettre dans `PYTHONPATH` fait qu'`import gi` charge le **faux** `gi`
(celui du stub, qui n'a pas `require_version`) au lieu du vrai paquet
système `python3-gi`. Corrigé en copiant uniquement
`tests/gtk_stub/asyncrdp.py` (pas tout le dossier) dans un répertoire
séparé (`/tmp/.../asyncrdp_stub_only/`) et en ne mettant que celui-là
dans `PYTHONPATH`. Une fois ce piège évité : **toute la suite
`test_gtk4_live.py` tourne sans avoir besoin du binding FreeRDP
compilé** — `gcm_gtk4_clipboard_bridge.py` importe bien `from asyncrdp
import Client as RdpClient, RemoteFileInfo` en tête de fichier, mais
aucun test de ce fichier n'instancie réellement un `RdpClient` (tous
utilisent `types.SimpleNamespace()` à la place) — le stub suffit à
satisfaire l'import. Confirmé : `4 passed, 1 xfailed` en 2.42 s,
résultat strictement identique à la session précédente qui avait, elle,
compilé FreeRDP pour rien de plus sur ce fichier précis. Note pour la
suite : gain de temps réel pour quiconque veut retravailler
spécifiquement ce fichier sans toucher au reste du binding.

**Hypothèse invalidée : « loader PNG/JPEG gdk-pixbuf manquant ».** La
session précédente n'avait vérifié que `loaders.cache`
(`gdk-pixbuf-query-loaders`), qui ne liste que les modules **chargés
dynamiquement** (`.so` séparés sous `loaders/`). Or PNG et JPEG, dans
ce paquet Ubuntu (`libgdk-pixbuf-2.0-0` 2.42.10), sont compilés
**directement dans** `libgdk_pixbuf-2.0.so.0` (symboles `png_*`
trouvés via `strings` sur la lib) plutôt que fournis comme modules
séparés — ils n'apparaissent donc jamais dans `loaders.cache`, mais
`GdkPixbuf.Pixbuf.get_formats()` (l'API qui fait réellement foi) les
liste bel et bien :

```
nom      writable  scalable  disabled  mime_types
png      True      False     False     ['image/png']
jpeg     True      False     False     ['image/jpeg']
bmp      True      False     False     ['image/bmp', ...]
```

Donc le module de sauvegarde PNG existe, est actif, et n'est pas
désactivé — l'ancienne hypothèse (« un paquet manque ») ne tient plus.

**Diagnostic plus précis, par élimination.** Script autonome
(`diag_contentprovider.py`, hors suite pytest, pour isoler la variable)
appelant directement `Gdk.ContentProvider.new_for_value(texture).
ref_formats().get_mime_types()` **avant même toute interaction avec un
`Gdk.Display`/`Gdk.Clipboard`** : déjà vide. Donc le problème n'est pas
côté transport X11/xclip, il est déjà là au niveau du `ContentProvider`
en mémoire, purement en-process. Trois constructions différentes
testées pour la texture/valeur (`diag_texture_variants.py`,
`diag_pixbuf_provider.py`), toutes avec le même résultat (`mimes: []`) :

| Construction | GType rapporté | mimes |
|---|---|---|
| `Gdk.Texture.new_for_pixbuf(pixbuf)` (code actuel de `_dib_to_texture`) | `GdkMemoryTexture` | `[]` |
| `Gdk.Texture.new_from_bytes(GLib.Bytes(png_reel))` (PNG déjà encodé, l'API "canonique" pour ce cas) | `GdkMemoryTexture` | `[]` |
| `GdkPixbuf.Pixbuf` passé tel quel comme valeur (sans aucune `Gdk.Texture`) | `GdkPixbuf` | `[]` |

Conclusion pratique : ni un paquet manquant, ni un choix d'API
différent dans `_dib_to_texture`/`ClipboardBridge` ne réglerait ce
problème — aucune des trois façons standard de représenter une image
pour un `ContentProvider` ne produit de mime-type sérialisable dans ce
GTK4 4.14.5 (Ubuntu `4.14.5+ds-0ubuntu0.10`) précompilé, dans cet
environnement (Xvfb, backend X11, pas de session D-Bus — le message
`Unable to acquire session bus: Failed to execute child process
"dbus-launch"` apparaît systématiquement, sans certitude que ce soit
lié).

**Piste non vérifiée, notée pour une session future** : une recherche
rapide a fait remonter un patch réel et indépendant du projet Vim
(« GTK4: does not support all clipboard formats », `gui_gtk4_cb.c`)
où les mainteneurs de Vim ont dû écrire leur propre sous-classe
`GdkContentProvider` (`VimContentProviderClass`, avec ses propres
`ref_formats`/`write_mime_type_async`/`write_mime_type_finish`) plutôt
que de compter sur la sérialisation GType→mime-type automatique de
GTK4 pour leur presse-papier — cohérent avec l'idée que cette
sérialisation automatique, pour les types image en tout cas, a de
vraies limites en amont plutôt que d'être une particularité cassée de
ce seul sandbox. Pas vérifié plus loin faute de temps (lecture du code
source C de GTK4 lui-même, hébergé sur `gitlab.gnome.org`, non
accessible depuis ce sandbox — seul `github.com` et ses miroirs le
sont). Si quelqu'un reprend ce sujet : la prochaine étape logique
serait soit de trouver un miroir GitHub du dépôt GTK, soit
d'implémenter et tester un `GdkContentProvider` custom minimal côté
`ClipboardBridge`, à la manière de Vim, pour voir si cela suffit à
faire apparaître un mime-type.

**Fichiers modifiés cette session** : uniquement le texte de la raison
`xfail` et du docstring d'en-tête dans `tests/test_gtk4_live.py` (pour
refléter ce diagnostic plus précis) — aucun changement de
comportement, le test reste `xfail` (toujours `strict=False`).
`pytest tests/test_gtk4_live.py -v` avec le stub `asyncrdp` minimal :
toujours `4 passed, 1 xfailed`, résultat inchangé.
