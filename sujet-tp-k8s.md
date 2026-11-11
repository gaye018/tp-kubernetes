# TP - Kubernetes avec Minikube
## Déploiement, résilience, mise à l’échelle et diagnostic


**Outils :** Docker, Minikube, kubectl, navigateur ou curl

---

## 1. Contexte professionnel

Vous êtes ingénieur DevOps dans une entreprise qui souhaite déployer une application web sur Kubernetes. L’équipe veut pouvoir déclarer l’état souhaité de l’application, rétablir automatiquement certaines pannes, faire évoluer le nombre d’instances et mettre à jour l’application sans gérer manuellement chaque conteneur.

Votre mission consiste à déployer une application web dans un cluster Kubernetes local, à expérimenter ses mécanismes d’orchestration et à documenter vos observations.

### Objectifs pédagogiques

À la fin du TP, vous etes capable de :

- expliquer le rôle des principaux composants Kubernetes ;
- créer et inspecter des ressources Kubernetes à partir de manifestes YAML ;
- distinguer Pod, ReplicaSet, Deployment et Service ;
- configurer une application web et l’exposer localement ;
- expérimenter la réconciliation et le remplacement d’un Pod ;
- configurer des probes de santé et des ressources CPU/mémoire ;
- mettre à l’échelle un Deployment ;
- effectuer une mise à jour et un retour arrière ;
- diagnostiquer des erreurs courantes à l’aide de `kubectl` ;
- discuter des limites d’un cluster local par rapport à un cluster de production.

---

## 2. Environnement de travail

### 2.1 Choisir une solution locale

Utilisez l’une des solutions suivantes :

- **Minikube avec Docker comme driver** — solution recommandée pour ce TP ;

Les commandes de ce sujet sont principalement prévues pour Minikube.

### 2.2 Prérequis conseillés

- Docker installé et démarré ;
- Minikube installé ;
- `kubectl` installé ;
- une connexion Internet pour télécharger les images nécessaires.

Documentation officielle :

- Minikube : https://minikube.sigs.k8s.io/docs/start/
- kubectl : https://kubernetes.io/docs/tasks/tools/
- Kubernetes dans Docker Desktop : https://docs.docker.com/desktop/features/kubernetes/

### 2.3 Vérifier les outils

```bash
docker --version
kubectl version --client
minikube version
```

### 2.4 Démarrer le cluster

```bash
minikube start --driver=docker --cpus=2 --memory=4096
```

Vérifiez son état :

```bash
kubectl cluster-info
kubectl get nodes
kubectl get pods -A
```

Le nœud doit être à l’état `Ready`.

**Si le démarrage échoue :** vérifiez que Docker fonctionne, que le poste dispose de suffisamment de mémoire et que la virtualisation requise par votre environnement est opérationnelle.

---

## 3. Partie 1 — Explorer l’architecture Kubernetes (30 min)

### Travail à réaliser

Exécutez les commandes suivantes et interprétez leurs résultats :

```bash
kubectl get nodes -o wide
kubectl get namespaces
kubectl get pods -A
kubectl get deployments -A
kubectl describe node
```

Pour consulter les composants du système :

