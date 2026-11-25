# Alternative à WDS : Clonezilla Server Edition (DRBL) sur Proxmox

TP3 – Déploiement des environnements de travail. Capture et déploiement d'une image Windows avec Clonezilla SE.

## Topologie du labo

![Topologie](images/topologie-clonezilla.svg)

- **clonezilla-srv** : serveur Ubuntu 24.04 (DRBL + Clonezilla). `ens18` = WAN `192.168.1.213`, `ens19` = LAN isolé `192.168.100.1`
- **win-modele** : poste modèle, sa capture devient l'image
- **win-test** : poste vide, reçoit l'image

## 1. Installation

**Machine : serveur Clonezilla** : ajouter le dépôt DRBL, puis installer le paquet.

```bash
sudo wget -O /etc/apt/trusted.gpg.d/drbl.asc https://drbl.org/GPG-KEY-DRBL
echo "deb http://free.nchc.org.tw/drbl-core drbl stable" | sudo tee /etc/apt/sources.list.d/drbl.list
sudo apt-get update
sudo apt-get install drbl
```

## 2. Configuration

**Machine : serveur Clonezilla** : configurer les deux cartes réseau (netplan), puis appliquer.

```bash
sudo netplan apply
```

`ens18` = `192.168.1.213/24` (WAN), `ens19` = `192.168.100.1/24` (LAN isolé, sans passerelle).

Préparer le serveur (installe Clonezilla et le noyau des clients) :

```bash
sudo drblsrv -i
```

Configurer le réseau et les services (carte clients `ens19`, plage DHCP `192.168.100.10` à `.14`, mode Clonezilla live) :

```bash
sudo drblpush -i
```

Vérifier que DHCP, TFTP et NFS tournent :

```bash
sudo systemctl status isc-dhcp-server tftpd-hpa nfs-kernel-server --no-pager
```

Captures de l'installation et de la configuration :

![Capture 1](images/01.png)

![Capture 2](images/02.png)

![Capture 3](images/03.png)

![Capture 4](images/04.png)

![Capture 5](images/05.png)

![Capture 6](images/06.png)

## 3. Capture de l'image

**Machine : serveur Clonezilla** : lancer le mode capture.

```bash
sudo dcs
```

Choix : `save-disk` → `Later_in_client` → compression `-z1p` → pas de découpage (`0`).

![Capture 7](images/07.png)

![Capture 8](images/08.png)

![Capture 9](images/09.png)

![Capture 10](images/10.png)

**Machine : win-modele** : démarrer en PXE, Clonezilla se charge. Nommer l'image (`winserver-modele`), choisir le disque, la capture démarre. Arrêter la VM à la fin.

![Capture 11](images/11.png)

![Capture 12](images/12.png)

![Capture 13](images/13.png)

![Capture 14](images/14.png)

![Capture 15](images/15.png)

## 4. Déploiement de l'image

**Machine : serveur Clonezilla** : relancer `sudo dcs`, puis `restore-disk` et la table de partitions de l'image (`-k0`).

![Capture 16](images/16.png)

![Capture 17](images/17.png)

Mode de transfert : **unicast** (un seul client à restaurer).

![Capture 18](images/18.png)

**Machine : win-test** : démarrer en PXE. L'image `winserver-modele` est restaurée sur le disque, puis la VM démarre sur le Windows cloné. Après la restauration, remettre `sata0` en premier dans l'ordre de démarrage.

![Capture 19](images/19.png)

![Capture 20](images/20.png)

![Capture 21](images/21.png)

![Capture 22](images/22.png)

![Capture 23](images/23.png)
## 5. Limites et conflits à éviter

### Limites de la solution
- **Pas de personnalisation après le déploiement.** Contrairement à WDS avec MDT ou un fichier de réponses, Clonezilla copie le disque bloc par bloc. Il ne nomme pas le poste, ne le joint pas au domaine et n'installe pas de pilotes.
- **Matériel identique requis.** Ça marche ici parce que win-modele et win-test ont le même matériel virtuel (i440fx, E1000, SATA). Sur du matériel différent, Windows peut planter au démarrage (BSOD) sans sysprep.
- **Le disque cible doit être au moins aussi grand que le disque source.**
- **PXE en BIOS legacy.** L'UEFI n'a pas été testé.
- **Un client à la fois.** Mode unicast utilisé. Le multicast permettrait plusieurs clients en parallèle, mais n'a pas été testé.
- **Le mode choisi dans `dcs` s'applique à tout client PXE du réseau isolé.** Ne pas démarrer win-modele pendant que le mode restauration est actif. Après la restauration, remettre `sata0` en premier dans l'ordre de démarrage de win-test et arrêter le mode actif sur le serveur, sinon la VM boucle sur le PXE (`-p reboot`).

### Conflits possibles sans sysprep

| Risque | Explication | Solution |
|---|---|---|
| Nom de l'ordinateur | Le clone porte le même nom que le modèle. Deux clones sur le même réseau entrent en conflit. | Renommer après le déploiement : `Rename-Computer -NewName WIN-TEST -Restart` |
| SID | Les clones partagent le même identifiant de sécurité. Sans conséquence en groupe de travail, problématique pour un domaine ou WSUS. | Ne pas joindre ces postes au domaine, ou faire un sysprep. |
| Adresse MAC | Pas de conflit : la MAC appartient à la carte réseau et n'est pas copiée avec le disque. Proxmox en génère une par VM. | Rien à faire. |
| Adresse IP | Une IP fixe sur le modèle serait héritée par tous les clones. | Laisser le modèle en DHCP, le serveur donne une IP différente à chaque poste. |
| Activation / licence | Le clone garde l'état d'activation du modèle. | Sans importance en labo. |

### Sysprep (non fait dans ce TP)
En production, on prépare le modèle avant la capture, sur win-modele :

```powershell
C:\Windows\System32\Sysprep\sysprep.exe /generalize /oobe /shutdown
```

Cette commande réinitialise le SID et le nom de l'ordinateur et relance la configuration initiale de Windows au premier démarrage. Elle permet aussi de déployer sur du matériel différent. Sur un Windows allégé (Tiny10), elle peut échouer parce que certains composants sont retirés. Dans ce TP, on l'a sauté et on renomme le poste après le déploiement.
