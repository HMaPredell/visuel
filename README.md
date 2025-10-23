# MANITOU DESADV D96A R 
## Schéma du flux


```mermaid
flowchart LR

  %% Styles
  %% Neutral styles
  classDef pastelA fill:#fff8f0,stroke:#cfc3b8,stroke-width:1px,color:#000;
  classDef pastelB fill:#f7fff6,stroke:#cfe8d8,stroke-width:1px,color:#000;
  classDef pastelC fill:#f3f7ff,stroke:#cfd8ec,stroke-width:1px,color:#000;

  %% large nodes, left-aligned preformatted text
  style A width:900px;font-size:13px;text-align:left
  style B width:900px;font-size:13px;text-align:left
  style C width:900px;font-size:13px;text-align:left
  style D width:900px;font-size:13px;text-align:left
  style E width:900px;font-size:13px;text-align:left

  A["<div style='white-space:pre; text-align:left;'>inbox/edifact/   <-- Fournisseur dépose ici</div>"]:::pastelA
  B["<div style='white-space:pre; text-align:left;'>[EDISEND: DESADV]  (binaire: MANITOU_DESADV_D96A_R)   <-- processeur initial</div>"]:::pastelB
  C["<div style='white-space:pre; text-align:left;'>tmp/split_MANITOU_DESADV_D96A_R/   <-- fichiers .txt crées par le splitter</div>"]:::pastelC
  D["<div style='white-space:pre; text-align:left;'>[EDISEND: TRAD_DESADV_MANI] (binaire: MANITOU_DESADV_D96A_R_trad)</div>"]:::pastelB
  E["<div style='white-space:pre; text-align:left;'>outbox/   <-- fichiers XML déposés ici</div>"]:::pastelC

  A --> B
  B -->|"<div style='white-space:pre; text-align:left;'>(1) lit `inbox/edifact/`<br/>(2) split : découpe le flux EDIFACT en messages (UNH..UNT)</div>"| C
  C --> D
  D -->|"<div style='white-space:pre; text-align:left;'>(1) lit chaque fichier split<br/>(2) traduit le message EDIFACT -> XML</div>"| E
  B -->|"(1) lit `inbox/edifact/`  (2) split : découpe le flux EDIFACT en messages (UNH..UNT)"| C
  C --> D
  D -->|"(1) lit chaque fichier split  (2) traduit le message EDIFACT -> XML"| E
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
