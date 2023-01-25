# Kubernetes Ingress
> Rakotomalala Rantoniaina Antonio
> Master 2 Cloud - Infra - Sécurité 

## Installer docker
* Créer un script bash dockerI.sh pour installer docker
* Lancer l'installation de docker 

## Installer kubectl
```bash=
$ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
$ curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
$ echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
kubectl: OK
$ sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```
* Lancer le script dockerI.sh
```bash=
$ chmod 755 dockerI.sh
$ ./dockerI.sh
```

## Installer Kind et créer votre premier cluster
https://kind.sigs.k8s.io/docs/user/quick-start/

### Installation de Kind
```bash=
$ curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.17.0/kind-linux-amd64
$ chmod +x ./kind
$ sudo mv ./kind /usr/local/bin/kind
```

### Création de cluster
```bash=
$ kind create cluster

Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.25.3) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/
```
### Vérification du cluster
```bash=
$ kubectl cluster-info --context kind-kind

Kubernetes control plane is running at https://127.0.0.1:38971
CoreDNS is running at https://127.0.0.1:38971/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

## Installer le Nginx Ingress Controller
https://kind.sigs.k8s.io/docs/user/ingress/#ingress-nginx

### Créer un cluster en mappant les ports 80 et 443 et en utilisant un node spécifique du cluster
```bash=
$ cat <<EOF | kind create cluster --config=-
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
EOF
```
### Déployer l'ingress Nginx
```bash=
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```
### Vérifier que le déploiement fonctionne
```bash=
$ kubectl get pods -A
NAMESPACE            NAME                                         READY   STATUS      RESTARTS   AGE
ingress-nginx        ingress-nginx-admission-create-g7zhb         0/1     Completed   0          3m44s
ingress-nginx        ingress-nginx-admission-patch-kpj6l          0/1     Completed   0          3m44s
ingress-nginx        ingress-nginx-controller-6bccc5966-pptjt     1/1     Running     0          3m44s
```
```bash=
$ kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s

pod/ingress-nginx-controller-6bccc5966-pptjt ondition met

```

## Schéma avec des objets Kubernetes
![](https://i.imgur.com/xN8x3Px.png)

## Builder et publier (à partir de l’image nginx) sur le DockerHub, une image docker pour chacun des sites web présent sur le schéma précédent. Vous devez avoir 3 images (une par magasin tacos, pizzas et burgers)

### Image docker pour tacos
* Créer un dossier tacos et configurer le fichier Dockerfile
```bash=
$ mkdir tacos
$ touch tacos/Dockerfile

FROM nginx:latest

COPY ./index.html /usr/share/nginx/html/index.html

EXPOSE 8080
```
* Créer un fichier index.html dans le dossier tacos
* Faire un build de l'image
```bash=
$ docker build -t tacosimg .
```

### Image docker pour burger
* Créer un dossier burger et configurer le fichier Dockerfile
```bash=
$ mkdir burger
$ touch burger/Dockerfile

FROM nginx:latest

COPY ./index.html /usr/share/nginx/html/index.html

EXPOSE 8080
```
* Créer un fichier index.html dans le dossier burger
* Faire un build de l'image
```bash=
$ docker build -t burgerimg .
```

### Image docker pour pizza
* Créer un dossier pizza et configurer le fichier Dockerfile
```bash=
$ mkdir pizza
$ touch pizza/Dockerfile

FROM nginx:latest

COPY ./index.html /usr/share/nginx/html/index.html

