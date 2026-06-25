# Projet DevSecOps — TerraGoat

Etudiant : DURBEC Luca - PINTO Axel
Formation : Mastere Infra — Ynov Aix-en-Provence 2025/2026
Module : DevSecOps
Encadrant : Damien Montmoulinex

---

## Presentation du projet

Ce projet a ete realise dans le cadre du module DevSecOps du Mastere Infrastructure
de Ynov Aix-en-Provence. L objectif principal est de mettre en place un pipeline
CI/CD complet integrant la securite a chaque etape du cycle de developpement,
en appliquant les bonnes pratiques DevSecOps sur TerraGoat.

TerraGoat est un projet open source fourni par Bridgecrew dont le but est de
demontrer comment des erreurs de configuration courantes peuvent se retrouver
dans des environnements cloud de production. Il contient volontairement des
vulnerabilites de securite dans son code Terraform, ce qui en fait un terrain
d application ideal pour mettre en oeuvre des outils d analyse de securite.

Le projet couvre les points suivants demandes par l encadrant :
- Mise en place d une chaine CI/CD complete via GitHub Actions
- Automatisation de l analyse de securite de l Infrastructure as Code
- Identification des mauvaises configurations de securite
- Correction des vulnerabilites les plus critiques
- Generation et conservation des rapports d analyse
- Documentation des risques identifies, corrections apportees et bonnes pratiques

---

## Environnement de travail

Systeme d exploitation : Windows 11 avec WSL2 (Ubuntu)
Outils installes en local :
- Python 3 et pip3
- Checkov 3.3.2
- Gitleaks 8.18.4
- Terraform 1.5.7
- Git 2.x

Depot GitHub : https://github.com/Zuka-Shooche/terragoat
Pipeline : GitHub Actions

---

## Outils utilises

| Outil | Version | Role | Categorie |
|---|---|---|---|
| Checkov | 3.3.2 | Analyse statique du code Terraform | SAST |
| Gitleaks | 8.18.4 | Detection de secrets dans le depot | Secret scanning |
| Terraform CLI | 1.5.7 | Validation syntaxique du code IaC | IaC |
| GitHub Actions | - | Orchestration automatique du pipeline | CI/CD |

---

## Architecture du pipeline

Le pipeline CI/CD est defini dans le fichier .github/workflows/devsecops.yml
Il se declenche automatiquement a chaque push ou Pull Request sur la branche master.
Il est compose de 3 jobs independants qui s executent en parallele.

Declencheurs :
- Push sur la branche master
- Pull Request vers la branche master

Job 1 - Analyse statique IaC avec Checkov :
  - Checkout du code source
  - Installation de Checkov
  - Execution de Checkov sur le dossier terraform/
  - Generation du rapport au format JSON
  - Archivage du rapport dans GitHub Actions Artifacts

Job 2 - Detection de secrets avec Gitleaks :
  - Checkout du code source
  - Execution de Gitleaks sur tout le depot
  - Detection des credentials et secrets exposes
  - Rapport des secrets trouves dans les logs CI

Job 3 - Validation Terraform :
  - Checkout du code source
  - Installation de Terraform 1.5.7
  - Execution de terraform init
  - Execution de terraform validate

---

## Resultats de l analyse Checkov

Outil : Checkov 3.3.2 by Prisma Cloud
Date d analyse : 2025
Perimetre : dossier terraform/ (AWS, Azure, GCP, AliCloud, Oracle)

Passed checks : 203
Failed checks : 467
Skipped checks : 0

Categories de vulnerabilites detectees :
- Chiffrement des donnees au repos non active
- Acces public non restreint sur les ressources
- Secrets et credentials hardcodes dans le code
- Logging et monitoring non configures
- Groupes de securite trop permissifs
- Authentification IAM non activee
- Versioning non configure sur les buckets
- Clusters Kubernetes mal configures
- Instances de base de donnees exposees publiquement

---

## Secrets detectes par Gitleaks

Gitleaks a detecte les secrets suivants dans le depot TerraGoat :

Fichier : terraform/aws/providers.tf
Type : Cle AWS Access Key hardcodee
Valeur exposee : AKIAIOSFODNN7EXAMPLE

Fichier : terraform/aws/providers.tf
Type : Chaine Base64 haute entropie
Description : Secret AWS hardcode dans la configuration du provider

