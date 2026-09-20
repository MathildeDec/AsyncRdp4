[← index](README.md) — session 19 — 2026-09-07

## Première session sur un vrai poste physique (2026-09-07, session suivante) — USB réel, écran réel, imprimante/série inventoriés

Contexte différent de toutes les sessions précédentes : celles-ci
tournaient dans des sandbox jetables sans aucun matériel réel (USB,
écran physique, imprimante, port série). Cette session-ci tourne
directement sur un vrai poste (`mathilde-HP-470-G7-Notebook-PC`, Ubuntu,
noyau 6.14, session GNOME/Wayland réelle avec XWayland) — l'occasion de
retester tout ce qui était marqué « non confirmé faute de matériel ».

**Inventaire matériel réel fait en premier** (`lsusb`, `lpinfo -v`,
`ls /dev/tty*`, `dpkg -l`) :
- USB : 8 périphériques réels (souris/clavier filaires Dell, récepteur
  sans-fil Logitech, webcam intégrée Foxlink, contrôleur Bluetooth Intel,
  deux hubs internes) — aucun périphérique « exotique » type
  imprimante/carte à puce USB.
- Imprimante : aucune file CUPS locale configurée, mais une vraie
  imprimante réseau détectable par `lpinfo -v` (`Brother MFC-L3770CDW`,
  découverte mDNS/IPP) — pas de gain possible sans un vrai serveur RDP
  Windows pour tester la redirection (xrdp reste structurellement
  bloqué, déjà diagnostiqué à fond une session précédente).
- Port série : uniquement `/dev/ttyS0`..`ttyS19`, les ports COM hérités
  standards émulés par le BIOS/ACPI sur ce type de portable (pas de vrai
  périphérique raccordé) — aucun `/dev/ttyUSB*`/`ttyACM*` réel. Même
  conclusion que l'imprimante : bloqué côté xrdp, pas d'adaptateur USB-
  série réel disponible de toute façon.
- Écran : session Wayland réelle (`DISPLAY=:0` via XWayland,
  `WAYLAND_DISPLAY=wayland-0`), GTK 4.18.5 avec le jeu complet de
  loaders gdk-pixbuf d'un vrai bureau (avif/gif/heif/jxl/tiff/webp/svg
  en plus des habituels png/jpeg/bmp compilés en dur) — bien plus riche
  que les sandbox minimalistes des sessions précédentes.
- `freerdp3-dev` 3.14.0 et `gir1.2-gtk-4.0` 4.18.5 déjà installés
  système — binding cffi recompilé sans erreur dans un venv dédié
  (`python3 -m venv --system-site-packages /tmp/asyncrdp-venv`, pour
  garder accès à `PyGObject` système sans le réinstaller par pip —
  celui-ci nécessite des libs de dev GTK4 supplémentaires pour compiler
  depuis pip, alors que le paquet système fonctionne directement).

**Point de méthode important** : pas d'accès `sudo` non interactif sur
cette machine (mot de passe requis) — impossible d'installer `xrdp`/
`xclip`/`imagemagick` dans cette session sans interrompre le flux pour
demander un mot de passe (à éviter, cf. consignes de sécurité). D'où
deux limites assumées ci-dessous : pas de nouveau test de connexion RDP
bout en bout, et un script de validation GTK4 ad hoc écrit à la main
(`/tmp/gtk4_real_display_test.py`, pas livré dans le paquet) plutôt que
la suite `tests/test_gtk4_live.py` existante, qui dépend de `xclip`/
`convert` (imagemagick) absents ici.

### USB — vrai bug trouvé sur du vrai matériel (`list_usb_devices()`)