EXPOSE 8080
```
* Créer un fichier index.html dans le dossier pizza
* Créer un fichier index.html dans le dossier burger
* Faire un build de l'image
```bash=
$ docker build -t pizzaimg .
```
### Vérifier les images
```bash=
$ docker images
REPOSITORY     TAG       IMAGE ID       CREATED              SIZE
tacosimg       latest    8cbbd1521c36   14 seconds ago       142MB
pizzaimg       latest    f8d9650f751d   30 seconds ago       142MB
burgerimg      latest    661ba0d93d43   About a minute ago   142MB
```

### Publier les images sur DockerHub
* Créer un compte sur dockerhub
* Créer un repository :
  * Nom : fastfood
  * Visibilité : Public 
* Se connecter à dockerhub en ligne de commandes
```bash=
$ docker login
```
* Créer un tag pour chaque image
```bash=
$ docker tag tacosimg:latest rakantonio/fastfood:tacos
$ docker tag pizzaimg:latest rakantonio/fastfood:pizza
$ docker tag burgerimg:latest rakantonio/fastfood:burger
```
* Pusher les images sur dockerhub
```bash=
$ docker push rakantonio/fastfood:tacos
$ docker push rakantonio/fastfood:pizza
$ docker push rakantonio/fastfood:burger
```
* Vérifier sur dockerhub que les images se sont bien ajoutées avec le bon tag
![](https://i.imgur.com/yWTus2D.png)
* Pour récupérer les images, utiliser la commande docker pull
```bash=
$ docker pull rakantonio/fastfood:burger
$ docker pull rakantonio/fastfood:pizza
$ docker pull rakantonio/fastfood:tacos
```

## Ecrire les fichiers yaml vous permettant de déployer sur votre cluster kind installé en local les composants décrits sur le schéma de la question 3 et les images crées à la question 4
* Créer et configurer les 3 fichiers de déploiements contenant les déploiements et les services de chaque déploiements
  * pizza-deployment.yml
  * burger-deployment.yml
  * tacos-deployment.yml

* Déployer les 3 pods
```bash=
$ kubectl apply -f pizza-deployment.yml
$ kubectl apply -f burger-deployment.yml
$ kubectl apply -f tacos-deployment.yml
```

 * Créer et configurer le fichier ingress pour permettre de faire un tunnel entre le monde extérieur et les services
   * ingress.yml
 
* Déployer l'ingress
```bash=
$ kubectl apply -f ingress.yml
```

* Vérifier que les deploiements, les services et l'ingress sont bien fonctionnels
```bash=
$ kubectl get deployments
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
burger-webapp   1/1     1            1           21m
pizza-webapp    1/1     1            1           21m
tacos-webapp    1/1     1            1           21m

$ kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
burger-webapp-5b5c49f6c6-gq8w8   1/1     Running   0          21m
pizza-webapp-846f44fcf7-mbf2c    1/1     Running   0          21m
tacos-webapp-5494fcdf7-jwhhl     1/1     Running   0          21m

$ kubectl get services
NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
burger-webservice   ClusterIP   10.96.74.117    <none>        80/TCP    22m
kubernetes          ClusterIP   10.96.0.1       <none>        443/TCP   4h17m
pizza-webservice    ClusterIP   10.96.4.218     <none>        80/TCP    21m
tacos-webservice    ClusterIP   10.96.200.249   <none>        80/TCP    21m

$ kubectl get ingress
NAME              CLASS    HOSTS                                            ADDRESS     PORTS   AGE
fastfood-webapp   <none>   mypizza.eatsout.com,burgerandtacos.eatsout.com   localhost   80      20m
```

* Modifier le fichier /etc/hots pour rediriger l'adresse IP de la machine vers les 2 noms de domaines

> Sur Linux
```bash=
$ vim /etc/hosts
192.168.233.132 mypizza.eatsout.com
192.168.233.132 burgerandtacos.eatsout.com

Tester ensuite avec la commande :
$ curl mypizza.eatsout.com
$ curl burgerandtacos.eatsout.com/burger
$ curl burgerandtacos.eatsout.com/tacos
```

> Sur Windows
```perl=
- Éditer le fichier C:\Windows\System32\drivers\etc\hosts en mode Administrateur.
Ajouter les 2 lignes suivantes :

192.168.233.132 mypizza.eatsout.com
192.168.233.132 burgerandtacos.eatsout.com

- mypizza.eatsout.com redirige vers un nom de domaine https
Il faut donc faire un flush du DNS sous Windows

> ipconfig /flushdns

Configuration IP de Windows
Cache de résolution DNS vidé.
```
* http://burgerandtacos.eatsout.com/tacos

![](https://i.imgur.com/686r1S0.jpg)

* http://burgerandtacos.eatsout.com/burger

![](https://i.imgur.com/hO7x70t.jpg)

* http://mypizza.eatsout.com/

![](https://i.imgur.com/vpRuk4J.jpg)

## Votre magasin de tacos devient très populaire (il va avoir 3 fois plus de commandes). Il va vous falloir gérer une charge importante sur le Service de commande des tacos. Comment gérez-vous cela ? Comment vérifier que la charge est bien répartie (avec quelle commande kubectl ?) ?

Pour gérer 3 fois plus de commandes, il faudra modifier l'option ReplicaSet du déploiement `tacos-deployment.yml`.
Il faudra donc passer de `replicas: 1` à `replicas: 3`.
Pour rappel, le ReplicaSet a pour but de maintenir un ensemble stable de Pods, permettant de garantir la disponibilité d'un certain nombre identiques de Pods.

* Modifier et relancer le déploiement du fichier tacos-deployment.yml
```bash=
$ vim tacos-deployment.yml
replicas: 3

$ kubectl apply -f tacos-deployment.yml
```
* Voir la liste des pods qui se sont créés (3 tacos-webapp liés au déploiement tacos-webapp 3/3)
```bash=
$ kubectl get deployments
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
burger-webapp   1/1     1            1           62m
pizza-webapp    1/1     1            1           62m
tacos-webapp    3/3     3            3           62m

