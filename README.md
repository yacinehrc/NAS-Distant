# NAS Distant — CasaOS, Wake-on-LAN & Tailscale

<img width="1235" height="691" alt="NAS Distant" src="https://github.com/user-attachments/assets/252b6313-dc7c-42c2-bbf1-7b634af0235c" />

> Un Acer Gateway DT55 sous Debian 13 (sans interface graphique), transformé en NAS accessible depuis n'importe où grâce à Tailscale, tout en restant éteint la majeure partie du temps (pour réduire la consommation électrique de ce PC) : il est réveillé à la demande via Wake-on-LAN, déclenché à distance depuis un HP EliteDesk sous Proxmox connecté au même tailnet.

---

## Sommaire

- [Architecture](#architecture)
- [Installation de CasaOS sur l'Acer](#installation-de-casaos-sur-lacer)
- [Wake-on-LAN — réveil à distance](#wake-on-lan--réveil-à-distance)
- [Tailscale — VPN mesh entre les machines](#tailscale--vpn-mesh-entre-les-machines)
- [Conclusion](#conclusion)

---

## Architecture

Le projet repose sur trois machines aux rôles complémentaires.

| Machine                      | OS                          | Rôle                                                                                         |
| ----------------------------- | --------------------------- | ------------------------------------------------------------------------------------------- |
| **Acer Gateway DT55**        | Debian 13 (sans GUI)        | NAS distant — CasaOS + Tailscale + Wake-on-LAN, éteint par défaut, réveillé à la demande     |
| **HP EliteDesk 800 G3 Mini** | Proxmox VE (Hyperviseur T1) | Tailscale installé sur l'hôte (shell PVE, en root) — sert de point de départ pour le WoL     |
| **HP portable**              | Windows 11                  | Poste de pilotage — navigateur pour Tailscale/CasaOS, PowerShell + SSH pour le réveil à distance |

---

## Installation de CasaOS sur l'Acer

**CasaOS** est un système d'exploitation orienté NAS avec interface web, installé par-dessus Debian pour simplifier la gestion des services et du stockage au quotidien.


## <img width="1919" height="951" alt="dashboard_2,68To" src="https://github.com/user-attachments/assets/2cbbd285-d2b5-4abd-b398-5480f15eb41b" />



Voici comment l'installer :

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl ethtool -y

# Récupérer l'adresse MAC et l'adresse IP de la carte réseau
ip a

# Installation de CasaOS
curl -fsSL https://get.casaos.io | sudo bash
```
##
Le système d'exploitation (Debian) étant installé sur le disque principal, les deux disques secondaires nécessitaient d'être initialisés et montés pour être exploitables. Cette configuration a été réalisée via l'interface de CasaOS afin de porter la capacité totale de stockage à environ 2,68 To.

Création d'un nouveau volume de stockage :

## <img width="1919" height="947" alt="choix_disque" src="https://github.com/user-attachments/assets/c6e2602b-c8fe-4303-afc9-605a7336ea4c" />


Sélection du disque brut, nommage et formatage :

<img width="1919" height="946" alt="nom_formattage_disque1" src="https://github.com/user-attachments/assets/384a5de0-ae41-49c7-be4d-dc98c940863c" />


L'opération est ensuite répétée pour le dernier disque, permettant ainsi d'exploiter pleinement l'architecture multi-disques du NAS.

---

## Wake-on-LAN — réveil à distance

Pour ne pas laisser l'Acer allumé en permanence, le Wake-on-LAN est activé au démarrage via un service systemd dédié, qui applique l'option `wol g` (réveil sur magic packet) à l'interface réseau `enp3s0` à chaque boot.

```bash
sudo bash -c 'cat <<EOF > /etc/systemd/system/wol.service
[Unit]
Description=Enable Wake-on-LAN
After=network.target

[Service]
Type=oneshot
ExecStart=/sbin/ethtool -s enp3s0 wol g

[Install]
WantedBy=basic.target
EOF'

sudo systemctl daemon-reload
sudo systemctl enable wol.service
sudo systemctl start wol.service
```

Le réveil à distance s'effectue ensuite en envoyant un magic packet à l'adresse MAC de l'Acer (*remplacer aa:aa:aa:aa:aa:aa par l'adresse MAC de la machine*) :

```bash
wakeonlan aa:aa:aa:aa:aa:aa
```

En pratique, cette commande est lancée depuis un terminal PowerShell sur le HP portable, après connexion en SSH au shell PVE (en root de Proxmox) sur le HP EliteDesk — c'est ce qui permet de réveiller l'Acer à distance sans avoir physiquement accès au boîtier.

---

## Tailscale — VPN mesh entre les machines
<img width="1902" height="947" alt="tailscale" src="" />

**Tailscale** relie les machines en réseau privé (VPN mesh basé sur WireGuard), ce qui permet d'atteindre CasaOS et le NAS depuis n'importe où sans exposer de port sur Internet.

- Sur le **HP EliteDesk**, Tailscale est installé directement au niveau de l'hyperviseur, dans le shell PVE (en root de Proxmox).
- Sur l'**Acer**, Tailscale est installé sur Debian, machine hôte de CasaOS.
- Le **HP portable** sert de poste de pilotage : navigateur pour l'admin console Tailscale et l'interface CasaOS, et terminal PowerShell pour se connecter en SSH et déclencher le Wake-on-LAN.

Installation côté Proxmox (shell PVE) :

```bash
# Désactiver les dépôts entreprise Proxmox/Ceph pour éviter les erreurs apt sans abonnement
echo "" > /etc/apt/sources.list.d/pve-enterprise.sources 2>/dev/null
echo "" > /etc/apt/sources.list.d/ceph.sources 2>/dev/null
rm -f /etc/apt/sources.list.d/*.list

# Installation de Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

La même procédure (`curl -fsSL https://tailscale.com/install.sh | sh` puis `tailscale up`) est appliquée sur l'Acer, avec connexion au **même compte Tailscale** que le HP EliteDesk, afin que les deux machines apparaissent sur le même tailnet.

---

## Conclusion

En définitive, je dispose maintenant d'un espace de stockage distant et multi-plateforme à la demande. Sa disponibilité continue repose simplement sur l'activité de l'hyperviseur relais (HP EliteDesk) et d'une connexion à un réseau.

Ce projet m'a permis :

- de découvrir CasaOS comme couche de gestion NAS par-dessus Debian
- de mettre en place le Wake-on-LAN via un service systemd dédié
- de comprendre et déployer Tailscale comme VPN mesh reliant plusieurs machines
- d'obtenir un accès distant complet à un NAS qui reste éteint la majeure partie du temps, réduisant la consommation électrique tout en gardant un accès à la demande depuis n'importe où

---

*Réalisé par [Yacine Harrache](https://github.com/yacinehrc) — BTS SIO SLAM | EPSI Lille*
