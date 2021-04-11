---
layout: post
title: "Ansible - Ansible cluster sandbox for testing, scaling and learning your playbooks"
categories: "blog"
header-img: "img/in-post/ansible-sandbox/ansible-docker.png"
tags:
    - Ansible
    - Docker
    - Sandbox
    - Easy to deploy
---

## 前言

前陣子，筆者因為工作需求，使用到了大量的 [Ansible](https://www.ansible.com/) 的腳本去完成一些佈署或是管理的目的，進而也學習到了很多 `Ansible` 的相關使用技巧。熟悉 `Ansible` 的大家一定都曾經為它的語法傷透腦筋，或是煩惱沒有測試環境能夠測試你的腳本。因此，筆者也開發了自己一套 [Ansible-Docker-Sandbox](https://github.com/jimmyw86878/ansible-docker-sandbox) 的工具，能夠讓使用者快速地建立起一套有多個主機的環境，並且能夠很簡單地測試 `Ansible` 的腳本。以下就來介紹一下筆者這套工具，筆者都已將此 project 放至 [github](https://github.com/jimmyw86878) 上囉! 二話不說馬上 clone 下來玩玩吧!

## Getting started

還不熟悉 `Ansible` 的大家可以先去官網或是搜尋網路一些大神的介紹，筆者這邊主要是針對已經對 `Ansible` 有相對使用經驗的使用者來說明。筆者所開發的 `Ansible-Docker-Sandbox` 是一個非常簡單且快速地能夠建構一個叢集，其中一個 host 為 master，在這台 master 上執行 `Ansible` 腳本來管理或是佈署其他 hosts.

### Build docker image

首先，我們要先來準備每台 host 所需要的 docker image。

```
sudo bash ansible_env.sh -- build
```
便會產生出一個 docker image: centos:ansible. 

```
docker images

REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
centos              ansible             6600335a2845        2 hours ago         897MB
```

有興趣的大家可以查看一下筆者所使用的 dockerfile，此 image 是利用 centos7 為基底打造出來的，其中安裝了一些必要的套件，SSH configuration 需要的。當然，若是使用者有額外需求想要安裝一些套件，修改 dockerfile 再重新 build 一次 image 即可喔!

dockerfile:
```
FROM centos:centos7

RUN yum install epel-release -y
RUN yum install ansible -y
RUN yum install openssh-server openssh-clients net-tools -y 
RUN yum install expect -y 

RUN /bin/echo "123456" | passwd --stdin root

RUN ssh-keygen -t rsa -f /etc/ssh/ssh_host_rsa_key \
    && ssh-keygen -t rsa -f /etc/ssh/ssh_host_ecdsa_key \
    && ssh-keygen -t rsa -f /etc/ssh/ssh_host_ed25519_key

RUN /bin/sed -i 's/.*session.*required.*pam_loginuid.so.*/session optional pam_loginuid.so/g' /etc/pam.d/sshd \
    && /bin/sed -i 's/UsePAM yes/UsePAM no/g' /etc/ssh/sshd_config \
    && /bin/sed -i "s/#UsePrivilegeSeparation.*/UsePrivilegeSeparation no/g" /etc/ssh/sshd_config

RUN ssh-keygen -q -t rsa -N '' <<< ""$'\n'"y" 2>&1 >/dev/null
COPY tools/ssh-copy-expect /root/

EXPOSE 22

CMD ["/usr/sbin/sshd","-D"]
```

### Start

就這麼簡單，我們就可以來建構叢集囉!

```
sudo bash ansible_env.sh --host-number 3 -- start
```

`--host-number` 可以用來指定你所需要的主機數量，若不指定此參數，預設就是為 1。亦即只會帶一個 master 主機起來。以上面為例子，便會帶起一個 master 加上二個 slave 主機。

```
docker ps -a

CONTAINER ID        IMAGE               COMMAND               CREATED             STATUS              PORTS               NAMES
9f66b01b8fc9        centos:ansible      "/usr/sbin/sshd -D"   About an hour ago   Up About an hour    22/tcp              slave-2
06ba37bb70de        centos:ansible      "/usr/sbin/sshd -D"   About an hour ago   Up About an hour    22/tcp              slave-1
303fea49a41c        centos:ansible      "/usr/sbin/sshd -D"   About an hour ago   Up About an hour    22/tcp              master
```

筆者這邊也說明一下此架構。一個 master 加上二個 slave 主機，都是利用 container 帶起來，master 對其餘 slave 主機 SSH key 的交換已經都在 start 的 command 中處理好了，所以在 start 的過程中，使用者會看到一些需要輸入的 ssh configuration 問題，這邊都會自動幫使用者填入，不需要額外再輸入任何資訊!

會需要 ssh key 的是因為 `Ansible` 基底就是透過 SSH 去做任何事情，所以這邊也是必須要先 config 好的。更多 detail 可以查看[官網](https://docs.ansible.com/ansible/latest/user_guide/connection_details.html#setting-up-ssh-keys)。

### Run playbook

當環境建構好之後，我們便可以利用 docker exec 進入 master container。

```
docker exec -it master bash
``` 
預設會有幾個 files 在 `/root` 底下 :

```
ls /root/

ansible/  ansiblehost
```

其中 `ansiblehost` 就是 inventory，儲存這個叢集所有的主機資訊。`ansible/` folder 中有一個預設的 `main.yml`，裡面的範例是去每一台主機上面取得 Hostname 並且打印出來。所以使用者可以直接操作:

```
ansible-playbook -i /root/ansiblehost /root/ansible/main.yml
```

便會看到以下結果:
```
PLAY [test area] *******************************************************************************************************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************************************************************************************
ok: [172.17.0.3]
ok: [172.17.0.4]
ok: [localhost]

TASK [get hostname] ****************************************************************************************************************************************************************************************
changed: [172.17.0.3]
changed: [localhost]
changed: [172.17.0.4]

TASK [debug] ***********************************************************************************************************************************************************************************************
ok: [172.17.0.3] => {
    "msg": "06ba37bb70de"
}
ok: [172.17.0.4] => {
    "msg": "9f66b01b8fc9"
}
ok: [localhost] => {
    "msg": "303fea49a41c"
}

PLAY RECAP *************************************************************************************************************************************************************************************************
172.17.0.3                 : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
172.17.0.4                 : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
localhost                  : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

如此一來，使用者就可以盡情在 `ansible/` folder 底下撰寫屬於自己的 playbooks。依照自己需要新增 `role`、`handler` 等等的。當然想要客製化 `/etc/ansible/ansible/cfg` 也可以。

### Teardown the hosts

當使用者確定不使用此叢集時，可以做以下:

```
sudo bash ansible_env.sh -- clear
```

此 command 會移除掉 ansible host 所有的 container.

## 總結

簡單且迅速地建構多台 `Anisble` 主機，使用者便不須再煩惱沒有測試環境能夠 run playbook，而且因為利用 docker 與宿主主機切開，不需要再額外裝其他套件在自己主機上面。只要有 `docker`，開發屬於你自己的 `Ansible` 腳本也可以簡單又快速 (最後這段怎麼描述得好像直銷XD)。總之，若是想要學習 `Ansible` 的各位不妨來試試喔!


### Contact me

有任何疑問或是建議歡迎聯絡 Jimmy!

Gmail: jimmyw86878@gmail.com

