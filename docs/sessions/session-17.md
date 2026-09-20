[← index](README.md) — session 17 — 2026-09-07

## Bug d'écriture du ContentProvider custom : la cause n'est pas là où on la cherchait (2026-09-07, nouvelle session)

Sandbox reparti de zéro (`apt-get install python3-gi gir1.2-gtk-4.0
xvfb xclip python3-pytest x11-utils`, puis `pip install loguru numpy`
pour que `gcm_gtk4_clipboard_bridge.py` s'importe — aucun binding
FreeRDP compilé n'a été nécessaire pour cette session, cf. la
technique déjà notée : stub minimal `asyncrdp`/`asyncrdp.tracing`
injecté dans `sys.modules` avant l'import, mêmes noms que
`tests/gtk_stub/asyncrdp.py`). Confirmation au passage du piège déjà
documenté ailleurs dans ce fichier (section PipeWire) : **aucun
process d'arrière-plan (`Xvfb`) ne survit d'un appel outil à l'autre
dans ce sandbox** — `Xvfb :99 &` suivi d'un tool call séparé pour le
test donne un socket X mort ; il faut lancer `Xvfb` et le script dans
la **même** invocation shell.

Tâche reprise : la piste « moyenne priorité » laissée ouverte le
2026-09-02 (bug d'écriture du `GdkContentProvider` custom, xfail
`test_clipboard_image_custom_content_provider_write_bug`). Deux
hypothèses restées non tranchées à l'époque ont été testées :

1. **`source_object=None` au lieu de `self` dans `Gio.Task.new()`**
   (hypothèse notée : PyGObject marshaling limité pour un
   `source_object` sous-classe Python d'un type GObject abstrait) —
   `experiment_source_object.py`. Échec identique.
2. **Référence Python forte explicite sur le `provider` lui-même**
   (pas seulement `self`/`task`/`stream` comme testé la fois
   précédente), pour écarter tout ramasse-miettes prématuré du wrapper
   PyGObject pendant l'opération asynchrone — même échec identique.
3. **`Gio.SimpleAsyncResult` à la place de `Gio.Task`**, piste
   explicitement notée comme prochaine étape ci-dessus —
   `experiment_simpleasyncresult.py`. Résultat clé : **échec identique,
   avec exactement le même message** `g_task_return_boolean: assertion
   'G_IS_TASK (task)' failed`, alors que ce code n'appelle **jamais**
   `Gio.Task` nulle part.

Ce troisième essai déplace le diagnostic : un `g_task_return_boolean`
qui échoue alors qu'aucun `Gio.Task` n'existe dans notre code élimine
d'un coup toutes les hypothèses de marshaling PyGObject côté
`ContentProvider` envisagées jusqu'ici (source_object, durée de vie,
stratégie d'écriture — les trois déjà testées, plus celle-ci). Ajout
de `print(..., flush=True)` dans les quatre méthodes (`do_ref_formats`,
`do_get_value`, `do_write_mime_type_async`, `do_write_mime_type_finish`)
pour vérifier ce qui s'exécute réellement avant l'assertion :

```
>>> do_write_mime_type_async CALLED image/png
>>> write_bytes OK, result set
>>> complete_in_idle() called
>>> do_write_mime_type_finish CALLED
[... puis, seulement ensuite ...]
GLib-GIO-CRITICAL: g_task_return_boolean: assertion 'G_IS_TASK (task)' failed
Warning: g_object_unref: assertion 'G_IS_OBJECT (object)' failed
```

Les quatre méthodes de notre `ContentProvider` s'exécutent donc
**intégralement et sans erreur**, dans l'ordre attendu, y compris
`do_write_mime_type_finish` qui retourne bien `True`
(`get_op_res_gboolean()` ne lève rien). L'assertion `G_IS_TASK`
apparaît **après**, côté GDK — nécessairement dans le code C interne
de GDK/GTK4 qui a appelé notre `write_mime_type_async`/`_finish` pour
le compte d'une requête de sélection X11 externe, et qui enveloppe
apparemment cet appel dans son propre `GTask` interne (probablement
dans le backend X11 du presse-papier, `gdkclipboard-x11.c` côté
sources GTK — non consulté, `gitlab.gnome.org` toujours hors de
portée réseau ici). Point notable additionnel : lors du tout premier
essai de cette session (variante `source_object=None`), le message
observé était légèrement différent —
`gdk_content_provider_write_mime_type_finish: assertion
'GDK_IS_CONTENT_PROVIDER (provider)' failed` en plus du `G_IS_TASK` —
cohérent avec l'idée que c'est bien un objet interne à GDK (pas notre
`provider` Python, vivant et valide tout du long d'après les prints)
qui se retrouve invalide au moment où GDK termine sa propre requête.

**Conclusion révisée** : le bug ne se situe très probablement **pas**
dans `gcm_gtk4_clipboard_bridge.py` ni dans la façon dont ce projet
sous-classe `Gdk.ContentProvider` en PyGObject — c'est la troisième
API d'écriture testée (après `Gio.Task` nu, puis avec
`source_object=None`, puis `Gio.SimpleAsyncResult`) à produire
exactement la même signature d'échec malgré des mécanismes internes
disjoints, ce qui pointe vers un bug ou une limitation du backend X11
du presse-papier de ce GTK4 4.14.5 (`libgtk-4-1` Ubuntu
`4.14.5+ds-0ubuntu0.10`) lui-même plutôt que du code appelant. Ceci
dépasse ce que ce projet peut corriger : la seule suite possible
serait de reproduire avec un GTK4 plus récent (non disponible via
`apt` sur Ubuntu 24.04 sans backport) ou de consulter le source GDK
directement pour confirmer l'hypothèse `gdkclipboard-x11.c`.

**Fichiers modifiés cette session** : uniquement la raison `xfail` de
`test_clipboard_image_custom_content_provider_write_bug` dans
`tests/test_gtk4_live.py` (pour refléter ce diagnostic affiné) et
`features.md` — aucun changement de comportement, le test reste
`xfail` (`strict=False`). Scripts d'expérimentation
(`/home/claude/work/experiment_source_object.py`,
`/home/claude/work/experiment_simpleasyncresult.py`) non versionnés,
même logique que les `diag_*.py` de la session du 2026-09-02.
