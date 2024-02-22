# ceph
dealing with ceph
```
Source info: https://computingforgeeks.com/install-ceph-storage-cluster-on-ubuntu-linux-servers/

Skema
===========================================================
admin-00 : Ceph Mon, MGR, MDS: 192.168.0.10
node-00: OSD: 192.168.0.11
node-01: OSD: 192.168.0.15
node-02: OSD: 192.168.0.14

Step 1:
===========================================================
-SSH to server admin-00 kemudian update hosts pada /etc/hosts
# BEGIN ANSIBLE MANAGED BLOCK
127.0.0.1    localhost
192.168.0.10 admin-00
192.168.0.11 node-00
192.168.0.15 node-01
192.168.0.14 node-02

-Update repository
$ apt-get update

-install base software
$ apt-get install software-properties-common git curl vim bash-completion ansible

-update path
$ echo "PATH=\$PATH:/usr/local/bin" >>~/.bashrc
$ source ~/.bashrc


Step 2:
===========================================================
-Generate SSH Keys
ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/id_rsa

- Copy ssh key to all nodes
ssh-copy-id root@admin-00
ssh-copy-id root@node-00
ssh-copy-id root@node-01
ssh-copy-id root@node-02


Step 3: 
===========================================================
-Create script for preparing provisioning ceph
$ cd ~/
$ vim prepare-ceph-nodes.yml

# Begin prepare ceph nodes.yml
---
- name: Prepare ceph nodes
  hosts: ceph_nodes
  become: yes
  become_method: sudo
  vars:
    ceph_admin_user: cephadmin
  tasks:
    - name: Set timezone
      timezone:
        name: Asia/Jakarta

    - name: Update system
      apt:
        name: "*"
        state: latest
        update_cache: yes

    - name: Install common packages
      apt:
        name: [vim,git,bash-completion,wget,curl,chrony]
        state: present
        update_cache: yes

    - name: Set authorized key taken from file to root user
      authorized_key:
        user: root
        state: present
        key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"

    - name: Install Docker
      shell: |
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
        echo "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable" > /etc/apt/sources.list.d/docker-ce.list
        apt update
        apt install -qq -y docker-ce docker-ce-cli containerd.io

    - name: Reboot server after update and configs
      reboot:

# End prepare-ceph.yaml 

-create update-hosts script
$ vim update-hosts.yml
# Start update-hosts.yml
---
- name: Prepare ceph nodes
  hosts: ceph_nodes
  become: yes
  become_method: sudo
  tasks:
    - name: Clean /etc/hosts file
      copy:
        content: ""
        dest: /etc/hosts

    - name: Update /etc/hosts file
      blockinfile:
        path: /etc/hosts
        block: |
           127.0.0.1    localhost
           192.168.0.10 admin-00
           192.168.0.11 node-00
           192.168.0.15 node-01
           192.168.0.14 node-02
# End update-hosts.yml


- Create Inventory Files
$ vim hosts
# Begin hosts node
[ceph_nodes]
admin-00
node-00
node-01
node-02
# End hosts node

-Run playbook prepare ceph nodes
ansible-playbook -i hosts prepare-ceph-nodes.yml --user root

- Update Hosts 
ansible-playbook -i hosts update-hosts.yml --user root


Step 4
===========================================================
- Download cephadm
curl --silent --remote-name --location https://github.com/ceph/ceph/raw/octopus/src/cephadm/cephadm

-give access
chmod +x cephadm

-copy to bin
cp cephadm /usr/local/bin

- Install cephadm
./cephadm install

- add repo
./cephadm add-repo --release octopus

-install ceph-common
cephadm install ceph-common


Step 5
===========================================================
- Bootstraping new cluster
mkdir -p /etc/ceph
cephadm bootstrap --mon-ip 192.168.0.10

akan muncul info akses admin dan passwordnya


Step 6 (Add another Monitor, OPTIONAL)
===========================================================
# Start Terminal
--- Copy Ceph SSH key ---
ssh-copy-id -f -i /etc/ceph/ceph.pub root@admin-01
ssh-copy-id -f -i /etc/ceph/ceph.pub root@admin-02


Step 7 
===========================================================
--- Label the nodes with mon ---
ceph orch host label add admin-00 mon

--- Add nodes to the cluster ---
ceph orch host add admin-00

--- Apply configs ---
ceph orch apply mon admin-00

# End of Terminal

-Verify
ceph orch host ls

-check docker
docker ps -a


Step 8 (Deploy OSD)
===========================================================
ssh-copy-id -f -i /etc/ceph/ceph.pub root@node-00
ssh-copy-id -f -i /etc/ceph/ceph.pub root@node-01
ssh-copy-id -f -i /etc/ceph/ceph.pub root@node-02

-Add host cluster
ceph orch host add node-00
ceph orch host add node-01
ceph orch host add node-02

- Give Labels
ceph orch host label add  node-00 osd
ceph orch host label add  node-01 osd
ceph orch host label add  node-02 osd

-Check device 
ceph orch device ls

-Add daemon osd
ceph orch daemon add osd node-00:/dev/sdb
ceph orch daemon add osd node-01:/dev/sdb
ceph orch daemon add osd node-02:/dev/sdb


-Check status
ceph -s


Step 9 
===========================================================
-create pool and mds
ceph osd pool create cephfs_data
ceph osd pool create cephfs_metadata

-create fs
ceph fs new cephfs cephfs_metadata cephfs_data

-check stat
ceph mds stat

-apply mds to node
ceph orch apply mds cephfs --placement="3 node-00 node-01 node-02"

-check fs ls
ceph fs ls

-check fs status
ceph fs status cephfs


Step 10 Preparing client nodes 
===========================================================
- Transfer public key
ssh-copy-id root@client-ceph

-Install fuse and ceph common
ssh client-ceph "apt-get install ceph-common && ceph-fuse -y"

-transfer required file to client host
scp /etc/ceph/ceph.conf client-ceph:/etc/ceph/
scp /etc/ceph/ceph.client.admin.keyring client-ceph:/etc/ceph/
ssh client-ceph "chown ceph. /etc/ceph/ceph.*"

Step 11 Mount
===========================================================
$ ceph-authtool -p /etc/ceph/ceph.client.admin.keyring > admin.key
$ chmod 600 admin.key
$ mount -t ceph node-02:/ /mnt -o name=admin,secretfile=admin.key

-check
df -h




=======================================================
Note:
-check all service in cephadm
cephadm ls

-Removing monitor
systemctl stop ceph.target
ceph mon remove admin-00syste
if take long time just ctrl+c
systemctl start ceph.target

- Redeploy bootstraping 
systemctl stop ceph.target
jalankan step bootstraping diatas lg

-stop all service ceph
systemctl stop ceph.target

-start all service ceph
systemctl start ceph.target

-checking ceph status detail
ceph -w
ceph
-remove ceph host
ceph orch host rm node-1

-remove ceph label tag
ceph orch host label rm node-04 osd


Trouble
==========================================================
- jika ceph sudah operational, kemudian kita mengganti ip ceph dengan segment 192. maka monitor tidak dikenali 
- jika ceph sedang berjalan kemudian salah satu node mati, maka masih bisa berjalan
```
- 
