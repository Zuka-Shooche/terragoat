# Projet DevSecOps — TerraGoat

Etudiant : DURBEC Luca - Pinto AXEL
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

Job 3 - Validation et Plan Terraform :
  - Checkout du code source
  - Installation de Terraform 1.5.7
  - Execution de terraform init
  - Execution de terraform validate
  - Execution de terraform plan (simulation de deploiement)
  - Archivage du rapport de plan dans GitHub Actions Artifacts

---

## Resultats de l analyse Checkov

Outil : Checkov 3.3.2 by Prisma Cloud
Date d analyse : 2025
Perimetre : dossier terraform/ (AWS, Azure, GCP, AliCloud, Oracle)

Passed checks : 218
Failed checks : 463
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
- Volumes EBS non chiffres
- Repositories ECR non securises
- Endpoints EKS publics

---

## Secrets detectes par Gitleaks

Gitleaks a detecte les secrets suivants dans le depot TerraGoat :

Fichier : terraform/aws/providers.tf
Type : Cle AWS Access Key hardcodee
Valeur exposee : AKIAIOSFODNN7EXAMPLE

Fichier : terraform/aws/lambda.tf
Type : Cle AWS Access Key hardcodee
Description : Credentials AWS hardcodes dans les variables Lambda

Fichier : terraform/aws/lambda.tf
Type : Chaine Base64 haute entropie
Description : Secret hardcode dans la configuration Lambda

Fichier : terraform/aws/ec2.tf
Type : Cle AWS hardcodee dans user_data
Valeur exposee : AKIAIOSFODNN7EXAMAAA
Description : Credentials AWS injectes en clair dans le script EC2

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

Correction : Suppression des cles hardcodees.
Bonne pratique : Utiliser AWS IAM Roles ou variables d environnement.

---

### Vulnerabilite 2 — Bucket S3 non securise

Checks Checkov : CKV_AWS_19, CKV_AWS_21, CKV2_AWS_6, CKV_AWS_18
Fichier : terraform/aws/s3.tf
Ressource : aws_s3_bucket.data
Criticite : HAUTE

Description :
Le bucket S3 data etait accessible publiquement, non chiffre,
sans versioning et sans logs d acces.

Corrections appliquees :
- Chiffrement KMS via aws_s3_bucket_server_side_encryption_configuration
- Versioning via aws_s3_bucket_versioning
- Blocage acces public via aws_s3_bucket_public_access_block
- Logs via aws_s3_bucket_logging

Bonne pratique : Tout bucket S3 sensible doit etre chiffre et prive.

---

### Vulnerabilite 3 — Security group ouvert a internet

Checks Checkov : CKV_AWS_24, CKV_AWS_260
Fichier : terraform/aws/ec2.tf
Ressource : aws_security_group.web-node
Criticite : HAUTE

Description :
Le security group autorisait SSH (port 22) et HTTP (port 80)
depuis n importe quelle IP sur internet (0.0.0.0/0).

Code vulnerable :
  cidr_blocks = ["0.0.0.0/0"]

Code corrige :
  cidr_blocks = ["10.0.0.0/8"]

Correction : Creation d un security group restreint au reseau interne.
Bonne pratique : Ne jamais ouvrir SSH a tout internet.

---

### Vulnerabilite 4 — Instance RDS accessible publiquement

Check Checkov : CKV_AWS_17
Fichier : terraform/aws/db-app.tf
Ressource : aws_db_instance.default
Criticite : CRITIQUE

Description :
L instance RDS etait configuree avec publicly_accessible = true,
ce qui expose la base de donnees directement sur internet.
N importe qui peut tenter de se connecter a la base de donnees.

Code vulnerable :
  publicly_accessible = true

Code corrige :
  publicly_accessible = false

Correction : Creation d une instance RDS securisee avec acces prive uniquement,
chiffrement active, Multi-AZ, protection contre la suppression et backup.
Bonne pratique : Une base de donnees ne doit jamais etre exposee sur internet.
Utiliser un bastion host ou VPN pour l acces administrateur.

---

### Vulnerabilite 5 — Instance RDS non chiffree

Check Checkov : CKV_AWS_16
Fichier : terraform/aws/db-app.tf
Ressource : aws_db_instance.default
Criticite : HAUTE

Description :
Les donnees stockees dans l instance RDS n etaient pas chiffrees au repos.
En cas d acces physique aux disques ou de compromission du stockage,
les donnees seraient lisibles en clair.

Code vulnerable :
  storage_encrypted = false (non defini)

Code corrige :
  storage_encrypted = true

Correction : Activation du chiffrement du stockage RDS.
Bonne pratique : Toutes les bases de donnees doivent avoir le chiffrement
au repos active, particulierement pour les donnees sensibles.

---

### Vulnerabilite 6 — Secrets hardcodes dans EC2 user_data

Check Checkov : CKV_AWS_46
Fichier : terraform/aws/ec2.tf
Ressource : aws_instance.web_host
Criticite : CRITIQUE

