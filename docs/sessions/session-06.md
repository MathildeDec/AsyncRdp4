[← index](README.md) — session 06 — 2026-09-02

## Tentative de correctif du bug mime-type presse-papier image (2026-09-02, nouvelle session)

Même sandbox que la section précédente (le conteneur avait cette fois
survécu entre les deux tours — GTK4/pytest/xclip/Xvfb déjà installés,
seul `Xvfb :99` avait besoin d'être relancé). Tâche choisie dans le
backlog moyenne-priorité de `features.md` : la piste « non vérifiée »
laissée en suspens la fois précédente — implémenter un
`GdkContentProvider` custom à la Vim plutôt que de compter sur la
sérialisation automatique de GTK4, pour voir si ça règle
`test_clipboard_image_available_to_external_reader`.

**Prototype** (`diag_custom_provider.py`, hors suite pytest) :

```python
class PngContentProvider(Gdk.ContentProvider):
    def __init__(self, texture):
        super().__init__()
        self._texture = texture
        self._png_bytes = texture.save_to_png_bytes()  # existe depuis GTK 4.6

    def do_ref_formats(self):
        return Gdk.ContentFormats.new(["image/png"])

    def do_get_value(self, gvalue):
        gvalue.set_object(self._texture)
        return True

    def do_write_mime_type_async(self, mime_type, stream, io_priority, cancellable, callback, user_data):
        task = Gio.Task.new(self, cancellable, callback, user_data)
        ...  # ecrit self._png_bytes dans stream, puis task.return_boolean(True)

    def do_write_mime_type_finish(self, result):
        return result.propagate_boolean()
```

**Bonne nouvelle, vérifiée directement (sans passer par xclip)** :
`provider.ref_formats().get_mime_types()` → `['image/png']`, et après
`clipboard.set_content(provider)`, `clipboard.get_formats().
get_mime_types()` → `['image/png']` aussi. C'est un changement net par
rapport à la session précédente (`[]` dans tous les cas testés) — la
première moitié du diagnostic de la fois précédente tenait donc bien :
le registre de sérialisation GType→mime-type automatique de GTK4 est
la cause de cette première partie du problème, et un `ContentProvider`
qui fait sa propre sérialisation la contourne complètement.

**Mauvaise nouvelle : la livraison des octets à un lecteur externe
échoue quand même**, mais pour une raison différente et nouvelle. Round-trip
complet (`diag_custom_provider_roundtrip.py`, `xclip -o -t image/png`
externe) → 0 octet reçu, `TIMEOUT`. Log :

```
GLib-GIO-CRITICAL: g_task_return_boolean: assertion 'G_IS_TASK (task)' failed
Warning: g_object_unref: assertion 'G_IS_OBJECT (object)' failed
```

**Élimination méthodique de trois causes possibles**, chacune testée
séparément (script dédié, log détaillé avec prints à chaque étape) :

1. *Réentrance de l'écriture asynchrone imbriquée*
   (`diag_custom_provider_debug.py`) : logs montrent que
   `write_bytes_async` → `write_bytes_finish` réussit bel et bien
   (« 82 octets écrits »), et que `do_write_mime_type_finish` est
   rappelé en réentrant *avant même* que l'appel Python à
   `_task.return_boolean(True)` ait fini de s'exécuter côté C — la
   warning apparaît épinglée sur cette ligne précise dans la trace.
   Hypothèse : l'imbrication d'un `write_bytes_async` dans le
   `GTask` du provider crée une réentrance.
2. *Test en écarquillant la variable* : écriture 100% synchrone
   (`stream.write_bytes(...)`, pas de `write_bytes_async` imbriqué du
   tout) dans `do_write_mime_type_async` avant `task.return_boolean(True)`
   (`diag_custom_provider_sync.py`) — **même échec, à l'identique**.
   Élimine l'hypothèse n°1 : ce n'est pas la réentrance de l'écriture
   imbriquée, l'écriture elle-même (async ou sync) n'est pas en cause.
3. *Durée de vie / ramasse-miettes Python prématuré* : les docs
   GTask elles-mêmes signalent un « thread-safety concern... if the
   main thread drops its last reference to the source object ».
   Test avec une référence Python forte explicite maintenue sur
   `(self, task, stream)` dans un dict module-level pendant toute
   l'opération, plus le `provider` lui-même gardé vivant séparément
   (`diag_task_lifetime.py`) — **même échec, à l'identique**. Élimine
   aussi cette hypothèse.

**Conclusion à ce stade** : ni la stratégie d'écriture (async
imbriqué vs. synchrone) ni une hypothèse de durée de vie côté Python
n'expliquent l'échec — les trois variantes produisent l'*exact même*
message d'assertion. Reste une hypothèse non vérifiée, plus difficile
à trancher sans lire le code C de GTK4/GIO lui-même (toujours hors de
portée réseau ici, `gitlab.gnome.org` non accessible) : une limitation
ou un bug du marshaling PyGObject pour le couple
`Gio.Task.new()`/`task.return_boolean()` spécifiquement quand
`source_object` (`self`, ici) est une sous-classe **Python** d'un
type GObject abstrait (`Gdk.ContentProvider`) plutôt qu'un GObject
natif ou une sous-classe C — un chemin de code potentiellement moins
testé en amont que le cas C-vers-C. Piste pour une session future :
essayer avec `Gio.SimpleAsyncResult` (API dépréciée mais peut-être
moins concernée par ce problème), ou chercher un exemple existant de
`GdkContentProvider` sous-classé en Python (PyGObject) qui fonctionne,
pour comparer précisément la marshaling attendue.

**Décision** : ne pas modifier `gcm_gtk4_clipboard_bridge.py` avec ce
correctif — il ne résout que la moitié du problème (l'annonce du
mime-type, pas la livraison réelle), et livrer un « correctif » qui ne
marche pas serait pire que de documenter honnêtement l'état
intermédiaire. Le prototype reste dans les scripts de diagnostic
(`/home/claude/work/diag_*.py`, non versionnés) et dans un nouveau
test `xfail` ajouté à la suite, pour qu'une session future dispose
d'un point de départ reproductible plutôt que de devoir tout
reconstruire : `test_clipboard_image_custom_content_provider_write_bug`
dans `tests/test_gtk4_live.py`, juste après
`test_clipboard_image_available_to_external_reader`. Vérifié
séparément avec `--runxfail` que l'échec se produit bien sur la
seconde assertion (les octets reçus), pas sur la première (les
mime-types annoncés, qui elle passe) — la raison `xfail` documente les
deux.

**Résultat final de la suite** : `pytest tests/test_gtk4_live.py -v`
(stub `asyncrdp` minimal, `ASYNCRDP_TEST_GTK4=1`, `DISPLAY=:99`,
`GDK_BACKEND=x11`) → **4 passed, 2 xfailed** (les 4 tests
préexistants qui passaient déjà, plus les 2 xfailed — l'ancien et le
nouveau).
