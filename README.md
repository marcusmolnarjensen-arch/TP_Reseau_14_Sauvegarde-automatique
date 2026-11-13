# TP Module 14 : Sauvegarde automatique de la configuration d'un routeur Cisco avec Ansible

Geneva Institute of Technology · Module 14 · Routeur R1 (GNS3, Cisco 7200 / IOS 15.2) · Contrôleur Ansible : VM Ubuntu `grafana`

> **Usage de l'IA** : ce TP a été réalisé avec l'aide de Claude (assistant IA) pour le dépannage de la liaison réseau (VMware / GNS3) et la mise en forme de ce README.
## 1. Objectif

Automatiser la sauvegarde de la `running-config` du routeur R1 depuis un contrôleur Ansible, enregistrer chaque sauvegarde horodatée sur la VM, puis planifier l'exécution quotidienne avec cron.

## 2. Architecture réalisée

| Équipement | Rôle | Adresse IP | Interface |
|---|---|---|---|
| R1 (Cisco 7200, GNS3) | Routeur cible | 192.168.10.1/26 | FastEthernet2/0 |
| grafana (Ubuntu, VMware Workstation) | Contrôleur Ansible, stockage des backups | 192.168.10.10/26 | ens33 |

```
[ VM grafana 192.168.10.10 ] --ens33-- VMnet2 (host-only) -- Cloud GNS3 -- Fa2/0 -- [ R1 192.168.10.1 ]
        |
        +-- ~/tp-ansible-backup/backups/R1_AAAAMMJJ_HHMMSS.cfg
```

### Écarts avec l'énoncé

