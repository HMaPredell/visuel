# MANITOU DESADV D96A R 
## Schéma du flux


```mermaid
flowchart TD
  %% Styles
  %% Neutral styles
  classDef pastelA fill:#fff8f0,stroke:#cfc3b8,stroke-width:1px,color:#000;
  classDef pastelB fill:#f7fff6,stroke:#cfe8d8,stroke-width:1px,color:#000;
  classDef pastelC fill:#f3f7ff,stroke:#cfd8ec,stroke-width:1px,color:#000;

  A["inbox/edifact/   &lt;-- Fournisseur d&eacute;pose ici"]:::pastelA
  B["[EDISEND: DESADV]  (binaire: MANITOU_DESADV_D96A_R)   &lt;- processeur initial"]:::pastelB
  C["tmp/split_MANITOU_DESADV_D96A_R/   &lt;-- fichiers .txt cr&eacute;es par le splitter"]:::pastelC
  D["[EDISEND: TRAD_DESADV_MANI] (binaire: MANITOU_DESADV_D96A_R_trad)"]:::pastelB
  E["outbox/   &lt;-- fichiers XML d&eacute;pos&eacute;s ici"]:::pastelC

  A --> B
  B -->|"(1) lit `inbox/edifact/`<br/>(2) split : d&eacute;coupe le flux EDIFACT en messages (UNH..UNT)"| C
  C --> D
  D -->|"(1) lit chaque fichier split<br/>(2) traduit le message EDIFACT -> XML"| E

  %% widen nodes to reduce text wrapping
  style A width:550px
  style B width:650px
  style C width:700px
  style D width:650px
  style E width:550px
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
