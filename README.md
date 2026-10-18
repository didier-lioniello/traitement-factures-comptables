# Traitement automatique des factures et relevés comptables (n8n)

## Objectif
Ne plus saisir les factures à la main. Les factures fournisseurs et les relevés bancaires arrivent par email ou dans un dossier Drive ; n8n les lit (OCR + extraction IA), vérifie les montants et la TVA, détecte les doublons, rapproche chaque facture de son paiement et exporte le tout vers le logiciel comptable. **Seules les anomalies demandent une intervention humaine.**

## Schéma

```mermaid
flowchart TD
    subgraph RECEP[1. Réception]
        A1[Gmail Trigger<br/>pièces jointes PDF / images] --> A3
        A2[Google Drive Trigger<br/>dossier Factures à traiter] --> A3
        A3[Classement : facture, avoir,<br/>relevé bancaire, autre] --> A4{Type ?}
    end
    A4 -- Autre --> A5[Ignorer + log]
    A4 -- Relevé bancaire --> RB
    A4 -- Facture ou avoir --> B1

    subgraph EXTRACT[2. OCR + extraction IA]
        B1{PDF avec texte ?} -- Oui --> B2[Extract from File]
        B1 -- Non, scan ou photo --> B3[OCR<br/>API OCR via HTTP Request]
        B2 --> B4[LLM : extraction JSON<br/>fournisseur, SIREN, n° facture, date,<br/>lignes, HT, TVA par taux, TTC, IBAN]
        B3 --> B4
        B4 --> B5{JSON conforme<br/>au schéma ?}
        B5 -- Non, 1er essai --> B4
        B5 -- Non, 2e échec --> ANO
    end

    B5 -- Oui --> C1

    subgraph CHECK[3. Contrôles]
        C1[Code : HT + TVA = TTC ?<br/>somme des lignes ?] --> C2[Taux de TVA autorisés ?<br/>liste paramétrée]
        C2 --> C3[Mentions obligatoires<br/>présentes ?]
        C3 --> C4{Doublon ?<br/>fournisseur + n° + montant<br/>ou empreinte du fichier}
        C4 -- Oui --> ANO
        C4 -- Non --> C5{Tous les contrôles OK ?}
        C5 -- Non --> ANO
    end

    C5 -- Oui --> D1[(Registre Postgres<br/>facture en attente de paiement)]

    subgraph RAPPRO[4. Rapprochement bancaire]
        RB[Import relevé<br/>CSV / OFX / API bancaire] --> RB2[Loop Over Items<br/>chaque opération]
        RB2 --> RB3[Recherche facture :<br/>montant, date proche,<br/>libellé, référence]
        D1 -.-> RB3
        RB3 --> RB4{Correspondance ?}
        RB4 -- Unique et exacte --> RB5[Marquer payée<br/>lier opération et facture]
        RB4 -- Plusieurs ou partielle --> ANO
        RB4 -- Aucune --> RB6[Opération sans justificatif<br/>liste à compléter]
    end

    RB5 --> E1
    D1 --> E1

    subgraph EXPORT[5. Export comptable]
        E1[Imputation proposée<br/>compte fournisseur, charge, TVA] --> E2[Export logiciel comptable<br/>API ou fichier d'import]
        E2 --> E3{Export accepté ?}
        E3 -- Non --> E4[Retry x3 puis alerte]
        E3 -- Oui --> E5[Archivage PDF<br/>Drive classé par mois]
    end

    subgraph HUMAIN[6. Validation humaine des anomalies]
        ANO[File d'anomalies<br/>motif + document + extraction] --> ANO2[Notification Slack / email<br/>au comptable]
        ANO2 --> ANO3[Wait : corriger,<br/>valider ou rejeter]
        ANO3 -- Corrigé --> C1
        ANO3 -- Rejeté --> ANO4[Statut rejeté + motif]
    end

    RB6 --> ANO2
    E4 --> ANO2
    ERR[Error Trigger] --> ERR2[Alerte + document<br/>déplacé dans À revoir]
    ERR2 --> ANO
```

## Explication étape par étape

