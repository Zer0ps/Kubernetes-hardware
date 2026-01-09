## Часть 1. Подготовка окружения

### Виртуальные машины

Для установки используется **4 виртуальные машины с Debian 12**:

- 1× jumpbox
- 1× control plane
- 2× worker-ноды
    
Роли машин фиксированные, переиспользовать одну VM под несколько ролей не рекомендуется.

---

### Сеть
Используется **две сети**:
#### 1. Host-only сеть (основная)
- Тип: **Host-only**
- Назначение:  
    внутренняя коммуникация между нодами Kubernetes
- IP-адреса: **статические**
- IP прописываются **вручную на каждой машине**
- Именно эта сеть используется:
    - kube-apiserver
    - kubelet
    - etcd
    - pod network
DHCP не используется.

---

#### 2. NAT сеть (вспомогательная)
- Тип: **NAT**
- Назначение:  
    доступ в интернет
- Используется только для:
    - скачивания Kubernetes-бинарников
    - загрузки репозитория
    - установки пакетов ОС

В кластере Kubernetes эта сеть **не участвует**.

---

### Важное замечание
- Kubernetes **никогда не использует NAT-интерфейс**
- Все IP-адреса и маршруты, которые настраиваются в процессе HardWay,  
    относятся **только к Host-only сети**
- NAT нужен исключительно как временный канал к интернету

## Настройка Jumpbox

В этой лабораторной работе вы настроите одну из четырех машин в качестве «джампбокса» (jumpbox). Эта машина будет использоваться для выполнения команд на протяжении всего руководства. Хотя здесь используется выделенная машина для обеспечения единообразия, эти команды можно запускать практически с любой машины, включая ваш личный компьютер на macOS или Linux.

Думайте о джампбоксе как об административной базе, которую вы будете использовать при создании кластера Kubernetes с нуля. Прежде чем начать, нам нужно установить несколько консольных утилит и клонировать git-репозиторий «Kubernetes The Hard Way», который содержит дополнительные файлы конфигурации.

Войдите на джампбокс:

```
ssh root@jumpbox
```
Все команды будут запускаться от имени пользователя **root** для удобства и сокращения количества вводимых команд.
Установка консольных утилит
Теперь, когда вы вошли под root, установите необходимые инструменты:

```
{
  apt-get update
  apt-get -y install wget curl vim openssl git
}
```

Синхронизация GitHub-репозитория

Скачайте копию руководства, содержащую файлы конфигурации и шаблоны:

```
git clone --depth 1 \
  https://github.com/kelseyhightower/kubernetes-the-hard-way.git
```

версия может отличаться
Перейдите в рабочую директорию:

```
cd kubernetes-the-hard-way
```

Это будет ваша рабочая директория до конца обучения. Если запутаетесь, используйте `pwd`, чтобы убедиться, что вы находитесь в `/root/kubernetes-the-hard-way`.

Загрузка бинарных файлов

В этом разделе вы скачаете исполняемые файлы (бинарники) компонентов Kubernetes. Они будут храниться в папке `downloads` на джампбоксе. Это сэкономит интернет-трафик, так как нам не придется скачивать их отдельно для каждой машины кластера.

Список файлов зависит от вашей архитектуры (amd64 или arm64). Скачайте их командой `wget`:

```
wget -q --show-progress \
  --https-only \
  --timestamping \
  -P downloads \
  -i downloads-$(dpkg --print-architecture).txt
```

Загрузка более 500 МБ может занять время. По завершении проверьте файлы командой `ls -oh downloads`.

Распакуйте архивы и распределите их по папкам:

```
{
  ARCH=$(dpkg --print-architecture)
  mkdir -p downloads/{client,cni-plugins,controller,worker}
  tar -xvf downloads/crictl-v1.32.0-linux-${ARCH}.tar.gz -C downloads/worker/
  tar -xvf downloads/containerd-2.1.0-beta.0-linux-${ARCH}.tar.gz --strip-components 1 -C downloads/worker/
  tar -xvf downloads/cni-plugins-linux-${ARCH}-v1.6.2.tgz -C downloads/cni-plugins/
  tar -xvf downloads/etcd-v3.6.0-rc.3-linux-${ARCH}.tar.gz -C downloads/ --strip-components 1 etcd-v3.6.0-rc.3-linux-${ARCH}/etcdctl etcd-v3.6.0-rc.3-linux-${ARCH}/etcd
  mv downloads/{etcdctl,kubectl} downloads/client/
  mv downloads/{etcd,kube-apiserver,kube-controller-manager,kube-scheduler} downloads/controller/
  mv downloads/{kubelet,kube-proxy} downloads/worker/
  mv downloads/runc.${ARCH} downloads/worker/runc
}
rm -rf downloads/*gz
```