`list_usb_devices()` plantait sur **chaque** périphérique réel avec
`ValueError: The device has no langid (permission issue, no string
descriptors supported or device error)`, levée par
`usb.util.get_string()` — jamais vue avant puisque toutes les sessions
précédentes n'avaient qu'une liste vide à énumérer (aucune exception ne
peut se produire dans une boucle qui ne s'exécute jamais). Cause :
lecture des descripteurs de chaîne USB (fabricant/produit/série) sans
droits udev sur `/dev/bus/usb/<bus>/<device>` en utilisateur non-root —
cas *normal*, pas une anomalie de cette machine précise. Le `except`
existant ne couvrait que `usb.core.USBError`/`NotImplementedError` ;
pyusb lève en réalité `ValueError` dans ce cas précis. Corrigé en
ajoutant `ValueError` au tuple d'exceptions capturées dans
`list_usb_devices()` (`src/asyncrdp/usb.py`).

Une fois corrigé : les 8 périphériques réels apparaissent dans le
résultat (VID:PID vérifiés identiques à `lsusb`, ex. `413c:301a` =
souris Dell, `413c:2113` = clavier Dell), `manufacturer`/`product` à
`None` comme attendu sans droits udev, `usb_device_args()` produit la
bonne chaîne `id,dev:VVVV:PPPP` pour chacun. `tests/test_usb.py` (9
tests, logique pure/mockée) repasse sans changement nécessaire — le bug
ne pouvait être détecté que par du vrai matériel, pas par des mocks.

### GTK4 — rendu sur le vrai écran, sans le contournement Xvfb

Script ad hoc (`RdpView._show_frame()` + fenêtre `Gtk.ApplicationWindow`
non décorée 200x200, capture `xwd -name <titre>` plutôt que `-root` pour
ne pas avoir à localiser la fenêtre sur le bureau, fichier XWD parsé à
la main avec `struct` — ni `xclip` ni `convert` disponibles sans sudo).

Résultat : **aucun segfault**, contrairement au comportement documenté
sous Xvfb sans `LIBGL_ALWAYS_SOFTWARE=1` (voir plus haut, « Segfault
GTK4 live diagnostiqué et contourné ») — confirme que ce contournement
était bien spécifique à l'absence de rendu GL accéléré exploitable sous
Xvfb (DRI3/EGL), pas une exigence générale du code. Les 4 couleurs
attendues (rouge/vert/bleu/jaune) apparaissent dans les 4 quadrants de
la capture d'écran réelle, à quelques valeurs près (ex. 252 au lieu de
255) dues au lissage bilinéaire de la mise à l'échelle 2x2 → 200x200,
sans rapport avec le code testé.

### Bug du mime-type presse-papier image — hypothèse « sandbox minimal » définitivement écartée

