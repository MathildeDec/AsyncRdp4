# Roadmap — AsyncRdp4

> Dernière mise à jour : 2026-09-21
> Source : GitHub issues, `docs/features-backlog.md`

## Vue d'ensemble

| Métrique | Valeur |
|----------|--------|
| Issues ouvertes | 5 (dont 4 bloquées par environnement) |
| Issues closes | 33 (sessions) |
| Code livré | Visualiseur RDP GTK4 + RdpSession + USB + imprimante |

## Phases d'intégration

### Phase 1 — Code livré ✅

| Composant | Statut | Détail |
|-----------|--------|--------|
| Visualiseur RDP GTK4 | ✅ Livré | `gcm_gtk4_demo_viewer.py` + `RdpSession` |
| Imprimante/série/parallèle | ✅ Code livré | Client prouvé correct pour imprimante |
| USB redirection | ✅ Code livré | Énumération + règle udev |
| Bug PyGObject contourné | ✅ Contourné | Sous-classe C GdkContentProvider |
| Bug FreeRDP documenté | ✅ Rapport rédigé | `docs/pygobject-async-vfunc-userdata-bug-report.md` |

### Phase 2 — Intégration GCM 🔄

| Issue | Titre | Dépend de | Statut |
|-------|-------|-----------|--------|
| #36 | Assembler le plugin RDP dans GCM | Gcm4 #71, #96, #97 | Bloqué (migration GTK4 Gcm4 en cours) |

Le visualiseur de référence est prêt. L'intégration dans GCM dépend de l'avancement de la migration GTK4 de Gcm4 :
- Les 3 chantiers GTK4 restants sur Gcm4 (#96, #97, #71) doivent être avancés
- Une fois le portage RDP (#71) en cours, l'intégration asyncrdp peut se faire en parallèle
- Si gtk-frdp n'a pas de chemin GTK4, asyncrdp remplace le plugin RDP natif

### Phase 3 — Validation environnement 🔜 (bloqué)

| Issue | Titre | Blocage | Parallélisable |
|-------|-------|---------|----------------|
| #34 | Imprimante/série/parallèle | Hôte Windows RDP requis | Avec #38 |
| #35 | Rapport bug PyGObject | Compte GitLab GNOME | Avec #38 |
| #37 | USB redirection | Machine + matériel USB + serveur RDP | Avec #34 |
| #38 | Rapport bug FreeRDP | Compte GitHub FreeRDP | Avec #35 |

Ces issues sont indépendantes du code — elles nécessitent uniquement l'accès aux environnements/comptes externes.

## Voir aussi

- [Backlog complet](features-backlog.md)
- Dépendance cross-repo : Gcm4 [#71](https://github.com/MathildeDec/Gcm4/issues/71), [#96](https://github.com/MathildeDec/Gcm4/issues/96), [#97](https://github.com/MathildeDec/Gcm4/issues/97)
