---
name: session-27
---

[← index](README.md)

# Session 27 (2026-09-14 → 2026-09-15) — Presse-papier image GTK4 : résolu par une sous-classe `GdkContentProvider` écrite en C ; bug de timing `GI_TYPELIB_PATH` trouvé en cours de route

## Contexte

Reprise depuis l'archive `asyncrdp-20260912-055040.zip` (export de fin de session 26), consigne inchangée : continuer les features à faire. Le seul point encore ouvert qui dépendait de code plutôt que d'un environnement indisponible était le bug presse-papier image GTK4, dont la session 25 avait laissé le verdict suivant : « correctif hors de portée sans réponse de PyGObject **ou une sous-classe `GdkContentProvider` écrite en C** ». C'est cette seconde piste, décrite depuis deux sessions comme « plus lourde » et jamais tentée, qui a été prise cette fois.

Rappel du diagnostic hérité (session 24, confirmé par lecture de `gdk/gdkclipboard.c` au tag 4.14.5 + expérimentation) : le `GTask` construit par `gdk_clipboard_write_async()` est authentique et valide **côté C**, mais `user_data` arrive déjà à `None` **côté Python** dans `do_write_mime_type_async()` d'une sous-classe de `Gdk.ContentProvider`, avant même que le corps de la méthode ne s'exécute. Trois stratégies d'écriture différentes en PyGObject avaient été essayées (sessions 6, 17, 24), toutes échouant identiquement sur `g_task_return_boolean: assertion 'G_IS_TASK (task)' failed`. La cause était donc pointée sur le marshaling PyGObject de **ce vfunc précis**, dans le sens C-appelle-override-Python.

## Le raisonnement qui rend la piste C décisive

Si la perte a lieu dans le marshaling de l'override Python, alors tout correctif écrit en Python la subit par construction, quelle que soit son ingéniosité — c'est ce que les trois tentatives précédentes avaient établi empiriquement. Une sous-classe GObject écrite **entièrement en C** ne traverse simplement jamais cette frontière pour cet appel : GDK invoque directement le pointeur de fonction inscrit dans la vtable du type, sans aucun mécanisme de marshaling vfunc-Python-override. C'est le seul chemin d'exécution qui contourne la cause identifiée plutôt que de la contourner de biais.

## Ce qui a été écrit

`integrations/gtk4/native/asyncrdp-image-provider.{h,c}` — un `GdkContentProvider` natif (`G_DECLARE_FINAL_TYPE` / `G_DEFINE_FINAL_TYPE`) :

