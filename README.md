---
runme:
  version: v3
---

# DEMO - CNPG

---

- PART1 : Les bases de CNPG
- PART2 : Backup vers S3 + Restauration CNPG

---

## DEMO - PART 1 - LES BASES DE CNPG

---

Création du cluster de démo

```bash {"terminalRows":"14"}
kind create cluster
```

Installation via helm (package manager de kubernetes)

```bash {"terminalRows":"39"}
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo update
helm install cnpg --namespace cnpg-system --create-namespace cnpg/cloudnative-pg
```

Création du namespace de démo

```bash
kubectl create ns demo
kn demo # alias pour se placer dans demo
```

Création du cluster PGSQL v1

```bash {"terminalRows":"14"}
cat psql-v1.yml
kubectl apply -f psql-v1.yml
```

Voir la composition

```bash {"terminalRows":"23"}
kubectl tree clusters.postgresql.cnpg.io/db # plugin krew
```

CRD cluster

```bash {"terminalRows":"34"}
kubectl get cluster
kubectl describe cluster db
```

---

Plugin kubectl cnpg - status

```bash {"terminalRows":"28"}
kubectl cnpg status db # --verbose pour encore + d'infos
```

Plugin kubectl cnpg - client pgsql

```bash {"terminalRows":"16"}
kubectl cnpg psql db
```

Plugin kubectl view-secret - secrets par défaut

```bash {"terminalRows":"16"}
kubectl view-secret
```

---

Fonctionnement des 3 services : R (all), RO (read replicas) et RW (primary)

```bash
kubectl get pods --show-labels # Vérifier primary (1er) et read replicas (le reste)
```

1 volume dédié par instance

```bash
kubectl get pv,pvc
```

---

## DEMO - PART 2 - Sauvegarde vers S3 + Restauration

---

**Pré-requis**

- CNPG >= 1.26

```bash
kubectl get deployment -n cnpg-system cnpg-cloudnative-pg -o jsonpath="{.spec.template.spec.containers[*].image}"
```

- Cert-manager pour activer la communication TLS entre le plugin et l’opérateur

```bash {"terminalRows":"32"}
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.18.2/cert-manager.yaml
```

- Installer le plugin (l'opérateur CNPG doit être installé dans le namespace "cnpg-system")

**[plugin-barman-cloud](https://github.com/cloudnative-pg/plugin-barman-cloud)**

```bash {"terminalRows":"20"}
kubectl apply -f https://github.com/cloudnative-pg/plugin-barman-cloud/releases/download/v0.6.0/manifest.yaml
```

- Bucket S3 pour stockage

```bash
aws s3api create-bucket \
  --bucket cnpg-demo-zone-infra \
  --region eu-west-3 \
  --create-bucket-configuration LocationConstraint=eu-west-3
```

- Secret pour accès S3

```bash
source .env
kubectl create secret generic aws-creds \
  --from-literal=ACCESS_KEY_ID=$ACCESS_KEY_ID \
  --from-literal=SECRET_ACCESS_KEY=$SECRET_ACCESS_KEY \
  --from-literal=AWS_DEFAULT_REGION=$AWS_DEFAULT_REGION
```

> Sur un cluster cloud managé : IRSA / EKS Pod Identity

---

**Sauvegarde/Restauration**

- ObjectStore

```bash {"terminalRows":"26"}
cat object-store.yml
kubectl apply -f object-store.yml
```

- Création du Cluster V2

```bash {"terminalRows":"26"}
cat psql-v2.yml
kubectl apply -f psql-v2.yml
```

> WAL (Write-Ahead Log) = Ecriture dans un journal (logs) pour conserver les différentes opérations à effectuer lors d'une restauration.

- Backup

```bash {"terminalRows":"12"}
cat backup.yml
kubectl apply -f backup.yml
```

Gestion backup

```bash
kubectl get backup # verification
kubectl delete backup db2-backup-manual # supression
```

- ScheduledBackup

```bash {"terminalRows":"16"}
cat scheduled-backup.yml
kubectl apply -f scheduled-backup.yml
```

Vérification

```bash
kubectl get scheduledbackups.postgresql.cnpg.io
kubectl get backups.postgresql.cnpg.io
```

Vérification S3 : Web UI

- Restoring (dans Cluster)

Récupérer le nom de la backup

```bash
kubectl get backup # nom
```

Récupérer l'ID de la backup

```bash {"terminalRows":"13"}
kubectl describe backup db2-backup-auto-20250905101432 # ID
```

Lancer la restauration (création du cluster "restore" avec paramètres de recovery)

```bash {"terminalRows":"39"}
cat restore.yml
kubectl apply -f restore.yml
```

> PITR (Point In Time Recovery) = Je veux récupérer l'état de la base à un moment précis.

Voir les logs

```bash
kubectl stern .
```

Vérifier qu'on retrouve bien la DB todolist

```bash {"terminalRows":"4"}
kubectl cnpg psql restore
```

> Résumé : En gros on créé 1 ObjectStore, et derrière on peut appeler le plugin "barman-cloud.cloudnative-pg.io" dans les autres ressources

> Bonne pratique : 1 object store par cluster PostgreSQL (ne pas utiliser 1 pour plusieurs cluster)

Vider le bucket S3 puis supprimer

```bash
aws s3api delete-bucket \
  --bucket cnpg-demo-zone-infra \
  --region eu-west-3
```

---
