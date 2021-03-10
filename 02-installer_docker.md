# Installation OS sur VPS pour conteneuriser Actions-Bidonvilles

## Infrastructure

* Un VPS a été commandé chez OVH.

* Il s'agit d'un VPS équivalent à celui de PROD pour éviter toute surprise au passage en PROD.

* Il faut noter que l'offre est évolutive et permet de doubler la RAM du serveur et du disque.

* Les sous-domaines ont été créés

## Installer l'OS

* L'OS (système d'exploitation) est installé par l'hébergeur: demander l’installation du VPS sous Debian 10.

## Installer Docker:

* Ouvrir une session sur le VPS avec le compte debian

* Supprimer toutes traces d'une éventuelle ancienne installation de Docker :

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
```

* Mettre à jour la liste des paquets:

```bash
sudo apt update
```

* Installer les paquets requis:

```bash
sudo apt install apt-transport-https ca-certificates curl gnupg lsb-release
```

* Ajouter la clé GPG pour l'accès au dépôt officiel Docker:

```bash
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

* Ajouter le dépôt Docker aux sources APT:

```bash
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

* Actualiser la base de données des paquets disponibles

```bash
sudo apt update
```

* Vérifier qu'on s'apprête bien à installer Docker depuis le dépôt Docker, et non depuis le dépôt Debian:

```bash
apt-cache policy docker-ce
```

* Vérfier que la version "candidate" est bien celle indiquée ci-dessous (/!\ à la date de rédaction de cet article):

```
docker-ce:
  Installed: 5:20.10.5~3-0~debian-buster
  Candidate: 5:20.10.5~3-0~debian-buster
  Version table:
 *** 5:20.10.5~3-0~debian-buster 500
        500 https://download.docker.com/linux/debian buster/stable amd64 Packages
  ...
```

* Lancer l'installation de Docker:

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

* Vérifier que Docker est installé et que le démon est bien lancé:

```bash
sudo systemctl status docker
```

* Résultat:

```
● docker.service - Docker Application Container Engine
   Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2021-03-10 13:28:15 UTC; 8min ago
     Docs: https://docs.docker.com
 Main PID: 3833 (dockerd)
    Tasks: 10
   Memory: 55.4M
   CGroup: /system.slice/docker.service
           └─3833 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```

* A ce stade, Docker est installé. Il faut maintenant sécuriser le serveur pour empêcher toute tentative de connexion non souhaitée.

## Sécuriser le système

### Ajouter un compte utilisateur

* Il s'agit de créer un compte utilisateur disposant des droits d'administration sur le serveur. Ceci permettra de désactiver le compte **root** et donc d'empêcher toute tentative de connexion avec ce compte en essayant des combinaisons différentes de mots de passes.

```bash
sudo adduser rbadmin
```

* Saisir un mot de passe et le confirmer:

```
Adding user `rbadmin' ...
Adding new group `rbadmin' (1001) ...
Adding new user `rbadmin' (1001) with group `rbadmin' ...
Creating home directory `/home/rbadmin' ...
Copying files from `/etc/skel' ...
New password: ****
Retype new password: ****
passwd: password updated successfully
```

* Après mise à jour du mot de passe, complêter éventuellement les informations relatives au compte en cours de création.

* Sur l'installation **Debian 10** fournie par OVH, il n'est pas nécessaire d'installer **sudo**: OVH l'a déjà fait pour nous. Mais si ce n'est pas le cas, il convient d'installer **sudo** de cette façon:

```bash
apt-get install sudo
```

* Il reste à ajouter l'utilisateur **rbadmin** au groupe **sudo** pour lui permettre de disposer des droit d'administration:

```bash
sudo usermod -aG root,sudo,adm rbadmin
```

* Vérifier que l'utilisateur est bien intégrés dans les groupes souhaités:

```bash
cat /etc/group | grep rbadmin
```

* Résultat:

```
root:x:0:rbadmin
adm:x:4:debian,rbadmin
sudo:x:27:debian,rbadmin
rbadmin:x:1001:
```

* Tout est conforme: rbadmin figure bien dans les listes d'utilisateurs des groupes **adm**, **sudo** et **root**.

### Interdire la connexion en tant que root

* Modifier la configuration du démon **ssh** pour qu'il n'accepte pas les connexions de l'utilisateur **root**:

```bash
sudo vi /etc/ssh/sshd_config
```

* Chercher la ligne contenant la chaîne de caractère "PermitRootLogin**:

```
#PermitRootLogin prohibit-password
# the setting of "PermitRootLogin without-password".
```

* On voit que la ligne contenant **PermitRootLogin** est commentée. Pour s'assurer que la connexion avec le compte **root** ne soit pas permise, on va décommentater la ligne et passer la valeur à **no**:

```
#PermitRootLogin prohibit-password
PermitRootLogin no
# the setting of "PermitRootLogin without-password".
```

* La ligne initiale commentée a été laissée à titre d'information.

* Il reste maintenant à appliquer cette modification en redémarrant le service **sshd**:

```bash
sudo service ssh restart
```

* On peut maintenant fermer la session en cours et se reconnecter avec le compte **rbadmin** et tester la commande **sudo**.

```bash
whoami
rbadmin
sudo ls -ial /
```

* Résultat:

```
We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for rbadmin: 
```

* Saisir à nouveau le mot de passe de l'utilisateur rbadmin.

* Résultat:

```
total 68
    2 drwxr-xr-x 18 root root  4096 Nov 19 09:22 .
    2 drwxr-xr-x 18 root root  4096 Nov 19 09:22 ..
  314 lrwxrwxrwx  1 root root     7 Sep 26 17:35 bin -> usr/bin
 8195 drwxr-xr-x  4 root root  4096 Sep 26 17:39 boot
    3 drwxr-xr-x 15 root root  2860 Nov 19 09:22 dev
