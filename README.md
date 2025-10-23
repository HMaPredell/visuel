# MANITOU DESADV D96A R

> Split & Traduction — vue d'ensemble

---

![status](https://img.shields.io/badge/status-ready-brightgreen) ![format](https://img.shields.io/badge/format-MD%20%2B%20Mermaid-blue)

## But

Document court pour expliquer comment le flux DESADV est traité :
- split (découpage UNH..UNT)
- traduction (EDIFACT -> XML)

Ce README est orienté exploitation : schéma, exemples de nommage, et réponses aux questions opérationnelles.

---

## Diagramme (Mermaid)

Pour afficher le diagramme dans VS Code : installe l'extension `Markdown Preview Mermaid Support` puis ouvrez l'aperçu Markdown (Ctrl+Shift+V).

```mermaid
flowchart TD
  inbox[inbox/edifact\n(fournisseur)]
  edisend_split[EDISEND: DESADV\n(MANITOU_DESADV_D96A_R) - split]
  split_dir[tmp/split_MANITOU_DESADV_D96A_R\n(messages UNH..UNT)]
  edisend_trad[EDISEND: TRAD_DESADV_MANI\n(MANITOU_DESADV_D96A_R_trad) - trad]
  outbox[outbox/\n(XML)]

  inbox --> edisend_split
  edisend_split -->|normalize -> split| split_dir
  split_dir --> edisend_trad
  edisend_trad -->|map -> xml| outbox
```

---

## Schéma ASCII (fallback)

```
inbox/edifact/        <-- Fournisseur dépose ici
    |
    v
[EDISEND: DESADV]  (MANITOU_DESADV_D96A_R)
    |
    v
tmp/split_MANITOU_DESADV_D96A_R/   <-- fichiers .txt créés par le splitter
    |
    v
[EDISEND: TRAD_DESADV_MANI] (MANITOU_DESADV_D96A_R_trad)
    |
    v
outbox/   <-- fichiers XML déposés ici
```

---

## Détails des composants

- Fournisseur : dépose l'interchange EDIFACT dans `inbox/edifact/`.
- EDISEND `DESADV` (`MANITOU_DESADV_D96A_R`) : lit/normalize, split UNH..UNT, écrit les fichiers split dans `tmp/split_MANITOU_DESADV_D96A_R/`.
- EDISEND `TRAD_DESADV_MANI` (`MANITOU_DESADV_D96A_R_trad`) : lit les splits, mappe les segments et écrit un XML validé dans `outbox/`.

---

## Conventions de nommage

- Splitter :
  - Dossier : `<sHOME>/split_MANITOU_DESADV_D96A_R/`
  - Pattern : `SPLIT_DESADV_<index>_<timestamp>_<n>.txt`
    - `<index>` : pINDEX (ou `0` si absent)
    - `<timestamp>` : `YYYYMMDDhhmmss` (ex: `20251022T143012`)
    - `<n>` : compteur incrémental
  - Exemple : `SPLIT_DESADV_0_20251022T143012_1.txt`

- Traducteur (XML) :
  - Dossier : `outbox/`
  - Pattern typique (voir `bfWriteMsg()` pour l'exact) :
    - `DESADV_<Sender>_<Receiver>_<DocNumber>_<DTM132>_<timestamp>.xml`

---

## Questions & réponses

### Q1 — Reprocess échoue sur `edidev` : `Not Found: Base archive is not installed`.

Cause probable : le composant DocumentTracker (ou sa base) n'est pas installé/configuré sur l'environnement.

Actions :
1. Vérifier la présence et l'accessibilité de `documenttracker.cfg`.
2. Vérifier que la DB / service d'archive est disponible.
3. Pour tests : ajouter un fallback dans le RTE (skip archive) temporairement, mais ce n'est pas recommandé en prod.

### Q2 — Peut-on supprimer le 2ème EDISEND et appeler le traducteur depuis le split ?

Oui, techniquement. Points à considérer :

- Avantages : moins d'I/O disque, traitement plus linéaire.
- Inconvénients : perte d'isolation; relance/diagnostic plus compliqué; plus de responsabilités dans un seul binaire.

Recommandation : garder la séparation pour faciliter l'exploitation et la relance de messages individuels.

---

## Astuces / opérations courantes

- Pour afficher correctement le README dans VS Code :
  1. Installer `Markdown Preview Mermaid Support`.
  2. Ouvrir le fichier et faire `Ctrl+Shift+V`.
- Pour lister les splits prêts à traduire :
  - PowerShell : `Get-ChildItem "$env:HOME\tmp\split_MANITOU_DESADV_D96A_R"`
- Nettoyage : configurer une tâche périodique pour purger `tmp/split_MANITOU_DESADV_D96A_R/` après X jours.

---

Si tu veux que je :
- génère une image PNG du diagramme et l'ajoute au README, ou
- écrive un petit script PowerShell pour relancer la traduction des fichiers split,

dis‑moi et je le fais.
