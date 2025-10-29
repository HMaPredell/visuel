# MANITOU DESADV D96A R — README

<small>Version: package_install_rte_20251027 — Documentation</small>

---

## Table des matières

- [MANITOU DESADV D96A R — README](#manitou-desadv-d96a-r--readme)
  - [Table des matières](#table-des-matières)
  - [1) Résumé rapide](#1-résumé-rapide)
  - [2) Ce qui change (avant / après)](#2-ce-qui-change-avant--après)
  - [3) Nouveautés et pourquoi](#3-nouveautés-et-pourquoi)
  - [4) Comment ça fonctionne (schéma)](#4-comment-ça-fonctionne-schéma)
  - [5) Arborescence attendue (exemple)](#5-arborescence-attendue-exemple)
  - [6) Commandes (install / rollback)](#6-commandes-install--rollback)
  - [7) Que fait exactement `install` et `rollback` (explication simple)](#7-que-fait-exactement-install-et-rollback-explication-simple)
    - [7.a) `install_manitou_update_20251027.sh`](#7a-install_manitou_update_20251027sh)
    - [7.b) `rollback_manitou_update_20251027.sh`](#7b-rollback_manitou_update_20251027sh)
  - [8) Vérifications post-installation (rapide et simple)](#8-vérifications-post-installation-rapide-et-simple)
  

---

## 1) Résumé rapide

- On a séparé le RTE monolithique en deux : un RTE split (découpe) et un RTE trad (traduction).
- But simple : rendre le traitement plus fiable et plus facile à gérer.

---

## 2) Ce qui change (avant / après)

- Avant : un seul RTE faisait tout.
- Maintenant :
  - `MANITOU_DESADV_D96A_R_split.rte` — fait le split (sortie : `/inbox/split_MANITOU_DESADV_D96A_R/`).
  - `MANITOU_DESADV_D96A_R.rte` — fait la traduction (lit le dossier split et écrit dans `outbox/`).

---

## 3) Nouveautés et pourquoi

- Séparation des tâches : split / trad.
- Moins de risques : une erreur touche seulement un message.
- Plus simple à tester et à corriger.

---

## 4) Comment ça fonctionne (schéma)

Le schéma ci-dessous montre le flux. J'ai simplifié les textes pour éviter que les boîtes ne coupent les mots.

<a id="4-comment-ca-fonctionne-schema"></a>
```mermaid
flowchart TD
  classDef bigBlock fill:#fffaf8,stroke:#d0a0a0,stroke-width:2px,color:#000,font-size:14px;
  classDef stepBlock fill:#fcfdf6,stroke:#e6d9b2,stroke-width:1px,color:#000,font-size:12px;

  FOURNISSEUR["FOURNISSEUR\n(ENVOIE)"]:::bigBlock
  EDISEND_SPLIT["EDISEND: DESADV\nMANITOU_DESADV_D96A_R_split"]:::stepBlock
  SPLIT_DIR["/inbox/split_MANITOU_DESADV_D96A_R/\n(.edi par message)"]:::stepBlock
  EDISEND_TRAD["EDISEND: TRAD_DESADV_MANI\nMANITOU_DESADV_D96A_R.rte"]:::stepBlock
  OUTBOX["outbox/\n(XML)"]:::stepBlock
  MEC["MEC\n(endpoint)"]:::bigBlock

  FOURNISSEUR -->|OFTP2 / AS2| EDISEND_SPLIT
  EDISEND_SPLIT -->|split| SPLIT_DIR
  SPLIT_DIR -->|lire| EDISEND_TRAD
  EDISEND_TRAD -->|xml| OUTBOX
  OUTBOX --> MEC
```

---

## 5) Arborescence attendue (exemple)

```
edidev/
├── rte/
│   ├── ARCHIVE/                          # archives des anciens RTE
│   ├── MANITOU_DESADV_D96A_R_split.rte   # split
│   └── MANITOU_DESADV_D96A_R.rte         # trad
├── inbox/
│   ├── edifact/                          # interchanges entrants
  └── split_MANITOU_DESADV_D96A_R/      # sorties du splitter (.edi)
└── outbox/                               # XML générés
└── package_install_rte_20251027/         # paquet d'installation (scripts + dumps)
```

---

## 6) Commandes (install / rollback)

Exécutez ces commandes depuis le dossier du paquet (`package_install_rte_20251027`) :

Installer :

```bash
cd /chemin/vers/edidev/package_install_rte_20251027
./install_manitou_update_20251027.sh
```

Rollback :

```bash
cd /chemin/vers/edidev/package_install_rte_20251027
./rollback_manitou_update_20251027.sh
```

---

## 7) Que fait exactement `install` et `rollback` (explication simple)

### 7.a) `install_manitou_update_20251027.sh`

1. Sauvegarde l'entrée edisend et le routing quand possible.
2. Modifie le routing pour utiliser le RTE split (édition directe du fichier de contrôle).
3. Copie les nouveaux RTE dans `rte/` et archive l'ancien dans `rte/ARCHIVE/`.
4. Importe le dump edisend `TRAD_DESADV_MANI` fourni si l'entrée manque.
5. Compile les RTE si l'outil `mktr` est disponible.

### 7.b) `rollback_manitou_update_20251027.sh`

1. Supprime les RTE installés (split + trad).
2. Supprime l'entrée edisend `TRAD_DESADV_MANI` si elle existe.
3. Remet `TRANSLATOR=` du routing sur `MANITOU_DESADV_D96A_R` (édition directe du fichier de contrôle).
4. Restaure la dernière archive RTE trouvée dans `rte/ARCHIVE/`.
5. IMPORTANT : rollback NE restaure PAS automatiquement les dumps ; toute réimportation reste manuelle.

---

## 8) Vérifications post-installation (rapide et simple)

1. Déposez un interchange EDIFACT de test dans `inbox/edifact/`.
2. Vérifiez que `inbox/split_MANITOU_DESADV_D96A_R/` contient un fichier `.edi` par message.
3. Vérifiez que `outbox/` reçoit les XML produits.
4. Si le routing n'a pas pris la modification, redémarrez le service TradExpress selon vos procédures.

---