Сделайте файлы исполняемыми:

```
chmod +x downloads/{client,cni-plugins,controller,worker}/*
```

Установка kubectl

Установите `kubectl` — официальный клиент Kubernetes — на джампбокс. Он понадобится для взаимодействия с кластером.

Скопируйте файл в системную директорию:

```
{
  cp downloads/client/kubectl /usr/local/bin/
}
```

Проверьте установку:

```
kubectl version --client
```

На этом этапе джампбокс полностью настроен и готов к работе.
Подготовка вычислительных ресурсов

Для работы Kubernetes требуется набор машин: для панели управления (control plane) и для рабочих узлов (worker nodes), где запускаются контейнеры. В этой лабораторной работе вы подготовите машины, необходимые для настройки кластера.

База данных машин

В этом руководстве используется текстовый файл в качестве «базы данных» для хранения атрибутов машин. Формат каждой строки в файле следующий:  
`IP-АДРЕС FQDN HOSTNAME POD_SUBNET`

- **IPV4_ADDRESS**: IP-адрес машины.
- **FQDN**: Полное доменное имя.
- **HOSTNAME**: Короткое имя хоста.
- **POD_SUBNET**: Уникальный диапазон IP-адресов (подсеть), выделяемый каждой машине для работы подов.

**Пример файла `machines.txt`:**  
(IP-адреса здесь скрыты, в вашем случае они должны быть доступны друг другу и с "джампбокса" — управляющей машины).



```
XXX.XXX.XXX.XXX server.kubernetes.local server
XXX.XXX.XXX.XXX node-0.kubernetes.local node-0 10.200.0.0/24
XXX.XXX.XXX.XXX node-1.kubernetes.local node-1 10.200.1.0/24
```

**Ваше задание:** Создайте файл `machines.txt` со своими данными для трех машин.

---

Настройка SSH-доступа

SSH используется для настройки всех машин. Убедитесь, что у вас есть доступ по SSH от имени пользователя **root** к каждой машине.

Включение доступа для root через SSH

По умолчанию в Debian доступ для root через SSH отключен. Чтобы его включить:

1. Зайдите на каждую машину под обычным пользователем и переключитесь на root:  
    `su - root`
2. Одной командой разрешите вход root в конфиге SSH:
    
    
    
    ```
    sed -i 's/^#*PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
    ```
    
    
    
3. Перезапустите службу SSH:  
    `systemctl restart sshd`

Генерация и распределение SSH-ключей

Выполните эти команды **на джампбоксе** (вашем основном ПК или управляющем узле):

1. Создайте ключ (нажимайте Enter на все вопросы):  
    `ssh-keygen`
2. Скопируйте публичный ключ на все машины из списка:
    
    
    
    ```
  ```
while read IP FQDN HOST SUBNET; do
  if [ -n "$IP" ]; then
    ssh-copy-id -o StrictHostKeyChecking=no root@${IP}
  fi
done < machines.txt
```
    ```
    
    
    
3. Проверьте доступ (команда должна вернуть имена хостов без ввода пароля):
    
    
    
    ```
    while read IP FQDN HOST SUBNET; do
      ssh -n root@${IP} hostname
    done < machines.txt
    ```
    
    
    

---

Имена хостов (Hostnames)

Теперь нужно присвоить машинам их имена. Это важно, так как Kubernetes использует имена хостов при регистрации узлов и в API.

Выполните **на джампбоксе**:

1. Установите имена на удаленных машинах:
    
    
    
    ```
    while read IP FQDN HOST SUBNET; do
        CMD="sed -i 's/^127.0.1.1.*/127.0.1.1\t${FQDN} ${HOST}/' /etc/hosts"
        ssh -n root@${IP} "$CMD"
        ssh -n root@${IP} hostnamectl set-hostname ${HOST}
        ssh -n root@${IP} systemctl restart systemd-hostnamed
    done < machines.txt
    ```
    
    
    
2. Проверьте результат:  
    `ssh -n root@${IP} hostname --fqdn`

---

