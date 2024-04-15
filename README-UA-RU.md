
# Инструкция по установки k8s кластера на 2 master ноды + HAproxy на сервер Proxmox VM
  
## Введение

![enter image description here](k8s-home.png)

Данную инструкцию было решено написать в качестве получения опыта работы с kubernetes и ansible. Большинство найденных инструкций либо уже неактуальны либо содержат ошибки. 

## Информация о виртуальных машинах

 - 192.168.1.233 - proxy
 - 192.168.1.230 - k8s-m1
 - 192.168.1.231 - k8s-m2
## Подготовка шаблона для виртуальных машин

Установка виртуальных машин осуществляется клонированием из готового шаблона. Вам необходимо заранее подготовить виртуальную машину и конвертировать ее в шаблон. Подробную инструкцию по созданию шаблона виртуальной машины вы найдете [тут](https://pve.proxmox.com/wiki/VM_Templates_and_Clones).

## Установка виртуальных машин

Для работы плейбука на сервере с установленным Proxmox VM вам необходимо установить [proxmoxer](https://pypi.org/project/proxmoxer/)

Для установки выполняем команду

    apt install python3-proxmoxer

## Структруа файла inventory
[pvenodes] - IP proxmox server
[k8s_master] - IP master VM
[k8s_master2] - IP master VM 2
[proxy] - IP HAproxy VM

## Структруа файла vm_vars.yml

- proxmox_node - название proxmox ноды
- proxmox_user - root пользователь proxmox ноды - пример root@pam
- proxmox_password - root пароль от proxmox ноды
- proxmox_bridge - имя бридж интерфейса - пример vmbr0
- proxmox_host - ip адрес proxmox ноды
- template_id - id шаблона для клонирования виртуальных машин
- cloudinit_user - имя пользователя которое будет создано на виртуальной машине
- cloudinit_password - пароль который будет создан пользователю
- sshkey - ssh ключ который будет добавлен при клонировании виртуальной машины
- vms - содержит id, name и ip для создания виртуальных серверов

## Создание виртуальных машин

Чтобы выполнить установку виртуальных машин необходимо запустить плейбук `create_vms.yml` с указанием файла инвентаря inventory, а также указать имя root пользователя от сервера Proxmox VM

    ansible-playbook create_vms.yml -i inventory --user=root

После запуска плейбука будут созданы виртуальные машины с настроенной сетью и заданным именем пользователя и паролем.

Для удаления всех созданных виртуальных машин используйте плейбук `delete_vms.yml`

    ansible-playbook delete_vms.yml -i inventory --user=root

## Настройка HAproxy

Установка и настройка HAproxy осуществляется с помощью плейбука `haproxy.yml`

    ansible-playbook haproxy.yml -i inventory --user=k8suser

### Ручная установка  haproxy на виртуальную машину proxy

    apt install -y haproxy

Откроем файл конфигурации haproxy `/etc/haproxy/haproxy.cfg` и добавим в конец файла следуюший текст

  

    frontend kubernetes-frontend
    	bind *:6443
    	mode tcp
    	option tcplog
    	default_backend kubernetes-backend
    
    backend kubernetes-backend
    	mode tcp
    	option tcp-check
    	balance roundrobin
    	server k8s-1 192.168.1.230:6443 check fall 3 rise 2
    	server k8s-2 192.168.1.231:6443 check fall 3 rise 2

Запустим haproxy выполнив следующие 3 команды

    systemctl start haproxy
    systemctl enable haproxy
    systemctl restart haproxy

## Настройка k8s-m1 и k8s-m2

  

На каждой из нод нам необходимо произвести следующие пакеты:

 - apt-transport-https
 - ca-certificates 
 - curl 
 - gpg
 - kubelet
 - kubeadm
 - kubectl
 - containerd

И настроить следующие параметры:
 - overlay 
 - br_netfilter 
 - net.bridge.bridge-nf-call-iptables = 1
 - net.bridge.bridge-nf-call-ip6tables = 1 
 - net.ipv4.ip_forward = 1
 - 
Все все необходимые команды для выполнения находятся в файле `kubectl_config.sh`
Для настройки всех нод выполним плейбук `install_kub.yml`

    ansible-playbook install_kub.yml -i inventory --user=k8suser

## Настройка кластера

Подключаеся к первой мастер ноде `k8s-m1` и создаем кластер, в `--control-plane-endpoint` необходимо указать ip адрес сервера proxy

    sudo kubeadm init --control-plane-endpoint=192.168.1.243:6443 --upload-certs
    
На второй и последующих нодах выполним команду добавления control-plane node

    You can now join any number of the control-plane node running the following command on each as root:
    
    kubeadm join 192.168.1.243:6443 --token 87oixr.l8kinulpdnnsc1z2 \
    --discovery-token-ca-cert-hash sha256:6ddac82ec84248505e4314ce95c547e9d41cd403a8d3841ed2ae0e8b58a4ea76 \
    --control-plane --certificate-key 2f6df124ab30e6780770ae70ec42a588bdc5bf84494f928aa77fe1f8a1128147

  

## настройка kubectl


Для настройки kubectl для получения данных от нашем кластере выполним следуюшую команду на одной из серверов кластера

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

  

Если же вы хотите получать данные о кластере на локальном компьютере то вам необходимо скопировать файл `/etc/kubernetes/admin.conf` на локальный компьютер в папку `$HOME/.kube/config`

## Проверка состояния кластера

Проверим состояние кластера выполнив команду

    kubectl get nodes

На этом настройка кластера k8s из 2х мастер нод завершена.

Для того чтобы сделать наши ноды как master так и worker выполним следующую команду

    kubectl get no -o name | xargs -n1 kubectl patch --type=json -p '[{
    'op': 'remove',
    'path': '/spec/taints'
    }]'
    kubectl get no -o name | xargs -n1 kubectl patch -p '{
    "metadata": {
    "labels": {
    "node-role.kubernetes.io/control-plane": "true",
    "node-role.kubernetes.io/worker": "true"
    }
    }
    }'