Description :
Des cles AWS etaient injectees en clair dans le script user_data
de l instance EC2. Ces donnees sont accessibles via l API de metadonnees
EC2 sans authentification (IMDSv1).

Valeurs exposees :
  AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMAAA
  AWS_SECRET_ACCESS_KEY=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

Code corrige :
  Suppression des credentials du user_data.
  Ajout d un IAM Instance Profile pour les permissions EC2.

Correction : Creation d un aws_iam_instance_profile pour gerer
les permissions EC2 sans credentials hardcodes.
Bonne pratique : Utiliser IAM Instance Profiles pour les permissions EC2.
Ne jamais injecter de credentials dans user_data.

---

## Vulnerabilites identifiees non corrigees

Ces vulnerabilites ont ete identifiees mais necessitent
une infrastructure AWS reelle pour etre corrigees en production.

| Check | Fichier | Description | Criticite |
|---|---|---|---|
| CKV_AWS_37 | eks.tf | Logs EKS non actives | Haute |
| CKV_AWS_38 | eks.tf | Endpoint EKS public accessible 0.0.0.0/0 | Haute |
| CKV_AWS_39 | eks.tf | Endpoint EKS public non desactive | Haute |
| CKV_AWS_58 | eks.tf | Chiffrement secrets EKS non active | Haute |
| CKV_AWS_51 | ecr.tf | Tags ECR non immuables | Moyenne |
| CKV_AWS_163 | ecr.tf | Scan ECR on push non active | Moyenne |
| CKV_AWS_136 | ecr.tf | ECR non chiffre avec KMS | Moyenne |
| CKV_AWS_84 | es.tf | Logs Elasticsearch non actives | Moyenne |
| CKV_AWS_115 | lambda.tf | Lambda sans limite de concurrence | Moyenne |
| CKV_AWS_157 | rds.tf | RDS sans Multi-AZ | Moyenne |
| CKV_AWS_130 | ec2.tf | Subnets assignent IP publique par defaut | Moyenne |

---

## Bonnes pratiques DevSecOps appliquees

### 1. Shift Left Security
La securite est integree des le debut du cycle de developpement.
Chaque push declenche automatiquement toutes les analyses de securite.

### 2. Gestion des secrets
Ne jamais stocker de credentials dans le code source.
Detection automatique via Gitleaks a chaque push.
Utiliser AWS IAM Roles et Instance Profiles.

### 3. Principe du moindre privilege
Security groups limites au reseau interne.
Buckets S3 bloques publiquement.
Bases de donnees non accessibles depuis internet.

### 4. Chiffrement des donnees
Chiffrement KMS sur les buckets S3.
Chiffrement active sur les instances RDS.
Backend Terraform avec chiffrement active.

### 5. Traçabilite et audit
Logging active sur les buckets S3.
Rapports archives dans GitHub Actions Artifacts.
Historique Git de toutes les modifications.

### 6. Automatisation de la securite
Checkov, Gitleaks et Terraform s executent automatiquement a chaque push.
Aucune intervention manuelle requise pour les analyses.

### 7. Infrastructure as Code
Toute infrastructure definie en code, versionnee et auditee.
Corrections documentees et tracees dans Git.

---

## Rapports de securite

Les rapports sont generes automatiquement a chaque execution
et archives dans GitHub Actions Artifacts.

Pour consulter les rapports :
1. Aller dans l onglet Actions du depot GitHub
2. Cliquer sur un run du Pipeline DevSecOps TerraGoat
3. Telecharger les artifacts disponibles :
   - checkov-report : rapport JSON Checkov complet
   - terraform-plan-report : rapport de simulation de deploiement

---

## Structure du depot

| Dossier / Fichier | Description |
|---|---|
| .github/workflows/devsecops.yml | Pipeline CI/CD GitHub Actions |
| docs/architecture.md | Schema detaille du pipeline |
| docs/vulnerabilites.md | Liste complete des vulnerabilites |
| docs/corrections.md | Corrections appliquees |
| docs/bonnes-pratiques.md | Bonnes pratiques DevSecOps |
| reports/ | Rapports bruts generes par le pipeline |
| terraform/ | Code TerraGoat avec corrections appliquees |

---

## Documentation complementaire

- docs/architecture.md — Architecture detaillee du pipeline CI/CD
- docs/vulnerabilites.md — Vulnerabilites identifiees et criticite
- docs/corrections.md — Corrections apportees au code Terraform
- docs/bonnes-pratiques.md — Bonnes pratiques DevSecOps appliquees

---

## References

- TerraGoat (Bridgecrew) : https://github.com/bridgecrewio/terragoat
- Checkov : https://www.checkov.io
- Gitleaks : https://github.com/gitleaks/gitleaks
- GitHub Actions : https://docs.github.com/en/actions
- CIS AWS Benchmark : https://www.cisecurity.org/benchmark/amazon_web_services
