[← index](README.md) — session 10 — 2026-09-05

## Audio réel, deuxième sens : capture (client → serveur) validée bout en bout (2026-09-05, nouvelle session)

Reprise du même sandbox (état disque persistant : binding compilé,
utilisateur `rdptest`, paquets `apt` déjà en place). Tâche choisie :
le seul point encore ouvert de la campagne audio de la session
précédente — « Capture (client → serveur) toujours non testée ».

### Confirmation immédiate d'un piège déjà documenté

Avant même de commencer le travail spécifique à l'audio, tous les
daemons en arrière-plan de la session précédente (`xrdp`, `xrdp-sesman`,
`pipewire`/`wireplumber`/`pipewire-pulse` des deux côtés) s'étaient
arrêtés — alors que `rdptest`, les paquets `apt` et le binding compilé
avaient bien persisté sur disque. Confirme et précise la note déjà
présente plus haut dans ce fichier : la non-persistance des processus
d'arrière-plan touche aussi deux réponses consécutives **au sein d'une
même tâche**, pas seulement des sessions de travail espacées dans le
temps. Tout relancé à l'identique (procédure inchangée, voir plus haut
dans ce fichier) : `xrdp-sesman`, `xrdp`, une connexion `asyncrdp`
minimale pour faire réapparaître `Xorg`/`chansrv` côté `rdptest`, puis
récupération des vraies variables d'environnement depuis
`/proc/<pid-chansrv>/environ` (le numéro de session/`DISPLAY` change à
chaque nouvelle connexion — `:11` cette fois, pas `:10` — donc bien
relire ces variables à chaque reprise plutôt que de réutiliser des
valeurs codées en dur d'une session précédente).

### Nouveauté par rapport à la session précédente : ordre des backends `audin`

Contrairement à `rdpsnd` (chargé une fois pour toutes avec le backend
demandé), le canal `audin` (capture) essaie plusieurs backends dans
l'ordre jusqu'à ce que l'un s'ouvre sans erreur. Constaté en clair dans
les logs `WLOG_LEVEL=DEBUG` :

```
[...] audin_pulse_connect: pa_context_connect failed (6)
[...] audin_load_device_plugin: pulse entry returned error 1359.
[...] audin_load_device_plugin: Loaded oss backend for audin
```

— **`pulse` est essayé en premier**, et seulement s'il échoue (pas de
serveur PulseAudio/PipeWire-Pulse joignable pour l'utilisateur qui
exécute le processus client) le code retombe sur `oss`, qui se charge
« avec succès » de façon purement déclarative (`/dev/dsp` n'existe pas
dans ce sandbox — l'échec réel, s'il y en a un, serait différé à la
première tentative d'ouverture du device, jamais atteinte dans ce test
puisque aucune donnée n'a été poussée dans ce cas). Point non documenté
avant cette session. Conséquence directe pour la suite : monter un vrai
serveur PulseAudio-compatible (ici `pipewire-pulse`) **côté client**
(pas seulement côté session serveur comme pour la lecture) est le
prérequis pour que `audin` utilise une voie réellement fonctionnelle.

### Mise en place du « faux micro » côté client

Le process client (`asyncrdp`, exécuté en `root` dans ce sandbox) n'a
par définition aucun micro physique. Plutôt que de chercher un module
noyau ALSA virtuel (déjà écarté pour la lecture, même limitation
`CAP_SYS_MODULE` indisponible), la solution retenue exploite PipeWire
lui-même : **en l'absence de toute carte audio réelle, PipeWire crée de
lui-même un sink `auto_null` et le pose comme sink ET source par défaut
(`auto_null.monitor`)** — pas besoin de charger explicitement
`module-null-sink` comme prévu initialement. Il suffit donc de :

1. Monter un `pipewire`/`wireplumber`/`pipewire-pulse` complet pour
   l'utilisateur `root` (même procédure que côté session `rdptest` pour
   la lecture : `XDG_RUNTIME_DIR=/run/user/0` créé manuellement,
   `dbus-launch` pour le bus de session — sans lui, `wireplumber`
   échoue immédiatement, comme déjà noté pour la session précédente).
2. Vérifier que `auto_null`/`auto_null.monitor` sont bien les
   sink/source par défaut (`pactl info`) — c'est le cas automatiquement,
   sans configuration supplémentaire.
