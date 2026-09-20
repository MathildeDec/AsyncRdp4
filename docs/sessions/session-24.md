---
name: session-24
---

[← index](README.md)

# Session 24 (2026-09-11) — Revérification complète (GTK4/Xvfb enfin disponibles) et diagnostic affiné du bug mime-type presse-papier image via le code source GDK

## Contexte

Reprise le lendemain de la session 23. `CLAUDE.md` indiquait que plus
rien n'était en attente au-delà des trois points bloqués par
l'environnement (USB de bout en bout, imprimante/série/parallèle,
presse-papier image GTK4) — à décider avec l'utilisateur avant de
repartir sur du travail de fond. Consigne de reprise, inchangée depuis
la session 23 : continuer les features à faire sans attendre cette
décision, un seul point avant livraison.

**Cadrage honnête avant de commencer** : contrairement à la session 23
(sandbox sans GTK4/Xvfb), ce sandbox-ci permet d'installer réellement
`freerdp3-dev`/`libwinpr3-dev` (3.31.0, identique aux sessions
précédentes malgré un cache `apt` non rafraîchi affichant d'abord un
candidat 3.5.1 — resolu par un `apt update`), `python3-gi`/
`gir1.2-gtk-4.0` (4.14.5, identique aux sessions précédentes) et
`xvfb`/`xclip`/`x11-apps`/`imagemagick`. Toujours pas de matériel USB
physique ni de serveur RDP tiers dans ce sandbox — ces deux points
n'ont donc pas pu être avancés cette session, seul le troisième
(presse-papier image GTK4) était à portée.

## Partie 1 — Revérification complète par exécution réelle

Binding recompilé (`pip install -e ".[test,gtk4,gtk4-test,usb]"`,
succès), `ruff check .` (0 erreur, inchangé). Piège `cd`/arrière-plan
documenté dans `docs/test-environment.md` reconfirmé à la dure : `cd
... && setsid Xvfb ... &` backgrounde en réalité **toute** la
commande composée (le `&&` est plus prioritaire que le `&` final), donc
le `cd` s'exécute dans le sous-shell arrière-plan et jamais dans le
shell principal — corrigé en séparant `cd` sur sa propre ligne avant
tout `setsid ... &`. Avertissement du 2026-09-02 dans
`docs/test-environment.md` reconfirmé plus tard dans cette même
session : `Xvfb` a survécu à plusieurs appels d'affilée avant de
finalement disparaître entre deux appels bien plus tard (constaté via
un `pgrep Xvfb` vide juste avant un segfault de la suite complète,
`DISPLAY=:99` pointant alors sur rien) — redémarré et réutilisé dans le
même appel, comme documenté. Confirme qu'il ne faut jamais supposer
Xvfb encore vivant d'un appel outil au suivant, même après plusieurs
succès d'affilée.

- `pytest tests/ --ignore=tests/test_integration_live.py --ignore=tests/test_gtk4_live.py`
  (unitaires + stubs, sans server/display) : **96 passed, 0 failed** —
  identique à la session 23, aucune régression.
- `ASYNCRDP_TEST_GTK4=1 DISPLAY=:99 GDK_BACKEND=x11 pytest
  tests/test_gtk4_live.py -v` : **5 passed, 2 xfailed** — sans même
  avoir besoin de `LIBGL_ALWAYS_SOFTWARE=1` cette fois (aucun segfault
  constaté ; cohérent avec le diagnostic du 2026-09-06 comme quoi ce
  contournement dépend de la disponibilité de DRI3/EGL, pas une
  exigence universelle). Première revalidation en conditions réelles de
  l'intégration GTK4 depuis la session 20 (2026-09-08) — la session 21
  (nettoyage ruff) et la session 23 (régression du stub) n'avaient pu
  rejouer que la suite sans display.

Ceci confirme : aucune régression introduite par le nettoyage ruff
(session 21), la règle udev USB (session 22) ni le correctif du stub
(session 23) sur l'intégration GTK4 réelle.

## Partie 2 — Diagnostic affiné du bug presse-papier image (xfail connu)

`CLAUDE.md`/`docs/features-backlog.md` documentaient ce bug comme
« hors de portée de ce projet : nécessiterait un GTK4 plus récent que
le 4.14.5 Ubuntu empaqueté ici, ou une lecture directe des sources
GDK — gitlab.gnome.org toujours hors de portée réseau ici » (session du
2026-09-07). Ce sandbox-ci n'a pas de GTK4 plus récent (4.14.5,
identique), mais dispose d'un accès réseau à `github.com` — qui héberge
un miroir en lecture seule de `GNOME/gtk` (`gitlab.gnome.org` reste
injoignable). Ressource jamais exploitée par les sessions précédentes.

**Lecture de source ciblée** (tag git `4.14.5`, correspondant exactement
au paquet installé) :

- `gdk/gdkclipboard.c::gdk_clipboard_write_async()` — crée un `GTask
  *task` réel via `g_task_new(clipboard, cancellable, callback,
  user_data)`, puis, quand le mime-type demandé correspond directement
  à un format annoncé (`gdk_content_formats_contain_mime_type`), appelle
  `gdk_content_provider_write_mime_type_async(priv->content, mime_type,
  stream, io_priority, cancellable, gdk_clipboard_write_done, task)` —
  **`task` est donc un pointeur C authentique et valide** au moment où
  notre vfunc `do_write_mime_type_async()` est invoquée côté Python.