```bash
kubectl get pods -n kube-system
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

La commande `kubectl get componentstatuses` est obsolète dans les versions récentes de Kubernetes. Utilisez plutôt les informations sur les Pods du système et les mécanismes de santé adaptés à la version du cluster.

### Questions

1. Quel est le rôle du control plane ?
Le control plane est le cerveau du cluster. Il reçoit les demandes (kube-apiserver), stocke l'état désiré et réel (etcd), décide sur quel nœud placer les Pods (scheduler) et exécute les contrôleurs qui corrigent en permanence l'écart entre l'état désiré et l'état réel (controller-manager).

2. Quelle différence existe-t-il entre un nœud Kubernetes et un Pod ?

Un nœud est une machine (physique ou virtuelle) qui fournit CPU, mémoire et réseau. Un Pod est une unité d'exécution logique, éphémère, qui tourne sur un nœud. Un nœud peut héberger plusieurs Pods.

3. Quel contrôleur maintient le nombre de réplicas demandé par un Deployment ?

Le contrôleur du ReplicaSet. Le Deployment crée et pilote un ReplicaSet, et c'est celui-ci qui maintient le nombre de Pods demandé.

4. Quelle différence existe-t-il entre un conteneur et un Pod ?
Un conteneur est un processus isolé lancé à partir d'une image. Un Pod est l'enveloppe Kubernetes qui contient un ou plusieurs conteneurs partageant la même IP, le même réseau et les mêmes volumes, et qui est planifiée comme un tout.

5. Quelles limites présente un cluster local à un seul nœud par rapport à un cluster de production ?

Un seul nœud signifie aucun point de reprise en cas de panne (le control plane et les applications tombent ensemble), pas de haute disponibilité du control plane, pas de multi-zone, des ressources limitées à celles du poste, pas de vrai LoadBalancer ni d'IP publique, et des composants de stockage, de réseau, de sécurité et de supervision simplifiés. Cette limite s'est vérifiée en pratique : après l'extinction de ma machine, kubectl répondait no route to host et plus rien ne fonctionnait avant minikube start.

### Livrable

Réalisez un schéma simple du cluster local et expliquez le rôle des composants que vous avez identifiés.

```text
Mon PC
└── Docker
    └── Nœud "minikube"
        ├── Control plane : kube-apiserver, etcd, kube-scheduler, kube-controller-manager
        ├── Sur le nœud : kubelet, kube-proxy, containerd
        ├── Réseau et DNS : kindnet, coredns
        ├── Stockage : storage-provisioner
        └── Namespace tp-k8s
            Service web → Deployment web → ReplicaSet → 2 Pods nginx (page via la ConfigMap)
```

- kube-apiserver : reçoit toutes les commandes kubectl.
- etcd : garde l'état du cluster.
- kube-scheduler : choisit le nœud de chaque Pod.
- kube-controller-manager : vérifie que l'état réel correspond à l'état demandé.
- kubelet : lance les conteneurs sur le nœud et fait passer les probes.
- kube-proxy : gère le réseau des Services.
- coredns : le DNS interne du cluster.
- kindnet : le réseau entre les Pods.
- storage-provisioner : crée les volumes de stockage.

---

## 4. Partie 2 — Déployer une application web (45 min)

### Objectif

Déployer Nginx avec deux réplicas, une page HTML personnalisée et un Service Kubernetes.

### 4.1 Créer l’arborescence

Créez le répertoire suivant :

```text
tp-kubernetes/
└── k8s/
    ├── 01-namespace.yaml
    ├── 02-configmap.yaml
    ├── 03-deployment.yaml
    └── 04-service.yaml
```

Placez-vous dans `tp-kubernetes` pour exécuter les commandes suivantes.

### 4.2 Créer un namespace

Créez `k8s/01-namespace.yaml` :

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tp-k8s
```

Appliquez le manifeste :

```bash
kubectl apply -f k8s/01-namespace.yaml
kubectl get namespaces
```

**Question 2.** Quel intérêt présente un namespace dans un environnement partagé entre plusieurs équipes ?
Un namespace sépare les ressources de chaque équipe ou projet. Il n'y a pas de conflit de noms, et on peut donner des droits et des limites de ressources différents à chaque équipe.

### 4.3 Créer une page web avec une ConfigMap

Créez `k8s/02-configmap.yaml` :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: page-web
  namespace: tp-k8s
data:
  index.html: |
    <!DOCTYPE html>
    <html lang="fr">
    <head>
      <meta charset="UTF-8">
      <title>TP Kubernetes</title>
    </head>
    <body>
      <h1>Application déployée sur Kubernetes</h1>
      <p> Cours sur le DevOps .</p>
    </body>
    </html>
```

Appliquez la configuration :

```bash
kubectl apply -f k8s/02-configmap.yaml
kubectl get configmap -n tp-k8s
kubectl describe configmap page-web -n tp-k8s
```

**Question 3.** Pourquoi utiliser une ConfigMap plutôt que d’intégrer directement la page dans le manifeste du Deployment ?

Une ConfigMap sépare la configuration du déploiement et de l'image. On peut modifier la page sans toucher au Deployment ni reconstruire d'image, la réutiliser dans plusieurs Pods ou environnements, et la versionner indépendamment. Le manifeste du Deployment reste court et lisible.
### 4.4 Créer le Deployment

Créez `k8s/03-deployment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: tp-k8s
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:stable-alpine
          ports:
            - name: http
              containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
            limits:
              cpu: 250m
              memory: 128Mi
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 2
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
          volumeMounts:
            - name: page-web
              mountPath: /usr/share/nginx/html/index.html
              subPath: index.html
      volumes:
        - name: page-web
          configMap:
            name: page-web