Таблица поиска хостов (/etc/hosts)

Чтобы машины могли обращаться друг к другу по именам (server, node-0, node-1), а не по IP, обновим файл `/etc/hosts`.

1. Создайте временный файл с записями:
    
    
    
    ```
    echo "" > hosts
    echo "# Kubernetes The Hard Way" >> hosts
    while read IP FQDN HOST SUBNET; do
        ENTRY="${IP} ${FQDN} ${HOST}"
        echo $ENTRY >> hosts
    done < machines.txt
    ```
    
    
    
2. Добавьте эти записи в `/etc/hosts` **на локальном джампбоксе**:  
    `cat hosts >> /etc/hosts`
3. Скопируйте эти записи на **все удаленные машины**:
    
    
    
    ```
    while read IP FQDN HOST SUBNET; do
      scp hosts root@${HOST}:~/
      ssh -n root@${HOST} "cat hosts >> /etc/hosts"
    done < machines.txt
    ```
    
    
    

**Итог:** Теперь вы можете подключаться к любой машине просто по её имени: `ssh root@node-0`.
Создание конфигурационных файлов Kubernetes для аутентификации

В этой лабораторной работе вы создадите файлы конфигурации клиентов Kubernetes, которые обычно называют **kubeconfigs**. Они позволяют клиентам Kubernetes подключаться к API-серверам и проходить проверку подлинности.

Конфигурации аутентификации клиентов

В этом разделе вы создадите kubeconfig-файлы для компонентов kubelet, kube-proxy, kube-controller-manager, kube-scheduler и для администратора (admin).

**Важно:** Все команды должны выполняться в той же директории на **jumpbox**, где вы генерировали TLS-сертификаты.

Kubeconfig для Kubelet

При генерации файлов для Kubelet необходимо использовать сертификат, имя которого совпадает с именем узла (node name). Это гарантирует правильную авторизацию через Node Authorizer.

**Создание kubeconfig для рабочих узлов (node-0 и node-1):**  
_(Если вы переименовали их в worker01/02, не забудьте изменить имена в цикле)_

```
for host in node-0 node-1; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=${host}.kubeconfig

  kubectl config set-credentials system:node:${host} \
    --client-certificate=${host}.crt \
    --client-key=${host}.key \
    --embed-certs=true \
    --kubeconfig=${host}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${host} \
    --kubeconfig=${host}.kubeconfig

  kubectl config use-context default \
    --kubeconfig=${host}.kubeconfig
done
```

Kubeconfig для kube-proxy

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.crt \
    --client-key=kube-proxy.key \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default \
    --kubeconfig=kube-proxy.kubeconfig
}
```

Kubeconfig для kube-controller-manager

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.crt \
    --client-key=kube-controller-manager.key \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default \
    --kubeconfig=kube-controller-manager.kubeconfig
}
```

Kubeconfig для kube-scheduler

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.crt \
    --client-key=kube-scheduler.key \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default \
    --kubeconfig=kube-scheduler.kubeconfig
}
```

Kubeconfig для администратора (admin)

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.crt \
    --client-key=admin.key \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default \
    --kubeconfig=admin.kubeconfig
}
```

---

Распределение конфигурационных файлов

**Копирование на рабочие узлы (node-0 и node-1):**

```
for host in node-0 node-1; do
  ssh root@${host} "mkdir -p /var/lib/{kube-proxy,kubelet}"

  scp kube-proxy.kubeconfig \
    root@${host}:/var/lib/kube-proxy/kubeconfig

  scp ${host}.kubeconfig \
    root@${host}:/var/lib/kubelet/kubeconfig
done
```

**Копирование на управляющий сервер (server):**

```
scp admin.kubeconfig \
  kube-controller-manager.kubeconfig \
  kube-scheduler.kubeconfig \
  root@server:~/
```
Создание конфигурации и ключа шифрования данных

Kubernetes хранит различные данные, включая состояние кластера, конфигурации приложений и секреты. Kubernetes поддерживает возможность шифрования данных кластера «в покое» (at rest), когда они записаны на диск.

В этой лабораторной работе вы создадите ключ шифрования и конфигурационный файл, подходящий для шифрования секретов (Secrets) Kubernetes.

Ключ шифрования

Сгенерируйте случайный ключ шифрования:

```
export ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

Файл конфигурации шифрования

Создайте файл конфигурации `encryption-config.yaml`:

```
envsubst < configs/encryption-config.yaml \
  > encryption-config.yaml