$ kubectl get pods
NAME                             READY   STATUS    RESTARTS       AGE
burger-webapp-5b5c49f6c6-gq8w8   1/1     Running   2 (170m ago)   5d20h
pizza-webapp-c75b477d6-frm7q     1/1     Running   0              7m51s
tacos-webapp-5494fcdf7-8f77m     1/1     Running   2 (170m ago)   5d19h
tacos-webapp-5494fcdf7-jwhhl     1/1     Running   2 (170m ago)   5d20h
tacos-webapp-5494fcdf7-v97bw     1/1     Running   2 (170m ago)   5d19h


$ kubectl get rs
NAME                       DESIRED   CURRENT   READY   AGE
burger-webapp-5b5c49f6c6   1         1         1       86m
pizza-webapp-846f44fcf7    1         1         1       86m
tacos-webapp-5494fcdf7     3         3         3       86m
```

* Vérifier que la charge est bien répartie dans les logs du déploiement
Ouvrir 3 terminales en faisant un kubectl logs -f des 3 pods tacos
```bash=
$ kubectl logs -f pod/tacos-webapp-5494fcdf7-8f77m
$ kubectl logs -f pod/tacos-webapp-5494fcdf7-v97bw
$ kubectl logs -f pod/tacos-webapp-5494fcdf7-jwhhl
```
En actualisant plusieurs fois le site http://burgerandtacos.eatsout.com/tacos, on peut constater que les requêtes se répartissent entre les 3 pods. La charge est donc bien répartie entre les 3 replicas. 

![](https://i.imgur.com/c2Lw5Te.png)

## Créer une nouvelle version de votre carte des pizzas et publiez-la dans une nouvelle version de votre image. Appliquer la modification à votre déploiement. Qu’observez vous sur la disponibilité du service qui présente la carte des pizzas pendant la mise à jour ? 

Pour mieux visualiser celà vous pouvez en parallèle de la mise à jour exécuter les commandes suivantes dans d’autres terminaux :
> watch -n 1 -c kubectl get pods  
> watch -n 1 -c curl mypizza.eatsout.com

* Modifier le fichier de déploiement en ajoutant l'option "imagePullPolicy". Cette option est nécessaire pour que le déploiement récupère la bonne image.
```bash=
$ vim pizza-deployment.yaml
spec:
      containers:
      - name: pizza-webapp
        image: rakantonio/fastfood:pizza
        imagePullPolicy: Always
```
* Relancer le déploiement
```bash=
$ kubectl apply -f pizza-deployment.yml
```
* Modifier le fichier index.html dans le dossier pizza
* Faire un build du Dockerfile dans le dossier pizza une fois le fichier index modifié
```bash=
$ docker build -t pizzaimg .
```
* Retagger la nouvelle image avec le même tag 'pizza' pour pouvoir le push sur le même repository
```bash=
$ docker tag pizzaimg:latest rakantonio/fastfood:pizza
```
* Faire un push de l'image sur le repository fastfood. L'image sera alors mise à jour avec les nouvelles modifications
```bash=
$ docker push rakantonio/fastfood:pizza 
```
* Faire un pull de la nouvelle image
```bash=
$ docker pull rakantonio/fastfood:pizza 
```
* Lancer les commandes `watch -n 1 -c kubectl get pods` et `watch -n 1 -c curl mypizza.eatsout.com`
```bash=
$ watch -n 1 -c kubectl get pods
```
![](https://i.imgur.com/oJfsevj.png)
```bash=
$ watch -n 1 -c curl mypizza.eatsout.com
```
![](https://i.imgur.com/gIzKVv1.png)

* Relancer le déploiement pour prendre en compte l'update de l'image
```bash=
$ kubectl rollout restart deployment/pizza-webapp
```
* Vérifier les changements avec les commandes `watch -n 1 -c kubectl get pods` et `watch -n 1 -c curl mypizza.eatsout.com`
```bash=
$ watch -n 1 -c kubectl get pods
```
![](https://i.imgur.com/y0sz25l.png)
> La commande **rollout restart** va donc lancer un nouveau déploiement. Une fois que le déploiement est en mode **running**, l'ancien déploiement se met en état **Terminated**.

```bash=
$ watch -n 1 -c curl mypizza.eatsout.com
```
> Le site est donc presque toujours disponible car le nouveau déploiement est déjà en état running avant que l'ancien se termine. Une fois que le déploiement s'est créé, la commande affiche les nouvelles modifications du index.html (du site web).

![](https://i.imgur.com/wQjmko7.png)

http://mypizza.eatsout.com/

![](https://i.imgur.com/v4krzPw.png)



