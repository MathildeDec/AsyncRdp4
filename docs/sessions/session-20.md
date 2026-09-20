[← index](README.md) — session 20 — 2026-09-08

## Revérification par vraie compilation + vraie connexion xrdp, contre un paquet Ubuntu (pas un snapshot) (2026-09-08)

Contrairement aux deux sessions précédentes sur ce sujet (2026-09-02,
l'une n'ayant accès qu'à une lecture statique d'un snapshot `master`
GitHub, l'autre ayant compilé mais sans avoir monté de serveur xrdp),
cette session dispose des deux à la fois : `apt`/`archive.ubuntu.com`
et `security.ubuntu.com` accessibles, droits root, compilateur
présent d'emblée. Tâche choisie en conséquence, reprenant exactement
la recommandation laissée le 2026-09-02 ci-dessus (« revérifier contre
une version taguée précise et faire une vraie compilation +
`test_connect_full.py` ») — avec un paquet Ubuntu réel plutôt qu'un
tag GitHub, ce qui teste directement ce qu'un utilisateur du paquet
installerait.

**Compilation réelle** : `apt install freerdp3-dev libwinpr3-dev`
installe la version candidate `3.31.0+dfsg-0ubuntu0.24.04.1` (la même
version majeure que celle déjà testée le 2026-09-02 et pendant la
campagne de tests par redirection — voir plus haut/plus bas dans ce
fichier). `pip install -e .` (donc `build_ffi.py` via le
`cffi_modules` de `setup.py`) compile `_shim.c` sans aucune erreur.
Seuls des avertissements de dépréciation apparaissent (capturés avec
`build_ffi.py` lancé directement, en verbose) : `WINPR_SLIST_HEADER`/
`WINPR_LIST_ENTRY`/`WINPR_SINGLE_LIST_ENTRY` (dépréciés depuis
3.27.0), `SEC_WINPR_NTLM_SETTINGS`/`SEC_WINPR_KERBEROS_SETTINGS`
(dépréciés depuis 3.31.0, donc nouveaux par rapport à la
revérification du 2026-09-02), `pVerifyCertificate`/
`pVerifyChangedCertificate`, `codecs_free`. Vérifié par `grep` : aucun
de ces huit symboles n'apparaît dans `_shim.c` lui-même — tous
proviennent de déclarations dans les headers inclus transitivement
(`freerdp/freerdp.h`, `freerdp/codecs.h`, `winpr/interlocked.h`,
`winpr/sspi.h`), parsées par le compilateur qu'elles soient utilisées
ou non. **Confirme qu'aucune modification de `_shim.c` n'est
nécessaire** pour cette version — les fonctions/champs réellement
appelés par le shim (les 39 symboles déjà recensés le 2026-09-02, plus
tout accès aux champs `RDPDR_PRINTER`/`CLIPRDR_HEADER` imbriqué) ne
génèrent aucun avertissement, seule leur présence dans des headers
transitifs en génère.

**Vraie connexion contre un vrai xrdp**, avec ce binding : `xrdp` +
`xrdp-sesman` + `xorgxrdp` + `openbox` installés et démarrés dans ce
sandbox (procédure identique à celle documentée en tête de ce
fichier — utilisateur `rdptest`/`testpass123`, `~/.xsession` avec
`exec openbox-session`, services lancés en `setsid` chacun dans le
même appel qui vérifie leur démarrage). `examples/
test_connect_minimal.py 127.0.0.1 rdptest testpass123
--ignore-certificate --frames 3` : connexion réussie, 3 frames
1280x800 reçues, déconnexion propre — même résultat que le tout
premier test de ce projet et que celui du 2026-09-07 contre un
troisième serveur externe. Poussé plus loin que
`test_connect_minimal.py` : `tests/test_integration_live.py` (les 5
tests que ce fichier existant proposait déjà mais qu'aucune session
précédente n'avait pu exécuter faute d'avoir à la fois un serveur ET
le binding compilé au même moment) lancé avec `ASYNCRDP_TEST_HOST=
127.0.0.1` — **5/5 passent** : connexion + frame exacte (1280x800,
taille en octets = largeur×hauteur×4, sans padding), clavier/souris
sans exception, resize dynamique du canal `disp` (1280x800 → 1024x640,
nouvelle frame reçue après), presse-papier texte (annonce sans
exception), déconnexion propre sans tâche asyncio en suspens. Suite
complète (`pytest tests/`, serveur xrdp toujours actif) : **95 passed,
8 skipped** (les 8 restants sont les tests `gtk4_live`, aucun GTK4 ni
Xvfb installés cette session — hors périmètre choisi ici).

**Referme définitivement** l'item resté ouvert depuis le 2026-09-02
(« Reste ouvert : rejouer une vraie connexion contre un `xrdp` réel
avec ce binding recompilé ») — voir aussi `features.md`. Aucun fichier
source du paquet n'a été modifié cette session : la tâche était de
vérification, pas de correction, et n'a trouvé aucun écart à corriger.

## Suite GTK4 live rejouée dans ce même sandbox, plus premier run combiné complet (2026-09-08, suite de la session ci-dessus)

Après la revérification du shim et la connexion xrdp réelle
ci-dessus, tâche suivante choisie dans la continuité : ce sandbox a
déjà `apt`/root, donc plutôt que de m'arrêter là, mise en place
complète de `tests/test_gtk4_live.py` (Xvfb, GTK4, xclip, imagemagick)
pour ajouter une nouvelle confirmation indépendante, sur un
environnement encore différent des précédents (ni le sandbox du
2026-09-01/02/05/06, ni le poste physique du 2026-09-07).

**Mise en place** : `apt install gir1.2-gtk-4.0 xvfb xclip x11-apps
imagemagick` → GTK **4.14.5** (même version majeure que la session du
2026-09-02/06, aucune version plus récente disponible via `apt` sur
cette Ubuntu 24.04 — cohérent avec ce qui avait déjà été constaté).
`pip install PyGObject gbulb` dans le venv a échoué une première fois
(`Dependency 'girepository-2.0' is required but not found` — PyGObject
3.58.0 depuis PyPI, plus récent que le `python3-gi` 3.48.2 fourni par
`apt`, exige les headers `girepository-2.0` et non plus seulement
`girepository-1.0`) : corrigé en ajoutant `libgirepository-2.0-dev` à
la liste des paquets système. **Détail utile pour une session future
qui repartirait d'un venv propre** (le paquet livré, lui, n'a pas
besoin de ce détail — `install.sh --gtk4` s'appuie sur `python3-gi`
du système, pas sur PyGObject via pip). `pip install Pillow` a
également été nécessaire (`test_display_bridge_renders_correct_colors_onscreen`
utilise `PIL.Image` pour décoder le PNG de la capture d'écran — dépendance
de test manquante du venv, pas du paquet).

