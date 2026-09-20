[← index](README.md) — session 16 — 2026-09-06

## Pont fichiers RDP -> local en conditions réelles GTK4 : gbulb intégré, vrai bug de dossier trouvé et corrigé (2026-09-06, nouvelle session)

Tâche choisie : le candidat explicitement identifié en fin de session
précédente (même jour, « Segfault GTK4 live diagnostiqué et
contourné ») — le pont fichiers RDP -> local
(`_download_and_set_files`), jusqu'ici couvert uniquement par de la
logique pure (`tests/test_gtk4_clipboard_file_download.py`,
2026-09-05), jamais exercé en conditions réelles GTK4 contrairement au
sens local -> RDP (couvert le 2026-09-02). Blocage déjà identifié à
deux reprises dans ce fichier : `_on_remote_files_changed` appelle un
`asyncio.ensure_future()` nu, qui ne s'exécute jamais sous un simple
`Gtk.Application.run(None)` (aucune boucle asyncio n'est «
en cours d'exécution » à ce moment-là) — il faut une vraie intégration
asyncio/GLib (gbulb, déjà envisagé mais jamais tenté).

### Environnement de cette session

Accès réseau `apt`/`pip` disponible. Sandbox non persistante pour les
process (confirmé une fois de plus : `Xvfb` de la session précédente
disparu, mais utilisateur/paquets/binding compilé sur disque
intacts) — juste relancé `Xvfb :99` comme d'habitude. Paquets `apt`
déjà tous présents depuis les sessions précédentes
(`gir1.2-gtk-4.0`/`freerdp3-dev`/`libwinpr3-dev`/`xclip`/etc.), donc
seul `pip install gbulb` était nouveau. `pip install -e ".[test]"`
(sans l'extra `gtk4` : `PyGObject` est déjà fourni par le paquet
système `python3-gi`, essayer de le réinstaller via pip échoue dans ce
sandbox faute de `girepository-2.0` pour Meson — non nécessaire de
toute façon, confirmé en importation directe).

### gbulb : trouver le bon usage AVANT de toucher au vrai test

Discipline habituelle de ce projet (vérifier un mécanisme isolément
avant de l'appliquer) : script jetable (`diag_gbulb_smoke2.py`, hors
suite pytest) avec juste une fenêtre GTK4 minimale et un
`asyncio.ensure_future()` déclenché depuis son callback `activate`.

**Premier essai, raté** : `gbulb.install(gtk=True)` puis appel direct
`app.run(sys.argv)` — la tâche asyncio ne s'exécute jamais (timeout de
l'outil, appel entier bloqué). Cause : `app.run()` appelé ainsi
pilote GLib directement, complètement en dehors de la boucle asyncio
que gbulb a mise en place — la loop gbulb créée par `get_event_loop()`
n'est jamais réellement exécutée (`run()`/`run_forever()` jamais
appelée dessus), donc aucun mécanisme asyncio n'est réellement actif
au moment du callback, quel que soit le contexte GLib par ailleurs
partagé.