Retesté directement (`Gdk.ContentProvider.new_for_value(texture).
ref_formats().get_mime_types()`) sur ce vrai bureau complet, avec un
jeu de loaders gdk-pixbuf bien plus riche que les sandbox précédents
(avif/heif/jxl/webp/tiff/svg en plus de png/jpeg/bmp) : **liste toujours
vide**, résultat rigoureusement identique à celui documenté depuis le
2026-09-02. Ceci ferme définitivement l'hypothèse déjà affaiblie
« paquet de loaders incomplet dans un sandbox minimal » — le
comportement est intrinsèque à cette version de GTK4/PyGObject, pas à
la pauvreté de l'environnement de test. Piste annexe vérifiée au
passage : `Gdk.Clipboard.set_texture()` (convenience API qui aurait pu
emprunter un chemin de sérialisation différent) n'existe même pas dans
l'introspection GObject de ce binding (`AttributeError:
'GdkX11Clipboard' object has no attribute 'set_texture'`) — donc pas
une alternative disponible ici. Aucun changement de code dans
`gcm_gtk4_clipboard_bridge.py` (le bug reste hors de portée de ce
projet, cf. diagnostic déjà détaillé le 2026-09-02/2026-09-07 plus haut).

### Bug de packaging trouvé et corrigé : `numpy` manquant pour l'extra `gtk4`

En import (via le mécanisme de stub de `tests/test_gtk4_bridges.py`,
mais le symptôme est général) : `gcm_gtk4_clipboard_bridge.py` fait
`import numpy as np` au niveau module, sans que `numpy` soit déclaré
nulle part — ni dans l'extra `gtk4` de `pyproject.toml`, ni dans
`install.sh`. Reproduit concrètement (`ModuleNotFoundError: No module
named 'numpy'`) en lançant la suite de tests dans un venv neuf n'ayant
que les dépendances déclarées. Quiconque suit `pip install
asyncrdp[gtk4]` ou `./install.sh --gtk4` à la lettre aurait heurté ce
même échec dès l'import du pont clipboard. Corrigé : `numpy` ajouté à
l'extra `gtk4` dans `pyproject.toml` (pas de changement nécessaire côté
`install.sh`, qui délègue déjà tous les extras pip à `pip install
-e ".${EXTRAS}"`). Vérifié en désinstallant `numpy` puis en réinstallant
uniquement via `pip install -e ".[gtk4]"` : `import numpy` fonctionne
de nouveau. Suite complète (`pytest tests/`) : 91 passed, 12 skipped
(intégration/GTK4-live nécessitant respectivement un serveur RDP et
Xvfb+xclip+imagemagick, non disponibles/pas requis dans cette session).

### Fichiers modifiés cette session

- `src/asyncrdp/usb.py` : `ValueError` ajouté aux exceptions capturées
  dans `list_usb_devices()` (bug réel trouvé sur du vrai matériel USB).
- `pyproject.toml` : `numpy` ajouté à l'extra `gtk4` (dépendance
  manquante, bug de packaging réel).
- `features.md` et ce fichier.
- Non modifié faute d'environnement adapté dans cette session précise
  (pas de `sudo` non interactif) : aucun nouveau serveur RDP monté,
  donc pas de nouveau test de redirection USB/imprimante/série bout en
  bout contre un vrai serveur — seule l'énumération côté client a pu
  être avancée cette fois.

## Addendum même jour (2026-09-07) — un vrai second serveur RDP externe signalé, testé en conditions réelles

L'utilisateur a signalé en cours de session un second serveur RDP
réellement joignable sur le LAN : `192.168.105.141:3389` (déjà vu
passivement plus tôt dans la découverte CUPS de la même session, sous
la forme `network socket://192.168.105.134` — adresse voisine mais PAS
identique, donc probablement une machine différente du même sous-réseau
plutôt que la même redécouverte deux fois). Connectivité confirmée
(`ping`, `nc -zv ... 3389`) avant tout test.

**Fausse piste corrigée rapidement, à ne pas répéter** : les tout
premiers logs FreeRDP contre ce serveur (canal dynamique `ainput`
chargé, erreur de négociation de licence `BB_ERROR_BLOB`) ont d'abord
fait conclure à tort à un **vrai serveur Windows** — ces deux signaux
sont pourtant couramment associés à des hôtes RDS Windows dans
l'expérience passée de ce projet. **Invalidé par une preuve visuelle
directe** : une frame réelle capturée après connexion (`client.
get_frame()` → reconstruction PNG via Pillow avec la bonne prise en
compte du stride, `Image.frombuffer("RGBA", (w,h), raw, "raw",
("BGRA", stride, 1))`) montre sans ambiguïté un bureau **Linux LXQt**
(barre des tâches, menu « Applications », terminal `xfce4-terminal`-
like) — le prompt shell affiche même l'hostname `d98f3259d39f`,
clairement un identifiant de conteneur Docker généré automatiquement,
pas un vrai poste Windows. Leçon retenue : `ainput`/erreurs de licence
ne sont PAS des signaux fiables à eux seuls pour distinguer Windows
d'un autre serveur RDP Linux qui les négocie de façon similaire — seule
une preuve visuelle (ou une commande shell distante) tranche vraiment.

**Connexion de base** : succès complet, comme le tout premier test
contre l'ancien sandbox xrdp de ce projet — 5 frames reçues à la
résolution demandée (1280x800), déconnexion propre. Bonne nouvelle de
robustesse : le binding cffi recompilé sur cette machine (FreeRDP
3.14.0, plus ancien que les 3.30/3.31 des sessions précédentes) parle
correctement à un troisième serveur RDP indépendant, jamais testé avant
ce jour.