- `ref_formats()` annonce `image/png` (là où `Gdk.ContentProvider.new_for_value()` renvoyait une liste de mime-types **vide** depuis le 2026-09-02, symptôme d'origine du ticket) ;
- l'encodage PNG est fait **une seule fois, à la construction**, via `gdk_texture_save_to_png_bytes()` — API GTK4 synchrone qui ne peut pas échouer pour une texture valide (contrairement à un chargement de fichier), donc pas de PNG fait main ni de chemin d'erreur à gérer au moment de l'écriture ;
- `write_mime_type_async()` / `write_mime_type_finish()` réutilisent le pattern `GTask` standard de GLib, mais entièrement en C. Le `callback`/`user_data` reçus viennent directement de `gdk_clipboard_write_async()` sans jamais passer par un override Python — exactement ce qui manquait aux trois tentatives précédentes ;
- `get_value()` délègue à la classe parente hors du cas `GDK_TYPE_TEXTURE`, pour ne pas casser les consommateurs `GValue` internes à GTK4.

Compilé en `-Wall -Wextra -Werror`, 0 avertissement. Un seul écart relevé au passage : `gdk_content_formats_new()` attend `const char **`, pas `const char *const *` — le tableau de mime-types est déclaré en conséquence.

`build_gir.py` (nouveau, à la racine, même esprit et même rôle que `build_ffi.py` pour le binding cffi principal) : compile la bibliothèque partagée, scanne le `.gir`, patche le `.gir`, compile le `.typelib`. Trois pièges rencontrés, tous silencieux ou trompeurs, tous documentés dans le script lui-même :

1. **Repasser les cflags bruts de `pkg-config` à `g-ir-scanner` échoue** — le scanner a son propre analyseur d'arguments (optparse) qui interprète `-mfpmath=sse`/`-msse2` comme des options courtes `-m` qui lui seraient destinées (`no such option: -m`). `--pkg=gtk4` délègue l'appel à `pkg-config` en interne, sans ce conflit.
2. **Sans `--identifier-prefix`/`--symbol-prefix` explicites, le scan sort un `.gir` vide** — le préfixe C réel (`Asyncrdp`/`asyncrdp_`) diffère du nom de namespace GI (`AsyncrdpClipboard`), et `g-ir-scanner` rejette alors **tous** les symboles comme hors namespace. L'échec ne se manifeste que par quatre lignes `Warning: Unknown namespace for symbol` au milieu d'une sortie par ailleurs normale, avec un code de retour 0 et un `.gir` bien formé mais sans aucune classe.
3. **`g-ir-scanner` n'écrit que le nom nu de la bibliothèque** dans l'attribut `shared-library`, résolu ensuite par le lieur dynamique via son chemin de recherche standard — jamais garanti pour une bibliothèque qui n'est pas installée à un emplacement système, ce qui est le cas de `_build/`. Le script réécrit cet attribut en chemin absolu, ce qui rend le typelib utilisable sans `LD_LIBRARY_PATH`.

## Vérification : un vrai lecteur externe, pas une vérification interne

Premier essai isolé (hors pytest, sous Xvfb), avec le protocole exact qui faisait échouer tout le reste depuis le 2026-09-02 — `xclip -selection clipboard -t image/png -o` comme lecteur X11 **externe** : en-tête PNG authentique (`\x89PNG\r\n\x1a\n`) reçu **dès la première tentative**, 76 octets, mime-type `image/png` correctement annoncé, zéro assertion `G_IS_TASK`. Le diagnostic de la session 24 est donc confirmé par la négative : ce n'était bien ni GDK, ni la méthode de construction de la texture, ni une maladresse de ce projet.

## Intégration dans le pont livré, et le bug trouvé en le faisant

`ClipboardBridge._on_remote_image_changed` (`gcm_gtk4_clipboard_bridge.py`) utilise désormais ce provider quand le typelib est disponible, via `_load_native_image_provider()`, avec repli propre vers `Gdk.ContentProvider.new_for_value()` sinon (`build_gir.py` jamais exécuté, ou `gi` qui n'est pas le vrai PyGObject — cas de `tests/gtk_stub/`). Jamais une erreur bloquante : un typelib absent dégrade vers l'ancien comportement (mime-types vides pour un lecteur externe, la limitation documentée depuis le 2026-09-02), il ne casse pas la classe.

**Bug distinct trouvé à ce moment-là, et c'est le point à retenir de cette session autant que le provider lui-même** : la première version de `_load_native_image_provider()` ajoutait le dossier `_build/` à `os.environ["GI_TYPELIB_PATH"]` avant d'appeler `gi.require_version("AsyncrdpClipboard", "1.0")`. Ça marchait dans le script de vérification isolé — et échouait dans la vraie suite de tests, avec un `ValueError: Namespace AsyncrdpClipboard not available` net (pas un repli silencieux, heureusement).

Raison : GObject-Introspection ne lit `GI_TYPELIB_PATH` qu'**une seule fois**, à la toute première opération de la `Repository` par défaut du process — c'est-à-dire au tout premier `gi.require_version()`, quel que soit le namespace. Dans la suite de tests, `_require_gtk4()` a déjà chargé `Gtk` bien avant que ce pont ne soit jamais importé ; dans une vraie application GTK4, `Gtk`/`Gdk` sont nécessairement chargés avant le pont aussi. La variable d'environnement arrive donc **toujours** trop tard dans les deux cas — le script de vérification isolé, qui fixait la variable avant tout import `gi`, était le seul contexte au monde où elle fonctionnait.

Remplacé par `GIRepository.Repository.prepend_search_path()`, qui modifie la même liste de recherche mais **à chaud**, à l'appel, quel que soit ce qui a déjà été chargé avant. C'est l'API prévue exactement pour ce cas (typelib hors des emplacements système). Vérifié par reproduction directe de l'ordre de chargement réel (`Gtk` d'abord, pont ensuite) avant et après le correctif.

Leçon transposable : une vérification isolée qui passe ne dit rien tant que le composant n'a pas été exercé dans l'ordre d'initialisation réel de l'application hôte.

## Tests

- `test_clipboard_image_available_to_external_reader` — **n'est plus un `xfail`** : c'est le test qui passe par le vrai `ClipboardBridge` livré, avec un vrai DIB BMP indexé en entrée, et vérifie la sortie via `xclip`. Le mime-type attendu est `image/png` et non `image/bmp` comme le demandait sa version précédente : `gdk_texture_save_to_png_bytes()` est l'API GTK4 la plus directe pour encoder en C à la construction, et `image/png` est de toute façon le format que la plupart des lecteurs externes réels demandent.
- `test_clipboard_image_native_c_provider_delivers_real_png` (nouveau) — l'expérience isolée sur le mécanisme lui-même, indépendante du pont, pour que la régression soit attribuable si elle survient.
- `test_clipboard_image_custom_content_provider_write_bug` et `test_clipboard_image_write_bug_survives_direct_callback_invocation` — **restent `xfail`, délibérément**. Le bug PyGObject sous-jacent n'est pas corrigé, il est contourné ; ces deux tests documentent fidèlement une impasse réelle et doivent continuer à échouer tant que PyGObject n'a pas bougé. Les transformer en tests passants aurait été une fausse bonne nouvelle.

Suite complète rejouée, GTK4 live inclus sous Xvfb : **103 passed, 5 skipped, 2 xfailed, 0 failed**. `ruff check .` : 0 erreur. Aucune régression.

Dépendance manquante rencontrée au passage, sans lien avec cette feature : `gbulb` (extra `gtk4-test`, requis par `test_clipboard_bridge_files_download_from_rdp_via_external_reader` depuis la session 16) n'était pas installé dans ce bac à sable — installé, le test repasse. L'extra existe bien dans `pyproject.toml` ; c'est `install.sh` qui n'a pas de drapeau pour lui (voir « Ce qui n'a pas été fait »).

## Packaging

`install.sh --gtk4` compile désormais ce composant automatiquement : ajout des paquets de compilation (`libgtk-4-dev`, `gobject-introspection`, `libgirepository1.0-dev` — absents d'un poste qui ne fait qu'utiliser un typelib déjà construit, d'où un bloc séparé) puis `python3 build_gir.py`. En cas d'échec de compilation : avertissement clair et le script continue, même politique que le bloc `udevadm` corrigé en session 26 — le pont se dégradera proprement, ce n'est pas une raison d'interrompre toute l'installation. Rejoué en entier depuis un `_build/` supprimé, code de sortie 0.

`MANIFEST.in` : ajout de `build_gir.py` et des sources `*.c`/`*.h` sous `integrations/`, par symétrie avec `build_ffi.py`/`src/asyncrdp/_shim.c` — sans quoi une distribution source ne contiendrait pas de quoi reconstruire le provider.

## Correction de dates dans les fichiers de suivi

Les paragraphes rédigés en début de session avaient été datés « 2026-09-12 » — la date de l'archive zip reprise, pas celle du travail. La date réelle (confirmée par `date` et les horodatages de fichiers) est le 2026-09-14 pour le provider natif et le 2026-09-15 pour l'intégration, le câblage `install.sh` et la vérification complète. Corrigé dans `CLAUDE.md`, `docs/features-backlog.md`, `tests/test_gtk4_live.py` et `gcm_gtk4_clipboard_bridge.py`. Les sessions archivées 20 à 26 n'ont pas été touchées (leurs mentions du 2026-09-12 sont légitimes : c'est la date de livraison de la session 26).

## Ce qui n'a pas été fait

- Le rapport de bug PyGObject rédigé en session 25 (`docs/pygobject-async-vfunc-userdata-bug-report.md`) n'est toujours pas déposé — aucun compte GitLab GNOME disponible ici. Il reste pertinent : cette session le contourne, elle ne le résout pas en amont, et la repro minimale qu'il contient reste valable.
- `install.sh` n'a toujours pas de drapeau pour l'extra `gtk4-test` (`gbulb`), installé à la main cette session. Petit manque de packaging repéré mais volontairement non traité ici pour ne pas mélanger deux sujets dans une même livraison.
- USB bout-en-bout (matériel physique) et imprimante/série/parallèle (serveur RDP alternatif à xrdp) : inchangés, toujours hors de portée de ce sandbox.
- Le `DeprecationWarning` sur `Gdk.pixbuf_get_from_texture` (sens local → RDP, chemin distinct de celui traité ici) est visible dans la sortie de tests et n'a pas été traité — candidat propre pour une prochaine session.

## Prochaine étape

Le backlog `docs/features-backlog.md` est désormais entièrement traité, à l'exception des deux points bloqués par l'environnement (USB matériel, imprimante/série/parallèle). Pistes concrètes restantes, par ordre de coût croissant : le drapeau `--gtk4-test` manquant dans `install.sh`, le `DeprecationWarning` `pixbuf_get_from_texture` ci-dessus, puis le dépôt du ticket PyGObject si un compte devient disponible.