```

_(Команда `envsubst` подставит значение вашей переменной `ENCRYPTION_KEY` в шаблон из папки `configs`)_.

Распределение файла

Скопируйте файл конфигурации шифрования на управляющий сервер (controller):

```
scp encryption-config.yaml root@server:~/
```

Компоненты Kubernetes не хранят состояние внутри себя (stateless) и используют **etcd** для хранения состояния всего кластера. В этой лабораторной работе вы развернете кластер etcd, состоящий из одного узла.

Предварительные условия

Скопируйте бинарные файлы etcd и юнит-файлы systemd на машину **server**:

```
scp \
  downloads/controller/etcd \
  downloads/client/etcdctl \
  units/etcd.service \
  root@server:~/
```

**Важно:** Все последующие команды в этом разделе должны выполняться непосредственно **на машине server**. Зайдите на неё по SSH:

```
ssh root@server
```

Развертывание кластера etcd

**Установка бинарных файлов etcd**  
Переместите сервер etcd и утилиту управления `etcdctl` в системную директорию:

```
{
  mv etcd etcdctl /usr/local/bin/
}
```

**Настройка сервера etcd**  
Создайте необходимые директории и скопируйте сертификаты:

```
{
  mkdir -p /etc/etcd /var/lib/etcd
  chmod 700 /var/lib/etcd
  cp ca.crt kube-api-server.key kube-api-server.crt \
    /etc/etcd/
}
```

Каждый участник кластера etcd должен иметь уникальное имя. Имя etcd будет соответствовать имени хоста текущей машины.

Установите юнит-файл `etcd.service`:

```
mv etcd.service /etc/systemd/system/
```

**Запуск сервера etcd**  
Перезагрузите конфигурацию systemd, включите автозагрузку и запустите службу:

```
{
  systemctl daemon-reload
  systemctl enable etcd
  systemctl start etcd
}
```

Проверка

Выведите список участников кластера etcd:

```
etcdctl member list
```

**Пример результата:**  
`6702b0a34e2cfd39, started, controller, http://127.0.0.1:2380, http://127.0.0.1:2379, false`
Развертывание панели управления Kubernetes

В этой лабораторной работе вы развернете панель управления Kubernetes. На машину **server** будут установлены следующие компоненты: API-сервер, Планировщик (Scheduler) и Менеджер контроллеров (Controller Manager).

Предварительные условия

Подключитесь к **jumpbox** и скопируйте бинарные файлы и юнит-файлы на машину **server**:

```
scp \
  downloads/controller/kube-apiserver \
  downloads/controller/kube-controller-manager \
  downloads/controller/kube-scheduler \
  downloads/client/kubectl \
  units/kube-apiserver.service \
  units/kube-controller-manager.service \
  units/kube-scheduler.service \
  configs/kube-scheduler.yaml \
  configs/kube-apiserver-to-kubelet.yaml \
  root@server:~/
```

**Важно:** Все последующие команды выполняются **на машине server**. Зайдите на неё по SSH:

```
ssh root@server
```

Развертывание панели управления

Создайте директорию для конфигураций:

```
mkdir -p /etc/kubernetes/config
```

**Установка бинарных файлов:**

```
{
  mv kube-apiserver \
    kube-controller-manager \
    kube-scheduler kubectl \
    /usr/local/bin/
}
```

**Настройка API-сервера:**

```
{
  mkdir -p /var/lib/kubernetes/

  mv ca.crt ca.key \
    kube-api-server.key kube-api-server.crt \
    service-accounts.key service-accounts.crt \
    encryption-config.yaml \
    /var/lib/kubernetes/
}
```

Установите юнит-файл службы:

```
mv kube-apiserver.service /etc/systemd/system/kube-apiserver.service
```

**Настройка Менеджера контроллеров:**  
Переместите kubeconfig на место и установите юнит-файл:

```
mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
mv kube-controller-manager.service /etc/systemd/system/
```

**Настройка Планировщика:**  
Переместите kubeconfig и файл конфигурации YAML, затем установите юнит-файл:

```
mv kube-scheduler.kubeconfig /var/lib/kubernetes/
mv kube-scheduler.yaml /etc/kubernetes/config/
mv kube-scheduler.service /etc/systemd/system/
```

**Запуск служб управления:**

