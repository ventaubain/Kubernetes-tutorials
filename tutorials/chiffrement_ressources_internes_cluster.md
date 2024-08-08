# Chiffrer les secrets du cluster Kubernetes *At Rest*

- [Chiffrer les secrets du cluster Kubernetes *At Rest*](#chiffrer-les-secrets-du-cluster-kubernetes-at-rest)
  - [Générer une clé de chiffrement](#générer-une-clé-de-chiffrement)
  - [Configurer le cluster Kubernetes](#configurer-le-cluster-kubernetes)
    - [Créer le fichier de configuration du chiffrement de etcd](#créer-le-fichier-de-configuration-du-chiffrement-de-etcd)
    - [Mettre à jour la configuration de Kubernetes](#mettre-à-jour-la-configuration-de-kubernetes)
    - [Mettre à jour les secrets existants](#mettre-à-jour-les-secrets-existants)
  - [Vérification du fonctionnement](#vérification-du-fonctionnement)

Par défaut, un cluster Kubernetes stocke les données issues de l'**API Server** (en charge des secrets, ConfigMap...) dans une base `etcd` **en clair**. Ainsi, tout accès à cette base permet d'extraire des informations sensibles du cluster. Afin de corriger ce problème, 2 approches sont envisageables:

- Utiliser un service externe pour stocker les données (ex: *Vault*)
- Chiffrer les données stockées dans `etcd`

Dans ce tutoriel, nous nous intéresserons à la 2nd approche, i.e le chiffrement de la donnée *At Rest* dans la base `etcd`.

## Générer une clé de chiffrement

Le chiffrement repose sur l'algorithme `SecretBox`. Il s'agit d'un algorithme de chiffrement symétrique qui utilise une clé de **32 bits**.

```bash
head -c 32 /dev/urandom | base64
```

## Configurer le cluster Kubernetes

### Créer le fichier de configuration du chiffrement de etcd

- encryption_config_etcd.yaml

```bash
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
# On chiffre les secrets et les configmaps
resources:
  - resources:
      - secrets
      - configmaps
    # On liste les clés pour déchiffrer le contenu
    # Plusieurs clés peuvent être listées. Elles sont testées les unes après les autres dans l'ordre
    providers:
      - secretbox:
          keys:
            - name: key1
              secret: <clé de 32 bits>
      # <identity> sert à lire le contenu natif non chiffré
      - identity: {}
```

### Mettre à jour la configuration de Kubernetes

- Se connecter au `controlplane`
- Copier `encryption_config_etcd.yaml` dans `/etc/kubernetes/pki`
- Editer `/etc/kubernetes/manifests/kube-apiserver.yaml`:
  - Dans la section `spec.command`, rajouter `- --encryption-provider-config=/etc/kubernetes/pki/encryption_config_etcd.yaml`
- Attendre le redémarrage (automatique) de `kube-apiserver`

### Mettre à jour les secrets existants

La mise en place du chiffrement de `etcd` ne fonctionne que pour les secrets crées a posteriori (non rétroactif). Il est nécessaire de recréer les secrets pour qu'ils soient chiffrés. C'est pour cela qu'il est conseillé de conserver `identity` dans le fichier de configuration `EncryptionConfiguration`. En effet, sans recréation, les secrets restent en clair et ne peuvent être lus sans `identity`.

Pour recréer l'ensemble des secrets: `kubectl get secrets --all-namespaces -o json | kubectl replace -f -`

## Vérification du fonctionnement

- Créer un secret: `kubectl create secret generic secret1 -n default --from-literal=mykey=mydata`
- Afficher les informations stockées dans `etcd` à propos de `secret1`:

```bash
ETCDCTL_API=3 sudo etcdctl \
   --cacert=/etc/kubernetes/pki/etcd/ca.crt   \
   --cert=/etc/kubernetes/pki/etcd/server.crt \
   --key=/etc/kubernetes/pki/etcd/server.key  \
   get /registry/secrets/default/secret1 | hexdump -C
```

```bash
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 73 65 63 72 65 74  |s/default/secret|
00000020  31 0a 6b 38 73 3a 65 6e  63 3a 73 65 63 72 65 74  |1.k8s:enc:secret|
00000030  62 6f 78 3a 76 31 3a 6b  65 79 31 3a 46 d5 17 05  |box:v1:key1:F...|
00000040  4d 5b 5b b1 34 63 24 c4  f8 9c 39 d7 a6 fb 4a af  |M[[.4c$...9...J.|
[...]
```

On observe que le contenu est bien chiffré.

- Lire le secret depuis *Kubernetes*: `kubectl get secret secret1 -n default -o yaml`

```bash
apiVersion: v1
data:
  mykey: bXlkYXRh
kind: Secret
metadata:
  creationTimestamp: "2023-11-03T09:11:02Z"
  name: secret1
  namespace: default
  resourceVersion: "10869560"
  uid: 07ffdc1a-eac2-4f0a-b207-ffae15bf786f
type: Opaque
```

La valeur `bXlkYXRh` correspond à `mydata` en base64. Le chiffrement et le déchiffrement sont donc fonctionnels. Supprimer le secret avec `kubectl delete secret/secret1`.
