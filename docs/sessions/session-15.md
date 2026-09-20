[← index](README.md) — session 15 — 2026-09-06

## Segfault GTK4 live diagnostiqué et contourné (2026-09-06, nouvelle session)

Tâche choisie dans le backlog ⚠️ Haute : la session précédente (même
jour) avait signalé un « segfault reproductible dès `RdpView.__init__()`
(`Gtk.Picture.set_content_fit()` ou à proximité) », non creusé
davantage, classé comme limitation d'environnement. Cette session avait
accès réseau `apt` (`archive.ubuntu.com`/`security.ubuntu.com`), donc le
diagnostic complet était possible.

### Mise en place

```
apt-get install -y gir1.2-gtk-4.0 xvfb xclip x11-apps imagemagick python3-gi
pip install --break-system-packages pytest
```

`gir1.2-gtk-4.0` installé : `4.14.5+ds-0ubuntu0.10` — même version que
la session précédente, donc pas un problème de version différente
entre deux sessions.

**Piège rencontré et corrigé en cours de route** : Xvfb démarré via
`(cd ... && setsid Xvfb ... &)` dans un seul appel outil composé — le
`cd` se retrouvait dans le sous-shell arrière-plané, donc le `cd` ne
s'appliquait jamais au shell principal qui exécutait la suite de la
commande (`pytest` cherchait alors les fichiers depuis le mauvais
répertoire). Corrigé en isolant le démarrage de Xvfb dans son propre
bloc (`nohup Xvfb ... > log 2>&1 < /dev/null &`), suivi d'une vérification
`ps aux | grep "Xvfb :99"` avant toute étape suivante.

### Première reproduction, et une fausse piste initiale

Une première tentative de lancer `pytest tests/test_gtk4_live.py`
directement a fait planter tout l'appel outil (code retour -1, aucune
sortie, pas même les commandes avant le `pytest` dans le même appel) —
signe d'un crash suffisamment violent pour perturber l'appel entier, pas
seulement le sous-processus Python. Reproduit une seconde fois à
l'identique avec une commande encore plus simple (démarrage de Xvfb
seul). Après isolement des commandes une par une (`ps aux`, `Xvfb` en
premier plan avec `timeout`), il est apparu que Xvfb lui-même
fonctionnait très bien isolément (`timeout 5 Xvfb :99 ...` termine
proprement au bout de 5 s, code 0).

**Première anomalie trouvée, distincte du vrai bug** : `Gdk.Display.
get_default()` renvoie `None` après un simple `Gtk.init()`, et
`Gtk.Window()` lève une `RuntimeError: Gtk couldn't be initialized`
même quand un appel manuel à `Gtk.init_check()` juste avant renvoie
`True` dans le même process. Cause identifiée par lecture du code
source de l'override PyGObject
(`/usr/lib/python3/dist-packages/gi/overrides/Gtk.py`) : la classe
`Window` (override) vérifie un drapeau **module-level** `initialized`,
calculé une seule fois à l'import du module
(`initialized = Gtk.init_check()`, ligne ~1702) — un appel ultérieur à
`Gtk.init_check()` par le code utilisateur renvoie bien une vraie
valeur à jour, mais ne remet jamais à jour ce drapeau déjà figé. Ceci
n'est PAS le bug principal (contourné pour l'investigation en
repatchant `gi.overrides.Gtk.initialized = True` directement), mais un
piège réel à documenter : ce contournement masque l'erreur propre sans
réparer la vraie cause, et `Gtk.Widget`/`Gtk.Picture` (contrairement à
`Gtk.Window`/`Gtk.Dialog`) ne sont de toute façon pas protégés par ce
garde-fou — ils foncent directement dans le constructeur natif même si
l'initialisation réelle a échoué.

### Isolation du vrai crash avec gdb

Avec le drapeau contourné, `Gtk.Window()` segfaultait toujours
(`Segmentation fault`, code 139). Script minimal isolant précisément le
point de crash (impressions avec `flush=True` après chaque étape,
piège initial : sortie Python bufferisée en pipe, donc `python3 -u` ou
`flush=True` indispensables pour voir où ça plante avant le crash) :

