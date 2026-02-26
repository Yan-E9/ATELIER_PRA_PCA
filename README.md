------------------------------------------------------------------------------------------------------
ATELIER PRA/PCA
------------------------------------------------------------------------------------------------------
L’idée en 30 secondes : Cet atelier met en œuvre un **mini-PRA** sur **Kubernetes** en déployant une **application Flask** avec une **base SQLite** stockée sur un **volume persistant (PVC pra-data)** et des **sauvegardes automatiques réalisées chaque minute vers un second volume (PVC pra-backup)** via un **CronJob**. L’**image applicative est construite avec Packer** et le **déploiement orchestré avec Ansible**, tandis que Kubernetes assure la gestion des pods et de la disponibilité applicative. Nous observerons la différence entre **disponibilité** (recréation automatique des pods sans perte de données) et **reprise après sinistre** (perte volontaire du volume de données puis restauration depuis les backups), nous mesurerons concrètement les RTO et RPO, et comprendrons les limites d’un PRA local non répliqué. Cet atelier illustre de manière pratique les principes de continuité et de reprise d’activité, ainsi que le rôle respectif des conteneurs, du stockage persistant et des mécanismes de sauvegarde.
  
**Architecture cible :** Ci-dessous, voici l'architecture cible souhaitée.   
  
![Screenshot Actions](Architecture_cible.png)  
  
-------------------------------------------------------------------------------------------------------
Séquence 1 : Codespace de Github
-------------------------------------------------------------------------------------------------------
Objectif : Création d'un Codespace Github  
Difficulté : Très facile (~5 minutes)
-------------------------------------------------------------------------------------------------------
**Faites un Fork de ce projet**. Si besoin, voici une vidéo d'accompagnement pour vous aider à "Forker" un Repository Github : [Forker ce projet](https://youtu.be/p33-7XQ29zQ) 
  
Ensuite depuis l'onglet **[CODE]** de votre nouveau Repository, **ouvrez un Codespace Github**.
  
---------------------------------------------------
Séquence 2 : Création du votre environnement de travail
---------------------------------------------------
Objectif : Créer votre environnement de travail  
Difficulté : Simple (~10 minutes)
---------------------------------------------------
Vous allez dans cette séquence mettre en place un cluster Kubernetes K3d contenant un master et 2 workers, installer les logiciels Packer et Ansible. Depuis le terminal de votre Codespace copier/coller les codes ci-dessous étape par étape :  