Fichier : terraform/aws/lambda.tf
Type : Cle AWS Access Key hardcodee
Description : Credentials AWS hardcodes dans les variables d environnement Lambda

Fichier : terraform/aws/lambda.tf
Type : Chaine Base64 haute entropie
Description : Secret hardcode dans la configuration Lambda

Fichier : terraform/aws/ec2.tf
Type : Cle AWS hardcodee dans user_data
Valeur exposee : AKIAIOSFODNN7EXAMAAA
Description : Credentials AWS injectes en clair dans le script de demarrage EC2

Fichier : terraform/azure/sql.tf
Type : Chaine Base64 haute entropie
Description : Secret hardcode dans la configuration SQL Azure

---

## Vulnerabilites critiques identifiees et corrigees

### Vulnerabilite 1 — Cles AWS hardcodees dans providers.tf

Check Checkov : CKV_AWS_41
Fichier : terraform/aws/providers.tf
Ressource : aws.plain_text_access_keys_provider
Criticite : CRITIQUE

Description :
Des cles d acces AWS etaient ecrites en clair directement dans le code source
Terraform. Toute personne ayant acces au depot pouvait recuperer ces credentials
et compromettre le compte AWS associe.

Code vulnerable :
  provider "aws" {
    alias      = "plain_text_access_keys_provider"
    region     = "us-west-1"
    access_key = "AKIAIOSFODNN7EXAMPLE"
    secret_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
  }

Code corrige :
  provider "aws" {
    alias  = "plain_text_access_keys_provider"
    region = "us-west-1"
  }

Correction appliquee :
Suppression des cles hardcodees. Les credentials sont desormais geres
via des variables d environnement ou le fichier ~/.aws/credentials.

Bonne pratique :
Ne jamais stocker de credentials dans le code source. Utiliser AWS IAM Roles,
des variables d environnement ou un gestionnaire de secrets comme AWS Secrets Manager.

---

### Vulnerabilite 2 — Bucket S3 non securise

Checks Checkov : CKV_AWS_19, CKV_AWS_21, CKV2_AWS_6
Fichier : terraform/aws/s3.tf
Ressource : aws_s3_bucket.data
Criticite : HAUTE

Description :
Le bucket S3 nomme "data" presentait plusieurs problemes de securite critiques.
Il etait accessible publiquement, les donnees n etaient pas chiffrees au repos,
il n y avait pas de versioning et aucun log d acces n etait configure.

Problemes identifies :
- CKV_AWS_19 : Pas de chiffrement des donnees au repos
- CKV_AWS_21 : Versioning non active
- CKV2_AWS_6 : Acces public non bloque
- CKV_AWS_18 : Pas de logs d acces

Corrections appliquees :
1. Ajout du chiffrement KMS via aws_s3_bucket_server_side_encryption_configuration
2. Activation du versioning via aws_s3_bucket_versioning
3. Blocage de tout acces public via aws_s3_bucket_public_access_block
4. Activation des logs d acces via aws_s3_bucket_logging

Bonne pratique :
Tout bucket S3 contenant des donnees sensibles doit imperativement
etre chiffre, prive, dispose de versioning et de logs d acces.

---

### Vulnerabilite 3 — Security group ouvert a internet

Checks Checkov : CKV_AWS_24, CKV_AWS_260
Fichier : terraform/aws/ec2.tf
Ressource : aws_security_group.web-node
Criticite : HAUTE

Description :
Le security group associe a l instance EC2 autorisait les connexions SSH
sur le port 22 et HTTP sur le port 80 depuis n importe quelle adresse IP
sur internet (0.0.0.0/0). Cela expose l instance a des attaques par force
brute, des scans de ports et des tentatives d intrusion depuis internet.

Code vulnerable :
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

Code corrige :
  ingress {
    description = "SSH depuis le reseau interne uniquement"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]
  }

Correction appliquee :
Creation d un nouveau security group securise limitant l acces SSH et HTTP
au reseau interne uniquement (10.0.0.0/8).

Bonne pratique :
Les ports SSH et RDP ne doivent jamais etre ouverts a tout internet.
Utiliser un bastion host ou un VPN pour les acces administrateurs.

---

## Vulnerabilites identifiees non corrigees