```python
import gi

gi.require_version("Gtk", "4.0")
from gi.repository import Gtk
import gi.overrides.Gtk as GtkOverrides

GtkOverrides.initialized = True
win = Gtk.Window()  # <- crash ICI, avant même le premier print après
```

Le crash a lieu dès `Gtk.Window()`, **pas** à
`Gtk.Picture.set_content_fit()` comme le formulait (avec réserve,
« ou à proximité ») la session précédente. Trace obtenue via :

```
gdb -batch -ex run -ex bt -ex "thread apply all bt" --args python3 crash_repro.py
```

```
Thread 1 "python3" received signal SIGSEGV, Segmentation fault.
0x00007ffff62e4a85 in ??? () from /lib/x86_64-linux-gnu/libgtk-4.so.1
#0  ??? () at libgtk-4.so.1
#1  ??? () at libgtk-4.so.1
#2  ??? () at libgtk-4.so.1
#3  ??? () at libgtk-4.so.1
#4  ??? () at libgtk-4.so.1
#5  g_type_create_instance () at libgobject-2.0.so.0
#6  ??? () at libgobject-2.0.so.0
#7  g_object_new_with_properties () at libgobject-2.0.so.0
#8  g_object_new () at libgobject-2.0.so.0
#9  ??? () at libgtk-4.so.1                    <- un 2e g_object_new imbriqué,
#10 g_type_create_instance () at libgobject-2.0.so.0   depuis l'intérieur du
#11 ??? () at libgobject-2.0.so.0                      constructeur de Window
#12 g_object_new_with_properties ()
#13 g_object_new ()
#14 ??? () at gi/_gi.cpython-312-x86_64-linux-gnu.so
... (pile Python normale jusqu'à Py_BytesMain)
```

