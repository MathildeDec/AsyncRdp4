[← index](README.md) — session 08 — 2026-09-05

## Audio réel : PipeWire monté, charge utile RDPSND confirmée (2026-09-05, nouvelle session)

Reprise du même sandbox (état conteneur persistant : binding asyncrdp
compilé — recompilé une fois de plus après le nettoyage du zip
précédent, voir note en fin de section —, utilisateur `rdptest`,
serveur `xrdp` — juste relancé comme d'habitude). Tâche choisie :
« Monter un vrai environnement PipeWire dans une session de test pour
valider l'audio au-delà de la négociation de canal », backlog moyenne
priorité jamais attaqué jusqu'ici.

### Ce qui existe déjà côté code (avant cette session)

`RdpOptions.audio_playback`/`audio_capture` ne font qu'activer les
réglages FreeRDP (`FreeRDP_AudioPlayback`/`FreeRDP_AudioCapture`) et
laisser `freerdp_client_load_addins` charger l'addin `rdpsnd`
standard. Vérifié : `rdpsnd`/`audin` (et leurs backends ALSA/Pulse) ne
sont **pas** des `.so` séparés chargés dynamiquement — ils sont
compilés statiquement dans `libfreerdp-client3.so` lui-même (`strings
libfreerdp-client3.so.3.31.0 | grep rdpsnd` les montre tous en dur).
Donc pas de plomberie supplémentaire à écrire côté `asyncrdp` pour
qu'un son reçu soit réellement traité : FreeRDP le fait lui-même,
nativement, dès que le channel est chargé — la question de cette
session était uniquement « le serveur envoie-t-il vraiment quelque
chose, et est-ce que ça arrive vraiment côté client ? », pas
« faut-il exposer un callback Python ? ».

### Mise en place de l'environnement audio serveur (côté session `rdptest`)

`xrdp` installe déjà `pipewire-module-xrdp` et
`libpipewire-0.3-modules-xrdp` comme dépendances (repéré dès la toute
première session xrdp de cette conversation), mais **pas** les
binaires `pipewire`/`wireplumber`/`pipewire-pulse` eux-mêmes ni les
utilitaires (`pactl`/`paplay`, `alsa-utils`) — à installer
explicitement : `apt-get install -y pipewire pipewire-pulse
wireplumber pulseaudio-utils alsa-utils dbus-x11`. `dbus-x11` (fournit
`dbus-launch`) est nécessaire : sans bus de session D-Bus,
`wireplumber` échoue immédiatement (`Error acquiring bus address:
Cannot autolaunch D-Bus without X11 $DISPLAY`), et le module `rt` de
PipeWire lui-même échoue à s'attacher à un bus (juste un `WARN`,
non bloquant, mais `wireplumber` sans bus, lui, ne démarre pas du
tout).

`openbox-session` ne fait **pas** d'autostart XDG (`.desktop` dans
`/etc/xdg/autostart/`) contrairement à GNOME/KDE — donc
`pipewire-xrdp.desktop` (fourni par le paquet) n'est jamais lancé tout
seul. Il faut, dans l'ordre, en tant que `rdptest` avec
`XDG_RUNTIME_DIR=/run/user/<uid>` (créé et `chown` manuellement, pas
fourni par défaut pour un utilisateur sans vraie session logind) :
1. `pipewire` (le serveur audio)
2. `wireplumber` (le session manager — sans lui, aucun routage
   automatique des flux, mais les nœuds bruts existent déjà)
3. `pipewire-pulse` (compatibilité PulseAudio, ce que `pactl`/`paplay`
   utilisent)
4. `/usr/libexec/pipewire-module-xrdp/load_pw_modules.sh -l <niveau>`
   — charge `libpipewire-module-xrdp` et crée les nœuds `xrdp-sink`/
   `xrdp-source`, puis les positionne comme sink/source par défaut via
   `pactl set-default-sink/source`.

### Piège trouvé : variables d'environnement inventées, échec silencieux

Première tentative avec des valeurs devinées pour `XRDP_SESSION=1` et
`XRDP_SOCKET_PATH=/tmp/.xrdp/xrdp_display_<N>` (raisonnable en
apparence, mais fausses) : le script `load_pw_modules.sh` s'exécute
sans erreur, `xrdp-sink`/`xrdp-source` apparaissent bien dans `pactl
list short sinks`, `paplay` d'un vrai fichier WAV réussit sans
erreur (`exit=0`) — **mais rien n'arrive jamais côté client asyncrdp**
(aucun `rdpsnd_recv_wave_info_pdu` dans les logs `WLOG_LEVEL=DEBUG`,
juste la négociation de format habituelle). Aucune erreur nulle part
pour l'indiquer : le module PipeWire-xrdp échoue silencieusement à
joindre `chansrv` avec un mauvais chemin de socket, et personne ne le
signale — `paplay` ne sait pas que le sink ne relaie nulle part, il
écrit juste dans le buffer PipeWire local avec succès.

Les **vraies** valeurs, à récupérer directement depuis l'environnement
du process `xrdp-chansrv` lui-même (`cat /proc/<pid-chansrv>/environ |
tr '\0' '\n'`) plutôt que devinées :
```
XRDP_SOCKET_PATH=/run/xrdp/sockdir
XRDP_PULSE_SINK_SOCKET=xrdp_chansrv_audio_out_socket_<display>
XRDP_PULSE_SOURCE_SOCKET=xrdp_chansrv_audio_in_socket_<display>
PULSE_SCRIPT=/etc/xrdp/pulse/default.pa
```
(`XRDP_SESSION=1` était la seule variable déjà correcte par chance.)
Ces noms trahissent que ce mécanisme a été conçu à l'origine pour
l'ancien pont **PulseAudio** natif d'xrdp (modules `module-xrdp-sink`/
`module-xrdp-source`, `PULSE_SCRIPT`), pas spécifiquement pour
`libpipewire-module-xrdp` — mais ce dernier lit apparemment les mêmes
variables (`XRDP_SOCKET_PATH` au minimum) pour trouver `chansrv`, donc
ça fonctionne une fois les bonnes valeurs fournies. Après avoir relancé
tout le pipeline (`pipewire`/`wireplumber`/`pipewire-pulse`/module)
avec ces vraies valeurs : succès immédiat, voir plus bas.