Ces vulnerabilites ont ete identifiees par Checkov mais n ont pas ete
corrigees car elles necessitent une infrastructure AWS reelle deployee
pour etre pleinement traitees en environnement de production.

| Check | Fichier | Description | Criticite |
|---|---|---|---|
| CKV_AWS_16 | db-app.tf | Chiffrement RDS non active | Haute |
| CKV_AWS_17 | db-app.tf | Instance RDS accessible publiquement | Haute |
| CKV_AWS_37 | eks.tf | Logs EKS control plane non actives | Haute |
| CKV_AWS_58 | eks.tf | Chiffrement secrets EKS non active | Haute |
| CKV_AWS_84 | es.tf | Logs Elasticsearch non actives | Moyenne |
| CKV_AWS_115 | lambda.tf | Lambda sans limite de concurrence | Moyenne |
| CKV_AWS_157 | rds.tf | RDS sans configuration Multi-AZ | Moyenne |
| CKV_AWS_161 | db-app.tf | Authentification IAM RDS non activee | Haute |
| CKV_AWS_92 | elb.tf | Logs ELB non actives | Moyenne |
| CKV_AWS_51 | ecr.tf | Tags ECR non immuables | Moyenne |

---

## Bonnes pratiques DevSecOps appliquees

### 1. Shift Left Security
La securite est integree des le debut du cycle de developpement.
Chaque push sur le depot declenche automatiquement les analyses de securite.

### 2. Gestion des secrets
Ne jamais stocker de credentials dans le code source.
Utiliser des variables d environnement ou un gestionnaire de secrets.
Detection automatique via Gitleaks a chaque push.

### 3. Principe du moindre privilege
Chaque ressource ne doit avoir acces qu a ce dont elle a besoin.
Security groups limites au reseau interne. Buckets S3 bloques publiquement.

### 4. Chiffrement des donnees
Toutes les donnees sensibles doivent etre chiffrees au repos et en transit.
Chiffrement KMS sur les buckets S3. Backend Terraform avec chiffrement active.

### 5. Traçabilite et audit
Toutes les actions doivent etre tracees et auditables.
Logging active sur les buckets S3. Rapports archives dans GitHub Actions.
Historique Git de toutes les modifications apportees.

### 6. Automatisation de la securite
La securite ne doit pas dependre d actions manuelles.
Checkov, Gitleaks et Terraform validate s executent automatiquement a chaque push.

### 7. Infrastructure as Code
Toute infrastructure doit etre definie en code, versionnee et auditee.
Permet la reproductibilite des environnements et la revue de code sur l infrastructure.

---

## Rapports de securite

Les rapports sont generes automatiquement a chaque execution du pipeline
et archives sous forme d Artifacts dans GitHub Actions.

Pour consulter les rapports :
1. Aller dans l onglet Actions du depot GitHub
2. Cliquer sur un run du Pipeline DevSecOps TerraGoat
3. Telecharger l artifact checkov-report dans la section Artifacts

Rapport disponible : checkov-report.json

---

## Structure du depot

| Dossier / Fichier | Description |
|---|---|
| .github/workflows/devsecops.yml | Pipeline CI/CD GitHub Actions |
| docs/architecture.md | Schema detaille et explication du pipeline |
| docs/vulnerabilites.md | Liste complete des vulnerabilites identifiees |
| docs/corrections.md | Corrections appliquees sur le code Terraform |
| docs/bonnes-pratiques.md | Bonnes pratiques DevSecOps mises en oeuvre |
| reports/ | Rapports bruts generes automatiquement par le pipeline |
| terraform/ | Code TerraGoat original intentionnellement vulnerable |

---

## Documentation complementaire

- docs/architecture.md — Architecture detaillee du pipeline CI/CD
- docs/vulnerabilites.md — Vulnerabilites identifiees et niveau de criticite
- docs/corrections.md — Corrections apportees au code Terraform
- docs/bonnes-pratiques.md — Bonnes pratiques DevSecOps appliquees

---

## References

- TerraGoat (Bridgecrew) : https://github.com/bridgecrewio/terragoat
- Checkov : https://www.checkov.io
- Gitleaks : https://github.com/gitleaks/gitleaks
- GitHub Actions : https://docs.github.com/en/actions
- CIS AWS Benchmark : https://www.cisecurity.org/benchmark/amazon_web_services