```

Appliquez le manifeste et observez le résultat :

```bash
kubectl apply -f k8s/03-deployment.yaml
kubectl get deployments -n tp-k8s
kubectl get replicasets -n tp-k8s
kubectl get pods -n tp-k8s -o wide
```

Attendez que les deux Pods soient prêts.

**À retenir :** `replicas: 2` indique l’état souhaité. Le contrôleur du Deployment s’appuie sur un ReplicaSet pour maintenir le nombre de Pods demandé.

### 4.5 Créer le Service

Créez `k8s/04-service.yaml` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: tp-k8s
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
    - name: http
      port: 80
      targetPort: http
```

Appliquez et inspectez le Service :

```bash
kubectl apply -f k8s/04-service.yaml
kubectl get services -n tp-k8s
kubectl get endpointslices -n tp-k8s
```

Le Service sélectionne les Pods qui possèdent le label `app: web`. Il fournit une adresse stable à l’intérieur du cluster.

### 4.6 Accéder à l’application

Dans un terminal, lancez :

```bash
kubectl port-forward -n tp-k8s service/web 8080:80
```

Laissez ce terminal ouvert et accédez à :

http://localhost:8080

Vous pouvez aussi tester depuis un second terminal :

```bash
curl -i http://localhost:8080
```

**Résultat attendu :** la page HTML personnalisée s’affiche.

### Questions

1. Quel est le rôle du Deployment par rapport au ReplicaSet ?

Le Deployment gère l'application (mises à jour, retour arrière) et s'appuie sur un ReplicaSet. Le ReplicaSet s'occupe juste de garder le bon nombre de Pods.

2. Pourquoi ne pas accéder directement aux adresses IP des Pods ?

Parce que l'IP d'un Pod change dès qu'il est remplacé. Le Service garde une adresse fixe et envoie le trafic vers les Pods prêts.

3. Quelle différence existe-t-il entre `ClusterIP`, `NodePort` et `LoadBalancer` ?
ClusterIP n'est accessible que dans le cluster. NodePort ouvre un port sur le nœud pour y accéder de l'extérieur. LoadBalancer utilise un répartiteur de charge du cloud avec une IP publique.

4. Quelle différence existe-t-il entre une readiness probe et une liveness probe ?
La readiness dit si le Pod peut recevoir du trafic : si elle échoue, le Pod est retiré du Service mais pas redémarré. La liveness dit si le conteneur est encore vivant : si elle échoue, il est redémarré.
5. Que peut-il se passer si une limite mémoire est trop basse ?

 Le conteneur dépasse la limite, il est tué (OOMKilled) puis redémarré, et ça peut tourner en boucle (CrashLoopBackOff).

---

## 5. Partie 3 — Résilience et auto-rétablissement (40 min)

### 5.1 Supprimer un Pod

Listez les Pods :

```bash
kubectl get pods -n tp-k8s
```

Supprimez un Pod en remplaçant `<nom-du-pod>` par son nom réel :

```bash
kubectl delete pod <nom-du-pod> -n tp-k8s
```

Dans un autre terminal, observez les changements :

```bash
kubectl get pods -n tp-k8s -w
```

Arrêtez l’observation avec `Ctrl+C`.

**Résultat attendu :** le Deployment demande le maintien de deux réplicas. Un Pod de remplacement est créé.

### 5.2 Examiner les journaux et les événements

```bash
kubectl describe pod <nom-du-pod> -n tp-k8s
kubectl logs <nom-du-pod> -n tp-k8s
kubectl get events -n tp-k8s --sort-by=.metadata.creationTimestamp
```

Si le Pod supprimé n’existe plus, sélectionnez un Pod encore présent.

### Questions d’analyse

1. Pourquoi le Pod supprimé est-il remplacé ?

Parce que le Deployment demande 2 Pods. Le ReplicaSet voit qu'il n'y en a plus qu'un et en recrée un.


2. Quelle différence existe-t-il entre la suppression d’un Pod et l’arrêt d’un processus dans un conteneur ?

Supprimer un Pod le fait disparaître, et un nouveau est créé avec un autre nom et une autre IP. Si le processus d'un conteneur s'arrête, le Pod reste et le conteneur est juste redémarré dedans (RESTARTS augmente).