- `gdk_clipboard_write_done()` (callback ci-dessus) : appelle
  `gdk_content_provider_write_mime_type_finish(content, result, &error)`
  puis, si succès, `g_task_return_boolean(task, TRUE)` — précisément la
  ligne dont l'assertion `G_IS_TASK (task)` échoue en pratique, ce qui
  colle exactement à l'endroit où les sessions précédentes situaient le
  crash (après un `write_mime_type_finish` qui, lui, réussit).
- `gdk/x11/gdkclipboard-x11.c` (gestion `SelectionRequest`) et
  `gdk/x11/gdkselectionoutputstream-x11.c`
  (`gdk_x11_selection_output_streams_request`/`_create`) : chemin
  X11 qui construit le flux de sortie et appelle
  `gdk_x11_clipboard_default_output_handler`, lequel appelle
  `gdk_clipboard_write_async` ci-dessus — rien dans ce chemin
  n'enveloppe `task` dans un objet supplémentaire ; la logique C, lue en
  entier, est saine.

**Expérimentation empirique** (script autonome, rejoué ensuite comme
test dans `tests/test_gtk4_live.py`, voir Partie 3) pour vérifier
l'hypothèse restante : le bug viendrait-il de la façon dont
`do_write_mime_type_async()` **refait transiter** le couple
`(callback, user_data)` reçu par un second constructeur GIO
(`Gio.Task.new(self, cancellable, callback, user_data)`, déjà en place
avant cette session) ? Testé : un `GTask` totalement indépendant
(`Gio.Task.new(self, cancellable, None, None)` — sans jamais lui faire
porter le couple reçu), puis appel **manuel et direct** de `callback`
avec ce couple reçu strictement intact — testé à la fois en 3
arguments (`callback(self, inner_task, user_data)`) et en 2
(`callback(self, inner_task)`, au cas où `gi.CCallback` porterait déjà
son propre `user_data` capturé en interne).

**Résultat, dans les deux cas : crash identique**
(`g_task_return_boolean: assertion 'G_IS_TASK (task)' failed`). Fait
nouveau confirmé par un simple `print()` : `user_data`, tel que reçu en
paramètre de `do_write_mime_type_async()` côté Python, **vaut déjà
`None`** — alors que la lecture de source ci-dessus établit que le
`task` C réel, lui, est valide à cet instant précis côté GDK.

**Conclusion affinée** (documentée dans le nouveau test, voir Partie 3) :
la perte a lieu **avant** que ce projet n'y touche, au moment où
PyGObject marshalle l'appel de ce vfunc précis depuis le C — pas dans la
façon dont le code Python réutilise ensuite ce qu'il croit avoir reçu.
Ceci écarte définitivement toute piste de correctif écrite uniquement en
Python dans ce projet, quelle que soit la technique d'écriture (les
sessions du 2026-09-02 et 2026-09-07 avaient déjà écarté plusieurs
variantes ; celle-ci écarte la famille complète des variantes
« retravailler ce que `do_write_mime_type_async` fait de ce qu'il
reçoit »). Un vrai correctif nécessiterait soit un correctif
PyGObject/gobject-introspection sur le marshaling du closure
`(callback, user_data)` de ce vfunc précis (hors de portée de ce
projet), soit une sous-classe `GdkContentProvider` écrite en C (un
vrai morceau de code C compilé, pas un pont Python — ampleur clairement
au-delà d'une seule tâche de session).

## Partie 3 — Nouveau test ajouté

`tests/test_gtk4_live.py::test_clipboard_image_write_bug_survives_direct_callback_invocation`,
`xfail` (pas `skip`) comme les deux tests existants sur ce même bug —
rejoue l'expérimentation ci-dessus dans le cadre habituel de la suite
(passe par `_make_indexed_bmp_dib`/`_dib_to_texture`/`_run_async` comme
les autres tests de ce fichier, pas un script à part). Vérifié : xfail
proprement (pas d'erreur de collection ni de crash du runner pytest
lui-même), suite complète rejouée sans régression : **101 passed, 3
xfailed, 0 failed** (`pytest tests/
--ignore=tests/test_integration_live.py`). Si ce test se met à xpass un
jour (nouvelle version de PyGObject qui corrige le marshaling), ce
serait une vraie confirmation à documenter, d'où `xfail` plutôt que
`skip`, comme pour les deux tests voisins.

## Ce qui n'a pas été fait, faute d'environnement disponible ici

Inchangé par rapport à `CLAUDE.md` : `tests/test_integration_live.py`
(vrai serveur RDP — non monté cette session, voir
`docs/test-environment.md` pour la recette si besoin), USB de bout en
bout (pas de matériel physique dans ce sandbox), imprimante/série/
parallèle (nécessiterait un serveur RDP structurellement différent de
xrdp, question déjà tranchée par lecture de source le 2026-09-05, voir
session 09). Rien dans cette session ne prétend le contraire.

## Prochaine étape

Toujours rien de plus en attente au-delà des trois points bloqués par
l'environnement — mais le point presse-papier image GTK4 est
désormais diagnostiqué avec une précision nettement supérieure
(cause pointée au marshaling PyGObject du closure `(callback,
user_data)` d'un vfunc précis, confirmée par lecture de source ET par
expérimentation, plutôt que « quelque part dans GDK »). Pour aller plus
loin sur ce point précis, deux pistes concrètes et hors de portée
d'une seule session : (1) chercher/ouvrir un rapport de bug PyGObject
existant sur ce marshaling, ou en ouvrir un ; (2) écrire un petit
`GdkContentProvider` en C (nouveau fichier, compilé et exposé via
cffi comme le reste du shim) plutôt qu'en PyGObject pur. USB et
imprimante/série/parallèle : toujours à décider avec l'utilisateur
comme indiqué depuis la session 22, aucun élément nouveau cette
session sur ces deux points faute de matériel/serveur disponibles ici.