**Résultat, `ASYNCRDP_TEST_GTK4=1 LIBGL_ALWAYS_SOFTWARE=1`** (le
contournement DRI3/EGL du 2026-09-06 reste nécessaire ici, comme
attendu sous Xvfb) : **7/7 tests passent** (5 réels + 2 `xfail`
attendus) :
- Rendu écran réel (`test_display_bridge_renders_correct_colors_onscreen`) :
  couleurs correctes dans les 4 quadrants, capture `xwd` indépendante de GDK.
- Presse-papier texte bidirectionnel avec `xclip` externe.
- Décodage DIB indexé 4bpp (régression du bug `bfOffBits`) toujours correct.
- Fichiers local→RDP et RDP→local (téléchargement asynchrone via `gbulb`) :
  les deux sens confirmés, y compris le cas dossier premier-niveau
  (correctif du 2026-09-06 dans `fetch_one`) toujours correct.
- Les 2 `xfail` connus (mime-type image absent du presse-papier système)
  se reproduisent à l'identique — **aucune régression, aucune amélioration**,
  cohérent avec le fait que la version de GTK4 disponible ici (4.14.5) est
  la même que celle où le bug avait été diagnostiqué comme probablement
  interne à GDK (voir plus bas dans ce fichier, section du 2026-09-07) ;
  ce diagnostic reste donc non vérifiable plus avant sans un GTK4 plus
  récent, toujours hors de portée du réseau autorisé dans ce sandbox
  (`gitlab.gnome.org` non accessible, `apt` ne propose rien de plus récent
  que 4.14.5 sur `noble`).

Nouveauté d'environnement notée au passage, sans impact sur les tests :
avertissement `Unable to acquire session bus: Failed to execute child
process "dbus-launch"` (pas de d-bus de session dans ce conteneur) —
purement cosmétique, GTK4 fonctionne quand même sans bus de session ici.

**Bonus** : avec `xrdp` relancé (tué entre deux appels outil — piège déjà
documenté en tête de ce fichier, confirmé une fois de plus) à côté de ce
même environnement GTK4, `pytest tests/` complet donne pour la première
fois de ce projet **100 passed, 1 skipped, 2 xfailed, 0 failed** en une
seule fois — unitaires + intégration xrdp réelle + GTK4 live réel, tout
au vert simultanément dans le même sandbox (le seul `skip` restant est
`test_full_suite.py`, qui prend des arguments CLI dédiés plutôt que de
tourner sous `pytest` directement). Aucune session précédente n'avait eu
les trois environnements disponibles en même temps. Aucun fichier source
du paquet modifié cette session non plus — uniquement de la vérification.
