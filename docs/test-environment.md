---
name: test-environment
---

# Reproduire l'environnement de test

Le sandbox de développement n'est pas persistant — cette recette est à rejouer à chaque reprise de session nécessitant un vrai serveur RDP. Lire le piège setsid avant de lancer quoi que ce soit en arrière-plan.

Pour retrouver un serveur de test :

```bash
apt install -y build-essential pkg-config freerdp3-dev libwinpr3-dev python3-dev python3-venv \
    xrdp xorgxrdp openbox xclip cups
useradd -m -s /bin/bash rdptest && echo "rdptest:testpass123" | chpasswd
su - rdptest -c "echo 'exec openbox-session' > ~/.xsession"

# IMPORTANT : les lancements en arrière-plan doivent être setsid, seuls, un par
# appel — combiner plusieurs services dans une seule commande bloque le
# terminal de façon intermittente dans ce sandbox (raison jamais élucidée).
# Confirmé à nouveau le 2026-09-02 avec Xvfb : un processus démarré en
# arrière-plan dans un appel ne survit PAS de façon fiable jusqu'à un appel
# suivant (parfois oui, parfois le process a disparu — pas élucidé non plus).
# Le seul pattern fiable observé : démarrer le service ET l'utiliser dans le
# MÊME appel outil.
setsid /usr/sbin/xrdp-sesman --nodaemon < /dev/null > /tmp/sesman.log 2>&1 &
# (vérifier que ça a démarré avant de continuer)
setsid /usr/sbin/xrdp --nodaemon < /dev/null > /tmp/xrdp.log 2>&1 &

chmod 666 /dev/fuse   # sinon xrdp-chansrv (tourne sous l'utilisateur de session) ne peut pas
                      # monter les disques redirigés — /dev/fuse est en 600 par défaut

cd <répertoire du projet>
python3 build_ffi.py
PYTHONPATH=. python3 test_connect_full.py 127.0.0.1 rdptest testpass123
```

Logs utiles : `/var/log/xrdp.log`, `/var/log/xrdp-sesman.log`, `~/.local/share/xrdp/xrdp-chansrv.<N>.log` (côté session rdptest — un fichier par session X, `<N>` est le numéro de display).

## Piège supplémentaire (trouvé le 2026-09-16, session 30) : le conteneur lui-même peut redémarrer en cours de session

Pas seulement les processus en arrière-plan qui « ne survivent pas de
façon fiable d'un appel outil à l'autre » (piège déjà documenté
ci-dessus) : le conteneur entier peut redémarrer pendant une session de
travail. Symptôme : `test_integration_live.py` échoue soudainement en
`ConnectLayer ... failed` alors que tout fonctionnait un instant plus
tôt. Diagnostic en une commande : `uptime` — un `up N min` avec `N`
petit confirme un redémarrage récent, pas juste un service mort. Ce qui
survit : tout ce qui est sur disque (dépôt, venv, paquets `apt`,
utilisateur `rdptest`). Ce qui ne survit pas : les processus
(`xrdp`, `xrdp-sesman`, `Xvfb`) et `/dev/fuse` (revient à `600`).

Conséquence pratique : ne pas démarrer les services un par un sur
plusieurs appels d'outil séparés en confiance — combiner leur démarrage
ET la suite de tests dans un seul et même appel, chaque étape vérifiée
vivante (`kill -0 $PID`) avant de passer à la suivante :

