# NFS
## Задание выполнялось с использованием VirtualBox 6.1.40. ВМ собирались с помощью vagrant на базе box-образа generic/debian10 (debian 10.13)
## В результате:
- **Vagrantfile** - основной конфигурационный файл стенда. Содержит описание 2 ВМ - `nfssrv` (сервер) и `nfscl` (клиент)
- **deployNFSSrv.yml** и **deployNFSCl.yml**  - файлы для **ansible**. На их основе соответственно настраиваються сервер и клиент
- **nfsSrvFW.rule** и **nfsClFW.rule**  - файлы собдержащие правила для **iptables** (см. замечание), применяются на этапе настройки ВМ
### Замечание:
1. В качестве файервола использовалася "голый" **iptables**. Вместе с тем, по умолчанию службы **rpc.mountd** и **rpc.statd** при запуске выбирают случайный порт для приема входящих подключений. Чтобы можно было ограничить доступ к данным службам с помощью файервола, в файлы **/etc/default/nfs-common**, **/etc/default/nfs-kernel-server** и **/etc/sysctl.d/nfs-nlm-ports.conf** вносятся определенные значения, в результате службы исопльзуют порты (tcp/udp) из диапазона 33330 - 33334
2. Не реализовано восстановление правил **iptables** после перезагрузки ВМ

## Основная часть (кратко). 
- Устанавливаются необходимые пакеты. На сервере - **nfs-kernel-server**, на клиенте - **nfs-common**
- На сервере создаем папку **/home/exportdir**, в ней папку **__upload__**, раздаем права
- На сервере в **/etc/exports** прописываем экспорт папки **/home/exportdir**
- На клиенте в **/etc/fstab** прописываем монтирование папки с сервера в локальную папку **/mnt**
- Всё вроде работает ...

Картинка с сервера
!["Видим на сервере"](https://github.com/mus-cat/otus-study-m1l6/blob/main/01.NFSServerState.png)

Картинка с клиента
!["Видим на клиенте"](https://github.com/mus-cat/otus-study-m1l6/blob/main/02.NFSClientState.png)

## Прикручиваем Kerberos к NFS
### В данном примере при подключении происходит аутентификация машин, а не конкретных пользователей
- Добавляем в /etc/hosts ("короткое" и "полное" имена следуют именно в такой последовательности, т.к. для создания spn мы использовали имена вида nfssrv.example.org)строки :
```
192.168.56.10 nfssrv.nfs.example.org nfssrv 
192.168.56.11 nfscl.nfs.example.org nfscl 
```

- На клиенте ставим пакет krb5-user (зависимости удовлетворяются автоматически)
```
apt install krb5-user
```

- На сервере ставим пакеты krb5-kdc и krb5-admin-server (зависимости удовлетворяются автоматически)
```
apt install krb5-kdc krb5-admin-server
```

- Правим на обоих машиных файл **/etc/krb5**  
!["Содержимое krb5.conf"](https://github.com/mus-cat/otus-study-m1l6/blob/main/03.mkKrbConfig.png)

- На сервере создаем файл **/etc/krb5kdc/kadm5.acl**, а также БД для kerberos. Запускаем необходимые службы.
```
touch /etc/krb5kdc/kadm5.acl
kdb5_util -r NFS.EXAMPLE.ORG create -s
systemctl start krb5-admin-server.service
systemctl start krb5-kdc.service
```

- На сервере запускаем **kadmin.local** и добавляем учётку для администратора, spn для сервера и клиента, затем выгружем ключ для сервера в файл **/etc/krb5.keytab**
```
ank admin
ank -randley nfs/nfssrv.example.org
ank -randley nfs/nfscl.example.org
ktadd nfs/nfssrv.example.org 
```
![""](https://github.com/mus-cat/otus-study-m1l6/blob/main/04.mkNewDB.png)

- На сервере правим  файл **/etc/exports**  

- На сервере в файл **/etc/default/nfs-kernel-server** добавиляем строку `NEED_SVCGSSD="yes"` и перезапускаем nfs-kernel-server
```
vi /etc/default/nfs-kernel-server
systemctl restart nfs-kernel-server
```
![""](https://github.com/mus-cat/otus-study-m1l6/blob/main/06.additionalConfServer.png)

- После перезапуска nfs-kernel-server убеждаемся, что серис rpc-svcgssd - работает. На этом настройка сервера завершена.
```
systemctl status rpc-svcgssd
```

- На клиенте запустив **kadmi** подключаемся к серверу kerberos с учёткой admin и выгружаем ключи клиента в файл **/etc/krb5.keytab**
```
kadmin -p admin
ktadd nfs/nfscl.example.org 
```
!["Сохраняем на клиенте keytab"](https://github.com/mus-cat/otus-study-m1l6/blob/main/05.mkPrincOnClient.png)

- На клиенте в файл **/etc/default/nfs-common** добавляем строку `NEED_GSSD="yes"`
!["Поднастраиваем клиент"](https://github.com/mus-cat/otus-study-m1l6/blob/main/07.clientAddConf.png)

- Перезапускаем службу **nfs-utils** и проверяем что запущен еще сервис **rpc-gssd** (если он не запущен, запускаем его непосредственно)
```
systemctl restart nfs-utils
systemctl status rpc-gssd
```

- Монтируем на клиенте папку с сервера
```
mount -t nfs4 192.168.56.10:/home/exportdir /mnt -o sec=krb5
```
!["Пытаемся смонтировать"](https://github.com/mus-cat/otus-study-m1l6/blob/main/08.tryMount.png)
