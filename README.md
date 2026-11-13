# TP Module 14 : Sauvegarde automatique de la configuration d'un routeur Cisco avec Ansible

Geneva Institute of Technology · Module 14 · Routeur R1 (GNS3, Cisco 7200 / IOS 15.2) · Contrôleur Ansible : VM Ubuntu `grafana`

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
