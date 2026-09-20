[← index](README.md) — session 09 — 2026-09-05

## Imprimante/série/parallèle : confirmation définitive via le code source de xrdp (2026-09-05, nouvelle session)

Session courte, purement documentaire — pas de sandbox technique
requise. Tâche choisie : le dernier point du backlog moyenne priorité
jamais attaqué, « Trouver ou monter un serveur RDP supportant
réellement imprimante/série/parallèle ». Plutôt que de retenter un
setup xrdp+CUPS (déjà fait, déjà documenté comme échouant avec
`(not supported)` — voir plus haut dans ce fichier), la question posée
cette fois : **est-ce vraiment définitif, ou juste une histoire de
configuration/version qu'on n'aurait pas trouvée ?**

Réponse trouvée en lisant directement le code source actuel de xrdp
upstream (`neutrinolabs/xrdp`, fichier
`sesman/chansrv/devredir.c`, miroir Fossies consulté :
version 0.10.6.1, datée du 7 juillet 2026 — donc la plus récente
disponible au moment de cette vérification, pas une vieille copie
figée) :

- Ligne 4, commentaire d'en-tête du fichier, inchangé depuis la
  création du fichier par son auteur original en 2013 : **« xrdp
  device redirection - only drive redirection is currently
  supported »**.
- Dans `devredir_proc_client_devlist_announce_req()` (la fonction qui
  traite l'annonce de chaque périphérique redirigé par le client),
  `response_status` est mis à `STATUS_NOT_SUPPORTED` par défaut avant
  même de regarder de quel type de périphérique il s'agit (ligne 936-937,
  commentaire : « Assume this device isn't supported by us »).
- Le `switch (device_type)` qui suit ne contient que **deux** `case`
  qui repassent `response_status` à `STATUS_SUCCESS` :
  `RDPDR_DTYP_FILESYSTEM` (disque — via `xfuse_create_share()`) et
  `RDPDR_DTYP_SMARTCARD` (si `scard_device_announce()` réussit).
- Le `default:` (ligne 987) qui reçoit `RDPDR_DTYP_SERIAL`,
  `RDPDR_DTYP_PARALLEL` et `RDPDR_DTYP_PRINT` construit uniquement une
  chaîne descriptive pour le message de log (« serial port »/
  « parallel port »/« printer »), logge « Detected remote %s '%s'
  (not supported) », et **ne fait rien d'autre** — pas d'appel à une
  fonction d'implémentation, pas de branchement conditionnel vers un
  quelconque backend CUPS ou série. Rien nulle part ailleurs dans ce
  fichier (2657 lignes au total) ne référence `RDPDR_DTYP_PRINT`,
  `RDPDR_DTYP_SERIAL` ou `RDPDR_DTYP_PARALLEL` en dehors de cette seule
  ligne de description textuelle.

**Verdict, avec certitude cette fois (pas juste une observation
empirique comme précédemment)** : imprimante/série/parallèle ne sont
**jamais implémentés** dans xrdp, à aucune version connue jusqu'ici,
indépendamment de la configuration serveur (CUPS installé ou non,
démarré ou non, `xrdp.ini`/`sesman.ini` quels qu'ils soient). Le
message `(not supported)` que nos tests précédents ont observé n'est
pas un refus conditionnel qu'on pourrait lever — c'est un code jamais
écrit. Toute tentative future de « mieux configurer xrdp » pour ces
trois device types serait donc une perte de temps certaine ; la seule
voie reste de trouver ou monter un serveur RDP structurellement
différent (un vrai hôte Windows, ou un autre serveur Linux qui
implémente réellement `RDPDR_DTYP_PRINT`/`SERIAL`/`PARALLEL`) — ce qui
reste hors de portée de ce sandbox et donc non résolu, mais au moins la
question « est-ce notre faute / notre config ? » est close, avec
preuve à l'appui, plutôt que de continuer à planer sur le backlog.

Aucun fichier de code modifié cette session (recherche documentaire
pure) — seuls `features.md` et ce fichier mis à jour.