16385 drwxr-xr-x 79 root root  4096 Nov 19 10:34 etc
 8197 drwxr-xr-x  4 root root  4096 Nov 19 10:28 home
31539 lrwxrwxrwx  1 root root    37 Sep 26 17:38 initrd.img -> boot/initrd.img-4.19.0-11-cloud-amd64
31537 lrwxrwxrwx  1 root root    37 Sep 26 17:38 initrd.img.old -> boot/initrd.img-4.19.0-11-cloud-amd64
  318 lrwxrwxrwx  1 root root     7 Sep 26 17:35 lib -> usr/lib
  320 lrwxrwxrwx  1 root root     9 Sep 26 17:35 lib32 -> usr/lib32
  322 lrwxrwxrwx  1 root root     9 Sep 26 17:35 lib64 -> usr/lib64
  324 lrwxrwxrwx  1 root root    10 Sep 26 17:35 libx32 -> usr/libx32
   11 drwx------  2 root root 16384 Sep 26 17:34 lost+found
 8206 drwxr-xr-x  2 root root  4096 Sep 26 17:35 media
 8203 drwxr-xr-x  2 root root  4096 Sep 26 17:35 mnt
 8205 drwxr-xr-x  3 root root  4096 Nov 19 09:41 opt
    1 dr-xr-xr-x 78 root root     0 Nov 19 09:22 proc
 8199 drwx------  3 root root  4096 Nov 19 10:58 root
 7385 drwxr-xr-x 23 root root   760 Nov 19 11:01 run
  316 lrwxrwxrwx  1 root root     8 Sep 26 17:35 sbin -> usr/sbin
 8204 drwxr-xr-x  2 root root  4096 Sep 26 17:35 srv
    1 dr-xr-xr-x 13 root root     0 Nov 19 09:22 sys
 8202 drwxrwxrwt  8 root root  4096 Nov 19 09:41 tmp
 8194 drwxr-xr-x 14 root root  4096 Nov 19 09:41 usr
 8193 drwxr-xr-x 11 root root  4096 Sep 26 17:35 var
31538 lrwxrwxrwx  1 root root    34 Sep 26 17:38 vmlinuz -> boot/vmlinuz-4.19.0-11-cloud-amd64
31536 lrwxrwxrwx  1 root root    34 Sep 26 17:38 vmlinuz.old -> boot/vmlinuz-4.19.0-11-cloud-amd64

```

* **sudo** fonctionne !

## Permettre l'exécution de la commande docker sans sudo

* Par défaut, la commande **Docker** n'est utilisable que par l'utilisateur **root** ou un utilisateur du groupe **docker** créé automatiquement pendant la phase d'installation de Docker. On va donc ajouter l'utilisateur rbadmin au groupe **docker** pour lui permettre de lancer les commandes docker sans utiliser **sudo**.

```bash
sudo usermod -aG docker rbadmin
```

* On vérifie:

```bash
sudo cat /etc/group | grep rbadmin
root:x:0:rbadmin
adm:x:4:debian,rbadmin
sudo:x:27:debian,rbadmin
docker:x:998:rbadmin
rbadmin:x:1001:
```

* **rbadmin** est bien dans le groupe **docker**.

* Pour que cette modification soit appliquée, il faut normalement fermer la session pouis la réouvrir ou simplement lancer la commande suivante:

```bash
su - rbadmin
```

* On vérifie:

```bash
id -nG
```

* Résultat:

```
rbadmin root adm sudo docker
```

* On vérifie que l'utilisateur **rbadmin** peut lancer la commande **docker**:

```bash
docker version
```

* Résultat attendu:

```
Client: Docker Engine - Community
 Version:           20.10.5
 API version:       1.41
 Go version:        go1.13.15
 Git commit:        55c4c88
 Built:             Tue Mar  2 20:17:50 2021
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true

Server: Docker Engine - Community
 Engine:
  Version:          20.10.5
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.13.15
  Git commit:       363e9a8
  Built:            Tue Mar  2 20:15:47 2021
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.4.4
  GitCommit:        05f951a3781f4f2c1911b05e61c160e9c30eaa8e
 runc:
  Version:          1.0.0-rc93
  GitCommit:        12644e614e25b05da6fd08a38ffa0cfe1903fdec
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```

## Installer docker-compose

* Vérifier la version la plus à jour sur la [page github de docker-compose](https://github.com/docker/compose/releases)

* Récupérer et installer l'exécutable en lançant la commande suivante:

```bash
sudo curl -L https://github.com/docker/compose/releases/download/1.28.5/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
[sudo] password for rbadmin: 
```

* Saisir le mot de passe de l'utilisateur **rbadmin**. Résultat:

```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   651  100   651    0     0  11625      0 --:--:-- --:--:-- --:--:-- 11625
100 11.6M  100 11.6M    0     0  6443k      0  0:00:01  0:00:01 --:--:-- 8240k
```

* Vérifier la version:

```bash
docker-compose --version
docker-compose version 1.28.5, build 40524192
```

* Si une erreur du typesuivant s'affiche:

```bash
-bash: /usr/local/bin/docker-compose: Permission denied
```

* Exécuter la commande suivante  pour rendre la commande **docker-compose** exécutable par **root** et les membres du gorupe **root*:

```bash
sudo chmod ug+x /usr/local/bin/docker-compose 
```

* **docker-compose** est installé.