3. Pourquoi un redémarrage de conteneur ne signifie-t-il pas nécessairement que le cluster entier est en panne ?
C'est un mécanisme normal de réparation qui ne touche qu'un conteneur, le reste continue de tourner. Par exemple storage-provisioner a planté au redémarrage de Minikube puis s'est relancé seul.
4. Que se passerait-il si le nœud unique de Minikube devenait indisponible ?

Tout s'arrêterait, les applications et le control plane, car aucun autre nœud ne peut reprendre. C'est ce qui s'est passé quand j'ai éteint ma machine (no route to host).

5. Pourquoi un cluster local à un seul nœud ne permet-il pas de démontrer la tolérance à la panne d’un nœud physique ?

Avec un seul nœud, on ne teste que la panne d'un Pod. Il faudrait au moins deux nœuds pour voir les Pods repartir sur une autre machine.

---

## 6. Partie 4 — Mise à l’échelle et découverte de services (35 min)

### 6.1 Augmenter le nombre de réplicas

Passez de deux à quatre instances :

```bash
kubectl scale deployment/web --replicas=4 -n tp-k8s
kubectl get deployment web -n tp-k8s
kubectl get pods -n tp-k8s -o wide
```

Réduisez ensuite le nombre d’instances à une :

```bash
kubectl scale deployment/web --replicas=1 -n tp-k8s
kubectl get pods -n tp-k8s
```

### 6.2 Examiner les points de terminaison du Service

Repassez à quatre réplicas :

```bash
kubectl scale deployment/web --replicas=4 -n tp-k8s
kubectl get endpointslices -n tp-k8s
```

Le Service peut distribuer les connexions vers les Pods prêts. La page étant identique sur toutes les instances, cette expérience ne permet pas à elle seule de mesurer la répartition du trafic.

### Questions

1. Quelle différence existe-t-il entre la mise à l’échelle manuelle et l’autoscaling ?

En manuel, je choisis moi-même le nombre de réplicas. Avec l'autoscaling, le système l'ajuste tout seul selon la charge.

2. Quel est le rôle d’un Horizontal Pod Autoscaler (HPA) ?

Le HPA augmente ou diminue le nombre de réplicas automatiquement, entre un minimum et un maximum, pour garder une métrique cible comme l'utilisation du CPU.

3. Pourquoi un HPA basé sur le CPU nécessite-t-il des métriques et des ressources correctement configurées ?

Il a besoin du metrics-server pour mesurer la consommation, et il calcule le pourcentage de CPU par rapport aux requests. Sans requests, ou avec de mauvaises valeurs, il décide mal.

4. Pourquoi quatre réplicas sur un seul nœud ne garantissent-ils pas la disponibilité en cas de panne de ce nœud ?
 Parce que les 4 Pods sont sur le même nœud. Si le nœud tombe, ils tombent tous.
---


## 7. Partie 5 — Mise à jour et retour arrière (35 min)

### 7.1 Modifier la page web

Modifiez le titre dans `k8s/02-configmap.yaml` :

```html
<h1>Version 2 — Application Kubernetes</h1>
```

Réappliquez la ConfigMap :

```bash
kubectl apply -f k8s/02-configmap.yaml
```

Le fichier est monté avec `subPath`. Dans cette configuration, les modifications de la ConfigMap ne sont pas automatiquement répercutées dans le fichier monté. Redémarrez donc le Deployment :

```bash
kubectl rollout restart deployment/web -n tp-k8s
kubectl rollout status deployment/web -n tp-k8s
```

Rechargez http://localhost:8080.

### 7.2 Examiner l’historique du Deployment

```bash
kubectl rollout history deployment/web -n tp-k8s
kubectl describe deployment web -n tp-k8s
```

### 7.3 Expérimenter une mise à jour d’image

Modifiez l’image du conteneur :

```bash
kubectl set image deployment/web nginx=nginx:alpine -n tp-k8s
kubectl rollout status deployment/web -n tp-k8s
```

Observez le remplacement progressif des Pods :

```bash
kubectl get pods -n tp-k8s -w
```

Pour revenir à la révision précédente du Deployment :

```bash
kubectl rollout undo deployment/web -n tp-k8s
kubectl rollout status deployment/web -n tp-k8s
```

**Attention :** les tags d’images tels que `alpine` sont mutables. En production, il faut contrôler les versions utilisées et envisager l’identification des images par digest pour renforcer la reproductibilité.

### Questions

1. Que signifie la stratégie `RollingUpdate` ?
  Les Pods sont remplacés petit à petit par la nouvelle version, quelques-uns à la fois, pour que l'application reste disponible. Chez moi, c'était un Pod sur 4 à la fois.