```
{
  systemctl daemon-reload
  systemctl enable kube-apiserver kube-controller-manager kube-scheduler
  systemctl start kube-apiserver kube-controller-manager kube-scheduler
}
```

Подождите около 10 секунд, пока API-сервер инициализируется. Проверить статус можно командой: `systemctl is-active kube-apiserver`. Логи доступны через `journalctl -u kube-apiserver`.

Проверка

Убедитесь, что компоненты работают, используя `kubectl`:

```
kubectl cluster-info --kubeconfig admin.kubeconfig
```

---

RBAC для авторизации Kubelet

В этом разделе вы настроите права RBAC, чтобы API-сервер мог обращаться к API Kubelet на рабочих узлах (для получения метрик, логов и запуска команд в подах).

Выполните на машине **server**:

```
kubectl apply -f kube-apiserver-to-kubelet.yaml --kubeconfig admin.kubeconfig
```

---

Финальная проверка (с Jumpbox)

Вернитесь на **jumpbox** и проверьте доступность API по сети, запросив версию Kubernetes:

```
curl --cacert ca.crt https://server.kubernetes.local:6443/version
```

Вы должны получить JSON-ответ с информацией о версии (например, `1.32.3`).
Развертывание рабочих узлов Kubernetes

В этой лабораторной работе вы настроите два рабочих узла. Будут установлены следующие компоненты: `runc`, плагины сетевого интерфейса контейнеров (CNI), `containerd`, `kubelet` и `kube-proxy`.

Предварительные условия

Команды в этом подразделе выполняются на **jumpbox**.

**1. Подготовка и копирование конфигураций сетей и kubelet:**  
Для каждого узла подставляется его уникальная подсеть подов из файла `machines.txt`.

```
for HOST in node-0 node-1; do
  SUBNET=$(grep ${HOST} machines.txt | cut -d " " -f 4)
  sed "s|SUBNET|$SUBNET|g" configs/10-bridge.conf > 10-bridge.conf
  sed "s|SUBNET|$SUBNET|g" configs/kubelet-config.yaml > kubelet-config.yaml
  scp 10-bridge.conf kubelet-config.yaml root@${HOST}:~/
done
```

**2. Копирование бинарных файлов и юнитов systemd:**

```
for HOST in node-0 node-1; do
  scp downloads/worker/* downloads/client/kubectl \
    configs/99-loopback.conf configs/containerd-config.toml \
    configs/kube-proxy-config.yaml units/containerd.service \
    units/kubelet.service units/kube-proxy.service root@${HOST}:~/
done
```

**3. Копирование CNI-плагинов:**

```
for HOST in node-0 node-1; do
  scp downloads/cni-plugins/* root@${HOST}:~/cni-plugins/
done
```

---

Настройка рабочего узла (выполнять на node-0 и node-1)

Зайдите на каждый узел по SSH (например, `ssh root@node-0`) и выполните следующие действия:

**1. Установка зависимостей ОС:**

```
{
  apt-get update
  apt-get -y install socat conntrack ipset kmod
}
```

_Утилита `socat` необходима для работы команды `kubectl port-forward`._

**2. Отключение файла подкачки (Swap):**  
Kubernetes требует отключения swap для корректного управления ресурсами.
(Начиная с Kubernetes 1.28 swap может использоваться в ограниченном режиме,  
но в HardWay он отключается для предсказуемости поведения.)


- Проверить: `swapon --show` (если пусто — отключен).
- Выключить немедленно: `swapoff -a`.
- _Примечание: Чтобы swap не включился после перезагрузки, закомментируйте его в `/etc/fstab`._

**3. Создание директорий и установка бинарных файлов:**

```
mkdir -p /etc/cni/net.d /opt/cni/bin /var/lib/kubelet /var/lib/kube-proxy /var/lib/kubernetes /var/run/kubernetes

{
  mv crictl kube-proxy kubelet runc /usr/local/bin/
  mv containerd containerd-shim-runc-v2 containerd-stress /bin/
  mv cni-plugins/* /opt/cni/bin/
}
```

**4. Настройка сети CNI:**

```
mv 10-bridge.conf 99-loopback.conf /etc/cni/net.d/

# Включение фильтрации трафика через мост (br-netfilter)
{
  modprobe br-netfilter
  echo "br-netfilter" >> /etc/modules-load.d/modules.conf
}
{
  echo "net.bridge.bridge-nf-call-iptables = 1" >> /etc/sysctl.d/kubernetes.conf
  echo "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.d/kubernetes.conf
  sysctl -p /etc/sysctl.d/kubernetes.conf
}
```