### Piège annexe : `speaker-test` n'est pas adapté à ce pipeline

`speaker-test` ouvre ALSA directement (`Playback open error: -2,No
such file or directory` — pas de device ALSA « default » réel dans ce
conteneur, et pas de plugin `pipewire-alsa` installé pour rediriger
ALSA vers PipeWire). `paplay` (client PulseAudio, compatible via
`pipewire-pulse`) fonctionne, lui, directement — c'est le bon outil
ici, pas `speaker-test`/`aplay` sans configuration ALSA supplémentaire.
Fichier de test généré à la volée avec le module `wave` de Python
(sinus 440 Hz, 44100 Hz, stéréo 16 bits) plutôt que de dépendre d'un
outil externe (`sox` non vérifié disponible).

### Résultat : charge utile RDPSND réelle confirmée

Avec les bonnes variables, connexion `asyncrdp` (`audio_playback=True`)
puis `paplay /home/rdptest/test_tone.wav` côté session pendant que la
connexion est active (`WLOG_LEVEL=DEBUG` côté client) :

```
[...] rdpsnd_recv_server_audio_formats_pdu: [static] Server Audio Formats
[...] rdpsnd_send_client_audio_formats: [static] Client Audio Formats
[...] rdpsnd_recv_training_pdu / rdpsnd_send_training_confirm_pdu   (négociation, déjà connue)

>>> lecture du son côté session via paplay maintenant

[...] rdpsnd_recv_wave_info_pdu: [static] WaveInfo: cBlockNo: 89 wFormatNo: 0 [WAVE_FORMAT_PCM]
[...] rdpsnd_ensure_device_is_open: [static] Opening device with format WAVE_FORMAT_PCM
```

`cBlockNo` (89, puis 176 sur une relance suivante — un compteur qui
avance réellement, pas une valeur figée) confirme qu'il s'agit bien
d'un flux d'échantillons réel généré par le vrai fichier WAV joué,
transporté depuis `xrdp-sink` → module PipeWire-xrdp → `chansrv` →
canal RDPSND → notre client `asyncrdp` → FreeRDP. C'est exactement ce
que le point du backlog demandait : « au-delà de la négociation de
canal ».

### Limite non résolue, isolée et documentée : lecture physique locale

FreeRDP échoue ensuite à `rdpsnd_alsa_open_mixer: snd_mixer_attach
failed`, empêchant la suite du pipeline de lecture locale (écriture
réelle des échantillons vers un device de sortie). Essayé : un
`~/.asoundrc` définissant un PCM `null` par défaut pour root (le
`speaker-test` en ligne de commande s'ouvre bien avec cette config,
prouvant que le PCM lui-même fonctionne) — insuffisant, car ALSA
tente en plus d'attacher un **mixer** (contrôle de volume), qui exige
une carte avec une interface de contrôle réelle. Sans le module noyau
`snd-dummy` (carte ALSA virtuelle complète, mixer inclus) — hors de
portée sans `CAP_SYS_MODULE`/privilèges noyau dans ce conteneur — ce
dernier maillon reste bloqué. **Ce n'est pas une limite du code
testé** : c'est une limite du poste client de ce sandbox précis,
analogue au constat déjà fait pour FUSE (bug 2 de la session
précédente) — un problème d'environnement, pas de protocole ni de
binding.

### Bonus : correction d'une note obsolète repérée au passage

En cherchant si un serveur xrdp réel était disponible (cette session
en a monté un pour la troisième fois consécutive), remarqué que la
revérification `_shim.c` du 2026-09-02 affirmait encore « un serveur
xrdp réel reste en revanche indisponible dans ce sandbox » — corrigé
dans `features.md` : c'était déjà faux à l'époque où ça a été écrit,
pour la même raison que l'erreur similaire sur `freerdp3-dev` corrigée
une ligne plus haut dans le même paragraphe (pas encore installé
**cette session-là** ≠ indisponible dans l'absolu).

### Fichiers modifiés cette session

Aucun fichier de code (`src/`, `examples/`, `tests/`, `integrations/`)
— cette session était une investigation d'infrastructure de test, pas
un changement de comportement d'`asyncrdp`. Seuls `features.md` et ce
fichier ont été mis à jour. Scripts de diagnostic
(`/home/claude/work/diag_audio*.py`, `/root/.asoundrc`,
`/tmp/rdptest_env.sh`) non versionnés, comme d'habitude.

Note pratique pour une session future : le binding compilé
(`asyncrdp/_asyncrdp_cffi*`, `src/asyncrdp.egg-info/`,
`src/asyncrdp/_asyncrdp_cffi.abi3.so`) est volontairement retiré du
zip livré à chaque fin de session (ce sont des artefacts de
compilation propres à l'environnement, pas du code source) — donc
`python3 build_ffi.py && pip install --break-system-packages -e .`
est à refaire à chaque reprise si un test contre un vrai serveur est
nécessaire, même quand le reste du conteneur (utilisateur `rdptest`,
paquets `apt`) a persisté d'une réponse à l'autre.