2. Pourquoi une mise à jour progressive ne garantit-elle pas, à elle seule, une disponibilité parfaite ?
Pendant la mise à jour, les deux versions tournent en même temps, des connexions peuvent être coupées, et une version cassée peut quand même passer les probes et recevoir du trafic.

3. Dans quelles conditions `rollout undo` fonctionne-t-il ?

Il faut que l'historique soit gardé (les anciens ReplicaSet). Il restaure le modèle du Pod  mais pas la ConfigMap ni les données de la base.
4. Pourquoi faut-il versionner les configurations et planifier les migrations de base de données séparément ?

 Versionner permet de savoir ce qui a changé et de revenir en arrière, ce que rollout undo ne fait pas pour une ConfigMap. Une migration de base de données ne s'annule pas avec l'application, donc il faut la planifier à part.




---

## 8. Partie 6 — Diagnostic et dépannage (30 min)

Pour chaque incident, suivez une démarche structurée : **symptôme → hypothèse → diagnostic → correction → vérification**.

### Incident A — Image introuvable

Modifiez temporairement l’image du Deployment pour utiliser un nom d’image inexistant.

Inspectez les Pods :

```bash
kubectl get pods -n tp-k8s
kubectl describe pod <nom-du-pod> -n tp-k8s
kubectl get events -n tp-k8s --sort-by=.metadata.creationTimestamp
```

**Questions :**

- Quel état observez-vous ?
Le nouveau Pod reste en `0/1`, d'abord en `ErrImagePull` puis en `ImagePullBackOff`. Les 2 anciens Pods restent en `1/1 Running`.

