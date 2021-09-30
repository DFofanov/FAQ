# Развертывание кластера Kubernetis на Proxmox 7 (Ubuntu v21).

### 1. Создание контейнера в Proxmox

    1.1. При создании контейнер должен быть не привелигированным
    1.2. Отключить подкачку (Swap=0)
    1.3. Kubernetes не поддерживает раздел ZFS
    1.4. Вставить в конфигурацию контейнера следующие настройки (nano /etc/pve/lxc/100.conf):
    ```bash
        lxc.apparmor.profile: unconfined
        lxc.cap.drop:
        lxc.cgroup2.devices.allow: a
        lxc.mount.auto: proc:rw sys:rw
        lxc.mount.entry: /dev/kmsg dev/kmsg none defaults,bind,create=file
    ```
####  Пример файла конфигурации:
```bash
        arch: amd64
        cores: 4
        hostname: k8s-master
        memory: 2048
        net0: name=eth0,bridge=vmbr0,firewall=1,gw=10.120.10.1,hwaddr=EA:1F:77:22:D4:E3,ip=10.120.10.20/24,type=veth
        onboot: 1
        ostype: ubuntu
        rootfs: local-btrfs:100/vm-100-disk-0.raw,size=50G
        swap: 0
        lxc.apparmor.profile: unconfined
        lxc.cap.drop:
        lxc.cgroup2.devices.allow: a
        lxc.mount.auto: proc:rw sys:rw
        lxc.mount.entry: /dev/kmsg dev/kmsg none defaults,bind,create=file
```    
    1.5. Перенести с родительского хоста из папки boot в папку boot контейнера файл конфигурации 
        Родительский хост /boot/config-5.11.22-4-pve -> Контейнер /boot/

2. Настройка контейнера k8s-master
    2.1. Раскомментировать в файле /etc/sysctl.conf следующие параметры:
        net.ipv4.ip_forward=1
        net.ipv6.conf.all.forwarding=1
        net.ipv4.conf.all.send_redirects=0
    2.2. Выполняем команду
        echo 'L /dev/kmsg - - - - /dev/console' > /etc/tmpfiles.d/kmsg.conf
    2.3. Настраиваем временую зону для синхронизации даты и времени (Europe/Moscow)
        dpkg-reconfigure tzdata
    2.4. Перезагружаемся для выполнения всех параметров
        reboot
    2.5. Устанавливаем Kubernetis
        apt-get install -y apt-transport-https ca-certificates curl
        sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
        echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
        apt-get update
        apt-get install -y kubelet kubeadm kubectl
        apt-mark hold kubelet kubeadm kubectl
    2.6. Установка Docker (https://docs.docker.com/engine/install/ubuntu/)
        apt-get install apt-transport-https ca-certificates curl gnupg lsb-release
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
        echo   "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
            $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
        apt-get update
        apt-get install docker-ce docker-ce-cli containerd.io
    2.6.  Удаляем в файле /etc/hosts ссыли на домен оставлая только имя кластера (k8s-master)
        Должно быть: 10.120.10.20 k8s-master
    2.7. Инициализация контейнера master
        export POD_CIDR="10.244.0.0/16"
        sudo kubeadm init --pod-network-cidr=$POD_CIDR
        mkdir -p $HOME/.kube
        sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config
        export KUBECONFIG=/etc/kubernetes/admin.conf
        kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
    2.8. Устанавливаем 