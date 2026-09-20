---
name: session-22
---

[← index](README.md)

# Session 22 (2026-09-10) — Règle udev USB + diagnostic de permissions ; rattrapage de documentation (README obsolète)

## Contexte

Reprise après la session 21 (nettoyage ruff, 2026-09-09). `CLAUDE.md`
indiquait qu'aucune feature de fond n'était en attente au-delà des
trois points bloqués par l'environnement (USB de bout en bout,
imprimante/série/parallèle, presse-papier image GTK4) — à décider avec
l'utilisateur avant de repartir sur du travail de fond. Consigne de
reprise pour cette session : continuer les features à faire, sans
attendre cette décision, en restant sur un seul point avant livraison.

Des trois points bloqués, deux sont fermés à du code : imprimante/
série/parallèle nécessite un serveur RDP structurellement différent de
xrdp (confirmé par lecture de son code source, session 09) ; le
presse-papier image GTK4 nécessite soit un GTK4 plus récent que 4.14.5,
soit une lecture des sources GDK elles-mêmes, toutes deux hors de
portée ici (déjà tenté en profondeur, sessions 05/06/17). Le troisième
— USB — était différent : le *blocage* (droits udev manquants sur
`/dev/bus/usb/...`) était bien identifié et documenté depuis le
2026-09-07, mais rien n'avait encore été livré côté projet pour le
lever. C'est une vraie feature actionnable sans infrastructure
supplémentaire, contrairement aux deux autres. Choix de cette session.

## Règle udev + diagnostic de permissions

**Vérification préalable, empirique plutôt que supposée** : avant
d'écrire la règle, inspection de `/usr/lib/udev/rules.d/70-uaccess.rules`
sur cette même image Ubuntu 24.04 (celle ciblée par `install.sh`) pour
confirmer qu'un mécanisme `TAG+="uaccess"` existe bel et bien nativement
sur ce système, et surtout qu'il ne couvre PAS déjà le bus USB générique
— seulement des classes précises (`ID_USB_INTERFACES` de capture
d'image, lecteurs média, capteurs...). Confirme qu'une règle dédiée est
nécessaire pour un outil comme `urbdrc` qui doit pouvoir rediriger
n'importe quel périphérique, pas une classe connue à l'avance — le même
besoin que dfu-util/OpenOCD/PlatformIO résolvent chacun avec leur propre
règle `uaccess`.

**`udev/70-asyncrdp-usb.rules`** (nouveau fichier, nouveau dossier) :
`SUBSYSTEM=="usb", TAG+="uaccess"`. Choix de `uaccess` (délégation à
`systemd-logind`, ACL accordée à la session graphique active) plutôt
qu'un groupe `plugdev` classique : pas de re-connexion nécessaire, pas
de groupe supplémentaire à gérer, cohérent avec l'usage visé (poste de
bureau GNOME interactif, exactement le contexte de GCM). Compromis
documenté explicitement dans le fichier : la règle s'applique à tout
périphérique USB, pas seulement à celui qu'on compte rediriger, faute
de connaître le VID:PID à l'avance côté GCM — une variante scopée à un
périphérique précis est donnée en commentaire pour qui préfère un accès
plus restreint.

**`usb_device_node_path()` / `usb_has_rw_access()`** (`src/asyncrdp/usb.py`) :
la première construit le chemin `/dev/bus/usb/BBB/DDD` à partir d'un
`UsbDeviceInfo` (logique pure). La seconde vérifie l'accès
lecture+écriture sur ce chemin via `os.access()` — sans ouverture
libusb — et renvoie `True`/`False`/`None` (`None` = device node absent,
pas déterminable, à ne pas confondre avec « refusé »). Objectif :
permettre à un appelant (le futur plugin GCM) de détecter le blocage
*avant* de tenter une redirection, plutôt que de recevoir un
`LIBUSB_ERROR_ACCESS` tardif en pleine session RDP.

**`install.sh --usb`** : installe désormais aussi la règle
(`cp` vers `/etc/udev/rules.d/` + `udevadm control --reload-rules` +
`udevadm trigger`), avec un avertissement propre (pas un échec du
script) si `udevadm` est introuvable.

## Rattrapage de documentation

En cherchant où documenter ce qui précède, `README.md` s'est révélé
significativement désynchronisé de `docs/features-backlog.md`/
`CLAUDE.md` — probablement resté figé depuis avant la mise en place du
flux `CLAUDE.md` court + backlog détaillé (sessions 1-2), jamais repris
depuis alors que le backlog, lui, est tenu à jour à chaque session.
Corrigé dans la même livraison plutôt que de laisser cette
incohérence à côté d'ajouts fraîchement documentés :
- Tableau de statut : Audio (disait « jamais testée » — en réalité
  confirmée dans les deux sens, sessions 08/10), USB (disait « jamais
  testé » — en réalité testé sur du vrai matériel, session 19), GTK4
  (disait « jamais exécutée » — en réalité testée sous Xvfb et sur un
  vrai écran physique, sessions 03/07) tous corrigés.
- Arborescence « Architecture » : `usb.py`, `tracing.py` et le nouveau
  `udev/` ajoutés (absents alors que présents dans le dépôt depuis
  plusieurs sessions) ; liste `tests/` complétée (6 fichiers de test
  existants n'y apparaissaient pas du tout).
- Section Installation : `--usb` ajouté aux exemples (le script le
  supportait déjà, mais n'était documenté nulle part dans le README).
- Limitations connues : entrée USB mise à jour pour mentionner la
  règle udev livrée cette session et `usb_has_rw_access()`.

Sans rapport direct mais remarqué en éditant `MANIFEST.in` pour y
ajouter `udev/` : une ligne `include features.md` y référençait un
fichier qui n'existe plus à la racine depuis le passage à
`docs/features-backlog.md` (déjà couvert de toute façon par
`recursive-include docs *.md`) — ligne morte supprimée au passage.

## Vérification finale

- `ruff check .` → `All checks passed!`
- **Binding cffi réellement recompilé** cette session : `freerdp3-dev`/
  `libwinpr3-dev` 3.31.0 + `pyusb` installés via `./install.sh --dev --usb`
  dans ce sandbox, `pip install -e ".[test,usb]"` sans erreur.
- `pytest tests/` → **96 passed, 12 skipped, 0 failed** (baseline avant
  cette session, rejouée à l'identique avant toute modification : 91
  passed, 12 skipped — les 5 nouveaux tests de `test_usb.py` expliquent
  exactement l'écart, aucune régression ailleurs).
- `bash -n install.sh` → syntaxe OK.
- **Limite honnête, non résolue par cette session** : ni `udevadm`, ni
  matériel USB physique, ni session `systemd-logind` active ne sont
  disponibles dans ce sandbox de développement. La règle udev a donc
  été vérifiée par lecture/convention (comparée au fichier
  `70-uaccess.rules` réellement présent sur cette même image Ubuntu
  24.04) et par les tests de logique pure de `usb_has_rw_access()`,
  mais **pas** par une exécution réelle de `install.sh --usb` suivie
  d'un branchement de périphérique sur une vraie session graphique.
  Reste donc ouvert, sur le fond inchangé : la confirmation matérielle
  complète (règle installée + vrai périphérique + vrai serveur RDP).
  Voir `docs/features-backlog.md`, section USB.

## Fichiers modifiés

`udev/70-asyncrdp-usb.rules` (nouveau), `src/asyncrdp/usb.py`,
`install.sh`, `tests/test_usb.py`, `docs/features-backlog.md`,
`CLAUDE.md`, `README.md`, `MANIFEST.in`, `docs/sessions/README.md`.