- Quel événement indique la cause probable ?
Failed to pull image "nginx:inexistant-xyz" ... not found` : l'image n'existe pas.
- Comment corriger le manifeste et vérifier le retour à la normale ?
J'ai remis `image: nginx:stable-alpine` dans `03-deployment.yaml` et refait `kubectl apply`. Ensuite `rollout status` a confirmé le déploiement et `get pods` montre 2 Pods en `1/1 Running`.

### Incident B — Service inaccessible

Modifiez temporairement le sélecteur du Service pour qu’il ne corresponde plus aux labels des Pods.

Inspectez les points de terminaison :

```bash
kubectl get services -n tp-k8s
kubectl get endpointslices -n tp-k8s
```


**Questions :**

- Pourquoi le Service n'a-t-il plus de destination exploitable ?
Parce que j'ai changé son sélecteur en `app: web-casse`, et mes Pods s'appellent `app: web`. Du coup le Service ne trouve plus aucun Pod, et sa liste d'endpoints est vide.

- Pourquoi les Pods peuvent-ils être opérationnels alors que l'application est inaccessible via le Service ?
Les Pods n'avaient rien de cassé, ils étaient toujours en `Running`. C'est juste que le Service passe par les labels pour les trouver, et là ils ne correspondaient plus. Donc personne ne pouvait les joindre par le Service.

- Quelle correction permet de rétablir le routage ?
Remettre `app: web` dans le sélecteur du Service, puis refaire `kubectl apply -f k8s/04-service.yaml`. Après ça, les 2 IP des Pods sont revenues dans les endpoints.

### Incident C — Sonde de santé incorrecte

Configurez temporairement une route HTTP inexistante dans la readiness probe, puis observez :

```bash
kubectl describe pod <nom-du-pod> -n tp-k8s
kubectl get pods -n tp-k8s
kubectl get endpointslices -n tp-k8s
```

**Questions :**

- Quel est l'effet d'une readiness probe en échec ?
Le Pod reste en `Running` mais il passe en `0/1` : Kubernetes considère qu'il n'est pas prêt. Il est retiré des endpoints du Service, donc il ne reçoit plus de trafic. Par contre il n'est pas redémarré.

- En quoi son effet diffère-t-il de celui d'une liveness probe en échec ?
Si c'est la liveness qui échoue, Kubernetes pense que le conteneur est bloqué et le redémarre (le compteur RESTARTS augmente). Avec la readiness, il ne fait que le mettre de côté sans toucher au conteneur.

- Pourquoi une sonde mal configurée peut-elle provoquer une indisponibilité ?
Si la readiness échoue sur tous les Pods, plus aucun n'est prêt et le Service n'a personne vers qui envoyer le trafic. Pendant une mise à jour, les nouveaux Pods n'arrivent jamais à `1/1`, donc la mise à jour reste bloquée. Une liveness mal réglée peut aussi redémarrer en boucle des conteneurs qui vont bien.


**Consigne :** restaurez tous les manifestes à un état fonctionnel après les expériences.

---

## 9. Mini-projet de synthèse

Faites évoluer le déploiement afin de satisfaire les exigences suivantes :

- [x] Déployer une application avec au moins deux réplicas.
- [x] Créer un namespace et un Service dédiés.
- [x] Configurer des probes de santé.
- [x] Définir des requests et des limits CPU/mémoire.
- [x] Démontrer le remplacement automatique d'un Pod supprimé.
- [x] Réaliser une mise à l'échelle manuelle.
- [x] Effectuer une mise à jour puis un retour arrière.
- [x] Provoquer et diagnostiquer un incident.
- [x] Documenter les commandes et les observations.
- [x] Justifier les choix d'architecture et leurs limites.



### Livrables attendus

1. Le répertoire `k8s/` contenant les manifestes YAML.

2. Une analyse des limites d’un cluster local par rapport à un cluster cloud de production.

Mon cluster n'a qu'un seul nœud. Si ma machine s'éteint, tout s'arrête, même le control plane : je l'ai vu avec le "no route to host". En production, il y a plusieurs nœuds, souvent dans plusieurs zones, donc les Pods peuvent repartir ailleurs si une machine tombe. Je ne peux donc tester que la panne d'un Pod, pas celle d'une machine.
Les ressources sont limitées à celles de mon PC (4 CPU et 16 Go d'après `describe node`, dont 2 CPU et 4 Go donnés à Minikube), alors qu'un cloud peut ajouter des nœuds quand la charge monte.
Pour accéder au site, j'ai dû utiliser `port-forward`. En production, on utilise un Service LoadBalancer ou un Ingress avec une vraie adresse.



**Justification des choix d'architecture et limites**

- 2 réplicas : je peux supprimer un Pod sans couper le site. Limite : les deux sont sur le même nœud, donc si le nœud tombe, tout tombe.
- Namespace `tp-k8s` : le TP reste séparé du reste et je peux tout supprimer d'un coup.
- ConfigMap pour la page : je change la page sans toucher à l'image. Limite : avec `subPath`, il faut redémarrer les Pods.
- Service en ClusterIP : suffisant pour le TP. Limite : il n'est pas accessible de l'extérieur, d'où le `port-forward`.
- Requests et limits : un Pod ne peut pas prendre toutes les ressources. Limite : si la limite mémoire est trop basse, le Pod est tué.
- Probes : pas de trafic vers un Pod pas prêt, et un conteneur bloqué est relancé. Limite : une probe mal réglée peut causer une panne, comme dans l'incident C.
---

## 12. Conclusion attendue

Répondez à la question suivante :

> En quoi Kubernetes apporte-t-il des mécanismes d’orchestration qui dépassent la simple exécution de conteneurs avec Docker, et quelles précautions restent nécessaires pour exploiter une application de manière fiable en production ?

Avec Docker, je lance un conteneur, mais s'il plante ou si j'en veux plusieurs, je dois tout faire moi-même. Avec Kubernetes, je dis ce que je veux (par exemple 2 réplicas) et il s'occupe de le maintenir. Pendant le TP, j'ai vu un Pod supprimé être recréé tout seul, j'ai changé le nombre de réplicas avec une commande, j'ai fait une mise à jour Pod par Pod puis un retour arrière, et j'ai retrouvé l'origine des erreurs avec `describe` et `get events`.
Mais Kubernetes ne suffit pas pour avoir une application fiable en production. Il faut plusieurs nœuds pour résister à la panne d'une machine, bien régler les probes et les ressources, versionner les images et les fichiers de configuration, gérer la sécurité, surveiller l'application, faire des sauvegardes et prévoir à part les migrations de base de données.

## 13. Nettoyage de l’environnement

Supprimez les ressources du TP :

```bash
kubectl delete namespace tp-k8s
```

Si vous utilisez Minikube et souhaitez simplement arrêter le cluster :

```bash
minikube stop
```

Pour supprimer entièrement le cluster local et ses données :

```bash
minikube delete
```

**Attention :** `minikube delete` supprime le cluster concerné. Ne l’exécutez pas si vous y avez conservé d’autres travaux.