**5. Настройка containerd, Kubelet и Kube-proxy:**  
Переместите конфигурационные файлы и юнит-файлы на свои места:

```
# containerd
mkdir -p /etc/containerd/
mv containerd-config.toml /etc/containerd/config.toml
mv containerd.service /etc/systemd/system/

# Kubelet
mv kubelet-config.yaml /var/lib/kubelet/
mv kubelet.service /etc/systemd/system/

# Kube-proxy
mv kube-proxy-config.yaml /var/lib/kube-proxy/
mv kube-proxy.service /etc/systemd/system/
```

**6. Запуск служб:**

```
{
  systemctl daemon-reload
  systemctl enable containerd kubelet kube-proxy
  systemctl start containerd kubelet kube-proxy
}
```

Проверьте статус: `systemctl is-active kubelet`.

---

Проверка (выполнять на Jumpbox)

Убедитесь, что узлы успешно зарегистрировались в кластере и перешли в состояние **Ready**:

```
ssh root@server "kubectl get nodes --kubeconfig admin.kubeconfig"
```

**Ожидаемый результат  :**



```
NAME     STATUS   ROLES    AGE    VERSION
node-0   Ready    <none>   1m     v1.32.3
node-1   Ready    <none>   10s    v1.32.3
```
Настройка kubectl для удаленного доступа

В этой лабораторной работе вы создадите файл `kubeconfig` для утилиты командной строки `kubectl` на основе учетных данных пользователя **admin**.

Все команды в этом разделе должны выполняться на **jumpbox**.

Конфигурационный файл администратора Kubernetes

Для работы каждого `kubeconfig` требуется адрес API-сервера Kubernetes для подключения.

Убедитесь, что вы можете «пинговать» адрес `server.kubernetes.local` (он должен быть доступен благодаря записи в `/etc/hosts`, сделанной в предыдущих лабораторных работах).

Проверьте доступность API-сервера:

```
curl --cacert ca.crt \
  https://server.kubernetes.local:6443/version
```

_(Вы должны увидеть JSON-ответ с информацией о версии 1.32.3)._

**Создайте файл kubeconfig для аутентификации от имени администратора:**

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443

  kubectl config set-credentials admin \
    --client-certificate=admin.crt \
    --client-key=admin.key

  kubectl config set-context kubernetes-the-hard-way \
    --cluster=kubernetes-the-hard-way \
    --user=admin

  kubectl config use-context kubernetes-the-hard-way
}
```

Результатом выполнения этих команд станет создание файла конфигурации в стандартном месте: `~/.kube/config`. Это позволит вам использовать команду `kubectl` без необходимости каждый раз указывать путь к файлу конфигурации.

Проверка

Проверьте версию удаленного кластера Kubernetes:

```
kubectl version
```

_(Должны отобразиться версии и клиента, и сервера — v1.32.3)._

Выведите список узлов в удаленном кластере:

```
kubectl get nodes
```

**Результат должен быть примерно таким:**



```
NAME     STATUS   ROLES    AGE    VERSION
node-0   Ready    <none>   10m    v1.32.3
node-1   Ready    <none>   10m    v1.32.3
```
Настройка маршрутов сети Подов

Поды, назначенные на узел, получают IP-адрес из диапазона Pod CIDR этого узла. На данный момент поды не могут связываться с подами на других узлах, так как отсутствуют необходимые сетевые маршруты.

В этой лабораторной работе вы создадите маршрут для каждого рабочего узла, который связывает диапазон Pod CIDR узла с его внутренним IP-адресом.

_Примечание: существуют и другие способы реализации сетевой модели Kubernetes (например, использование CNI-плагинов вроде Calico или Flannel в режиме VXLAN)._

Таблица маршрутизации

В этом разделе вы соберете информацию, необходимую для создания маршрутов в вашей сети.

**1. Подготовка переменных (выполняется на jumpbox):**  
Сначала извлечем IP-адреса и подсети из файла `machines.txt`:

```
{
  SERVER_IP=$(grep server machines.txt | cut -d " " -f 1)
  NODE_0_IP=$(grep node-0 machines.txt | cut -d " " -f 1)
  NODE_0_SUBNET=$(grep node-0 machines.txt | cut -d " " -f 4)
  NODE_1_IP=$(grep node-1 machines.txt | cut -d " " -f 1)
  NODE_1_SUBNET=$(grep node-1 machines.txt | cut -d " " -f 4)
}
```

**2. Создание маршрутов на каждой машине:**  
Мы добавим статические маршруты, чтобы каждая машина знала, куда отправлять трафик, предназначенный для подсетей подов других узлов.

На сервере управления (server):

```
ssh root@server <<EOF
  ip route add ${NODE_0_SUBNET} via ${NODE_0_IP}
  ip route add ${NODE_1_SUBNET} via ${NODE_1_IP}
