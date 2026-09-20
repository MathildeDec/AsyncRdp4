---
name: session-29
---

[← index](README.md)

# Session 29 (2026-09-16) — Imprimante/série/parallèle : la piste de la session 28 menée à son terme ; le « blocage sandbox » était un mauvais diagnostic

## Contexte

Reprise depuis l'archive `asyncrdp-20260915-193114.zip`. Consigne
inchangée (uv, ruff à 0, continuer les features, tenir les fichiers de
suivi/tests/doc, livrer un zip horodaté).

Point de départ hérité de la session 28 : le harnais de test (serveur
d'exemple FreeRDP + canal serveur RDPDR générique) compilait et linkait,
mais la connexion réelle n'avait jamais pu être établie — conclusion
retenue à l'époque, explicitement marquée « non confirmée » :
*« probablement une restriction du bac à sable sur les sockets en
écoute »*.

## Outillage : rien à migrer, tout revérifié

`uv` était déjà en place depuis la session 28, et ce projet n'a **jamais
utilisé Poetry** (constat déjà établi et redocumenté en session 28 :
`[build-system]` a toujours été `setuptools.build_meta`). Revérifié
concrètement dans ce bac à sable plutôt que relu :

- `uv sync` → recrée `.venv/`, recompile le shim cffi contre FreeRDP
  3.31.0 (installé via `apt` cette session), installe les 12 paquets du
  groupe `dev`.
- `uv run ruff check .` → **0 erreur**, rien à corriger.
- `uv run pytest` (hors serveur réel/GTK4/extras) → **64 passed,
  1 skipped, 0 failed**.
- `install.sh` amorce déjà `uv` s'il est absent (`python3 -m pip install
  uv`), donc le script ne dépend pas d'un `uv` préinstallé.

## Le blocage de la session 28 était un mauvais diagnostic

Trois constats, dans l'ordre où ils ont été faits, qui démontent
l'hypothèse « restriction réseau du bac à sable » :

1. **L'écoute TCP locale fonctionne parfaitement ici.** Un
   `bind()`/`listen()`/`accept()` Python sur `127.0.0.1:13389`, avec un
   vrai aller-retour d'octets depuis un thread client, passe du premier
   coup. C'est le test le plus simple possible et il aurait suffi à
   invalider l'hypothèse.

2. **`--local-only` n'ouvre pas de socket TCP du tout.** Lecture de
   `libfreerdp/core/listener.c` : cette option appelle
   `freerdp_listener_open_local()`, qui fait un
   `socket(AF_UNIX, SOCK_STREAM, 0)` sur un chemin temporaire. Le socket
   observé en session 28 via `/proc/<pid>/fd` était donc bien réel — et
   son absence de `/proc/net/tcp` parfaitement normale, puisque ce n'est
   pas un socket TCP. L'observation était juste ; l'interprétation non.

3. **Le message d'erreur `[freerdp_listener_open]: socket` n'est pas
   fatal.** Dans `freerdp_listener_open()`, il est suivi d'un `continue`
   dans la boucle sur les `addrinfo` : l'échec de la tentative IPv6
   n'empêche pas l'IPv4 de réussir, et la fonction retourne
   `num_sockfds > 0`. Relancé ici sans `--local-only` : le serveur
   affiche l'erreur **et** `Listening on [0.0.0.0]:13389`, confirmé dans
   `/proc/net/tcp` (port `0x344D`).

Leçon à retenir pour les sessions futures : ce message d'erreur ressemble
à un échec d'ouverture, n'en est pas un, et a coûté une session entière.

## Harnais : deux obstacles réels, tous deux hors sujet RDPDR

Le harnais a été reconstruit à l'identique des indications de la session
28 (`sf_rdpdr.c`/`sf_rdpdr.h` contre `server/Sample` de FreeRDP 3.31.0,
sources récupérées depuis `codeload.github.com`, tag `3.31.0`, même
version que le paquet système). Compilation et link : OK du premier coup,
comme annoncé.

Deux échecs de connexion successifs, tous deux causés par le caractère
« démo graphique » du serveur d'exemple, sans rapport avec la redirection
de périphériques :

1. `test_peer_load_icon()` refuse le peer si le client n'annonce ni
   `RemoteFxCodec` ni `NSCodec`. `asyncrdp` fait du GDI + RDPGFX et
   n'expose aucun de ces deux réglages hérités —
   `rdp_peer_handle_state_active: PostConnect for peer failed`.
2. Une fois l'icône dégradée en avertissement : même cause un cran plus
   loin, `test_peer_draw_background()` retourne `FALSE` pour la même
   raison, ce qui fait échouer `tf_peer_activate()` et coupe la session
   avant tout échange RDPDR.

Les deux corrigés côté harnais (avertissement au lieu de `return FALSE`),
et l'initialisation RDPDR déplacée **avant** le bloc icône dans
`tf_peer_post_connect`, pour que le canal soit câblé même si un futur
échec apparaît plus haut dans cette fonction. Ce sont des modifications du
harnais, pas d'`asyncrdp` : le serveur d'exemple FreeRDP veut dessiner une
image de démonstration, ce qui n'a aucun rapport avec ce qu'on mesure.

## Résultat empirique

Sonde : un vrai client `asyncrdp` annonçant une imprimante (`PRN1`), un
port série (`COM3`), un port parallèle (`LPT1`) et un disque témoin
(`probe`). Sortie serveur, **identique sur trois exécutions
consécutives** :

```
SF_RDPDR|INIT|supported_default=0xffff
SF_RDPDR|STARTED
SF_RDPDR|ANNOUNCE|type=PRINT|id=1|name=PRN1|datalen=74|supported_mask=0x000f
SF_RDPDR|CREATE|PRINT|id=1|name=PRN1|reply=STATUS_SUCCESS
SF_RDPDR|ANNOUNCE|type=FILESYSTEM|id=2|name=probe|datalen=6|supported_mask=0x000f
SF_RDPDR|CREATE|FILESYSTEM|id=2|name=probe|reply=STATUS_SUCCESS
SF_RDPDR|ANNOUNCE|type=SERIAL|id=3|name=COM3|datalen=5|supported_mask=0x000f
SF_RDPDR|ANNOUNCE|type=PARALLEL|id=4|name=LPT1|datalen=5|supported_mask=0x000f
```

Trois choses s'y lisent.

**1. L'imprimante est acceptée. C'est une première dans ce projet.**
`STATUS_SUCCESS` sur l'annonce `RDPDR_DTYP_PRINT`, là où xrdp répond
`STATUS_NOT_SUPPORTED` depuis le 2026-09-05. Le constat n°1 de la session
28, jusqu'ici tiré d'une lecture de source, est donc confirmé
empiriquement — et l'annonce construite par `asyncrdp` (74 octets de
DeviceData, conforme à MS-RDPEPC) est protocolairement correcte, ce
qu'aucun serveur n'avait jamais permis de vérifier autrement que par
« le client parle bien le protocole ».

**2. La négociation de capacités laisse bien passer les trois types.**
`supported` part de `0xffff` (tous types) et redescend à `0x000f` après
intersection avec les capacités du client :
`SERIAL|PARALLEL|PRINT|FILESYSTEM`. Les trois types en question sont donc
bien dans l'ensemble négocié — ce n'est pas la négociation qui bloque.

**3. Série et parallèle échouent, mais pas pour la raison attendue : les
deux moitiés de FreeRDP sont en désaccord.** Aucun callback `CREATE` pour
eux, et la cause est explicite côté serveur :

```
[WARN][...rdpdr.server] - [rdpdr_server_receive_device_list_announce_request]:
    [rdpdr] RDPDR_DTYP_SERIAL::DeviceDataLength != 0 [5]
    [rdpdr] RDPDR_DTYP_PARALLEL::DeviceDataLength != 0 [5]
```

Le canal **serveur** de FreeRDP exige `DeviceDataLength == 0` pour ces
deux types et répond `ERROR_INVALID_DATA` sinon
(`channels/rdpdr/server/rdpdr_main.c`). Le canal **client** de FreeRDP y
écrit le nom du périphérique suivi d'un NUL — vérifié dans la source :
`channels/serial/client/serial_main.c` alloue
`Stream_New(nullptr, len + 1)` puis y écrit `name[0..len]` octet par
octet, `channels/parallel/client/parallel_main.c` fait exactement pareil.
D'où les 5 octets observés (`COM3\0`, `LPT1\0`), sérialisés tels quels par
`channels/rdpdr/client/rdpdr_main.c`.

C'est donc un désaccord interne à FreeRDP entre son propre client et son
propre serveur, pas une limitation d'`asyncrdp` : ce champ est rempli par
le plugin de canal chargé via `freerdp_client_load_addins()`, jamais par
du code de ce projet. Laquelle des deux moitiés a tort n'est **pas**
tranché par ce test — il faudrait un vrai serveur Windows pour le dire, et
le fait que la redirection série de `xfreerdp` fonctionne en pratique
contre Windows suggère (sans le prouver) que c'est le contrôle côté
serveur FreeRDP qui est plus strict que la réalité.

## Ce qui reste hors de portée, et pourquoi c'est structurel

Même pour l'imprimante, désormais acceptée à l'annonce, aucun job ne peut
circuler : l'API publique `RdpdrServerContext` n'expose des fonctions
d'E/S appelables par le serveur que pour deux familles —
`DriveOpenFile`/`DriveReadFile`/`DriveWriteFile`/… et les
`Smartcard*`. **Rien** pour l'imprimante, le série ou le parallèle
(vérifié en énumérant les champs de la struct dans
`/usr/include/freerdp3/freerdp/server/rdpdr.h`). Ça complète le constat
n°2 de la session 28 (« les E/S génériques CREATE/READ/WRITE ne sont pas
implémentées ») par un point plus fort : il n'y a même pas de surface
d'API par où un serveur bâti sur ce canal pourrait envoyer un job
d'impression, quand bien même ces fonctions le seraient.

Conclusion du point backlog, désormais appuyée sur de l'observation et
plus seulement sur de la lecture :

- **Imprimante** : le client est correct, prouvé contre un serveur qui
  l'accepte. Ce qui manque est côté serveur FOSS — personne ne consomme
  le job.
- **Série / parallèle** : bloqués plus tôt encore, par une incohérence de
  format d'annonce interne à FreeRDP. Rien à corriger dans ce projet ;
  un rapport de bug FreeRDP serait le geste utile (non déposé ici, même
  raison qu'en session 25 : pas de compte).

## Livré dans le dépôt

- `examples/probe_rdpdr_devices.py` — la sonde, en exemple de première
  classe plutôt qu'en script jetable, avec le résultat obtenu inscrit
  dans sa docstring et la marche à suivre pour l'observer côté serveur.
  Sa configuration est extraite dans `build_probe_options()` pour être
  testable sans serveur.
- `src/asyncrdp/_core.py` — `PrinterMapping`, `SerialMapping` et
  `ParallelMapping` documentent maintenant ce qui a été constaté, à
  l'endroit où un appelant (le futur plugin RDP de GCM) le lira.
- `tests/test_rdp_options.py` — deux tests :
  `test_probe_rdpdr_options_announce_the_four_device_types` fige la
  composition de la sonde (sans le disque témoin, un serveur muet et un
  serveur qui refuse tout seraient indiscernables) ;
  `test_serial_and_parallel_document_the_freerdp_announce_limitation`
  est une régression de documentation — la limitation n'étant pas
  détectable à l'exécution côté client, le seul garde-fou possible est
  que l'avertissement reste attaché aux dataclasses. Ce second test a
  d'ailleurs immédiatement trouvé un vrai oubli en le rédigeant
  (`ParallelMapping` ne renvoyait pas vers cette session), corrigé.

## Reconstruire le harnais

Non versionné ici, à dessein : c'est du code contre les sources FreeRDP
upstream, pas du code `asyncrdp`. Procédure complète :

1. `apt install freerdp3-dev libwinpr3-dev` (donne `freerdp-server3`
   en pkg-config).
2. Récupérer les sources au même tag que le paquet système
   (`codeload.github.com/FreeRDP/FreeRDP/tar.gz/refs/tags/3.31.0`),
   copier `server/Sample/`.
3. Ajouter `sf_rdpdr.c`/`sf_rdpdr.h` : `sf_peer_rdpdr_init()` crée le
   contexte (`rdpdr_server_context_new(context->vcm)`), pose
   `rdpcontext`/`data`, **laisse `supported` à sa valeur par défaut à
   dessein**, enregistre `ReceiveDeviceAnnounce` + les paires
   `OnPrinterCreate`/`Delete`, `OnSerialPortCreate`/`Delete`,
   `OnParallelPortCreate`/`Delete`, `OnDriveCreate`/`Delete` (chacune
   journalise sur stdout avec un préfixe `SF_RDPDR|` parsable, plutôt que
   de dépendre du format WLog), puis `Start()`. `sf_peer_rdpdr_uninit()`
   fait le `Stop()`/`rdpdr_server_context_free()` symétrique, appelé
   depuis `test_peer_context_free`.
4. Dans `sfreerdp.h` : `#include <freerdp/server/rdpdr.h>` et
   `RdpdrServerContext* rdpdr;` dans `struct test_peer_context`.
5. Dans `sfreerdp.c` : inclure `sf_rdpdr.h` ; dans
   `tf_peer_post_connect`, insérer le bloc
   `WTSVirtualChannelManagerIsChannelJoined(context->vcm,
   RDPDR_SVC_CHANNEL_NAME)` → `sf_peer_rdpdr_init(context)` **avant**
   l'appel à `test_peer_load_icon()` ; dégrader en avertissement l'échec
   de `test_peer_load_icon()` **et** celui de
   `test_peer_draw_background()` dans `tf_peer_activate()` (sans ces deux
   corrections, aucun client sans RemoteFX/NSCodec ne peut activer sa
   session).
6. Compiler : `gcc -c *.c $(pkg-config --cflags freerdp-server3)
   -DSAMPLE_RESOURCE_ROOT='"."' -O1` puis linker avec
   `$(pkg-config --libs freerdp-server3) -lm -lpthread`.
7. Certificat : `openssl req -x509 -newkey rsa:2048 -keyout server.key
   -out server.crt -nodes -subj "/CN=..."` dans le répertoire de
   lancement (le serveur cherche `server.crt`/`server.key` par défaut).
8. Lancer `./sfreerdp-rdpdr --port=13389` — **pas** `--local-only`
   (socket AF_UNIX, voir plus haut) — et ignorer l'erreur `socket` sur
   IPv6.
9. `python3 examples/probe_rdpdr_devices.py --port 13389`, puis lire la
   sortie serveur.

## État après cette session

- ruff : 0 erreur.
- Tests unitaires (hors extras/serveur réel/GTK4) : 66 passed,
  1 skipped, 0 failed (+2 cette session). Avec les tests GTK4 de logique
  pure (`numpy` installé à la main dans le venv dev — il relève de
  l'extra `gtk4`, hors du groupe `dev` de `uv sync`) : **97 passed,
  1 skipped, 0 failed**.
- Imprimante/série/parallèle : la question ouverte depuis le 2026-09-05
  est tranchée sur le fond. Plus rien à explorer côté serveur FOSS sans
  écrire soi-même l'implémentation protocolaire manquante.
- USB : inchangé, toujours bloqué par l'absence de matériel physique.
- GTK4 live et intégration contre un vrai serveur RDP : non rejoués cette
  session (ni Xvfb ni xrdp montés ici — le harnais RDPDR occupait le
  temps disponible).