**USB (webcam réelle, VID:PID 05c8:03d2, sélectionnée précisément pour
NE JAMAIS toucher souris/clavier/Bluetooth réels utilisés pour
contrôler cette machine)** : le canal `urbdrc` se charge, le client
tente bien la redirection (`asyncrdp_add_usb_device` avec
`usb_device_args()`), mais échoue proprement côté libusb :
`LIBUSB_ERROR_ACCESS`/`LIBUSB_ERROR_IO`, `« Could not find or redirect
any usb devices by id 05c8:03d2 »`. Cohérent avec le bug déjà trouvé
plus tôt cette même session (`list_usb_devices()` : lecture des
descripteurs de chaîne déjà bloquée sans droits udev) — la redirection
USB effective, une étape plus loin (ouverture exclusive du device),
nécessite les mêmes droits et échoue pour la même raison structurelle.
Confirme que cette limitation est bien celle de CET environnement
(droits udev), pas du code, et rassure sur un point important : sans
ces droits, `urbdrc` échoue à *ouvrir* le périphérique plutôt que de le
détacher silencieusement d'un mauvais device — aucun risque constaté
de « voler » par erreur un périphérique d'entrée critique dans cet état
de permissions.

**Imprimante et port série, avec confirmation SERVEUR cette fois (pas
seulement les logs client)** : le client annonce toujours correctement
les deux (`device_announce: registered [serial] device #1: COM3`,
`registered [printer] device #2: PRN1`), comme dans toutes les
sessions précédentes contre l'ancien xrdp. Nouveauté méthodologique
cette session : plutôt que de s'arrêter aux seuls logs FreeRDP côté
client (qui ne prouvent que l'ANNONCE, jamais la réponse serveur),
utilisation du clavier/souris déjà fonctionnels du binding pour piloter
le VRAI bureau distant et ouvrir un VRAI terminal dedans (clic sur
l'icône de la barre des tâches, capture d'écran pour repérer les
coordonnées, un `\n` dans `Keyboard.write()` ne suffit PAS à valider
une ligne de commande — il faut un `key_press("return")` séparé,
piège noté pour la suite) puis exécution de `lpstat -p` et `ls -la
/dev/ttyS0` **directement dans la session serveur**. Résultat, capturé
en image (voir `/tmp/rdp_after_enter.png`, non conservé dans le
paquet) : `bash: lpstat: command not found` (CUPS n'est même pas
installé sur ce serveur — conteneur minimal) et `ls: cannot access
'/dev/ttyS0': No such file or directory`. Conclusion honnête : ce
troisième environnement de test ne permet pas plus que les précédents
de valider une redirection effective — pour une raison différente
cette fois (absence de CUPS plutôt qu'un refus explicite comme
documenté pour xrdp), mais le résultat pratique reste identique :
aucun environnement de test rencontré à ce jour par ce projet
n'implémente réellement ces deux redirections côté serveur. Ceci
n'invalide PAS le diagnostic définitif déjà fait à la lecture du code
source de xrdp (toujours la preuve la plus rigoureuse dont on dispose
sur le « pourquoi ») — ça ajoute simplement un point de données
indépendant, cohérent, sur un serveur différent.

**Point resté ouvert** : la nature exacte du serveur RDP tournant dans
ce conteneur (`xrdp` avec un DE LXQt au lieu d'openbox ? un autre
serveur RDP Linux ? ) n'a pas été déterminée précisément — aurait
nécessité une commande serveur supplémentaire (`ps aux`, `dpkg -l`)
non exécutée faute de temps dans cette session. Sans conséquence sur
les conclusions ci-dessus (déjà tranchées par l'absence de CUPS et de
device série, quel que soit le serveur exact).

### Fichiers modifiés dans cet addendum

Aucun — cette partie de la session n'a produit que des observations
(scripts de test ad hoc dans `/tmp/`, non livrés dans le paquet, cf.
figure ci-dessus). `features.md` mis à jour en conséquence (voir plus
bas).