3. Jouer un vrai fichier WAV (sinus 440 Hz, 8 s, généré avec le module
   `wave` de Python, même recette que le ton de test de la session
   précédente) avec `paplay -d auto_null` **pendant que la connexion
   `asyncrdp` est active** : le contenu de ce sink se retrouve sur son
   *monitor*, qui est la source par défaut que le backend `pulse`
   d'`audin` capture.

### Côté session serveur : réutilisation telle quelle de la recette de lecture

`xrdp-sink`/`xrdp-source` remontés exactement comme pour la session
précédente (`load_pw_modules.sh` avec les vraies variables lues sur
`chansrv`) — cette partie de la procédure s'est révélée directement
réutilisable sans adaptation, seul le numéro de session/`DISPLAY`
changeant d'une reprise à l'autre.

### Validation : un `parec` externe sur `xrdp-source`, pas seulement les logs client

Contrairement à `rdpsnd` (qui expose un compteur de bloc directement
dans les logs FreeRDP, `cBlockNo`), aucune chaîne de log équivalente
n'existe côté `audin` pour un transfert réussi (uniquement des messages
d'erreur en cas d'échec) — la preuve ne peut donc pas venir des logs du
client. Preuve construite autrement, à l'extérieur du process client :
un `parec --device=xrdp-source --format=s16le --rate=44100
--channels=2 --raw` lancé côté session `rdptest`, en parallèle de la
connexion `asyncrdp`, enregistre tout ce qui transite réellement par
`xrdp-source` pendant que le faux micro joue le ton de test.

Analyse du fichier obtenu (script Python ad hoc, aucune dépendance
externe) :
- Découpage en fenêtres de 0,5 s et calcul du RMS : silence strict
  (RMS = 0) avant et après une fenêtre d'environ 8 s au RMS parfaitement
  stable (~11584), correspondant exactement à la durée du fichier WAV
  joué — pas de fuite, pas de résidu.
- Sur une seconde de signal stable : 440 passages par zéro montants,
  soit une fréquence mesurée de **440,0 Hz exactement**.
- Amplitude crête mesurée : 16384, contre 16383,5 attendu
  (0,5 × 32767, l'amplitude programmée dans le générateur du ton de
  test) — écart d'une unité, imputable à l'arrondi entier.
- Canaux gauche et droit rigoureusement identiques (différence maximale
  nulle sur 500 échantillons comparés) — cohérent avec un WAV généré en
  dupliquant le même échantillon mono sur les deux canaux.
- Comparaison échantillon par échantillon contre la sinusoïde idéale
  (meilleur déphasage entier recherché sur les 100 premiers
  échantillons) : erreur RMS résiduelle d'environ 2 % de l'amplitude
  crête. Attribuée à la chaîne de rééchantillonnage interne de
  PipeWire (natif 48 kHz flottant, converti en 44,1 kHz entier 16 bits
  pour correspondre au format négocié par le canal RDPSND/`audin`) plutôt
  qu'à une perte du canal RDP — un déphasage à la fraction d'échantillon
  près (non cherché, seul le déphasage entier l'a été) expliquerait
  déjà l'essentiel de cet écart résiduel.

**Verdict** : la capture audio (client → serveur) est validée de bout
en bout avec un niveau de preuve plus complet que celui déjà établi
pour la lecture — jusqu'à un vrai consommateur côté session
(`parec`, qui est exactement le rôle que jouerait n'importe quelle
application de la session distante voulant lire le micro redirigé),
et pas seulement jusqu'à la négociation de canal ou un compteur de
bloc dans les logs. Aucune modification de code n'a été nécessaire :
`RdpOptions.audio_capture` (déjà présent, un simple booléen câblé sur
`FreeRDP_AudioCapture`) suffisait — le travail de cette session était
entièrement environnemental (mise en place du faux micro + preuve),
pas un correctif.

### Fichiers modifiés cette session

Aucun fichier de code (`src/`, `examples/`, `tests/`, `integrations/`)
— uniquement `features.md` et ce fichier. Scripts de test ad hoc
(`/tmp/test_audio_capture.py`, script d'analyse RMS/fréquence) non
versionnés, comme d'habitude pour ce type d'investigation ponctuelle.