1. **Réception** : un `Gmail Trigger` récupère les pièces jointes des emails reçus sur une adresse dédiée (par exemple `factures@…`) ; un `Google Drive Trigger` surveille un dossier « Factures à traiter » où l'on dépose les scans.
2. **Classement** : un `Text Classifier` distingue facture, avoir, relevé bancaire et document hors sujet.
3. **Lecture du document** : si le PDF contient du texte, `Extract from File` suffit. S'il s'agit d'un scan ou d'une photo, on passe par un service OCR appelé en `HTTP Request` (au choix : OCR cloud, modèle vision, outil auto-hébergé).
4. **Extraction IA en JSON** : un `Information Extractor` (ou LLM + `Structured Output Parser`) produit un JSON strict : fournisseur, SIREN/SIRET, numéro et date de facture, lignes, total HT, TVA par taux, TTC, échéance, IBAN. En cas de JSON invalide, un second essai est tenté, puis la facture part en anomalie.
5. **Contrôles arithmétiques** : un nœud `Code` vérifie que HT + TVA = TTC et que la somme des lignes correspond au total, avec une tolérance d'arrondi de quelques centimes à paramétrer.
6. **Contrôle de la TVA** : les taux détectés sont comparés à une liste autorisée. En France métropolitaine, les taux courants sont 20 %, 10 %, 5,5 % et 2,1 % ; **cette liste et les cas particuliers (autoliquidation, intracommunautaire, franchise) sont à faire valider par l'expert-comptable** avant mise en production.
7. **Mentions obligatoires** : présence du numéro de facture, de la date, de l'identité du fournisseur et des montants. La liste exacte est à définir avec le comptable.
8. **Doublons** : on cherche une facture existante avec le même fournisseur, le même numéro et le même montant, et on compare aussi l'empreinte du fichier (nœud `Crypto`). Un doublon n'est jamais exporté.
9. **Registre** : les factures valides sont enregistrées dans Postgres avec le statut « en attente de paiement ».
10. **Rapprochement bancaire** : le relevé (CSV, OFX ou API bancaire) est importé, puis chaque opération (`Loop Over Items`) est comparée aux factures en attente : montant, date proche, libellé et référence. Une correspondance unique et exacte marque la facture comme payée ; une correspondance multiple ou partielle part en anomalie ; une opération sans facture rejoint la liste des justificatifs manquants.
11. **Imputation et export** : n8n propose le compte fournisseur, le compte de charge et le compte de TVA (règles par fournisseur), puis exporte vers le logiciel comptable : nœud natif s'il existe (`QuickBooks Online`, `Xero`), sinon API via `HTTP Request` ou fichier d'import CSV.
12. **Archivage** : le PDF est renommé (date, fournisseur, numéro) et classé par mois dans Drive.
13. **Validation humaine des anomalies** : chaque anomalie arrive au comptable avec le motif, le document et l'extraction. Il corrige (le dossier repasse dans les contrôles), valide ou rejette, via un formulaire et un nœud `Wait`.
14. **Gestion d'erreurs** : un `Error Trigger` alerte l'équipe et déplace le document dans un dossier « À revoir » pour qu'aucune pièce ne se perde.

## Nœuds n8n principaux
`Gmail Trigger` / `Google Drive Trigger` → `Text Classifier` → `IF` → `Extract from File` / `HTTP Request` (OCR) → `Information Extractor` + `Structured Output Parser` → `Code` (contrôles HT/TVA/TTC) → `Crypto` (empreinte) → `Postgres` (registre, doublons) → `Loop Over Items` → `Merge` / `Compare Datasets` (rapprochement) → `QuickBooks Online` / `Xero` / `HTTP Request` → `Google Drive` (archivage) → `Slack` + `n8n Form` + `Wait` (anomalies) ; `Error Trigger` → `Slack`.

## Garde-fous
- **L'IA extrait, les règles décident** : tous les calculs et contrôles sont faits en code, pas par le LLM.
- **Validation humaine** : toute anomalie (écart de montant, TVA inhabituelle, doublon possible, rapprochement ambigu, nouveau fournisseur, changement d'IBAN) est revue par une personne. Au démarrage, faire valider aussi un échantillon de factures « OK ».
- **Fraude** : un changement d'IBAN chez un fournisseur connu bloque toujours la facture jusqu'à vérification humaine par un autre canal.
- **Aucun paiement automatique** : le workflow prépare et rapproche, il ne déclenche jamais de virement.
- **RGPD et conservation** : les factures contiennent des données personnelles (noms, adresses, IBAN) ; accès restreint, durée de conservation conforme aux obligations comptables à confirmer avec le comptable, choix d'un fournisseur OCR/LLM compatible avec ces données.
- **Coûts** : OCR seulement pour les scans, modèle léger pour le classement, traitement par lots.
- **Sécurité** : adresse de réception dédiée, pièces jointes limitées aux PDF et images, credentials stockés dans n8n, journal d'audit de chaque modification.
