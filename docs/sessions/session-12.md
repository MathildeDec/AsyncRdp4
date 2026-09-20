[← index](README.md) — session 12 — 2026-09-05

## Comparaison avec `PATTERNS.md` d'un autre projet (squelette Python/GTK4, 2026-09-05)

Un catalogue de règles non métier (`PATTERNS.md`, projet dérivé
« switch-capture »/squelette générique, threading + GTK4 pur, pas de
cffi ni d'asyncio) a été fourni pour comparaison. La majorité de ses
sections (architecture single-exécutable, pages GTK4 multiples,
diagnostic sudo+GUI+RDP, chaîne de résolution de secrets, i18n,
packaging) ne sont pas des règles de code propre/test au sens strict
et ne s'appliquent de toute façon pas ici : `asyncrdp` est une
bibliothèque cffi/asyncio consommée par un futur plugin GCM, pas une
application GTK4 autonome avec sa propre CLI/fenêtre/persistance de
préférences. Seules les sections tests (7) et discipline de code (10,
11, 12) étaient potentiellement pertinentes ; vérification faite dans
le code réel (`grep`/lecture), pas par supposition :

**Déjà en place ici, et déjà documenté :**
- Suite pytest hermétique (logique pure, stub `gi`/`asyncrdp`) séparée
  des scénarios qui ont vraiment besoin d'un environnement réel
  (`ASYNCRDP_TEST_GTK4=1` + Xvfb) — même principe que PATTERNS.md
  section 7. Voir « Stub GTK4 réalisé » plus haut et `features.md`.
- Isolation `sys.modules` lors du mock de `gi`/`asyncrdp`
  (sauvegarde/restauration exacte dans `_import_bridges_with_stub()`)
  — même motif que PATTERNS.md section 7. Voir « Stub GTK4 réalisé »
  plus haut.
- Journal de session chronologique (PATTERNS.md section 9) — c'est ce
  que ce fichier fait déjà depuis le début du projet.

**Absent du code, et non documenté nulle part — écart réel, pas un
choix explicite :**
- Pas de décorateur de traçabilité systématique entrée/sortie
  (`tracing.py::traced` dans PATTERNS.md section 10). `_core.py` pose
  des `logger.debug()` ponctuels à la main à certains points
  (négociation multi-écran, RD Gateway, RemoteApp...), pas une
  couverture garantie structurellement sur chaque fonction.
- Pas de test dédié d'absence de dépendances circulaires (analyse
  `ast` + parcours DFS, PATTERNS.md section 11).
- Pas de hook pre-commit relançant la suite à chaque commit
  (PATTERNS.md section 12) — aucun `.pre-commit-config.yaml` dans le
  projet ; la discipline « un test par motif » existe, mais son
  exécution reste manuelle.

**Non transposable tel quel (différence de conception, pas un
manque) :**
- Le garde-fou `require_gtk4()` contre `gi.require_version()` qui
  bloquerait toute la collecte pytest (PATTERNS.md section 7) :
  `asyncrdp` n'importe jamais `gi` au niveau module dans ses tests —
  uniquement à l'intérieur du corps de chaque test, derrière le
  fixture `gtk4_live_enabled` qui skip avant d'y arriver (voir
  `tests/test_gtk4_live.py`/`conftest.py`). Le problème que ce
  garde-fou résout ailleurs ne se pose donc pas ici par construction,
  mais ce n'était pas un choix explicite tracé avant cette
  comparaison — désormais noté pour éviter qu'un futur test GTK4
  réintroduise l'import au niveau module sans y repenser.
- Le principe « fakes conçus pour échouer bruyamment sur un appel
  illégal » (PATTERNS.md section 7, `FakeKeyringModule`/
  `FakePyKeePass`) : sans objet ici (`asyncrdp` ne gère aucun secret/
  trousseau côté code testé) — le stub GTK4 (`tests/gtk_stub/`) fait
  d'ailleurs l'inverse par nécessité, un `_AnyCall` passe-partout qui
  accepte n'importe quel appel, puisqu'il doit couvrir toute la
  surface d'API GTK4 non testée en détail, pas une poignée de méthodes
  précises comme un backend de secrets.

**Pas d'action de code prévue à ce stade** : ces trois derniers points
restent ouverts, à traiter dans une session future si le projet en a
besoin (le décorateur de traçabilité et le test anti-cycle sont les
deux candidats les plus simples à porter tels quels si souhaité).
