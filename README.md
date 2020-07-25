## Домашнее задание
Практика с SELinux  
Цель: Тренируем умение работать с SELinux: диагностировать проблемы и модифицировать политики SELinux для корректной работы приложений, если это требуется.  
1. Запустить nginx на нестандартном порту 3-мя разными способами:  
- переключатели setsebool;  
- добавление нестандартного порта в имеющийся тип;  
- формирование и установка модуля SELinux.  
К сдаче:  
- README с описанием каждого решения (скриншоты и демонстрация приветствуются).   
  
2. Обеспечить работоспособность приложения при включенном selinux.  
- Развернуть приложенный стенд  
https://github.com/mbfx/otus-linux-adm/blob/master/selinux_dns_problems/  
- Выяснить причину неработоспособности механизма обновления зоны (см. README);  
- Предложить решение (или решения) для данной проблемы;  
- Выбрать одно из решений для реализации, предварительно обосновав выбор;  
- Реализовать выбранное решение и продемонстрировать его работоспособность.  
  
## Решение 
Пакеты для анализа логов SELinux
```
yum install -y setools setroubleshoot-server
```

### Первая часть
Для выполнения первой части задания и назначения нестандартного порта добаляем строку `listen 5081;` в соответствующий контектст в файле `/etc/nginx/nginx.conf`      
```
    server {
        listen       5081 ;
        listen       80 default_server;
        listen       [::]:80 default_server;
        server_name  _;
        root         /usr/share/nginx/html;
```
Перезапустив `nginx`, он выпадет в ошибку    
```
[root@SELinux vagrant]# systemctl restart nginx.service
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```
статус демона следующий:   
![](https://github.com/dbudakov/11.SELinux/blob/master/images/1.1/status%20nginx%201.png)   
Обращаем внимание на строку `nginx: [emerg] bind() to 0.0.0.0:5081 failed (13: Permission denied)`  
Далее зачистим(необязательно, выполняется для экономии времени и локализации ошибки) и проанализируем `audit.log`, при помощи `sealert`, после зачистки ОБЯЗАТЕЛЬНО вновь перезапусть `nginx` чтобы было что анализировать.   
```
[root@SELinux vagrant]# >/var/log/audit/audit.log
[root@SELinux vagrant]# systemctl restart nginx.service
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@SELinux vagrant]# sealert -a /var/log/audit/audit.log

```
В выводе мы увидим  способы решения ошибки с запуском nginx. Во-первых добавление указанного порта в тип указанного контекста, в выводе указаны какие типы можно расширить      
```
Do
# semanage port -a -t PORT_TYPE -p tcp 5081
    where PORT_TYPE is one of the following: http_cache_port_t, http_port_t, jboss_management_port_t, jboss_messaging_port_t, ntop_port_t, puppet_port_t.
```
Во-вторых разрешение использования нестандартных портов, по сути, открывает всем сервисам такую возможноть, что сравнимо с отключением SELinux   
```
Do
setsebool -P nis_enabled 1
```
В-третьих команды формирующие модуль, на основе анализа лога audit.log, довольно дорогой способ, но локализует и решает ошибки с разрешениями  
```
Do
allow this access for now by executing:
# ausearch -c 'nginx' --raw | audit2allow -M my-nginx
# semodule -i my-nginx.pp
```

Итак первый пункт, добавляем порт в нужный тип контекста, перезапускаем nginx, смотрим завязанные порты на nginx  
```
semanage port -a -t http_port_t -p tcp 5081
systemctl restart nginx
ss -ntpl |grep nginxf
```
![](https://github.com/dbudakov/11.SELinux/blob/master/images/main/1_nginx.png)    
Удаляем порт из указанного типа, для теста других методов  
```
semanage port -d -t http_port_t -p tcp 5081
```

Для второго пункта сформируем модуль для сервиса на порт 5081. Для начала посмотрим какая информация передается для формирования модуля   
```
ausearch -c 'nginx' --raw 
```
![](https://github.com/dbudakov/11.SELinux/blob/master/images/main/1_ausearch.png)     
из вывода видно что она зависит от содержимого в файле `audit.log`, поэтому для формирования только нужного лога его желательно зачистить, и перезапустить рассматриваемый сервис. Запускаем формирование модуля и его включение. Включение происходит через файл с расширением `.pp`   
```
ausearch -c 'nginx' --raw | audit2allow -M my-nginx
semodule -i my-nginx.pp
```
Для проверки перезапускаем `nginx` и смонтри на открытые для него порты    
```
systemctl restart nginx
ss -ntpl | grep nginx
```
![](https://github.com/dbudakov/11.SELinux/blob/master/images/main/nginx_2.png)    
Отключаем модуль:    
```
semodule -r my-nginx
```
Для третьего пункта разрешить использование нестандартных портов можно по команде  
```
setsebool -P nis_enabled 1
```
Перезапускаем `nginx` и проверяем открытые порты для `nginx`   
```
systemctl restart nginx
ss -ntpl | grep nginx
```
![](https://github.com/dbudakov/11.SELinux/blob/master/images/main/nginx_3.png)    
Отключить данную функцию можно так:  
```
setsebool -P nis_enabled 0
```

### Вторая часть  
ИЗМЕНЕНО:
Можно решить двумя способами, это правкой SELinux отталкиваясь от конфигов и сохраняя строй ФС, либо перемещением файла DNS-зоны, и изменением конфига named.conf, без изменения SELinux. Выбирая из двух вариантов, я выбиру второй, потому что он прозрачен, хоть и неочевиден, всегда можно посмотреть что где лежит, и будет легче объяснить настройку другому специалисту, чем изменение контекстов SELinux.
#### Вариант 1. Правим SELinux
Проблема невозможности добавления зоны на стенде https://github.com/mbfx/otus-linux-adm/blob/master/selinux_dns_problems/ , заключается в типе контекста для файлов содержащих записи зон, на это указывает анализ файла `/var/log/audit/audit.log`   
```
type=AVC msg=audit(1589369529.013:2012): avc:  denied  { create } for  pid=7288 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0
```
а также просмотр контекста нужных файлов
```
[root@ns01 vagrant]# ls -Z /etc/named/dynamic/
-rw-rw----. named named system_u:object_r:etc_t:s0       named.ddns.lab
-rw-rw----. named named system_u:object_r:etc_t:s0       named.ddns.lab.view1
```
Причем просто рекомендация `sealert audit.log` выполнить `semanage fcontext -a -t FILE_TYPE 'named.ddns.lab.view1.jnl'` не поможет,как я понял это из-за типа `etc_t`, в котором нет нужных разрешений, поэтому предварительно меняем тип контекста через `chcon` и после запускаем `semanage fcontext`   
```
chcon -R -t named_zone_t /etc/named/dynamic/
semanage fcontext -a -t named_zone_t /etc/named/dynamic/
```
После чего контекст для файлов в каталоге `/etc/named/dynamic/` будет изменён на постоянной основе и запись днс зоны с клиента отработает  
```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key<<EOF
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> EOF
```
#### Вариант 2. Правим конфигурацию DNS.  
После попытки добавить зону, обратим внимание на вывод команды   
```
sealert -a /var/log/audit/audit.log
```
и заострим внимание на этих строках
```
Additional Information:
Source Context                system_u:system_r:named_t:s0
Target Context                system_u:object_r:etc_t:s0
Target Objects                named.ddns.lab.view1.jnl [ file ]
```
Как мы видим разные права на файл зоны, идем в `named.conf` для уточнения местонахождения журнала.  
```
cat /etc/named.conf
> // labs ddns zone
> zone "ddns.lab" {
>   type master;
>   allow-transfer { key "zonetransfer.key"; };
>   allow-update { key "zonetransfer.key"; };
>   file "/etc/named/dynamic/named.ddns.lab.view1";
> };
```
Увидим местоположение файла и посмотрим на контекст каталога.  
```
ls -Z /etc/named/dynamic/
> -rw-rw----. named named system_u:object_r:etc_t:s0       named.ddns.lab
> -rw-rw----. named named system_u:object_r:etc_t:s0       named.ddns.lab.view1
```
Видим что тип контекста несоответствует. Посмотрим контексты для каталога  
```
sudo semanage fcontext -l | grep /etc/named/dynamic # Для этого контекста вывода не будет, смотрим контекст только на конечный каталог
sudo semanage fcontext -l | grep dynamic
> /var/named/dynamic(/.*)?                           all files          system_u:object_r:named_cache_t:s0 
> /var/named/chroot/var/named/dynamic(/.*)?          all files          system_u:object_r:named_cache_t:s0 
```
Из вывода видим что нужный нам контекст настроен на пути `/var/named/dynamic/`, переносим нужный журнал зоны в настроеный каталог меняя пользователя на `named` чтобы `bind` мог с ним работать, а также редактируем `named.conf`  
```
mv /etc/named/dynamic/named.ddns.lab.view1  /var/named/dynamic/ 
chown named:named /var/named/dynamic/*
systemctl restart named

## приводим настройку для зоны в /etc/named.conf  к следующему виду
> // labs ddns zone
> zone "ddns.lab" {
>   type master;
>   allow-transfer { key "zonetransfer.key"; };
>   allow-update { key "zonetransfer.key"; };
>   file "/var/named/dynamic/named.ddns.lab.view1";
> };
```
Смотрим изменения контекста на файле  
```
ls -Z  /var/named/dynamic/named.ddns.lab.view1.jnl 
> -rw-r--r--. named named system_u:object_r:named_cache_t:s0 named.ddns.lab.view1.jnl
```
Теперь добавление записи для DNS-зоны отработает  
```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key<<EOF
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> EOF
```

В результате с клиентской машины пройдет пинг до сервера по днс имени `ping www.ddns.lab`  
![](https://github.com/dbudakov/12.SELinux/blob/master/images/2/ddns.png)  
решение оформлено в виде дополнительных задач для `ansible` при деплое стенда, решение можно проверить проверив доступность узла "www.ddns.lab" по ДНС имени, с клиентской машины.  