EOF
```

На первом рабочем узле (node-0):

```
ssh root@node-0 <<EOF
  ip route add ${NODE_1_SUBNET} via ${NODE_1_IP}
EOF
```

На втором рабочем узле (node-1):

```
ssh root@node-1 <<EOF
  ip route add ${NODE_0_SUBNET} via ${NODE_0_IP}
EOF
```

Проверка

Проверьте таблицы маршрутизации на каждой машине. Команда `ip route` должна показать новые маршруты к подсетям `10.200.0.0/24` и `10.200.1.0/24`.

**Пример для сервера:**

```
ssh root@server ip route
```

Вы должны увидеть строки вида:  
`10.200.0.0/24 via [IP_NODE_0] dev ens160`  
`10.200.1.0/24 via [IP_NODE_1] dev ens160`

Аналогично проверьте на `node-0` и `node-1`. Теперь поды смогут «общаться» напрямую между узлами.
Дымовой тест (Smoke Test)

В этой лабораторной работе вы выполните серию задач, чтобы убедиться, что ваш кластер Kubernetes функционирует правильно.

Шифрование данных

В этом разделе вы проверите возможность шифрования секретных данных «в покое» (at rest).

**1. Создайте обычный секрет:**

```
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```

**2. Выведите дамп секрета, хранящегося в etcd (выполняется через сервер):**

```
ssh root@server \
    'etcdctl get /registry/secrets/default/kubernetes-the-hard-way | hexdump -C'
```

В дампе вы должны увидеть префикс `k8s:enc:aescbc:v1:key1`. Это подтверждает, что данные зашифрованы провайдером `aescbc` с использованием ключа `key1`.

---

Развертывания (Deployments)

Проверка создания и управления объектами Deployment.

**1. Создайте развертывание для веб-сервера nginx:**

```
kubectl create deployment nginx --image=nginx:latest
```

**2. Проверьте статус пода:**

```
kubectl get pods -l app=nginx
```

_(Статус должен быть **Running**)._

---

Проброс портов (Port Forwarding)

Проверка возможности удаленного доступа к приложениям через `port-forward`.

**1. Получите полное имя пода nginx:**

```
POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath="{.items[0].metadata.name}")
```

**2. Пробросьте порт 8080 вашего локального ПК на порт 80 пода:**

```
kubectl port-forward $POD_NAME 8080:80
```

**3. В новом терминале выполните HTTP-запрос:**

```
curl --head http://127.0.0.1:8080
```

_(Вы должны получить ответ `HTTP/1.1 200 OK`)._ После проверки вернитесь в первый терминал и нажмите `Ctrl+C`, чтобы остановить проброс.

---

Логи

Проверка возможности получения логов контейнера.

```
kubectl logs $POD_NAME
```

---

Выполнение команд (Exec)

Проверка возможности запускать команды внутри контейнера.

```
kubectl exec -ti $POD_NAME -- nginx -v
```

_(Должна отобразиться версия nginx)._

---

Сервисы (Services)

Проверка возможности предоставления доступа к приложению через сервис типа **NodePort**.

**1. Опубликуйте deployment через NodePort:**

```
kubectl expose deployment nginx --port 80 --type NodePort
```

_(Тип LoadBalancer не используется, так как у нас нет интеграции с облачным провайдером)._

**2. Получите номер назначенного порта:**

```
NODE_PORT=$(kubectl get svc nginx --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```

**3. Узнайте имя узла, на котором запущен под, и выполните запрос:**

```
NODE_NAME=$(kubectl get pods -l app=nginx -o jsonpath="{.items[0].spec.nodeName}")

curl -I http://${NODE_NAME}:${NODE_PORT}
```

_(Вы получите HTTP-ответ 200 OK через сетевой адрес узла и NodePort)._