## Тест работы кластера

В качестве базового теста кластера и понимания работы проброса портов для доступа к приложениями, установим web сервер nginx.

На первой ноде установим сетевой плагин calico

    kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

Установим nginx

    kubectl create deployment nginx-app --image=nginx

Теперь если мы выполним команду `kubectl get pods` то увидим, что у нас развернутый pod nginx-app и он запущен и работает.

    k8suser@k8s-1:~$ kubectl get pods
    NAME 						READY   STATUS 	    RESTARTS 	AGE
    nginx-app-5777b5f95-gsbdd 	1/1 	Running 	0 			2m27s

В данный момент pod доступен только во внутренней сети кластера, чтобы пустить pod наружу и сделать его доступным внес публичной сети кластера, выполним следующую команду

    kubectl expose deployment nginx-app --type=NodePort --port=80

Чтобы просмотреть все открытые порты выполним команду `kubectl get svc`

    k8suser@k8s-1:~$ kubectl get svc
    NAME TYPE  CLUSTER-IP EXTERNAL-IP    PORT(S) AGE
    kubernetes ClusterIP  10.96.0.1      <none>  443/TCP 15m
    nginx-app  NodePort   10.104.151.153 <none>  80:30809/TCP 3s

Порт который был присвоен нашему pod - 30809
Проверим, доступен ли pod по внутреннему адресу и порту

    curl http://10.104.151.153:80
    <!DOCTYPE  html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
    html { color-scheme: light  dark; }
    body { width: 35em; margin: 0  auto;
    font-family: Tahoma, Verdana, Arial, sans-serif; }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working. Further configuration is required.</p>
    <p>For online documentation and support please refer to
    <a  href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a  href="http://nginx.com/">nginx.com</a>.</p>
    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>


Для того чтобы мы смогли получить доступ к nginx, нам необходимо внести правила в haproxy.
Подключимся к серверу proxy
В файл конфигурации `/etc/haproxy/haproxy.cfg` добавим следующее

    frontend nginx
    	bind *:80
    	mode tcp
    	option tcplog
    	default_backend nginx_nodes
    
    backend nginx_nodes
    	mode tcp
    	option tcp-check
    	balance roundrobin
    	server k8s-1 192.168.1.240:30809 check
    	server k8s-2 192.168.1.241:30809 check

Замените порт 30809 на тот который который будет выдан вашему pod и ip адреса серверов.
Для того чтобы новые правила вступили в силу необходимо перезапустить сервис haproxy

    systemctl restart haproxy

Перейдем в браузере по адресу http://192.168.1.243:80 заменив 192.168.1.243 на адрес вашего proxy сервера и мы увидем стандартную страницу nginx.

На этом, установка кластера k8s для домашнего использования завершена. 

## Дополнительно
Вы можете отредактировать плейбук под свои нужны, добавив в него большее количество виртуальных серверов.