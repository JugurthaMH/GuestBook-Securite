# GuestBook Sécurité

Le guestbook des jours précédents part en production... et il est truffé d'erreurs de sécurité, toutes vues en cours. Votre objectif: **l'attaquer vous-même** (Partie A), puis **le sécuriser** jusqu'à ce que chacune de vos attaques échoue (Partie B). Et d'apprendre un maximum pendant cette session.

Vous travaillez **seul, sur votre propre cluster minikube**. Toutes les attaques visent uniquement votre machine: n'attaquez jamais un système qui ne vous appartient pas ou pour lequel vous n'avez pas d'autorisation écrite.

Ce dépôt contient tout ce qu'il faut:

```text
.
├── cours/                        les slides du Jour 5
├── guestbook/                    le code du guestbook (celui du Jour 4, + REDIS_PASSWORD)
├── deploy/
│   ├── namespaces.yaml           namespaces guestbook et data                        (ex. 0)
│   ├── redis/                    Redis, son volume, son Service et un "debug" RBAC   (ex. 0, 6, 8)
│   └── guestbook/                chart Helm du guestbook                             (ex. 0, 4, 6)
├── attack/
│   ├── paths.txt                 chemins web à tester                                (ex. 1)
│   ├── fake-miner.sh             faux mineur: ne mine rien, consomme du CPU          (ex. 2, 9)
│   └── privileged-pod.yaml       pod privilégié qui monte le disque du node          (ex. 2, 6)
├── starter/                      fichiers de départ à compléter                      (ex. 4, 6, 7, 8)
└── README.md
```

