# MANITOU DESADV D96A R 
## Schéma du flux


```mermaid
flowchart TD
  %% Styles
  classDef inboxStyle fill:#ffefef,stroke:#b22222,stroke-width:1px;
  classDef splitStyle fill:#fff7cc,stroke:#b8860b,stroke-width:1px;
  classDef tradStyle fill:#e8fff0,stroke:#228b22,stroke-width:1px;
  classDef outStyle fill:#e6f0ff,stroke:#1e90ff,stroke-width:1px;

  inbox["inbox/edifact<br/> (fournisseur)"]:::inboxStyle
  edisend_split["EDISEND: DESADV<br/>MANITOU_DESADV_D96A_R<br/>(split)"]:::splitStyle
  split_dir["tmp/split_MANITOU_DESADV_D96A_R<br/>(messages UNH..UNT)"]:::splitStyle
  edisend_trad["EDISEND: TRAD_DESADV_MANI<br/>MANITOU_DESADV_D96A_R_trad<br/>(trad)"]:::tradStyle
  outbox["outbox/<br/>(XML)"]:::outStyle

  inbox --> edisend_split
  edisend_split -->|normalize → split| split_dir
  split_dir --> edisend_trad
  edisend_trad -->|map → xml| outbox
```


## Que fait chaque composant ?

- Fournisseur : place un fichier EDIFACT (interchange contenant possiblement plusieurs messages UNH..UNT) dans `inbox/edifact`.
- EDISEND DESADV (MANITOU_DESADV_D96A_R) : prend le fichier d'entrée, puis découpe chaque message UNH..UNT et écrit un fichier par message dans `tmp/split_MANITOU_DESADV_D96A_R/`.
- EDISEND TRAD_DESADV_MANI (MANITOU_DESADV_D96A_R_trad) : lit chaque fichier split, applique la logique de mapping et produit un fichier XML, écrit dans `outbox/`.

## Conventions de nommage des fichiers

- Fichiers split produits par le splitter :
  - Chemin : `<sHOME>/split_MANITOU_DESADV_D96A_R/`
  - Nom : `SPLIT_DESADV_<index>_<timestamp>_<n>.txt`
    - `<index>` : pINDEX (si disponible dans l'environnement), sinon valeur vide ou `0`.
    - `<timestamp>` : format `YYYYMMDDhhmmss` (ex: 20251022...)
    - `<n>` : compteur incrémental pour chaque message extrait pendant l'exécution du splitter.
  - Exemple : `SPLIT_DESADV_0_20251022T143012_1.txt`

- Fichiers XML produits par le traducteur :
  - Emplacement : `outbox/`
  - Le traducteur écrit le nom selon `bfWriteMsg()` : il combine typiquement des informations comme `taDocTracker["SenderCode"]`, `taDocTracker["ReceiverCode"]`, `taDocTracker["DocNumber"]`, la date DTM132, et un timestamp. Exemple de pattern :
    - `DESADV_<Sender>_<Receiver>_<DocNumber>_<DTM132>_<timestamp>.xml`



## Questions et réponses

Q1 — Le reprocess ne marche pas sur edidev (erreur: `Not Found: Base archive is not installed`).

- Cause probable (par GPT) : la base `documenttracker.cfg` ou la base d'archive attendue par le code n'est pas installée/configurée sur l'environnement `edidev`. Le message `Base archive is not installed` indique que le runtime cherche une base (BDD locale ou config) qui n'existe pas.


Q2 — On pourrait enlever le deuxième edisend et appeler le traducteur directement depuis le RTE split (faire l'appel au traducteur après le split).

- Avantages de supprimer le 2ème edisend :
  - Moins de latence / Un edisend en moins qui fait du as file drive.

- Inconvénients / risques :
  - Perte de séparation claire des responsabilités : splitter vs traducteur. En cas d'erreur de traduction, il peut être plus difficile de relancer uniquement la traduction sans refaire le split.