**Création du cluster K3d**  
```
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```
```
k3d cluster create pra \
  --servers 1 \
  --agents 2
```
**vérification de la création de votre cluster Kubernetes**  
```
kubectl get nodes
```
**Installation du logiciel Packer (création d'images Docker)**  
```
PACKER_VERSION=1.11.2
curl -fsSL -o /tmp/packer.zip \
  "https://releases.hashicorp.com/packer/${PACKER_VERSION}/packer_${PACKER_VERSION}_linux_amd64.zip"
sudo unzip -o /tmp/packer.zip -d /usr/local/bin
rm -f /tmp/packer.zip
```
**Installation du logiciel Ansible**  
```
python3 -m pip install --user ansible kubernetes PyYAML jinja2
export PATH="$HOME/.local/bin:$PATH"
ansible-galaxy collection install kubernetes.core
```
  
---------------------------------------------------
Séquence 3 : Déploiement de l'infrastructure
---------------------------------------------------
Objectif : Déployer l'infrastructure sur le cluster Kubernetes
Difficulté : Facile (~15 minutes)
---------------------------------------------------  
Nous allons à présent déployer notre infrastructure sur Kubernetes. C'est à dire, créér l'image Docker de notre application Flask avec Packer, déposer l'image dans le cluster Kubernetes et enfin déployer l'infratructure avec Ansible (Création du pod, création des PVC et les scripts des sauvegardes aututomatiques).  

**Création de l'image Docker avec Packer**  
```
packer init .
packer build -var "image_tag=1.0" .
docker images | head
```
  
**Import de l'image Docker dans le cluster Kubernetes**  
```
k3d image import pra/flask-sqlite:1.0 -c pra
```
  
**Déploiment de l'infrastructure dans Kubernetes**  
```
ansible-playbook ansible/playbook.yml
```
  
**Forward du port 8080 qui est le port d'exposition de votre application Flask**  
```
kubectl -n pra port-forward svc/flask 8080:80 >/tmp/web.log 2>&1 &
```
  
---------------------------------------------------  
**Réccupération de l'URL de votre application Flask**. Votre application Flask est déployée sur le cluster K3d. Pour obtenir votre URL cliquez sur l'onglet **[PORTS]** dans votre Codespace (à coté de Terminal) et rendez public votre port 8080 (Visibilité du port). Ouvrez l'URL dans votre navigateur et c'est terminé.  

**Les routes** à votre disposition sont les suivantes :  
1. https://...**/** affichera dans votre navigateur "Bonjour tout le monde !".
2. https://...**/health** pour voir l'état de santé de votre application.
3. https://...**/add?message=test** pour ajouter un message dans votre base de données SQLite.
4. https://...**/count** pour afficher le nombre de messages stockés dans votre base de données SQLite.
5. https://...**/consultation** pour afficher les messages stockés dans votre base de données.
  
---------------------------------------------------  
### Processus de sauvegarde de la BDD SQLite

Grâce à une tâche CRON déployée par Ansible sur le cluster Kubernetes (un CronJob), toutes les minutes une sauvegarde de la BDD SQLite est faite depuis le PVC pra-data vers le PCV pra-backup dans Kubernetes.  

Pour visualiser les sauvegardes périodiques déposées dans le PVC pra-backup, coller les commandes suivantes dans votre terminal Codespace :  

```
kubectl -n pra run debug-backup \
  --rm -it \
  --image=alpine \
  --overrides='
{
  "spec": {
    "containers": [{
      "name": "debug",
      "image": "alpine",
      "command": ["sh"],
      "stdin": true,
      "tty": true,
      "volumeMounts": [{
        "name": "backup",
        "mountPath": "/backup"
      }]
    }],
    "volumes": [{
      "name": "backup",
      "persistentVolumeClaim": {
        "claimName": "pra-backup"
      }
    }]
  }
}'
```
```
ls -lh /backup
```
**Pour sortir du cluster et revenir dans le terminal**
```
exit
```

---------------------------------------------------
Séquence 4 : 💥 Scénarios de crash possibles  
Difficulté : Facile (~30 minutes)
---------------------------------------------------
### 🎬 **Scénario 1 : PCA — Crash du pod**  
Nous allons dans ce scénario **détruire notre Pod Kubernetes**. Ceci simulera par exemple la supression d'un pod accidentellement, ou un pod qui crash, ou un pod redémarré, etc..

**Destruction du pod :** Ci-dessous, la cible de notre scénario   
  
![Screenshot Actions](scenario1.png)  

Nous perdons donc ici notre application mais pas notre base de données puisque celle-ci est déposée dans le PVC pra-data hors du pod.  

Copier/coller le code suivant dans votre terminal Codespace pour détruire votre pod :
```
kubectl -n pra get pods
```
Notez le nom de votre pod qui est différent pour tout le monde.  
Supprimez votre pod (pensez à remplacer <nom-du-pod-flask> par le nom de votre pod).  
Exemple : kubectl -n pra delete pod flask-7c4fd76955-abcde  
```
kubectl -n pra delete pod <nom-du-pod-flask>
```
**Vérification de la suppression de votre pod**
```
kubectl -n pra get pods
```
👉 **Le pod a été reconstruit sous un autre identifiant**.  
Forward du port 8080 du nouveau service  
```
kubectl -n pra port-forward svc/flask 8080:80 >/tmp/web.log 2>&1 &
```
Observez le résultat en ligne  
https://...**/consultation** -> Vous n'avez perdu aucun message.
  
👉 Kubernetes gère tout seul : Aucun impact sur les données ou sur votre service (PVC conserve la DB et le pod est reconstruit automatiquement) -> **C'est du PCA**. Tout est automatique et il n'y a aucune rupture de service.
  
---------------------------------------------------
### 🎬 **Scénario 2 : PRA - Perte du PVC pra-data** 
Nous allons dans ce scénario **détruire notre PVC pra-data**. C'est à dire nous allons suprimer la base de données en production. Ceci simulera par exemple la corruption de la BDD SQLite, le disque du node perdu, une erreur humaine, etc. 💥 Impact : IL s'agit ici d'un impact important puisque **la BDD est perdue**.  

**Destruction du PVC pra-data :** Ci-dessous, la cible de notre scénario   
  
![Screenshot Actions](scenario2.png)  

🔥 **PHASE 1 — Simuler le sinistre (perte de la BDD de production)**  
Copier/coller le code suivant dans votre terminal Codespace pour détruire votre base de données :
```
kubectl -n pra scale deployment flask --replicas=0
```
```
kubectl -n pra patch cronjob sqlite-backup -p '{"spec":{"suspend":true}}'
```
```
kubectl -n pra delete job --all
```
```
kubectl -n pra delete pvc pra-data
```
👉 Vous pouvez vérifier votre application en ligne, la base de données est détruite et la service n'est plus accéssible.  

✅ **PHASE 2 — Procédure de restauration**  
Recréer l’infrastructure avec un PVC pra-data vide.  
```
kubectl apply -f k8s/
```
Vérification de votre application en ligne.  
Forward du port 8080 du service pour tester l'application en ligne.  
```
kubectl -n pra port-forward svc/flask 8080:80 >/tmp/web.log 2>&1 &
```
https://...**/count** -> =0.  
https://...**/consultation** Vous avez perdu tous vos messages.  

Retaurez votre BDD depuis le PVC Backup.  
```
kubectl apply -f pra/50-job-restore.yaml
```
👉 Vous pouvez vérifier votre application en ligne, **votre base de données a été restaureé** et tous vos messages sont bien présents.  

Relance des CRON de sauvgardes.  
```
kubectl -n pra patch cronjob sqlite-backup -p '{"spec":{"suspend":false}}'
```
👉 Nous n'avons pas perdu de données mais Kubernetes ne gère pas la restauration tout seul. Nous avons du protéger nos données via des sauvegardes régulières (du PVC pra-data vers le PVC pra-backup). -> **C'est du PRA**. Il s'agit d'une stratégie de sauvegarde avec une procédure de restauration.  

---------------------------------------------------
Séquence 5 : Exercices  
Difficulté : Moyenne (~45 minutes)
---------------------------------------------------
**Complétez et documentez ce fichier README.md** pour répondre aux questions des exercices.  
Faites preuve de pédagogie et soyez clair dans vos explications et procedures de travail.  

**Exercice 1 :**  
Quels sont les composants dont la perte entraîne une perte de données ?  
  
*Les composants dont la perte entraîne une perte de données sont le PVC pra-data, qui contient la base SQLite de production, le PVC pra-backup, qui stocke l’historique des sauvegardes, et le stockage physique sous-jacent qui supporte ces volumes. Si pra-data est perdu sans sauvegarde, les données applicatives sont perdues ; si pra-backup est perdu, on ne peut plus restaurer après un sinistre ; et si le stockage commun aux deux PVC est impacté, on perd à la fois la production et les backups.*

**Exercice 2 :**  
Expliquez nous pourquoi nous n'avons pas perdu les données lors de la supression du PVC pra-data  
  
*Nous n’avons pas perdu les données lors de la suppression du PVC pra-data parce qu’elles étaient régulièrement copiées sur un autre volume et que nous avons appliqué une procédure de restauration. Le CronJob sauvegarde la base de production de pra-data vers pra-backup toutes les minutes, ce qui crée des fichiers de backup indépendants. Après la suppression, nous recréons un PVC pra-data vide puis lançons un Job qui recopie le dernier backup depuis pra-backup vers ce nouveau volume, ce qui reconstruit la base. La conservation des données vient donc du mécanisme de sauvegarde et de restauration.*

**Exercice 3 :**  
Quels sont les RTO et RPO de cette solution ?  
  
*Dans cette solution, le RTO est le temps nécessaire pour remettre l’application en service après un sinistre, c’est-à-dire le temps de recréer l’infrastructure, d’exécuter le Job de restauration et de rendre l’URL à nouveau accessible ; dans ce lab, cela se mesure en quelques minutes. Le RPO correspond à l’ancienneté maximale des données que l’on accepte de perdre : comme les sauvegardes sont effectuées toutes les minutes par le CronJob, on peut perdre au maximum les écritures réalisées depuis la dernière sauvegarde, ce qui donne un RPO d’environ une minute.*

**Exercice 4 :**  
Pourquoi cette solution (cet atelier) ne peux pas être utilisé dans un vrai environnement de production ? Que manque-t-il ?   
  
*Cette solution n’est pas directement utilisable en production car elle repose sur un seul cluster local et sur des volumes non répliqués, ce qui expose à la perte simultanée de la production et des sauvegardes en cas de sinistre sur le stockage. Il n’existe ni réplication géographique ni site de secours, et la procédure de PRA est essentiellement manuelle, sans orchestration ni tests réguliers. De plus, des aspects essentiels comme la sécurité, la supervision, l’alerting et la gestion des SLA ne sont pas pris en compte, ce qui limite fortement la fiabilité et la robustesse de l’architecture pour un contexte réel.*
  
**Exercice 5 :**  
Proposez une archtecture plus robuste.   
  
*Une architecture plus robuste consisterait à déployer l’application sur un cluster Kubernetes managé utilisant un stockage persistant managé, redondé et pouvant faire des snapshots, tout en externalisant les sauvegardes vers un stockage objet répliqué (par exemple dans une autre région). On ajouterait un second cluster ou un site de secours capable de restaurer l’application à partir de ces sauvegardes, avec un runbook de PRA largement automatisé pour réduire le RTO. Enfin, on intégrerait du monitoring, de l’alerting et des tests réguliers de restauration afin de vérifier en continu que les objectifs de RTO et de RPO sont respectés dans des conditions proches de la production.*

---------------------------------------------------
Séquence 6 : Ateliers  
Difficulté : Moyenne (~2 heures)
---------------------------------------------------
### **Atelier 1 : Ajoutez une fonctionnalité à votre application**  
**Ajouter une route GET /status** dans votre application qui affiche en JSON :
* count : nombre d’événements en base
* last_backup_file : nom du dernier backup présent dans /backup
* backup_age_seconds : âge du dernier backup

*..**Déposez ici une copie d'écran**!
!![alt text](image-2.png)*

---------------------------------------------------
### **Atelier 2 : Choisir notre point de restauration**  
Aujourd’hui nous restaurobs “le dernier backup”. Nous souhaitons **ajouter la capacité de choisir un point de restauration**.

*..Décrir ici votre procédure de restauration (votre runbook)..

Procédure de restauration (Runbook) – Atelier 2

Prérequis : fichier pra/55-job-restore-specific.yaml créé avec le contenu suivant :

text
apiVersion: batch/v1
kind: Job
metadata:
  name: sqlite-restore-specific
  namespace: pra
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: restore
        image: pra/flask-sqlite:1.2  # Remplace par ta dernière image
        command: ["/bin/sh"]
        args:
        - -c
        - |
          echo "🔄 Restauration depuis \$BACKUP_FILE"
          cp /backup/\$BACKUP_FILE /data/app.db
          echo "✅ Restauration terminée"
        env:
        - name: BACKUP_FILE
          value: "CHANGE-MOI"  # ← Nom du backup à restaurer
        volumeMounts:
        - name: data
          mountPath: /data
        - name: backup
          mountPath: /backup
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: pra-data
      - name: backup
        persistentVolumeClaim:
          claimName: pra-backup
Étapes de la procédure :

Lister les backups disponibles :

bash
kubectl -n pra run tmp-list --rm -it --image=alpine \
  --overrides='{"spec":{"containers":[{"name":"list","image":"alpine","command":["ls","-la"],"volumeMounts":[{"name":"backup","mountPath":"/backup"}],"volumes":[{"name":"backup","persistentVolumeClaim":{"claimName":"pra-backup"}}]}]}}' /backup
Exemple de sortie : app-1772098381.db, app-1772098441.db, etc.

Choisir un point de restauration (ex. app-1772098381.db)

Préparer le Job :

bash
# Supprimer l'ancien Job s'il existe
kubectl -n pra delete job sqlite-restore-specific --ignore-not-found

# Éditer le YAML pour mettre le bon nom de fichier
vim pra/55-job-restore-specific.yaml  # Ligne 17 : value: "app-1772098381.db"
Lancer la restauration :

bash
kubectl apply -f pra/55-job-restore-specific.yaml
Suivre l’exécution :

bash
kubectl -n pra get jobs
kubectl -n pra logs job/sqlite-restore-specific
Sortie attendue :

text
🔄 Restauration depuis app-1772098381.db
✅ Restauration terminée
Vérifier la restauration :

bash
kubectl -n pra port-forward svc/flask 8080:80 >/tmp/web.log 2>&1 &
text
https://.../count     # Nombre d'événements au moment du backup choisi
https://.../status   # Confirme le bon backup restauré
https://.../consultation  # Données correspondant au point choisi*  
  
---------------------------------------------------
Evaluation
---------------------------------------------------
Cet atelier PRA PCA, **noté sur 20 points**, est évalué sur la base du barème suivant :  
- Série d'exerices (5 points)
- Atelier N°1 - Ajout d'un fonctionnalité (4 points)
- Atelier N°2 - Choisir son point de restauration (4 points)
- Qualité du Readme (lisibilité, erreur, ...) (3 points)
- Processus travail (quantité de commits, cohérence globale, interventions externes, ...) (4 points) 

