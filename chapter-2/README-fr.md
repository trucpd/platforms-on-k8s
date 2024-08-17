# Chapitre 2 :: Défis des Applications Cloud-Native

---
_🌍 Disponible en_: [English](README.md) | [中文 (Chinois)](README-zh.md) | [Português (Portugais)](README-pt.md) | [日本語 (Japonais)](README-ja.md) | [Español](README-es.md) | [Français](README-fr.md)

> **Remarque :** Proposé par les [ 🌟 contributeurs](https://github.com/salaboy/platforms-on-k8s/graphs/contributors) de la fantastique communauté cloud-native!

Dans ce court tutoriel, nous allons installer l'`Application Conférence` en utilisant Helm dans un cluster Kubernetes KinD local.

> [!NOTE]
> Les Helm Charts peuvent être publiés dans des dépôts Helm Chart ou, depuis Helm 3.7, en tant que conteneurs OCI dans des registres de conteneurs.

## Création d'un cluster local avec Kubernetes KinD

> [!Important]
> Assurez-vous de disposer des prérequis pour tous les tutoriels. Vous pouvez les trouver [ici](../chapter-1/README-fr.md#prérequis-pour-les-tutoriels).

Utilisez la commande ci-dessous pour créer un cluster KinD avec trois nœuds de travail et 1 Plan de Contrôle.

```shell
cat <<EOF | kind create cluster --name dev --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
- role: worker
- role: worker
- role: worker
EOF

```

![3 nœuds de travail](imgs/cluster-topology.png)

### Chargement des images de conteneurs avant l'installation de l'application et d'autres composants

Le script `kind-load.sh` précharge, c'est-à-dire télécharge et charge les images de conteneurs que nous utiliserons pour notre application dans notre cluster KinD.

L'idée ici est d'optimiser le processus pour notre cluster, afin que lorsque nous installerons l'application, nous n'ayons pas à attendre plus de 10 minutes pendant que toutes les images de conteneurs nécessaires sont récupérées. Avec toutes les images déjà préchargées dans notre cluster KinD, l'application devra démarrer en environ 1 minute, ce qui est le temps nécessaire pour que PostgreSQL, Redis et Kafka se lancent.

Maintenant, récupérons les images requises dans notre cluster KinD.

> [!Important]
> En exécutant le script indiqué à l'étape suivante, vous téléchargerez et chargerez toutes les images nécessaires dans chaque nœud de votre cluster KinD. Si vous exécutez les exemples sur un fournisseur cloud, cette étape pourrait ne pas être nécessaire, car ces fournisseurs bénéficient de connexions haut débit aux registres de conteneurs, permettant de récupérer les images en quelques secondes.

Dans votre terminal, accédez au répertoire `chapter-2`, puis exécutez le script :

```shell
./kind-load.sh
```

> [!Note]
> Si vous utilisez Docker Desktop sur MacOS et avez défini une taille plus petite pour le disque virtuel, vous pourriez rencontrer l'erreur suivante :
>
> ```shell
> $ ./kind-load.sh
> ...
> Command Output: Error response from daemon: write /var/lib/docker/...
> /layer.tar: no space left on device
> ```
>
> Vous pouvez modifier la valeur de la limite du disque virtuel dans le menu ``Settings -> Resources``.
>   ![Limites du disque virtuel Docker Desktop sur MacOS](imgs/macos-docker-desktop-virtual-disk-setting.png)

### Installation du NGINX Ingress Controller

Nous avons besoin du NGINX Ingress Controller pour diriger le trafic de notre ordinateur portable vers les services exécutés à l'intérieur du cluster. Le NGINX Ingress Controller fonctionne comme un routeur: il opère au sein du cluster tout en étant accessible depuis l'extérieur.

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

Vérifiez que les pods à l'intérieur du `ingress-nginx` sont correctement démarrés avant de continuer:
 
```shell
> kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-cflcl        0/1     Completed   0          62s
ingress-nginx-admission-patch-sb64q         0/1     Completed   0          62s
ingress-nginx-controller-5bb6b499dc-7chfm   0/1     Running     0          62s
```

Cela devrait vous permettre d'acheminer le trafic de `http://localhost` vers les services à l'intérieur du cluster. Notez que pour que KinD fonctionne de cette manière, nous avons fourni des paramètres et des labels supplémentaires pour le nœud de plan de contrôle lorsque nous avons créé le cluster:
```yaml
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true" #This allow the ingress controller to be installed in the control plane node
  extraPortMappings:
  - containerPort: 80 # Cela nous permet de lier le port 80 sur l'hôte local à l'Ingress Controller, afin qu'il puisse acheminer le trafic vers les services exécutés à l'intérieur du cluster.
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
```

Une fois notre cluster et notre Ingress Controller installés et configurés, nous pouvons passer à l'installation de notre application.

### Installation de l'Application de Conférence

Depuis Helm 3.7+, nous pouvons utiliser des images OCI pour publier, télécharger et installer des Helm Charts. Cette approche utilise Docker Hub comme registre Helm Chart.

Pour installer l'Application de Conférence, il vous suffit d'exécuter la commande suivante:

```shell
helm install conference oci://docker.io/salaboy/conference-app --version v1.0.0
```

Vous pouvez également exécuter la commande suivante pour voir les détails du chart:

```shell
helm show all oci://docker.io/salaboy/conference-app --version v1.0.0
```

Vérifiez que tous les pods de l'application sont en cours d'exécution.

> [!Note]
> Notez que si votre connexion Internet est lente, il peut falloir un certain temps avant que l'application ne démarre. Étant donné que les services de l'application dépendent de certains composants d'infrastructure (Redis, Kafka, PostgreSQL), ces composants doivent démarrer et être prêts pour que les services puissent se connecter.
>
> Les composants comme Kafka sont assez lourds, avec environ 335+ Mo, PostgreSQL 88+ Mo, et Redis 35+ Mo.

Vous devriez éventuellement voir quelque chose comme ceci. Cela peut prendre quelques minutes:

```shell
kubectl get pods
NAME                                                           READY   STATUS    RESTARTS      AGE
conference-agenda-service-deployment-7cc9f58875-k7s2x          1/1     Running   4 (45s ago)   2m2s
conference-c4p-service-deployment-54f754b67c-br9dg             1/1     Running   4 (65s ago)   2m2s
conference-frontend-deployment-74cf86495-jthgr                 1/1     Running   4 (56s ago)   2m2s
conference-kafka-0                                             1/1     Running   0             2m2s
conference-notifications-service-deployment-7cbcb8677b-rz8bf   1/1     Running   4 (47s ago)   2m2s
conference-postgresql-0                                        1/1     Running   0             2m2s
conference-redis-master-0                                      1/1     Running   0             2m2s
```

La colonne `RESTARTS` des pods indique que Kafka a peut-être mis du temps à démarrer, ce qui a conduit Kubernetes à démarrer le service en premier et à le redémarrer en attendant que Kafka soit prêt.

Vous pouvez maintenant ouvrir votre navigateur et accéder à [http://localhost](http://localhost) pour visualiser l'application.

![app conférence](imgs/conference-app-homepage.png)

------
## [Important] Nettoyage - _!!!À LIRE ABSOLUMENT!!_
Étant donné que l'Application de Conférence installe PostgreSQL, Redis et Kafka, si vous souhaitez supprimer et réinstaller l'application (ce que nous ferons au fur et à mesure des guides), vous devez vous assurer de supprimer les PersistenceVolumeClaims (PVC) associés.

Ces PVC sont les volumes utilisés pour stocker les données des bases de données et de Kafka. Ne pas supprimer ces PVC entre les installations peut entraîner l'utilisation d'anciens identifiants pour se connecter aux nouvelles bases de données provisionnées.

Vous pouvez lister tous les PVC avec:

```shell
kubectl get pvc
```

Vous devriez voir quelque chose comme ceci:

```shell
NAME                                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-conference-kafka-0                Bound    pvc-2c3ccdbe-a3a5-4ef1-a69a-2b1022818278   8Gi        RWO            standard       8m13s
data-conference-postgresql-0           Bound    pvc-efd1a785-e363-462d-8447-3e48c768ae33   8Gi        RWO            standard       8m13s
redis-data-conference-redis-master-0   Bound    pvc-5c2a96b1-b545-426d-b800-b8c71d073ca0   8Gi        RWO            standard       8m13s
```

Puis supprimez-les avec:

```shell
kubectl delete pvc  data-conference-kafka-0 data-conference-postgresql-0 redis-data-conference-redis-master-0
```

Les noms des PVC changeront en fonction du nom de la release Helm que vous avez utilisé lors de l'installation du chart.

Enfin, si vous souhaitez supprimer complètement le cluster KinD, vous pouvez exécuter:

```shell
kind delete clusters dev
```

## Étapes suivantes

Je vous recommande vivement de vous familiariser avec un vrai cluster Kubernetes hébergé par un fournisseur de cloud. Vous pouvez essayer la plupart des fournisseurs de cloud, car ils offrent un essai gratuit permettant de créer des clusters Kubernetes et de réaliser tous ces exemples [consultez ce dépôt pour plus d'informations](https://github.com/learnk8s/free-kubernetes).

Si vous parvenez à créer un cluster chez un fournisseur de cloud et à faire fonctionner l'application, vous acquérirez une expérience concrète sur tous les sujets abordés dans le Chapitre 2.


## Résumé et Contribuer
Dans ce court tutoriel, nous avons réussi à installer le squelette fonctionnel de l'application Conférence. Nous utiliserons cette application comme exemple tout au long des chapitres suivants. Assurez-vous que cette application fonctionne pour vous, car elle couvre les bases de l'utilisation et de l'interaction avec un cluster Kubernetes.

Vous souhaitez améliorer ce tutoriel? Créez un [ticket](https://github.com/salaboy/platforms-on-k8s/issues/new), envoyez-moi un message sur [Twitter](https://twitter.com/salaboy), ou soumettez une [pull request](https://github.com/salaboy/platforms-on-k8s/compare).