- **Liaison VM ↔ R1** : la VM est branchée sur le réseau VMware host-only `VMnet2` (192.168.10.0/26, sans DHCP). Un nœud **Cloud** GNS3 relie ce réseau à R1.
- **Interface de R1** : `FastEthernet2/0` à la place de `GigabitEthernet0/0` (c'est celle reliée au Cloud dans ma topologie).
- Aucune passerelle par défaut : R1 est dans le même sous-réseau que la VM.
- L'utilisateur `ansible` est en `privilege 15` : la connexion SSH arrive directement sur `R1#` (et non `R1>`), le `enable` n'est donc pas nécessaire en session interactive.
- La VM n'a pas d'accès internet sur ce réseau : Ansible, `python3-paramiko` et les collections `cisco.ios` / `ansible.netcommon` étaient déjà installés.

### Test de connectivité (prérequis)

```text
grafana@grafana:~$ ping -c 5 192.168.10.1
64 bytes from 192.168.10.1: icmp_seq=1 ttl=255 time=6.14 ms
64 bytes from 192.168.10.1: icmp_seq=2 ttl=255 time=17.2 ms
64 bytes from 192.168.10.1: icmp_seq=3 ttl=255 time=6.71 ms
64 bytes from 192.168.10.1: icmp_seq=4 ttl=255 time=16.2 ms
64 bytes from 192.168.10.1: icmp_seq=5 ttl=255 time=8.60 ms

5 packets transmitted, 5 received, 0% packet loss
```

## 3. Étape 1 : SSH sur R1

```text
R1> enable
R1# configure terminal
R1(config)# hostname R1
R1(config)# ip domain-name lab.local
R1(config)# crypto key generate rsa modulus 2048
R1(config)# ip ssh version 2
R1(config)# username ansible privilege 15 secret <MDP_USER_ANSIBLE>
R1(config)# enable secret <MDP_ENABLE>
R1(config)# line vty 0 4
R1(config-line)# login local
R1(config-line)# transport input ssh
R1(config-line)# end
R1# write memory
```

Vérifications :

```text
R1# show ip ssh
SSH Enabled - version 2.0
Authentication timeout: 120 secs; Authentication retries: 3
Minimum expected Diffie Hellman key size : 1024 bits
IOS Keys in SECSH format(ssh-rsa, base64 encoded): <clé publique RSA>

R1# show run | include username|enable secret
enable secret 5 <HASH>
username ansible privilege 15 secret 5 <HASH>

R1# show run | section line vty
line vty 0 4
 login local
 transport input ssh
```

Les hashes sont masqués volontairement (données sensibles).

## 4. Étape 2 : Test SSH depuis la VM

Un premier essai a échoué avec `REMOTE HOST IDENTIFICATION HAS CHANGED` : la VM connaissait une ancienne clé pour `192.168.10.1`, alors que la clé RSA de R1 venait d'être régénérée. L'ancienne entrée a été supprimée :

```bash
ssh-keygen -f '/home/grafana/.ssh/known_hosts' -R '192.168.10.1'
```

Connexion réussie :

```text
grafana@grafana:~$ ssh -o KexAlgorithms=+diffie-hellman-group14-sha1 \
    -o HostKeyAlgorithms=+ssh-rsa \
    -c aes128-cbc \
    ansible@192.168.10.1
The authenticity of host '192.168.10.1 (192.168.10.1)' can't be established.
RSA key fingerprint is: SHA256:zrv7JqlF4efEEo7Uj4+flFA9WwRN2qbzPIrGCK8lUpY
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.10.1' (RSA) to the list of known hosts.
(ansible@192.168.10.1) Password:

R1#
```

## 5. Étape 3 : Ansible et collections

```text
grafana@grafana:~$ ansible --version
ansible [core 2.20.1]
  python version = 3.14.4
  jinja version = 3.1.6

grafana@grafana:~$ ansible-galaxy collection list | grep -E "cisco.ios|netcommon"
ansible.netcommon                        8.2.0
cisco.ios                                11.1.1
```

## 6. Étape 4 : Arborescence et fichiers

```bash
mkdir -p ~/tp-ansible-backup/{inventory,playbooks,backups}
cd ~/tp-ansible-backup
```

### `ansible.cfg`

```ini
[defaults]
inventory = inventory/hosts.ini
host_key_checking = False
deprecation_warnings = False
```

### `inventory/hosts.ini` (mots de passe masqués)

```ini
[cisco]
R1 ansible_host=192.168.10.1

[cisco:vars]
ansible_user=ansible
ansible_password=<MDP>
ansible_connection=ansible.netcommon.network_cli
ansible_network_os=cisco.ios.ios
ansible_become=yes
ansible_become_method=enable
ansible_become_password=<MDP>
ansible_ssh_common_args='-o KexAlgorithms=+diffie-hellman-group14-sha1 -o HostKeyAlgorithms=+ssh-rsa -c aes128-cbc'
```

### `playbooks/backup_r1.yml`

```yaml
---
- name: Sauvegarde de la configuration du routeur R1
  hosts: cisco
  gather_facts: false

  vars:
    backup_dir: "{{ playbook_dir }}/../backups"

  tasks:
    - name: Créer le dossier de sauvegarde sur la VM
      ansible.builtin.file:
        path: "{{ backup_dir }}"
        state: directory
        mode: "0755"
      delegate_to: localhost
      run_once: true

    - name: Sauvegarder la running-config
      cisco.ios.ios_config:
        backup: true
        backup_options:
          filename: "{{ inventory_hostname }}_{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}.cfg"
          dir_path: "{{ backup_dir }}"
```

## 7. Étape 5 : Test de connectivité Ansible

```text
grafana@grafana:~/tp-ansible-backup$ ansible cisco -m cisco.ios.ios_command -a "commands='show version'"
[WARNING]: ansible-pylibssh not installed, falling back to paramiko
R1 | SUCCESS => {
    "changed": false,
    "stdout": [
        "Cisco IOS Software, 7200 Software (C7200-ADVENTERPRISEK9-M), Version 15.2(4)S7, RELEASE SOFTWARE (fc4) [...]"
    ]
}
```

(Sortie tronquée : le reste est le détail de `show version`.) Le warning `ansible-pylibssh` est sans gravité : Ansible utilise paramiko.

## 8. Étape 6 : Lancement de la sauvegarde

```text
grafana@grafana:~/tp-ansible-backup$ ansible-playbook playbooks/backup_r1.yml

PLAY [Sauvegarde de la configuration du routeur R1] ****************************

TASK [Créer le dossier de sauvegarde sur la VM] ********************************
changed: [R1 -> localhost]

TASK [Sauvegarder la running-config] *******************************************
[WARNING]: ansible-pylibssh not installed, falling back to paramiko
changed: [R1]

PLAY RECAP *********************************************************************
R1                         : ok=2    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Vérification du fichier créé :

```text
grafana@grafana:~/tp-ansible-backup$ ls -l backups/
total 4
-rw-rw-r-- 1 grafana grafana 1677 Nov 13 14:08 R1_20261113_140814.cfg

grafana@grafana:~/tp-ansible-backup$ head -20 backups/R1_*.cfg
Building configuration...

Current configuration : 1617 bytes
!
! Last configuration change at 13:55:11 UTC Fri Nov 13 2026
!
version 15.2
service timestamps debug datetime msec
service timestamps log datetime msec
!
hostname R1
!
boot-start-marker
boot-end-marker
!
!
enable secret 5 <HASH>
!
no aaa new-model
no ip icmp rate-limit unreachable
```

On retrouve `hostname R1`, la version 15.2 et la configuration du routeur.

Sécurisation du dossier :

```bash
chmod 700 ~/tp-ansible-backup/backups
chmod 600 ~/tp-ansible-backup/backups/*.cfg
```

## 9. Étape 7 : Planification avec cron

Service cron actif :

```text
grafana@grafana:~$ systemctl status cron --no-pager
● cron.service - Regular background program processing daemon
     Loaded: loaded (/usr/lib/systemd/system/cron.service; enabled; preset: enabled)
     Active: active (running) since Fri 2026-11-13 13:31:32 CET
```

Contenu de `crontab -l` (chemins adaptés à l'utilisateur `grafana`, sauvegarde à 02:00, rotation à 02:30) :

```text
grafana@grafana:~$ crontab -l
0 2 * * * cd /home/grafana/tp-ansible-backup && /usr/bin/ansible-playbook playbooks/backup_r1.yml >> /home/grafana/tp-ansible-backup/backup.log 2>&1
30 2 * * * find /home/grafana/tp-ansible-backup/backups -name "R1_*.cfg" -mtime +30 -delete

grafana@grafana:~$ which ansible-playbook
/usr/bin/ansible-playbook
```

Test en conditions cron (environnement minimal) :

```text
grafana@grafana:~$ env -i HOME=/home/grafana PATH=/usr/bin:/bin bash -c 'cd /home/grafana/tp-ansible-backup && /usr/bin/ansible-playbook playbooks/backup_r1.yml'

PLAY [Sauvegarde de la configuration du routeur R1] ****************************

TASK [Créer le dossier de sauvegarde sur la VM] ********************************
changed: [R1 -> localhost]

TASK [Sauvegarder la running-config] *******************************************
changed: [R1]

PLAY RECAP *********************************************************************
R1                         : ok=2    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

`failed=0` : la tâche planifiée fonctionnera.

## 10. Étape 8 : Preuve qu'une sauvegarde reflète l'état réel

Modification sur R1 (interface reliée à la VM) :

```text
R1# configure terminal
R1(config)# interface FastEthernet2/0
R1(config-if)# description TEST-BACKUP
R1(config-if)# end
R1# write memory
Building configuration...
[OK]
```

Nouvelle sauvegarde puis comparaison des deux derniers backups :

```text
grafana@grafana:~/tp-ansible-backup$ ansible-playbook playbooks/backup_r1.yml
[...]
PLAY RECAP *********************************************************************
R1                         : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

grafana@grafana:~/tp-ansible-backup/backups$ ls -t R1_*.cfg | head -2
R1_20261113_141104.cfg
R1_20261113_140829.cfg

grafana@grafana:~/tp-ansible-backup/backups$ diff $(ls -t R1_*.cfg | head -2 | tail -1) $(ls -t R1_*.cfg | head -1)
3c3
< Current configuration : 1617 bytes
---
> Current configuration : 1642 bytes
5c5
< ! Last configuration change at 13:55:11 UTC Fri Nov 13 2026
---
> ! Last configuration change at 14:10:48 UTC Fri Nov 13 2026
67a68
>  description TEST-BACKUP
```

Le `diff` contient la ligne `description TEST-BACKUP` : la sauvegarde reflète bien l'état réel du routeur.

## 11. Livrables

- [x] Configuration SSH de R1 (`show ip ssh`, `show run | section line vty`)
- [x] `ansible.cfg` et inventaire (mots de passe remplacés par `<MDP>`)
- [x] Playbook `backup_r1.yml`
- [x] Sortie complète de `ansible-playbook playbooks/backup_r1.yml`
- [x] Contenu de `crontab -l`
- [x] Sortie de `diff` de l'étape 8

## 12. Sécurité

- Aucun mot de passe réel ni hash dans ce dépôt (remplacés par `<MDP>` et `<HASH>`). Les fichiers de sauvegarde (`backups/*.cfg`) contiennent les hashes de R1 : ne pas les versionner.
- Dossier de sauvegarde en `chmod 700`, fichiers en `chmod 600`.
- Les mots de passe de démonstration sont à changer après le TP.
- En production : `ansible-vault` pour les secrets, sauvegardes copiées hors du contrôleur.

---

# TP Module 14 part.2 : Sujets de TP Ansible

Suite du module 14 : sujets de TP Ansible (niveau débutant à avancé) réalisés sur la même maquette que la partie 1 (VM `grafana` sous VMware, R1 sous GNS3).

| TP | Sujet | État |
|---|---|---|
| TP 1 | Préparer une VM Ubuntu | Réalisé |
| TP 2 | Standardiser un routeur | Réalisé (adapté à ma topologie) |
| TP 3 | Auditer la conformité d'un routeur | Réalisé (avec l'exercice de sabotage) |
| TP 4 | Configurer des VLAN sur un switch | Non réalisé : la maquette ne contient pas de switch IOS (le switch Ethernet intégré à GNS3 n'a pas de CLI ni de SSH, Ansible ne peut donc pas le configurer) |

## 1. Préparation commune

### Accès internet de la VM

Le réseau `VMnet2` (host-only) ne donne pas accès à internet. Une seconde carte réseau en NAT (`VMnet8`) a été ajoutée à la VM : `ens33` reste sur `VMnet2` (lien vers R1), `ens37` sert à internet.

```bash
sudo nmcli con add type ethernet ifname ens37 con-name internet ipv4.method auto
sudo nmcli con up internet
ping -c 3 8.8.8.8
ping -c 3 archive.ubuntu.com
ping -c 3 192.168.10.1
```

Les trois pings répondent (0 % de perte) : internet via `ens37` (192.168.100.x), R1 via `ens33` (192.168.10.10/26). La collection `community.general` était déjà installée (`ansible-galaxy collection install community.general` : « Nothing to do »).

### `sudo` sans mot de passe (VM de labo uniquement)

`become` d'Ansible n'arrivait pas à répondre à l'invite de mot de passe de `sudo` (voir la section « Problèmes rencontrés »). Contournement, à retirer en fin de TP :

```bash
echo 'grafana ALL=(ALL) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/90-lab-ansible
sudo chmod 440 /etc/sudoers.d/90-lab-ansible
sudo visudo -cf /etc/sudoers.d/90-lab-ansible
```

### Arborescence et fichiers

```bash
mkdir -p ~/tp-ansible/{inventory,playbooks}
cd ~/tp-ansible
```

`ansible.cfg` :

```ini
[defaults]
inventory = inventory/hosts.ini
host_key_checking = False
deprecation_warnings = False
```

`inventory/hosts.ini` (mots de passe masqués) :

```ini
[routers]
R1 ansible_host=192.168.10.1

[switches]
SW1 ansible_host=192.168.10.2

[cisco:children]
routers
switches

[cisco:vars]
ansible_user=ansible
ansible_password=<MDP>
ansible_connection=ansible.netcommon.network_cli
ansible_network_os=cisco.ios.ios
ansible_become=yes
ansible_become_method=enable
ansible_become_password=<MDP>
ansible_ssh_common_args='-o KexAlgorithms=+diffie-hellman-group14-sha1 -o HostKeyAlgorithms=+ssh-rsa -c aes128-cbc'

[linux]
grafana ansible_host=localhost ansible_connection=local
```

`SW1` figure dans l'inventaire comme dans l'énoncé, mais n'est utilisé que par le TP 4 (non réalisé).

## 2. TP 1 : Préparer une VM Ubuntu

### Playbook `playbooks/tp1_ubuntu_base.yml`

```yaml
---
- name: TP1 - Configuration de base Ubuntu
  hosts: linux
  become: yes
  tasks:
    - name: Créer l'utilisateur étudiant
      ansible.builtin.user:
        name: etudiant
        shell: /bin/bash
        groups: sudo
        append: yes

    - name: Installer les outils de base
      ansible.builtin.apt:
        name:
          - curl
          - net-tools
          - tree
        state: present
        update_cache: yes

    - name: Régler le fuseau horaire
      community.general.timezone:
        name: Europe/Zurich
```

### Exécutions

```text
grafana@grafana:~/tp-ansible$ ansible-playbook playbooks/tp1_ubuntu_base.yml

PLAY [TP1 - Configuration de base Ubuntu] ***

TASK [Gathering Facts] ***
ok: [grafana]

TASK [Créer l'utilisateur étudiant] ***
ok: [grafana]

TASK [Installer les outils de base] ***
changed: [grafana]

TASK [Régler le fuseau horaire] ***
changed: [grafana]

PLAY RECAP ***
grafana                    : ok=4    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

La tâche utilisateur affiche `ok` (et non `changed`) car `etudiant` avait déjà été créé par une tentative précédente, interrompue à la tâche `apt` (voir « Problèmes rencontrés »). L'utilisateur avait alors bien été créé (`changed`).

Seconde exécution (idempotence) :

```text
PLAY RECAP ***
grafana                    : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### Vérifications

```text
grafana@grafana:~/tp-ansible$ id etudiant
uid=1001(etudiant) gid=1001(etudiant) groups=1001(etudiant),27(sudo)

grafana@grafana:~/tp-ansible$ timedatectl | grep "Time zone"
                Time zone: Europe/Zurich (CET, +0100)
```

### Critères de réussite

- [x] `failed=0` à la première exécution
- [x] L'utilisateur `etudiant` appartient au groupe `sudo`
- [x] Le fuseau horaire est `Europe/Zurich`
- [x] Idempotence : la seconde exécution affiche `changed=0`

### Questions

1. **Pourquoi la seconde exécution ne modifie-t-elle rien ?** Les modules Ansible sont idempotents : ils comparent l'état voulu à l'état réel et n'agissent que s'il y a un écart. L'utilisateur, les paquets et le fuseau horaire étant déjà conformes, aucune tâche ne change rien (`changed=0`).
2. **Que se passerait-il sans `append: yes` ?** `groups: sudo` remplacerait tous les groupes secondaires de l'utilisateur par `sudo` seul, au lieu d'ajouter `sudo` aux groupes existants.

## 3. TP 2 : Standardiser un routeur

### Adaptations à ma topologie

| Énoncé | Ma maquette | Raison |
|---|---|---|
| `GigabitEthernet0/0` (LAN gestion) | `FastEthernet2/0` | R1 n'a pas de Gi0/0 ; Fa2/0 est reliée au Cloud (VMnet2) |
| `GigabitEthernet1/0` (LAN utilisateurs) | `GigabitEthernet1/0` | Identique |
| Passerelle `192.168.10.254` | `192.168.10.20` | `192.168.10.254` est hors du réseau `192.168.10.0/26` (de .0 à .63) ; l'énoncé demande d'adapter la passerelle |

### Playbook `playbooks/tp2_routeur.yml`

```yaml
---
- name: TP2 - Standardisation du routeur R1
  hosts: routers
  gather_facts: false
  tasks:
    - name: Bannière de connexion
      cisco.ios.ios_banner:
        banner: login
        text: |
          Acces autorise uniquement au personnel GIT.
        state: present

    - name: Description des interfaces
      cisco.ios.ios_interfaces:
        config:
          - name: FastEthernet2/0
            description: "LAN gestion - VM grafana"
          - name: GigabitEthernet1/0
            description: "LAN utilisateurs"
        state: merged

    - name: Route statique vers le réseau distant
      cisco.ios.ios_static_routes:
        config:
          - vrf: null
            address_families:
              - afi: ipv4
                routes:
                  - dest: 10.99.0.0/24
                    next_hops:
                      - forward_router_address: 192.168.10.20
        state: merged
```

### Exécutions

```text
grafana@grafana:~/tp-ansible$ ansible-playbook playbooks/tp2_routeur.yml

TASK [Bannière de connexion] ***
changed: [R1]

TASK [Description des interfaces] ***
changed: [R1]

TASK [Route statique vers le réseau distant] ***
changed: [R1]

PLAY RECAP ***
R1                         : ok=3    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Seconde exécution (idempotence) :

```text
TASK [Bannière de connexion] ***
ok: [R1]
TASK [Description des interfaces] ***
ok: [R1]
TASK [Route statique vers le réseau distant] ***
ok: [R1]

PLAY RECAP ***
R1                         : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### Vérifications

```text
grafana@grafana:~/tp-ansible$ ansible routers -m cisco.ios.ios_command -a "commands='show ip route static'"
R1 | SUCCESS => {
    "stdout_lines": [
        [
            "[...]",
            "Gateway of last resort is not set",
            "",
            "      10.0.0.0/24 is subnetted, 1 subnets",
            "S        10.99.0.0 [1/0] via 192.168.10.20"
        ]
    ]
}

grafana@grafana:~/tp-ansible$ ansible routers -m cisco.ios.ios_command -a "commands='show interfaces description'"
R1 | SUCCESS => {
    "stdout_lines": [
        [
            "Interface                      Status         Protocol Description",
            "Fa0/0                          admin down     down     ",
            "Gi1/0                          up             up       LAN utilisateurs",
            "Fa2/0                          up             up       LAN gestion - VM grafana",
            "Fa3/0                          admin down     down     ",
            "Fa3/1                          admin down     down     ",
            "Gi4/0                          admin down     down     ",
            "Se5/0                          admin down     down     ",
            "Se5/1                          admin down     down     ",
            "Se5/2                          admin down     down     ",
            "Se5/3                          admin down     down"
        ]
    ]
}
```

Bannière affichée lors d'une nouvelle connexion SSH à R1 :

```text
grafana@grafana:~/tp-ansible$ ssh -o KexAlgorithms=+diffie-hellman-group14-sha1 -o HostKeyAlgorithms=+ssh-rsa -c aes128-cbc ansible@192.168.10.1

Acces autorise uniquement au personnel GIT.
(ansible@192.168.10.1) Password:

R1#
```

### Critères de réussite

- [x] La route `S 10.99.0.0/24` apparaît dans la table de routage
- [x] Les descriptions des interfaces sont visibles
- [x] La bannière s'affiche à la connexion SSH (tâche `changed`, puis `ok` à la seconde exécution)

### Questions

1. **Différence entre `merged` et `replaced` ?** `merged` ajoute ou modifie uniquement ce qui est décrit et laisse le reste de la configuration intact. `replaced` remplace la configuration des objets listés (par exemple une interface) par exactement ce qui est décrit, en supprimant ce qui est en trop sur ces objets.
2. **Pourquoi se méfier de `overridden` en production ?** `overridden` aligne tout l'équipement sur la liste fournie : tout ce qui n'y figure pas est supprimé. Avec une liste incomplète, on peut retirer des interfaces, des routes ou la configuration qui permet à Ansible de se connecter, et couper l'accès au routeur.

## 4. TP 3 : Auditer la conformité d'un routeur

### Playbook `playbooks/tp3_audit.yml`

```yaml
---
- name: TP3 - Audit de conformité
  hosts: routers
  gather_facts: false
  tasks:
    - name: Lire la configuration
      cisco.ios.ios_command:
        commands: show running-config
      register: cfg

    - name: Lire l'état SSH
      cisco.ios.ios_command:
        commands: show ip ssh
      register: ssh_state

    - name: Vérifier la politique
      ansible.builtin.assert:
        that:
          - "'enable secret' in cfg.stdout[0]"
          - "'ip ssh version 2' in cfg.stdout[0]"
          - "'transport input telnet' not in cfg.stdout[0]"
          - "'SSH Enabled - version 2' in ssh_state.stdout[0]"
        fail_msg: "Non conforme : corriger la configuration"
        success_msg: "Conforme à la politique"
```

### Audit initial (configuration conforme)

```text
grafana@grafana:~/tp-ansible$ ansible-playbook playbooks/tp3_audit.yml

PLAY [TP3 - Audit de conformité] ***

TASK [Lire la configuration] ***
ok: [R1]

TASK [Lire l'état SSH] ***
ok: [R1]

TASK [Vérifier la politique] ***
ok: [R1] => {
    "changed": false,
    "msg": "Conforme à la politique"
}

PLAY RECAP ***
R1                         : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### Exercice de sabotage

1. Activation volontaire de Telnet sur R1 :

```text
R1# configure terminal
R1(config)# line vty 0 4
R1(config-line)# transport input ssh telnet
R1(config-line)# end
R1# write memory
Building configuration...
[OK]
```

2. L'audit échoue, comme attendu :

```text
grafana@grafana:~/tp-ansible$ ansible-playbook playbooks/tp3_audit.yml

TASK [Lire la configuration] ***
ok: [R1]

TASK [Lire l'état SSH] ***
ok: [R1]

TASK [Vérifier la politique] ***
[ERROR]: Task failed: Action failed: Non conforme : corriger la configuration

fatal: [R1]: FAILED! => {
    "assertion": "'transport input telnet' not in cfg.stdout[0]",
    "changed": false,
    "evaluated_to": false,
    "msg": "Non conforme : corriger la configuration"
}

PLAY RECAP ***
R1                         : ok=2    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
```

3. Restauration de la configuration correcte :

```text
R1# configure terminal
R1(config)# line vty 0 4
R1(config-line)# transport input ssh
R1(config-line)# end
R1# write memory
Building configuration...
[OK]
```

4. L'audit réussit de nouveau :

```text
grafana@grafana:~/tp-ansible$ ansible-playbook playbooks/tp3_audit.yml

TASK [Lire la configuration] ***
ok: [R1]

TASK [Lire l'état SSH] ***
ok: [R1]

TASK [Vérifier la politique] ***
ok: [R1] => {
    "changed": false,
    "msg": "Conforme à la politique"
}

PLAY RECAP ***
R1                         : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### Critères de réussite

- [x] Audit réussi sur la configuration correcte
- [x] Audit échoué avec Telnet actif, puis réussi après correction
- [x] L'échec vient de l'assertion `'transport input telnet' not in cfg.stdout[0]` évaluée à `false`
- [x] Telnet n'est plus actif en fin d'exercice (`transport input ssh` uniquement)

### Questions

1. **Pourquoi lit-on `cfg.stdout[0]` et non `cfg.stdout` ?** `stdout` est une liste avec un élément par commande exécutée par `ios_command`. Une seule commande (`show running-config`) est lancée, sa sortie est donc à l'indice 0. `cfg.stdout` seul est une liste, pas une chaîne de texte : le test `in` ne chercherait pas dans le texte de la configuration.
2. **Comment ajouter une règle sur la longueur minimale du mot de passe ?** Configurer d'abord `security passwords min-length 10` sur R1, puis ajouter une condition à l'`assert`, par exemple `"'security passwords min-length' in cfg.stdout[0]"`. Pour contrôler la valeur exacte, on peut extraire la ligne avec le filtre `regex_search` et comparer le nombre à la longueur voulue.

## 5. TP 4 : Configurer des VLAN sur un switch

Non réalisé

## 6. Problèmes rencontrés et résolutions

### Liaison réseau VM ↔ R1 (parties 1 et 2)

| Problème | Cause | Résolution |
|---|---|---|
| `docker: failed to connect to the docker API` | Docker Desktop non démarré | Démarrer Docker Desktop (voir ci-dessous : l'approche Docker a finalement été abandonnée) |
| `mkdir C:\WINDOWS\system32\tp-ansible-backup: Access is denied` | Commande lancée depuis le dossier système de Windows ; les commandes prévues pour l'intérieur du conteneur ont aussi été tapées dans PowerShell | Se placer dans le dossier utilisateur (`cd $HOME`) ; commandes du conteneur tapées dans le conteneur |
| Ping impossible entre le PC et R1 avec le Cloud GNS3 sur le Wi-Fi | La carte Wi-Fi ne relaie pas les trames d'un routeur virtuel ; le Wi-Fi n'a pas d'IP dans 192.168.10.0/26 | Utiliser une carte dédiée (loopback Windows, puis réseau VMware) |
| R1 répond à l'ARP mais Windows ne reçoit pas les réponses (loopback, puis adaptateur VMnet2 de l'hôte) | L'adaptateur de l'hôte Windows ne sert pas de point d'arrivée fiable pour le Cloud GNS3 (captures Wireshark : requête ARP et réponse de R1 visibles, ping Windows en échec) | Mettre le contrôleur Ansible dans une VM branchée sur `VMnet2`, comme le prévoit l'énoncé, et abandonner Docker sur l'hôte |
| Conflit d'adresse : l'adaptateur hôte `VMnet2` prenait 192.168.10.1 (IP de R1), puis 192.168.10.10 (IP de la VM) | VMware attribue la première adresse du réseau à l'adaptateur hôte | Adaptateur hôte passé en 192.168.10.11, sans passerelle ; cache ARP de R1 vidé (`clear arp`) |
| Ping impossible alors que `show ip interface brief` semblait correct | IP de R1 configurée sur la mauvaise interface (le câble Cloud est sur Fa2/0, pas Gi1/0) et Fa2/0 en `administratively down` après rechargement du projet | IP `192.168.10.1/26` mise sur `FastEthernet2/0` avec `no shutdown` |
| VM : `ping: connect: Le réseau n'est pas accessible` | Aucune IPv4 sur `ens33` : passerelle `192.168.10.0` invalide (adresse réseau) et profil NetworkManager non appliqué (`nmcli con up` : « La configuration IP n'a pas pu être réservée ») | Passerelle supprimée (inutile dans le même sous-réseau) ; IP posée avec `sudo ip addr add 192.168.10.10/26 dev ens33` (non persistante au redémarrage) |
| 66 % de perte de paquets puis 100 % depuis la VM | Même IP 192.168.10.10 sur la VM et sur l'adaptateur hôte VMnet2 (R1 gardait la MAC de Windows en cache) | Voir le conflit d'adresse ci-dessus |

### SSH et Ansible (partie 1)

| Problème | Cause | Résolution |
|---|---|---|
| `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED` à la première connexion SSH | La VM connaissait une ancienne clé pour 192.168.10.1 ; la clé RSA de R1 venait d'être régénérée | `ssh-keygen -f ~/.ssh/known_hosts -R 192.168.10.1` puis nouvelle connexion (acceptation de la nouvelle empreinte) |
| `R1# exit` refusé / `enable` sans effet | L'utilisateur `ansible` est en `privilege 15` : la connexion arrive directement sur `R1#` ; la commande avait été collée avec l'invite | Taper `exit` seul ; le `enable` n'est pas nécessaire en session interactive |
| `[WARNING]: ansible-pylibssh not installed, falling back to paramiko` | Bibliothèque SSH optionnelle absente | Sans gravité, Ansible utilise paramiko comme prévu par l'énoncé |

### Accès internet de la VM (partie 2)

| Problème | Cause | Résolution |
|---|---|---|
| Après avoir passé `VMnet2` en NAT, `VMnet8` est devenu host-only | VMware n'autorise qu'un seul réseau NAT ; `VMnet2` en NAT aurait aussi pris l'adresse 192.168.10.2 (passerelle VMware), prévue pour SW1 | `VMnet2` remis en Host-only (192.168.10.0/26, sans DHCP) et `VMnet8` en NAT (192.168.100.0/24) |
| `nmcli con up internet` : « No suitable device found... mismatching interface name » | La VM n'avait qu'une carte (`ens33`) : `ens37` n'existait pas | Ajout d'une seconde carte réseau en NAT à la VM |
| `apt` / playbook : `Failed to update apt cache after 5 retries`, `Échec temporaire dans la résolution du nom` | `ens37` était `déconnecté` sans profil : aucune route par défaut, donc pas de DNS ni d'internet | Création du profil DHCP : `sudo nmcli con add type ethernet ifname ens37 con-name internet ipv4.method auto`, puis `sudo nmcli con up internet` |

### Ansible sur la VM (partie 2)

| Problème | Cause | Résolution |
|---|---|---|
| `Timed out waiting for become success or become password prompt` (même avec `--ask-become-pass`) | La VM Ubuntu 26.04 utilise `sudo-rs` (confirmé) : son invite de mot de passe n'est pas reconnue par `become` | Contournement de labo : `NOPASSWD` pour l'utilisateur dans `/etc/sudoers.d/90-lab-ansible`, puis exécution sans `--ask-become-pass`. À retirer en fin de TP |
| Playbook non créé : dossier `playbooks/` vide | Collage du bloc `cat > ... << 'EOF'` interrompu avant la ligne `EOF` ; le terminal attendait la suite | Recréation du playbook, vérification avec `ls -R` et `cat` |
| Messages `commande introuvable` en rafale | Sortie d'une commande collée dans le terminal comme si c'étaient des commandes ; `!!` interprété par bash comme « rejouer la dernière commande » | Ne coller que les commandes, jamais leur résultat |
| TP 2 : `GigabitEthernet0/0` inexistante et passerelle `192.168.10.254` hors du réseau | L'énoncé suppose une autre topologie que la mienne | Adaptation : `FastEthernet2/0` et passerelle `192.168.10.20` (voir la section TP 2) |

## 7. Livrables de la partie 2

- [x] Playbooks `tp1_ubuntu_base.yml`, `tp2_routeur.yml`, `tp3_audit.yml`, sans mot de passe
- [x] Sorties complètes d'`ansible-playbook` (exécution et seconde exécution pour l'idempotence, échec puis réussite pour l'audit)
- [x] Sorties des commandes `show` et de vérification demandées
- [x] Réponses aux questions des TP 1, 2 et 3
- [ ] TP 4 : non réalisé (pas de switch IOS)

## 8. Sécurité (partie 2)

- Aucun mot de passe réel dans le README : l'inventaire est présenté avec `<MDP>`.
- La règle `NOPASSWD` est un contournement de labo à retirer après le TP : `sudo rm /etc/sudoers.d/90-lab-ansible`.
- Les mots de passe de démonstration de R1 sont à changer après le TP.
- L'IP `192.168.10.10` de `ens33` posée avec `ip addr add` n'est pas persistante : à rendre permanente via netplan avant tout redémarrage de la VM.