**Bon usage, trouvé en lisant `glib_events.py`/`gtk.py` de gbulb
directement (pas de documentation en ligne consultée, aucune n'était
nécessaire)** : `GLibEventLoop.run_forever(application=app, argv=...)`
appelle en interne `self._application.run(self._argv)` **depuis
l'intérieur de son propre `run()`**, entre deux appels
`events._set_running_loop(self)`/`(None)` — c'est cette
séquence précise qui fait que `asyncio.ensure_future()` (qui cherche
la boucle *en cours d'exécution*) trouve bien la boucle gbulb pendant
tout le temps où `app.run()` tourne. Donc : ne jamais appeler
`app.run()` soi-même une fois gbulb en jeu — toujours passer par
`loop.run_forever(application=app)`. Découverte annexe : la variante
`gtk=True` de gbulb (`GtkEventLoopPolicy`/`GtkEventLoop`) s'appuie sur
`Gtk.main()`/`Gtk.main_quit()`, des fonctions **GTK3**, absentes de
GTK4 — sans conséquence tant qu'aucune récursion de boucle n'est
utilisée (`is_running()` reste `False` dans ce test, donc le chemin
`Gtk.main()` n'est jamais atteint), mais la policy simple
`gbulb.GLibEventLoopPolicy()` (sans `gtk=True`) suffit et évite
d'ailleurs ce risque : retenue pour le vrai test.

Une fois corrigé (`asyncio.set_event_loop_policy(gbulb.GLibEventLoopPolicy())`
puis `loop = asyncio.new_event_loop(); asyncio.set_event_loop(loop);
loop.run_forever(application=app, argv=[])`) : le smoke test passe,
`asyncio.ensure_future()` appelé depuis le callback GTK s'exécute bien
avant que l'app ne quitte. Politique de boucle asyncio restaurée dans
un `finally` (`asyncio.set_event_loop_policy(previous_policy)`) une
fois appliqué au vrai fichier de test, pour ne pas contaminer le reste
de la suite (même discipline que les substitutions `sys.modules`
ailleurs dans ce projet) — vérifié : suite complète (`pytest tests/`)
verte après, aucune fuite constatée sur les autres fichiers.

### Vrai bug trouvé en testant contre un vrai presse-papier GTK4 (pas supposé, pas deviné)

Script de diagnostic (`diag_rdp_to_local_files.py`) : sélection à
trois entrées annoncées côté serveur — `rapport.txt` (fichier de
premier niveau), `sous-dossier` (dossier de premier niveau),
`sous-dossier\notes.bin` (fichier imbriqué, plage d'octets 0-255
complète pour détecter une troncature). `ClipboardBridge` réel branché
sur un `RdpClient` factice (`types.SimpleNamespace`, même convention
que les autres tests de ce fichier), `request_remote_file_contents`
factice retournant les contenus attendus.

**Résultat inattendu** : les trois fichiers sont bien recréés sur
disque avec le bon contenu (`call_log` confirme que seuls les index 0
et 2 sont demandés au serveur, jamais le dossier) — mais un lecteur
externe (`xclip -o -t text/uri-list`) ne récupère qu'**une seule** URI
(`rapport.txt`), alors que deux entrées de premier niveau étaient
attendues (`rapport.txt` ET `sous-dossier`).

**Diagnostic, par lecture du code plutôt que supposition** : dans
`fetch_one` (`_download_and_set_files`), la branche `is_directory`
fait `dest.mkdir(...); return None`. Plus bas, le filtre de
`local_paths` est `if path is not None and "\\" not in info.name` —
`path is not None` exclut donc **systématiquement** tout dossier de
premier niveau du résultat final, quelle que soit sa profondeur
d'imbrication. Confirmé en relisant `tests/test_gtk4_clipboard_file_download.py::test_download_only_top_level_entries_reach_file_list` :
son commentaire d'origine rationalisait déjà ce comportement (« ni le
dossier (déjà recréé sur disque, pas un GFile "sélectionné") ») — une
justification a posteriori d'un comportement jamais vérifié en
conditions réelles, pas un choix de conception délibéré validé par un
usage réel. Concrètement, côté utilisateur : copier un dossier + un
fichier isolé depuis une session RDP puis coller localement ne faisait
apparaître QUE le fichier isolé — le dossier (et tout ce qu'il
contient) restait invisible, orphelin dans un répertoire temporaire
que rien ne signale à l'utilisateur. Un dossier vide de premier niveau
ne produisait même aucun dépôt presse-papier du tout (`set_content`
jamais appelé), silencieusement.

**Correctif** (`gcm_gtk4_clipboard_bridge.py`, `fetch_one`) : la
branche `is_directory` renvoie désormais `str(dest)` au lieu de `None`
— le dossier existe déjà réellement sur disque à ce stade (`mkdir`
juste avant), rien n'empêche de le proposer comme un `GFile` de plus,
exactement comme un fichier. Un dossier **imbriqué** (nom contenant
`\\`) reste correctement exclu du résultat final grâce au filtre déjà
en place — seul le comportement des dossiers de **premier niveau**
change.

**Tests pure logique mis à jour en conséquence**
(`tests/test_gtk4_clipboard_file_download.py`), avec vérification par
mutation (fix retiré temporairement, tests en échec confirmé, fix
restauré, tests de nouveau verts — même discipline que les autres
mutations manuelles de ce projet) :
- `test_download_only_top_level_entries_reach_file_list` : attend
  désormais 2 entrées de premier niveau (`dossier` ET `racine.txt`),
  pas 1 seule.
- `test_download_no_top_level_entries_skips_clipboard_entirely`
  renommé `test_download_empty_top_level_folder_is_still_pasted` et
  inversé : un dossier de premier niveau vide compte maintenant comme
  une entrée utilisable (`set_content` appelé), là où il ne l'était
  pas avant. Un nouveau test reprend l'ancien nom
  (`test_download_no_top_level_entries_skips_clipboard_entirely`) pour
  le seul cas réellement vide qui reste : une sélection sans aucune
  entrée annoncée (`files = []`).

### Nouveau test live ajouté : `tests/test_gtk4_live.py::test_clipboard_bridge_files_download_from_rdp_via_external_reader`

Rejoue exactement le scénario du script de diagnostic, cette fois dans
la suite pytest permanente, avec gbulb scopé au test (policy restaurée
en `finally`). Vérifie : (1) les indices demandés au serveur sont
exactement `{0, 2}`, jamais le dossier ; (2) le contenu sur disque des
deux fichiers (dont la plage d'octets 0-255 complète pour le fichier
imbriqué) ; (3) le lecteur externe `xclip -o -t text/uri-list` reçoit
exactement les deux bons chemins de premier niveau (`rapport.txt` et
`sous-dossier`) — **l'assertion qui aurait échoué avant le correctif**
(un seul chemin reçu). Confirme au passage un point qui restait
implicite jusqu'ici : contrairement au cas de l'image
(`Gdk.ContentProvider.new_for_value(texture)`, mime-types vides —
bug non résolu documenté plus haut), la sérialisation GType -> mime-type
automatique de GTK4 fonctionne correctement pour `Gdk.FileList` — le
problème rencontré ici était un vrai bug applicatif dans
`fetch_one`, pas une limitation de GTK4/GDK comme pour l'image.

**Résultat** : `pytest tests/test_gtk4_live.py -v` (stub non utilisé
ici — vrai binding compilé, `LIBGL_ALWAYS_SOFTWARE=1` requis comme
depuis la session précédente) → **5 passed, 2 xfailed** (les 2 xfailed
inchangés, bug image sans lien). Suite complète
(`pytest tests/ --ignore=tests/test_integration_live.py`) → **87
passed, 2 xfailed**, `Xvfb` toujours vivant après coup.

### Dépendances

`gbulb` ajouté comme extra pip dédié (`gtk4-test`, distinct de `gtk4`
qui ne concerne que `PyGObject`) dans `pyproject.toml` — utile
uniquement pour ce test précis, pas une dépendance des ponts GTK4
eux-mêmes ni du reste de la suite. Instructions de mise en place
mises à jour en tête de `tests/test_gtk4_live.py` (`pip install
gbulb`), avec un rappel du contournement `LIBGL_ALWAYS_SOFTWARE=1`
déjà documenté la session précédente (redécouvert nécessaire ici
aussi, comme attendu — c'est un trait de cet environnement Mesa/Xvfb,
pas quelque chose de spécifique à un fichier de test).

### Fichiers modifiés cette session

- `integrations/gtk4/gcm_gtk4_clipboard_bridge.py` : correctif d'une
  ligne dans `fetch_one` (`return str(dest)` au lieu de `return None`
  pour un dossier), commenté avec le diagnostic complet.
- `tests/test_gtk4_clipboard_file_download.py` : 2 tests mis à jour,
  1 nouveau test ajouté (voir détail ci-dessus), docstring d'en-tête
  mise à jour.
- `tests/test_gtk4_live.py` : nouveau test
  `test_clipboard_bridge_files_download_from_rdp_via_external_reader`,
  docstring du test symétrique (local -> RDP) et docstring d'en-tête
  de fichier mises à jour pour ne plus dire ce chemin non couvert.
- `pyproject.toml` : nouvel extra `gtk4-test = ["gbulb>=0.6.0"]`.
- `features.md` et ce fichier.
- Scripts de diagnostic (`diag_gbulb_smoke2.py`,
  `diag_rdp_to_local_files.py`, `diag_gbulb_smoke.py` — ce dernier
  gardé tel quel comme trace de l'usage INCORRECT de gbulb, pour
  mémoire) non versionnés, comme d'habitude.