Instruction fautive (`x/3i $pc`) : `mov (%r12),%rdi` juste avant un
`call g_type_interface_peek@plt` — déréférencement d'un pointeur
nul/invalide passé en premier argument à `g_type_interface_peek`.
Ubuntu ne fournit pas de symboles de debug pour `libgtk-4` sur les
dépôts accessibles depuis ce sandbox (`ddebs.ubuntu.com` hors liste
d'autorisation réseau), donc impossible d'aller plus loin que
« quel objet/quelle fonction précise », mais la pile (deux
`g_object_new` imbriqués — la construction de `Gtk.Window` construit
elle-même un autre `GObject` en interne, probablement un composant
interne du style/du thème/de l'icon theme) est cohérente avec un
sous-système non initialisé correctement à ce stade.

Un deuxième thread tournait en parallèle au moment du crash, en train
de parser la configuration fontconfig (`FcInit` → `libexpat`) — piste
envisagée (course avec l'initialisation asynchrone de fontconfig) puis
**écartée par test** : `fc-cache -f` en préchauffage avant de relancer
ne change rien, le crash reste identique.

### Cause confirmée : échec du rendu GL accéléré sous Xvfb

Test décisif : le contournement `LIBGL_ALWAYS_SOFTWARE=1` (déjà
mentionné, sans succès, par la session précédente — mais celle-ci
n'avait apparemment pas dépassé l'erreur `initialized=False` pour
l'essayer en conditions identiques) **évite le crash** :

```
DISPLAY=:99 GDK_BACKEND=x11 LIBGL_ALWAYS_SOFTWARE=1 python3 crash_repro.py
# -> "window ok", code 0, Xvfb toujours vivant après coup
```

Testé également `GSK_RENDERER=cairo` seul : fonctionne aussi, mais avec
un avertissement explicite qui confirme le diagnostic —
`libEGL warning: DRI3 error: Could not get DRI3 device`. Les deux
variables fonctionnent indépendamment ; `LIBGL_ALWAYS_SOFTWARE=1` seul
est le contournement retenu (plus propre, aucun avertissement resituel).
Cause donc confirmée : dans ce conteneur, Xvfb ne fournit pas
d'extension DRI3 exploitable, et le renderer GL par défaut de GSK4
(passage par EGL) plante au lieu d'échouer proprement en repli logiciel
— contrairement à ce qu'on pourrait attendre d'un échec « propre » de
négociation GL.

**Effet de bord noté** : quand le crash se produit (sans le
contournement), **Xvfb meurt aussi**, pas seulement le processus
Python — `ps aux | grep Xvfb` ne montre plus rien après. Donc chaque
essai raté nécessite de relancer Xvfb avant de réessayer ; oublié une
fois pendant l'investigation, ce qui a causé une fausse piste
temporaire (« `Gdk.Display.open` échoue même via le bon chemin
`Gtk.Application.run()` ») qui s'est révélée être un Xvfb déjà mort par
un essai précédent, pas un nouveau bug — reconfirmé sain en revérifiant
d'abord avec un vrai client X11 externe (`xclock`, qui tourne sans
erreur jusqu'au `timeout`) avant de rejouer le scénario GTK4.

### Validation complète avec le contournement

Binding cffi compilé pour de vrai cette session (`freerdp3-dev` et
`libwinpr3-dev` 3.31.0, installables via `apt` — le nom exact du paquet
est `freerdp3-dev`, pas `libfreerdp3-dev` qui n'existe pas sous ce nom ;
`pip install --break-system-packages -e .` compile sans erreur), plus
`loguru`/`numpy` (dépendances runtime des ponts GTK4, absentes au
départ). Avec `LIBGL_ALWAYS_SOFTWARE=1` :

```
DISPLAY=:99 GDK_BACKEND=x11 LIBGL_ALWAYS_SOFTWARE=1 ASYNCRDP_TEST_GTK4=1 \
  pytest tests/test_gtk4_live.py -v
```

→ **4 passed, 2 xfailed** (les 2 xfailed sont le bug connu, sans lien,
de mime-type presse-papier image — voir plus haut dans ce fichier) :
rendu écran réel (`test_display_bridge_renders_correct_colors_onscreen`),
presse-papier texte bidirectionnel via `xclip` externe, décodage DIB
indexé avec le vrai `GdkPixbuf`, presse-papier fichiers sens local→RDP
via `xclip -t text/uri-list` externe. Xvfb toujours vivant après coup.

Suite complète (`pytest tests/`, binding réel chargé) : **85 passed,
5 skipped (tests `test_integration_live.py`, nécessitent un vrai
serveur RDP non monté cette session), 2 xfailed, 0 failed** — aucune
régression, y compris sur les 24 tests de traçabilité et le test
anti-cycle ajoutés la session précédente.

### Ce que ceci ne couvre toujours pas

- La cause profonde exacte du défaut DRI3 dans ce conteneur précis
  (configuration Mesa ? absence de `/dev/dri/*` exploitable pour Xvfb ?
  autre chose) n'a pas été creusée plus loin que « DRI3 échoue,
  contournable en forçant le logiciel » — hors périmètre choisi pour
  cette tâche (le but était de débloquer les tests, pas d'auditer la
  pile graphique Mesa/DRI de ce conteneur).
- Le pont fichiers RDP→local (téléchargement asynchrone,
  `_download_and_set_files`) reste non couvert en conditions réelles
  GTK4 — seule la logique pure l'est (`tests/test_gtk4_clipboard_file_download.py`,
  session du 2026-09-05). L'environnement est désormais opérationnel
  pour l'écrire, mais ce n'était pas la tâche choisie cette session.
- Le bug de mime-type presse-papier image (`xfail`, section dédiée
  plus haut dans ce fichier) reste ouvert, sans lien avec ce segfault.

### Fichiers modifiés cette session

- `features.md` et ce fichier uniquement.
- Aucun fichier de code de production (`src/`, `integrations/`)
  modifié — le contournement (`LIBGL_ALWAYS_SOFTWARE=1`) est une
  variable d'environnement à positionner autour des invocations de
  `pytest tests/test_gtk4_live.py`, pas un changement de code ; il n'y
  a pas de fichier `conftest.py`/CI dans ce projet qui fixe déjà des
  variables d'environnement pour ces tests, donc rien à y modifier non
  plus dans le périmètre choisi.