Pour chaque exercice, créez un dossier `exercise-N/` avec un `README.md` (votre doc et votre fiche d'apprentissage) et les fichiers que vous écrivez (politiques, NetworkPolicies, règles...). Le guestbook et ses manifestes restent dans `guestbook/` et `deploy/`: quand vous les modifiez, notez dans le `README.md` de l'exercice ce que vous avez changé et pourquoi (ou le `git diff`).

## Fiche d'apprentissages

Pour l'évaluation, merci de remplir une fiche d'apprentissage pour chaque exercice réalisé, dans le `README.md` de son dossier `exercise-N/`.
Cette fiche aura le template suivant :

```markdown
# Apprentissages

## Titre d'exo

Court résumé de l’objectif visé

### Problème rencontré et pourquoi il est survenu

### Solution appliquée et pourquoi cette solution fonctionne

### Ce que j'ai appris

```

## N'oubliez pas !

Tenez votre doc dans le `README.md` de chaque `exercise-N/`, pour garder une trace de ce que vous faites, des commandes que vous utilisez, des erreurs que vous rencontrez et comment vous les avez résolues.

Gardez précieusement les commandes d'attaque de la Partie A: dans la Partie B, vous les rejouerez pour **prouver** que chaque correction fonctionne.

---

# Partie A - Attaquer

## 0. Préparer votre environnement

### Pourquoi

Il vous faut un cluster dont le réseau applique vraiment les NetworkPolicies (exercice 8), et la stack telle qu'elle a été livrée, avec toutes ses erreurs.

### Comment

**Fichiers**: `guestbook/`, `deploy/`

1. Installez les outils suivants, en plus de ceux des jours précédents (minikube, kubectl, Docker, Helm) :

  - [nmap](https://nmap.org/download): scanner de ports
  - [redis-cli](https://redis.io/docs/latest/develop/tools/cli/): client Redis (paquet `redis-tools` ou `redis`)
  - [gitleaks](https://github.com/gitleaks/gitleaks), [Trivy](https://trivy.dev/latest/getting-started/installation/), [Cosign](https://docs.sigstore.dev/cosign/system_config/installation/): pour la Partie B

2. Créez un cluster minikube avec le CNI **kindnet** (le plugin réseau du cluster: c'est lui qui appliquera les NetworkPolicies de l'exercice 8; selon votre installation, le réseau choisi par défaut, `bridge`, les ignore) :

  ```bash
  minikube start -p security --cni=kindnet --cpus 4 --memory 6144
  minikube -p security addons enable metrics-server
  minikube -p security addons enable ingress
  kubectl wait -n ingress-nginx --for=condition=Ready pod -l app.kubernetes.io/component=controller --timeout=5m
  ```

3. Construisez le guestbook et chargez-le dans minikube :

  ```bash
  docker build -t guestbook:v1.0.0 guestbook/
  minikube -p security image load guestbook:v1.0.0
  ```

4. Déployez la stack telle quelle :

  ```bash
  kubectl apply -f deploy/namespaces.yaml
  kubectl apply -n data -f deploy/redis/
  helm install guestbook deploy/guestbook -n guestbook
  ```

5. Dans un **autre terminal**, laissez tourner le tunnel (il donne une IP aux Services `LoadBalancer`; sous Linux, il vous demande votre mot de passe `sudo`) :

  ```bash
  minikube -p security tunnel
  ```

### Tips

- Arrêtez les clusters des jours précédents pour libérer de la mémoire: `minikube profile list`, puis `minikube stop -p <profil>`
- Le guestbook est servi par l'Ingress avec l'hôte `guestbook.local`: `curl -H "Host: guestbook.local" http://<IP>/`. L'IP est `minikube -p security ip` sous Linux, `127.0.0.1` (avec le tunnel) sous macOS et Windows
- L'addon ingress de minikube utilise ingress-nginx, dont le projet Kubernetes a arrêté la maintenance en mars 2026: suffisant pour un TP, mais en production on choisit aujourd'hui une implémentation de Gateway API
- Lisez les fichiers de `deploy/` avant de les appliquer: les commentaires *"temporary"* et *"TODO"* sont des indices pour la suite

### Vérifications

- [ ] `kubectl get pods -A` n'affiche que des pods `Running`, dont un pod `kindnet-*` dans `kube-system`
- [ ] Le guestbook fonctionne dans le navigateur ou avec `curl` et je peux poster un message
- [ ] `kubectl get svc -n data` affiche une `EXTERNAL-IP` pour `redis`

## 1. Reconnaissance

### Pourquoi

Un attaquant ne commence jamais par attaquer: il cherche d'abord ce qui est exposé. Les robots qui scannent Internet font exactement ça, en permanence (cours: *"Penser comme un attaquant"*).

### Comment

Notez les commandes, leurs résultats et vos réponses aux questions dans `exercise-1/README.md`.

1. **Préparez deux variables**: l'IP du cluster (le guestbook) et l'IP de Redis (donnée par le tunnel)

  ```bash
  IP=$(minikube -p security ip)          # macOS / Windows: IP=127.0.0.1
  REDIS_IP=$(kubectl get svc -n data redis -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
  echo $IP $REDIS_IP
  ```

2. **Scannez tous les ports** du cluster, puis identifiez les services derrière les ports ouverts

  ```bash
  nmap -p- $IP                           # macOS / Windows: sautez cette étape, 127.0.0.1 est votre propre machine
  nmap -sV -p <ports trouvés> $IP
  nmap -sV -p 6379 $REDIS_IP
  ```

  - Quels ports appartiennent à minikube lui-même, lesquels à notre application ?
  - Quel service tourne sur le port 6379 ? Pourquoi est-il visible depuis votre machine ?

3. **Testez les chemins web** de `attack/paths.txt` sur le guestbook

  ```bash
  while read path; do
    curl -s -o /dev/null -w "%{http_code} $path\n" -H "Host: guestbook.local" "http://$IP$path"
  done < attack/paths.txt
  ```

  - Quels chemins répondent `200` alors qu'ils ne devraient pas exister en production ?
  - Ouvrez-les avec `curl -H "Host: guestbook.local" http://$IP/<chemin>`: que révèle chacun ?

4. **Listez votre surface d'attaque**: un tableau *point d'entrée / ce qu'il révèle / pourquoi c'est un problème*

### Tips

- Un 404 n'est pas intéressant, un 200 sur un chemin qui ne devrait pas exister l'est beaucoup
- `nmap` sans `-p-` ne scanne que les 1000 ports les plus courants
- Le guestbook est servi par l'Ingress avec l'hôte `guestbook.local`: d'où le `-H "Host: guestbook.local"` sur chaque `curl`

### Vérifications

- [ ] J'ai trouvé au moins 2 endpoints HTTP qui n'auraient pas dû être publics
- [ ] J'ai trouvé un service non HTTP exposé, et identifié lequel
- [ ] Ma surface d'attaque est documentée et j'ai répondu aux questions

## 2. Exploitation

### Pourquoi

Une faille n'est grave que si elle est exploitable. En l'exploitant vous-même, vous mesurez son impact réel, et vous saurez ensuite prouver que votre correction fonctionne.

### Comment

Chaque étape correspond à une étape de la kill chain du cours. Notez commandes, résultats et réponses dans `exercise-2/README.md`, en masquant les clés (`AKIA****`).

1. **Vol d'identifiants**: lisez ce que publie `/env`

  ```bash
  curl -s -H "Host: guestbook.local" http://$IP/env
  ```

  - Quelles variables sont sensibles ?
  - D'où viennent-elles ? Cherchez-les dans `deploy/guestbook/` (`values.yaml`, `templates/`)

2. **Accès initial**: connectez-vous à Redis depuis votre poste, sans mot de passe

  ```bash
  redis-cli -h $REDIS_IP PING
  redis-cli -h $REDIS_IP LRANGE guestbook 0 -1
  redis-cli -h $REDIS_IP RPUSH guestbook "Redis est ouvert a tous"
  redis-cli -h $REDIS_IP CONFIG GET '*'
  ```

  - Rechargez le guestbook dans le navigateur: que voyez-vous ?
  - Qu'aurait pu faire un attaquant avec la commande `FLUSHALL` ? (ne la lancez pas)
  - Quels sont les deux problèmes de configuration qui rendent ceci possible ?

3. **Exécution**: un vrai attaquant obtiendrait un shell dans le pod Redis (via un module Redis malveillant, que nous ne ferons pas). Simulez-le en lançant le faux mineur dans le pod :

  ```bash
  kubectl exec -i -n data deploy/redis -- sh < attack/fake-miner.sh
  ```

  - Lisez `attack/fake-miner.sh`: où le "mineur" s'installe-t-il, sous quel nom, et pourquoi ce nom ?

4. **Impact**: pendant que le mineur tourne (il s'arrête tout seul au bout de 2 minutes), dans un autre terminal :

  ```bash
  kubectl top pods -n data
  kubectl top nodes
  ```

  - Pourquoi Redis peut-il consommer autant de CPU ? Qu'est-ce qui manque dans `deploy/redis/redis-deployment.yaml` ?
  - Dans le cloud, qui paierait ce CPU ?

5. **Élévation de privilèges**: quelle identité Kubernetes un attaquant récupère-t-il dans le pod Redis, et que peut-elle faire ?

  ```bash
  kubectl exec -n data deploy/redis -- ls /var/run/secrets/kubernetes.io/serviceaccount/
  kubectl get pod -n data -l app=redis -o jsonpath='{.items[0].spec.serviceAccountName}'
  kubectl auth can-i --list --as=system:serviceaccount:data:default
  kubectl get secrets -A --as=system:serviceaccount:data:default
  kubectl get secret -n guestbook guestbook-aws --as=system:serviceaccount:data:default \
    -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 -d
  ```

  - Quel ServiceAccount utilise le pod Redis ? Quel fichier de `deploy/` lui donne ces droits ?
  - Le `Secret` était-il chiffré ?

6. **Évasion**: avec cette même identité, lancez un pod privilégié et lisez le disque du node

  ```bash
  kubectl apply --as=system:serviceaccount:data:default -f attack/privileged-pod.yaml
  kubectl wait -n data --for=condition=Ready pod/node-shell
  kubectl exec -n data node-shell -- cat /host/etc/hostname
  kubectl exec -n data node-shell -- ls /host/var/lib/kubelet/pods
  kubectl delete -n data pod node-shell
  ```

  - Lisez `attack/privileged-pod.yaml`: quels réglages donnent accès au node ?
  - Pourquoi Kubernetes a-t-il accepté ce pod ?

### Tips

- `--as=...` demande à Kubernetes d'agir *comme* une autre identité (on appelle ça l'*impersonation*): c'est exactement ce que pourrait faire un attaquant avec le token monté dans le pod
- Les valeurs d'un `Secret` sont en base64: `base64 -d` pour les lire
- Le faux mineur tourne dans `/tmp/.x/` sous le nom `kworker`: retenez-le, vous le chercherez avec Falco (exercice 9)

### Vérifications

- [ ] J'ai récupéré les clés AWS (factices) du guestbook sans aucun accès au cluster
- [ ] J'ai lu et modifié les données de Redis depuis mon poste
- [ ] J'ai observé l'impact du faux mineur sur le CPU
- [ ] J'ai listé les secrets de tous les namespaces avec l'identité du pod Redis
- [ ] J'ai lu un fichier du node depuis un pod
- [ ] J'ai répondu aux questions de chaque étape

## 3. (Optionnel) Rapport d'attaque

### Pourquoi

Une attaque n'a de valeur pour l'équipe que si elle est décrite et qualifiée: c'est ce qu'on attend d'un rapport de pentest.

### Comment

Exercice optionnel, mais rapide: il suffit de compléter, dans `exercise-3/README.md`, ces deux tableaux à partir des exercices 1 et 2. Il vous servira aussi de plan pour la Partie B.

1. **Décrivez votre attaque** étape par étape, et associez à chaque étape une technique [MITRE ATT&CK](https://attack.mitre.org/) :

  ```markdown
  | # | Étape | Ce que j'ai fait | Tactique ATT&CK | Technique (ID) |
  |---|-------|------------------|-----------------|----------------|
  | 1 | Reconnaissance | scan des ports et des chemins web | Reconnaissance | T1595 Active Scanning |
  | 2 | Vol d'identifiants | ... | ... | ... |
  ```

2. **Qualifiez chaque faille** avec la grille du cours (impact × facilité d'exploitation ⇒ Mineur, Important, Majeur, Critique), et **proposez une remédiation**: ce sera votre plan pour la Partie B

  ```markdown
  | Faille | Impact | Exploitation | Qualification | Remédiation |
  |--------|--------|--------------|---------------|-------------|
  | /env publie les clés cloud | Critique | Facile (un GET) | Critique | ... |
  ```

### Tips

- La matrice [ATT&CK for Containers](https://attack.mitre.org/matrices/enterprise/containers/) couvre l'essentiel; la reconnaissance est dans la matrice Enterprise. Le cours montre l'attaque du fil rouge lue avec ATT&CK
- Une même faille peut servir à plusieurs étapes (ex. le RBAC trop large sert à la fois à lire les secrets et à créer le pod privilégié)

### Vérifications

- [ ] Chaque étape est associée à une technique ATT&CK
- [ ] Chaque faille est qualifiée, avec une justification
- [ ] Chaque faille a une remédiation proposée

---

# Partie B - Sécuriser

À la fin de chaque exercice, **rejouez l'attaque correspondante** de la Partie A et collez la preuve qu'elle échoue (la commande et sa sortie) dans le `README.md` de l'exercice.

Pour les exercices 4, 6, 7 et 8, le dossier `starter/` contient des fichiers de départ: copiez-les dans votre `exercise-N/` et complétez les `TODO`.

## 4. Les secrets et l'image

### Pourquoi

Les clés AWS sont dans Git et dans une variable d'environnement affichée par `/env`. Et l'image du guestbook tourne en root. On corrige à la source (cours: *"Sécuriser la chaîne de livraison"*).

### Comment

1. **Détectez les secrets** du dépôt avec `gitleaks`
2. **Supprimez la fuite** :
  - retirez les endpoints de debug `/env` et `/info` du guestbook: `git apply starter/exercise-4/remove-debug-endpoints.diff` (lisez le diff avant: quelles lignes disparaissent ?)
  - retirez les clés du chart: le bloc `aws` de `deploy/guestbook/values.yaml`, le fichier `deploy/guestbook/templates/secret.yaml`, et le `secretRef` qui le charge dans `deploy/guestbook/templates/deployment.yaml`
3. **Durcissez l'image**: modifiez le `Dockerfile` pour que le conteneur tourne avec un utilisateur non-root, et construisez une `v1.1.0`
4. **Scannez les images** `guestbook:v1.1.0` et `redis:7` avec Trivy, et choisissez un tag de Redis plus sûr
5. **Analysez les CVE du guestbook**: pour chacune, quel paquet est touché, quelle attaque permet-elle, et est-elle **exploitable dans le guestbook** ? (cours: *"Les CVE"*, le score en contexte)
6. **Corrigez-les** en mettant à jour la dépendance, puis reconstruisez et rescannez :

  ```bash
  cd guestbook/
  go get golang.org/x/net@latest   # la version "Fixed Version" de Trivy, ou plus récente
  go mod tidy
  cd ..
  docker build -t guestbook:v1.1.0 guestbook/
  trivy image --severity HIGH,CRITICAL guestbook:v1.1.0
  ```

7. **Redéployez** et rejouez l'étape 1 de l'exercice 2

### Tips

- `gitleaks dir .` scanne les fichiers, `gitleaks git .` tout l'historique Git: les deux ne donnent pas forcément le même résultat. Que faut-il faire d'un secret qui a été commité, même s'il est supprimé ensuite ?
- Supprimer `/env` du code ne suffit pas si le pod a toujours les clés en variable d'environnement: à quoi servaient-elles ? Le guestbook en a-t-il besoin ?
- L'image actuelle part de `scratch`: elle n'a pas d'utilisateur. `USER 65532:65532` fonctionne sans créer d'utilisateur, ou partez d'une image `distroless` `:nonroot`
- Trivy a besoin de télécharger sa base de vulnérabilités au premier lancement
- Dans le rapport Trivy, la ligne `app/app (gobinary)` liste les modules Go compilés dans le binaire; la colonne *Fixed Version* donne la version à installer. `go mod why -m <module>` explique pourquoi le guestbook en dépend (directement ou via une autre librairie)
- Une CVE `HIGH` n'est pas forcément exploitable chez vous: le guestbook utilise-t-il vraiment le code vulnérable (lit-il du HTML ? quel serveur HTTP utilise-t-il ?)
- gitleaks scanne aussi vos `exercise-N/README.md`: si vous y collez les clés volées à l'exercice 2, masquez-les (`AKIA****`)

### Vérifications

- [ ] `gitleaks` ne trouve plus de secret dans les fichiers du dépôt
- [ ] `/env` et `/info` retournent 404 (ou 405)
- [ ] Le guestbook ne tourne plus en root: `minikube -p security ssh -- ps -eo user,comm | grep app` affiche `65532` (l'image n'a pas de shell, `kubectl exec ... id` ne marche pas)
- [ ] J'ai un rapport Trivy avant/après pour Redis, et j'ai justifié mon choix de tag
- [ ] J'ai analysé les CVE du guestbook (exploitables ou non, pourquoi), et le rescan après correction n'en montre plus

## 5. Signer l'image avec Cosign

### Pourquoi

Comment le cluster peut-il être sûr que l'image qu'il lance est bien celle construite par vous, et pas une image modifiée dans le registry ? [Cosign](https://github.com/sigstore/cosign) (écosystème Sigstore) signe les images, et sait y attacher une SBOM.

### Comment

1. **Activez le registry** de minikube et poussez-y `guestbook:v1.1.0`
2. **Générez une paire de clés** Cosign
3. **Générez la SBOM** de l'image, affichez-la, et attachez-la à l'image (`cosign attest`)
4. **Signez l'image** puis **vérifiez** la signature
5. **Simulez une attaque**: poussez une autre image sous le même tag, et vérifiez à nouveau

### Tips

- `minikube -p security addons enable registry`, puis `kubectl port-forward -n kube-system svc/registry 5000:80` pour pousser vers `localhost:5000`
- Signez le **digest** (`@sha256:...`), pas le tag: un tag peut être déplacé vers une autre image, un digest non. Pour l'obtenir: `docker buildx imagetools inspect localhost:5000/guestbook:v1.1.0`
- `trivy image --format cyclonedx` génère une SBOM CycloneDX
- Le registry de minikube est en HTTP: Cosign aura besoin de `--allow-insecure-registry`
- Ne commitez jamais `cosign.key`

### Vérifications

- [ ] L'image est signée et `cosign verify` réussit
- [ ] La SBOM est attachée et je sais retrouver la version d'un paquet dedans
- [ ] Après remplacement de l'image sous le même tag, `cosign verify` échoue
- [ ] `cosign.key` n'est pas dans Git

## 6. Pod Security et RBAC

### Pourquoi

Avec l'identité de Redis, on pouvait tout lire et lancer un pod privilégié. On applique le moindre privilège aux identités et aux pods (cours: *"Sécuriser le cluster"*).

### Comment

1. **Supprimez le droit "debug"** donné au ServiceAccount de `data`
2. **Désactivez le montage du token** dans les pods qui n'appellent pas l'API Kubernetes
3. **Ajoutez un `securityContext`** au guestbook et à Redis: non-root, pas d'élévation de privilèges, capacités supprimées, système de fichiers en lecture seule, profil seccomp (départ: `starter/exercise-6/security-context.yaml`)
4. **Ajoutez des `resources`** à Redis (requests et limits)
5. **Appliquez le Pod Security Standard `restricted`** aux namespaces `guestbook` et `data`, puis réappliquez vos manifestes :

  ```bash
  kubectl delete clusterrolebinding data-default-debug
  kubectl apply -f deploy/namespaces.yaml
  kubectl apply -n data -f deploy/redis/
  helm upgrade guestbook deploy/guestbook -n guestbook
  ```

6. Rejouez les étapes 3, 5 et 6 de l'exercice 2

### Tips

- Commencez par `pod-security.kubernetes.io/warn=restricted`: le label `enforce` ne touche pas les pods déjà lancés, il bloque la création des **nouveaux**
- Si `kubectl apply` d'un Deployment répond `configured` mais qu'aucun nouveau pod n'apparaît, ce n'est pas le Deployment qui est refusé: cherchez dans `kubectl get events -n <ns>` ou `kubectl describe rs`
- `runAsNonRoot: true` ne choisit pas d'utilisateur, il vérifie seulement. Indiquez donc aussi l'utilisateur: `runAsUser: 999`, `runAsGroup: 999` et `fsGroup: 999` pour Redis (l'utilisateur `redis` de l'image, `fsGroup` lui donne le droit d'écrire dans `/data`), `runAsUser: 65532` pour le guestbook
- `kubectl auth can-i --list --as=...` pour vérifier les droits après correction

### Vérifications

- [ ] L'identité de Redis ne peut plus lister les secrets
- [ ] Le token n'est plus monté dans le pod Redis
- [ ] Le faux mineur ne peut plus être copié dans `/tmp`, et un processus ne peut pas dépasser la limite CPU
- [ ] `attack/privileged-pod.yaml` est refusé
- [ ] Le guestbook et Redis fonctionnent toujours

## 7. Admission policies avec OPA Gatekeeper

### Pourquoi

Le Pod Security Standard ne couvre que les pods. Le Service `LoadBalancer` *"temporaire"* de Redis, le tag `latest`, une image d'un registry inconnu: on veut que l'API Kubernetes les **refuse** (*"policy as code"*).

### Comment

1. **Installez [OPA Gatekeeper](https://open-policy-agent.github.io/gatekeeper/website/docs/install)** dans votre cluster
2. **Écrivez une politique** qui interdit le tag `latest` (et l'absence de tag): complétez `starter/exercise-7/disallow-latest.yaml`
3. **Appliquez une politique** qui n'autorise que les images de registries approuvés: le template `k8sallowedrepos` de la bibliothèque + `starter/exercise-7/allowed-repos-constraint.yaml`
4. **Appliquez une politique** qui interdit les Services `LoadBalancer` et `NodePort`: les templates de la bibliothèque + `starter/exercise-7/block-services-constraints.yaml`
5. **Testez** chaque politique avec une ressource conforme et une non conforme, puis supprimez le `LoadBalancer` de Redis

### Tips

- Un `ConstraintTemplate` (la règle, en Rego) + une `Constraint` (où l'appliquer). Écrivez vous-même celui du tag `latest` (le cours le montre), et partez de la [bibliothèque Gatekeeper](https://open-policy-agent.github.io/gatekeeper-library/website/) pour les autres: `k8sallowedrepos`, `k8sblockloadbalancer`, `k8sblocknodeport`
- Commencez avec `enforcementAction: dryrun` ou `warn`, et lisez `kubectl get <kind> <nom> -o yaml` (champ `status.violations`): vous verrez ce qui serait refusé dans le cluster existant
- Limitez vos contraintes à vos namespaces avec `spec.match.namespaces: ["guestbook", "data", "policy-test"]`: les composants du cluster (`kube-system`, `ingress-nginx`...) ne sont pas concernés
- Testez dans un namespace `policy-test` (`kubectl create ns policy-test`), sans Pod Security: dans `guestbook` et `data`, Pod Security refuse un pod non conforme **avant** Gatekeeper, et vous ne verriez jamais le message de votre politique
- Préfixes d'images approuvés: `guestbook:` (votre image, chargée dans minikube) et `redis:` (image officielle). En entreprise, ce serait le registry de l'entreprise (`registry.example.com/`)
- Comme pour Pod Security: si votre contrainte vise des `Pod`, un Deployment non conforme est accepté et ce sont ses pods qui sont refusés

### Vérifications

- [ ] Gatekeeper est déployé et fonctionnel
- [ ] `kubectl run test -n policy-test --image=redis:latest` est refusé avec un message clair
- [ ] Une image d'un registry non approuvé est refusée
- [ ] Redis ne peut plus être exposé en `LoadBalancer`
- [ ] Le guestbook et Redis se déploient toujours

## 8. Network Policies

### Pourquoi

Par défaut, tous les pods peuvent parler à tous les pods, et à Internet. Un pod compromis n'importe où peut joindre Redis, et Redis peut télécharger un mineur. On applique la micro-segmentation.

### Comment

1. **Interdisez tout** (`ingress` et `egress`) dans les namespaces `guestbook` et `data`: `starter/exercise-8/default-deny.yaml` est fourni, appliquez-le et constatez que le guestbook ne répond plus
2. **Autorisez uniquement** :
  - `ingress-nginx` ⇒ guestbook (port 3000)
  - guestbook ⇒ Redis (port 6379)
  - les résolutions DNS nécessaires
   Complétez les trois fichiers `starter/exercise-8/allow-*.yaml`
3. **Testez la connectivité**, dans les deux sens: ce qui doit passer passe, ce qui doit être bloqué l'est

### Tips

- Les labels à utiliser: `app: guestbook` pour le guestbook, `app: redis` pour Redis, `kubernetes.io/metadata.name: <nom>` pour sélectionner un namespace (`kubectl get pods --show-labels` pour vérifier)
- Bloquer l'egress bloque aussi le DNS: le guestbook ne trouvera plus `redis.data.svc.cluster.local` tant que le port 53 (UDP **et** TCP) vers `kube-dns` n'est pas autorisé. Avec kindnet, le DNS peut sembler marcher encore un moment (kindnet garde un cache DNS), puis casser au prochain redémarrage d'un pod: ajoutez la règle dans tous les cas
- Pour tester: `kubectl run attacker -n default --rm -it --image=busybox:1.36 -- nc -zv -w 2 redis.data 6379`

### Vérifications

- [ ] Le guestbook fonctionne via l'Ingress
- [ ] Le guestbook peut joindre Redis
- [ ] Un pod de `default` ne peut pas joindre Redis (donnez la preuve)
- [ ] Redis ne peut pas joindre Internet (donnez la preuve)

## 9. (Optionnel) Détecter avec Falco

### Pourquoi

On ne bloquera jamais tout. Il faut **voir** l'attaque quand elle passe: un shell dans un conteneur, un binaire lancé depuis `/tmp` (cours: *"Détecter et répondre"*).

### Comment

0. **Vérifiez que votre machine est compatible** (voir ci-dessous)
1. **Installez [Falco](https://falco.org/docs/setup/kubernetes/)** avec Helm
2. **Ouvrez un shell** dans un pod et retrouvez l'alerte correspondante dans les logs de Falco
3. **Écrivez une règle custom** qui détecte un binaire lancé depuis `/tmp`, avec le pod, le namespace et l'image dans le message
4. **Rejouez le faux mineur** et montrez l'alerte
5. **(Bonus)** envoyez les alertes vers un webhook ou un Slack avec Falcosidekick

### Tips

- Falco observe les appels système du noyau: il a besoin d'un noyau Linux **≥ 5.8 avec BTF**. Avec minikube en driver Docker, c'est le noyau de votre machine (sous macOS et Windows, celui de la VM de Docker Desktop ou de WSL2, en général récent). Vérifiez dans le node :

  ```bash
  minikube -p security ssh -- 'uname -r; ls /sys/kernel/btf/vmlinux'
  ```

  Une version ≥ 5.8 et le fichier `/sys/kernel/btf/vmlinux` présent: tout va bien. Sinon, Falco ne démarrera pas (pod en `CrashLoopBackOff`, logs avec `failed to open` ou `BTF`): faites cet exercice en binôme sur la machine d'un camarade compatible, ou passez à la suite, c'est un exercice optionnel
- Le driver `modern_ebpf` est le plus simple, il n'a rien à télécharger ni compiler: `--set driver.kind=modern_ebpf`. Les autres drivers (`kmod`, `ebpf`) doivent télécharger ou compiler un composant pour votre noyau exact, et échouent le plus souvent dans minikube
- Les règles custom s'ajoutent via la valeur `customRules` du chart, et sont rechargées au redémarrage du pod Falco
- Le champ `proc.exepath` donne le chemin de l'exécutable; `container.id != host` limite la règle aux conteneurs
- Par défaut, Falco n'affiche que la **première** règle qui correspond à un événement: une règle intégrée peut masquer la vôtre. Ajoutez `falco.rule_matching: all` à vos valeurs Helm
- Si vous avez appliqué l'exercice 6, `/tmp` n'est plus inscriptible: pour tester votre règle, lancez le faux mineur dans un pod de test sans `readOnlyRootFilesystem` (et c'est une bonne nouvelle)

### Vérifications

- [ ] J'ai vérifié la compatibilité de mon noyau (ou noté pourquoi je n'ai pas pu faire l'exercice)
- [ ] Falco tourne sur le node
- [ ] Un `kubectl exec ... sh` génère une alerte
- [ ] Ma règle custom détecte le faux mineur, avec le contexte Kubernetes
- [ ] Je sais lire une alerte Falco (règle, priorité, processus, pod)

## 10. (Optionnel) Construire une matrice de risque

### Pourquoi

L'analyse de risque permet d'identifier, d'évaluer et de prioriser les menaces. Une matrice de risque permet de visualiser et de hiérarchiser les actions de sécurité selon leur impact et leur probabilité.

### Comment

1. **Analysez l'infrastructure** mise en place jusqu'ici, et les failles de votre rapport d'attaque (exercice 3)
2. **Identifiez les risques potentiels** pour chaque composant, y compris ceux que vous n'avez pas corrigés
3. **Évaluez l'impact et la probabilité** de chaque risque
4. **Construisez la matrice de risque** selon le modèle fourni
5. **Définissez les remédiations** appropriées pour chaque risque
6. **Priorisez les actions** selon les critères P0 à P4

### Tips

**Étapes de construction de la matrice :**

    | ID | Risque | Impact | Probabilité | Gestion du risque | Remédiation associée |
    |----|--------|--------|-------------|-------------------|----------------------|
    | 1  |        |        |             |                   |                      |

  - **Impact**

      **Faible**: effet quasi nul sur les opérations, la mission, les actifs, les personnes, les clients ou d'autres organisations.

      **Moyen**: effet limité: mission dégradée mais fonctions principales assurées, dommages ou pertes mineurs, impact cantonné à des ressources non critiques.

      **Élevé**: effet grave: forte dégradation ou perte de capacité, une ou plusieurs fonctions clés indisponibles, dommages ou pertes majeurs, impact étendu incluant des ressources critiques.

      **Critique**: effets graves multiples et d'ampleur générale, touchant presque toutes les ressources et parties prenantes.

  - **Probabilité**

      **Faible**: si peu probable qu'on peut considérer que l'événement ne surviendra pas (source peu motivée ou incapable, ou contrôles très efficaces).

      **Moyenne**: improbable mais possible (motivation ou capacité insuffisante, ou contrôles qui freinent l'exploitation).

      **Élevée**: probable (source motivée et capable, contrôles insuffisants pour empêcher l'exploitation).

      **Critique**: quasi certain ou imminent, possiblement récurrent (contrôles inexistants ou inefficaces).

  - **Gestion du risque** :

      **Acceptation**: Accepter le risque tel quel, sans prendre de mesures supplémentaires.

      **Mitigation**: Prendre des mesures pour réduire la probabilité ou l'impact du risque.

      **Transfert**: Transférer le risque à une autre partie, par exemple en sous-traitant.

      **Évitement**: Modifier les plans pour éliminer le risque ou son impact.

  - **Remédiation associée** : Faire le lien avec une remédiation dans la matrice de remédiation.

Documentez chaque remédiation dans un tableau.

    | Remédiation | Description | Priorité | Estimation en JH |
    |-------------|-------------|----------|------------------|
    |             |             |          |                  |

 - **Priorité:**

     P0 = Essentielles, à mettre en place dès le départ

     P1 = Critiques, à mettre en place avant le déploiement en production

     P2 = Majeures, à mettre en place dans les 3 premier mois du projet

     P3 = Importantes, à mettre en place dans les 6 premiers mois du projet

     P4 = Mineures, à mettre en place par la suite

### Vérifications

- [ ] Vous avez identifié au moins 5 risques pertinents
- [ ] Chaque risque a une évaluation d'impact et de probabilité
- [ ] La matrice de risque est complète et documentée
- [ ] Les remédiations sont définies avec des priorités
- [ ] Les failles corrigées dans la Partie B apparaissent avec leur risque résiduel

## 11. (Optionnel) Gérer les secrets avec External Secrets et Vault

### Pourquoi

Redis a maintenant besoin d'un mot de passe, et il ne doit être ni dans Git ni en clair dans le chart. Un gestionnaire de secrets le garde, et le cluster vient le chercher.

### Comment

1. **Déployez [Vault](https://developer.hashicorp.com/vault/docs/platform/k8s/helm)** en mode dev dans un namespace `vault`
2. **Stockez le mot de passe** de Redis dans Vault
3. **Installez [External Secrets Operator](https://external-secrets.io/latest/)** et configurez un `ClusterSecretStore` vers Vault
4. **Créez des `ExternalSecret`** qui créent le Secret du mot de passe dans `data` **et** dans `guestbook`
5. **Configurez Redis** (`--requirepass`) et le guestbook (`REDIS_PASSWORD`) pour l'utiliser
6. Rejouez l'étape 2 de l'exercice 2 sans mot de passe

### Tips

- En mode dev, Vault est en mémoire et son token root est `root`: parfait pour un TP, interdit en production
- Le mot de passe est passé à Redis via une variable d'environnement: `args: ["--requirepass", "$(REDIS_PASSWORD)"]`
- Vos NetworkPolicies (exercice 8) bloquent External Secrets ⇒ Vault ? Pas forcément: regardez dans quels namespaces ils tournent

### Vérifications

- [ ] Aucun mot de passe en clair dans Git
- [ ] Le Secret est créé et mis à jour par External Secrets
- [ ] `redis-cli` sans mot de passe est refusé (`NOAUTH`)
- [ ] Le guestbook fonctionne toujours

## 12. (Optionnel) Activer un WAF avec ModSecurity

### Pourquoi

Un Web Application Firewall (WAF) lit le contenu des requêtes HTTP et bloque les attaques connues: injections SQL, XSS, path traversal, scanners. ingress-nginx embarque [ModSecurity](https://github.com/owasp-modsecurity/ModSecurity) et l'[OWASP Core Rule Set](https://coreruleset.org/) (CRS), le jeu de règles cité dans le cours: il suffit de les activer.

### Comment

1. **Rejouez votre reconnaissance** (exercice 1) et ces requêtes, et notez les codes HTTP: `/lrange/guestbook?id=1' OR '1'='1`, `/lrange/guestbook?f=../../etc/passwd`, `/?q=<script>alert(1)</script>`, `/.git/config`, puis une requête avec l'en-tête `User-Agent: Nikto`
2. **Activez ModSecurity et le CRS** dans la configuration de l'ingress controller, en mode blocage
3. **Rejouez les mêmes requêtes**: les attaques doivent être bloquées, les requêtes normales du guestbook doivent passer
4. **Retrouvez dans les logs** de l'ingress controller la règle qui a bloqué chaque requête

### Tips

- La configuration globale d'ingress-nginx est la ConfigMap `ingress-nginx-controller` du namespace `ingress-nginx`. Les clés utiles: `enable-modsecurity`, `enable-owasp-modsecurity-crs` et `modsecurity-snippet` (pour passer `SecRuleEngine On`: par défaut ModSecurity ne fait que **détecter**, sans bloquer)
- Les annotations `modsecurity-snippet` sur l'Ingress sont refusées (*"Snippet directives are disabled"*): c'est voulu, un snippet permet d'injecter n'importe quelle configuration nginx. Passez par la ConfigMap
- Encodez les espaces et les quotes dans `curl` (`%20`, `%27`) ou utilisez `--data-urlencode` avec `-G`
- Les blocages apparaissent dans `kubectl logs -n ingress-nginx deploy/ingress-nginx-controller` avec `ModSecurity: Access denied`
- Le CRS travaille par score: chaque règle ajoute des points, la requête est bloquée au-delà d'un seuil (règle `949110`)

### Vérifications

- [ ] Les requêtes malveillantes reçoivent un `403`, le guestbook fonctionne toujours (lecture et écriture de messages)
- [ ] Je sais retrouver dans les logs la règle qui a bloqué une requête
- [ ] Je sais expliquer pourquoi un WAF ne remplace pas la correction du code (défense en profondeur)

## 13. (Optionnel) Mesurer les coûts avec OpenCost

### Pourquoi

Le faux mineur a consommé du CPU... qui coûterait de l'argent dans le cloud. Et vos `requests` sont-elles justes ? (cours: Partie 2, *"FinOps"*)

### Comment

1. **Déployez Prometheus** (kube-prometheus-stack, comme au Jour 4) et **[OpenCost](https://opencost.io/docs/installation/helm)**
2. **Affichez le coût** par namespace sur la dernière heure
3. **Relancez le faux mineur** dans un pod de test, et observez son effet sur le coût et sur l'efficacité
4. **Proposez de nouvelles `requests`** pour le guestbook, à partir de son usage réel (P95)

### Tips

- Sur minikube, OpenCost utilise une grille de prix par défaut: les montants sont indicatifs, les **proportions** sont justes
- L'efficacité = usage réel / requests
- Ajoutez `includeIdle=true` à la requête de l'API `/allocation` pour voir la ligne `__idle__`

### Vérifications

- [ ] J'ai le coût par namespace
- [ ] Je sais expliquer la ligne `__idle__`
- [ ] J'ai proposé des `requests` justifiées par des mesures

## Objectifs pédagogiques

### Savoir

- [ ] Je sais en quoi consiste un pentest
- [ ] Je sais distinguer un pentest en boîte noire, grise ou blanche
- [ ] Je connais les tâches de la blue team, de la red team et de la purple team
- [ ] Je sais ce qu’est un bug bounty
- [ ] Je sais ce qu’est l’OWASP et je connais son Top 10
- [ ] Je sais lire une CVE et un score CVSS
- [ ] Je connais au moins 5 tactiques MITRE ATT&CK
- [ ] Je sais ce que sont un SIEM et un SOC
- [ ] Je connais le rôle d’un WAF et je sais expliquer son fonctionnement
- [ ] Je sais expliquer le fonctionnement du mTLS
- [ ] Je sais expliquer le moindre privilège et Zero Trust
- [ ] Je comprends le besoin de fédération d’identité
- [ ] Je comprends le fonctionnement d’External Secrets
- [ ] Je comprends la sécurité réseau dans le cloud et dans Kubernetes
- [ ] Je connais les principes et le cycle de vie FinOps

### Savoir-faire

- [ ] Je sais cartographier une surface d'attaque
- [ ] Je sais exploiter une mauvaise configuration (endpoint de debug, service exposé, RBAC trop large)
- [ ] Je sais qualifier une faille
- [ ] Je sais détecter des secrets et scanner une image
- [ ] Je sais signer une image Docker et générer sa SBOM
- [ ] Je sais appliquer Pod Security et le moindre privilège avec RBAC
- [ ] Je sais écrire une admission policy avec Gatekeeper
- [ ] Je sais mettre en place une network policy et prouver qu'elle fonctionne
- [ ] Je sais déployer un IDS (Falco) et écrire une règle
- [ ] Je sais rédiger une matrice de risques

## Conseils pour aller plus loin

### Ressources d'apprentissage
- **OWASP Top 10**: <https://owasp.org/Top10/>
- **MITRE ATT&CK**: <https://attack.mitre.org/>
- **Kubernetes Security Checklist**: <https://kubernetes.io/docs/concepts/security/security-checklist/>
- **Root Me** (challenges, en français): <https://www.root-me.org/>
- **Kubernetes Goat** (cluster volontairement vulnérable): <https://madhuakula.com/kubernetes-goat/>

### Prochaines étapes
1. **Mettez gitleaks, Trivy et Cosign dans votre CI**, en mode bloquant
2. **Vérifiez les signatures à l'admission** (Kyverno ou le policy-controller de Sigstore)
3. **Envoyez les alertes Falco vers un SIEM** et écrivez des règles de corrélation

### Bonnes pratiques à retenir
- Rien d'inutile exposé: ni endpoint de debug, ni port
- Pas de secret dans Git ni dans l'image; un secret commité est un secret compromis
- Moindre privilège pour les humains, les pods et la CI
- Refusez par défaut (Pod Security, Gatekeeper, NetworkPolicies), puis ouvrez au cas par cas
- Prouvez chaque correction en rejouant l'attaque