```bash
#!/bin/bash
set -uo pipefail

chmod 666 /dev/fuse

pkill -f "xrdp-sesman" 2>/dev/null; pkill -f "/usr/sbin/xrdp " 2>/dev/null
pkill -f "Xvfb :99" 2>/dev/null
sleep 1

setsid /usr/sbin/xrdp-sesman --nodaemon < /dev/null > /tmp/sesman.log 2>&1 &
SESMAN_PID=$!; sleep 2
kill -0 "$SESMAN_PID" 2>/dev/null || { echo "SESMAN_DEAD"; cat /tmp/sesman.log; exit 1; }

setsid /usr/sbin/xrdp --nodaemon < /dev/null > /tmp/xrdp.log 2>&1 &
XRDP_PID=$!; sleep 2
kill -0 "$XRDP_PID" 2>/dev/null || { echo "XRDP_DEAD"; cat /tmp/xrdp.log; exit 1; }

nohup Xvfb :99 -screen 0 1280x1024x24 > /tmp/xvfb.log 2>&1 < /dev/null &
XVFB_PID=$!; sleep 2
kill -0 "$XVFB_PID" 2>/dev/null || { echo "XVFB_DEAD"; cat /tmp/xvfb.log; exit 1; }

# Cache de certificat FreeRDP potentiellement périmé (clé hôte xrdp
# régénérée à chaque démarrage) — purge préventive.
rm -rf /root/.config/freerdp/server

cd <répertoire du projet>
ASYNCRDP_TEST_HOST=127.0.0.1 ASYNCRDP_TEST_USER=rdptest \
    ASYNCRDP_TEST_PASSWORD=testpass123 ASYNCRDP_TEST_GTK4=1 \
    DISPLAY=:99 GDK_BACKEND=x11 \
    uv run pytest tests/ -v
```

Résultat le 2026-09-16 avec cette recette : les trois services vivants
du début à la fin d'un run de 112 tests (~20s), aucune mort en cours de
route.

## Détecter GLX dans Xvfb (trouvé le 2026-09-18, session 32)

`dpkg -L xvfb | grep glx` (le test utilisé jusqu'ici, voir le
paragraphe dédié en tête de `tests/test_gtk4_live.py`) reste **vide
dans tous les cas rencontrés à ce jour, y compris quand GLX est bel et
bien présent** — ce n'est donc pas un indicateur fiable, contrairement
à ce que son usage précédent suggérait. Test direct et fiable :

```bash
Xvfb :99 -screen 0 400x300x24 -nolisten tcp &
sleep 2
DISPLAY=:99 xdpyinfo -queryExtensions | grep -i GLX
```

Une ligne `GLX  (opcode: ...)` en sortie confirme l'extension. Ce
résultat varie d'un bac à sable à l'autre (absent en session 31, présent
en session 32, même paquet `xvfb` Ubuntu 24.04 dans les deux cas) — à
revérifier à chaque reprise plutôt qu'à supposer réglé ou bloqué par
défaut.

## Piège PyGObject vs uv (trouvé le 2026-09-18, session 32)

`install.sh --gtk4` installe `python3-gi` via apt (PyGObject 3.48.2 sur
Ubuntu 24.04 au moment de cette session), puis appelait jusqu'ici
`uv pip install --system -e ".[gtk4]"` sans contrainte de version.
`uv` ne voit pas le paquet apt (pas de métadonnées pip pour un paquet
dpkg) et tente donc de compiler la toute dernière version PyPI
(3.58.0), qui exige `girepository-2.0` au lien — absent des dépôts
Ubuntu 24.04 (seul `girepository-1.0` y est empaqueté). Échec net à
l'étape meson :

```
Run-time dependency girepository-2.0 found: NO
ERROR: Dependency 'girepository-2.0' is required but not found.
```

`install.sh` était donc réellement cassé avec `--gtk4` sur une Ubuntu
24.04 neuve, pas seulement une hypothèse de sandbox. Corrigé dans
`install.sh` : la version PyGObject qu'apt vient d'installer
(`python3 -c "import gi; print(gi.__version__)"`) est désormais passée
en contrainte explicite à `uv pip install` — évite à la fois la
recompilation contre `girepository-2.0` et toute divergence entre le
PyGObject système et celui que `uv` installe. Si ce blocage
réapparaît malgré tout (nouvelle version d'Ubuntu ayant basculé vers
`girepository-2.0` en dépôt standard, par exemple), la contrainte
devient inutile mais inoffensive — `uv` retombera simplement sur la
version demandée si elle est toujours disponible sur PyPI, sinon
l'échec redeviendra visible tel quel.
