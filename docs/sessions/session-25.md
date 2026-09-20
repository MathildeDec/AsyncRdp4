---
name: session-25
---

[← index](README.md)

# Session 25 (2026-09-11) — Recherche d'un rapport de bug PyGObject existant et rédaction d'un rapport prêt à déposer pour le bug presse-papier image

## Contexte

Reprise après la session 24 (veille). `CLAUDE.md` proposait deux pistes
concrètes pour aller plus loin sur le bug presse-papier image
(marshaling PyGObject du closure `(callback, user_data)` de
`write_mime_type_async`, diagnostiqué la veille) : (1) chercher ou
ouvrir un rapport de bug PyGObject existant sur ce marshaling ; (2)
écrire une sous-classe `GdkContentProvider` en C. La première est à
portée d'une seule session (recherche + rédaction), la seconde
nécessiterait un vrai fichier C compilé et intégré au shim existant —
gardée pour plus tard. USB et imprimante/série/parallèle : toujours
hors de portée de ce sandbox (pas de matériel physique, pas de serveur
RDP alternatif), inchangé.

## Recherche d'un rapport existant

Plusieurs recherches ciblées (message d'assertion exact, combinaisons
de mots-clés `PyGObject`/`GAsyncReadyCallback`/`vfunc`/`user_data`/
`GdkContentProvider`/`write_mime_type_async`) sur le web et directement
sur `gitlab.gnome.org` (accessible depuis les outils de recherche web,
contrairement à `curl` dans le sandbox bash — confirmé cette session :
`web_fetch` a pu charger des pages `gitlab.gnome.org` sans blocage).
Tentative de requête sur l'index de recherche natif de GitLab
(`?scope=all&search=...`) : rejetée (erreur HTTP 406, rendu
probablement dépendant de JavaScript) ; l'API REST GitLab
(`/api/v4/projects/.../issues`) n'a pas pu être essayée non plus — une
URL construite en modifiant le chemin d'une URL déjà vue est refusée
par l'outil `web_fetch` (protection anti-devinette d'URL), et cette
URL d'API précise n'était apparue dans aucun résultat de recherche.
**Aucun rapport existant trouvé correspondant exactement à ce
scénario** (implémentation Python d'un vfunc GDK async avec paire
`(callback, user_data)`) — recherche par mots-clés uniquement, donc pas
une certitude absolue.

En revanche, trouvé en cours de route, deux éléments qui corroborent
le diagnostic de la session 24 sans le remplacer :
- Une page d'analyse GNOME (« Object Reference Counting for VFuncs and
  Closures », 2012-2013) recensant neuf bugs historiques distincts
  autour du marshaling GObject pour les vfuncs/closures Python — la
  zone est documentée de longue date comme fragile, même si aucun de
  ces neuf bugs ne correspond précisément au nôtre (ils portent sur le
  comptage de références d'objets retournés/passés, pas sur la perte
  d'un `gpointer user_data` non typé).
- Une merge request PyGObject (`!158`) décrivant le mécanisme
  `GI_SCOPE_TYPE_ASYNC` utilisé pour les fonctions `GAsyncReadyCallback`
  — mais uniquement testé/exercé dans le sens Python-appelle-C (ex.
  `clipboard.read_async(...)`), jamais dans le sens inverse (C appelle
  un vfunc Python qui reçoit ce couple en argument) : hypothèse de
  travail plausible pour expliquer pourquoi ce chemin précis est moins
  éprouvé, mais le code interne de PyGObject lui-même n'a pas été lu
  pour la confirmer — présentée comme piste, pas comme diagnostic
  confirmé.
- Un billet de blog d'un développeur GNOME senior (projet Builder)
  témoignant, indépendamment de ce projet, avoir abandonné PyGObject
  pour plusieurs de ses greffons internes précisément à cause de
  problèmes de binding autour de ce style de code asynchrone — pas une
  preuve technique, mais une corroboration indépendante que la
  difficulté rencontrée ici n'est pas isolée à ce projet.

## Rapport de bug rédigé

`docs/pygobject-async-vfunc-userdata-bug-report.md` — contenu en
anglais (langue de travail du projet GNOME), prêt à coller dans un
nouveau ticket sur `gitlab.gnome.org/GNOME/pygobject/-/issues/new`.
Reprend, sous une forme indépendante de ce projet et de sa suite de
tests (repro minimale autonome, une seule classe, pas de dépendance à
`asyncrdp`) : environnement exact (PyGObject 3.48.2, GTK 4.14.5, Python
3.12.3, Ubuntu 24.04.4), résumé, étapes de reproduction, résultat
attendu/obtenu, analyse de cause (extraits de source GTK déjà réunis le
2026-09-10), liste des pistes déjà écartées empiriquement, et
l'hypothèse `GI_SCOPE_TYPE_ASYNC` ci-dessus clairement présentée comme
non confirmée. Ce projet (`asyncrdp`) n'a pas de compte GitLab GNOME
configuré ici pour déposer le ticket lui-même — document laissé prêt à
l'emploi pour que l'utilisateur (ou une session future avec les accès
nécessaires) le dépose.

Référence ajoutée dans la raison `xfail` du test
`test_clipboard_image_write_bug_survives_direct_callback_invocation`
(`tests/test_gtk4_live.py`) pointant vers ce nouveau document.

## Vérification

Binding recompilé (`_asyncrdp_cffi.abi3.so` supprimé en fin de session
24 lors du nettoyage avant zip — recompilé ici avec succès,
`freerdp3-dev`/`libwinpr3-dev` 3.31.0 toujours disponibles via `apt`).
`ruff check .` : 0 erreur. Xvfb relancé (mort entre les deux réponses,
comme attendu — voir `docs/test-environment.md`). Suite complète
rejouée : **101 passed, 3 xfailed, 0 failed**, identique à la fin de la
session 24 — aucune régression, cette session n'a touché que de la
documentation et une chaîne de caractères dans un `xfail`.

## Ce qui n'a pas été fait

Le ticket n'a pas été déposé sur `gitlab.gnome.org` (pas d'accès/compte
disponible ici) — seul le contenu prêt à déposer a été produit. La
sous-classe `GdkContentProvider` en C (deuxième piste possible) n'a pas
été tentée cette session, gardée pour une session dédiée si le rapport
de bug ne suffit pas à débloquer les choses côté PyGObject. USB
(matériel physique) et imprimante/série/parallèle (serveur RDP
alternatif) : toujours inchangés, hors de portée de ce sandbox.

## Prochaine étape

Trois points toujours ouverts, inchangés dans leur nature : USB de bout
en bout (matériel), imprimante/série/parallèle (serveur RDP
alternatif), presse-papier image GTK4 (dépose du ticket PyGObject
ci-dessus par l'utilisateur, et/ou tentative d'une sous-classe
`GdkContentProvider` en C si une session future veut aller plus loin
sans attendre une réponse upstream). Continuer sur les features à
faire sans attendre de décision sur ces trois points reste la consigne
en vigueur.
