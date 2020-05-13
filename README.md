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
В выводе мы увидим  способы решения ошибки с запуском nginx. Во-первых добавление указанного порта в тип указанного контекста, в выводе указаны каие типы можно расширить      
```
Do
# semanage port -a -t PORT_TYPE -p tcp 5081
    where PORT_TYPE is one of the following: http_cache_port_t, http_port_t, jboss_management_port_t, jboss_messaging_port_t, ntop_port_t, puppet_port_t.
```
Во-вторых разрешение использования нестандартных портов, по сути открывает всем сервисам такую возможноть, что сравнимо с отключением SELinux   
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
из вывода видно что она зависит от содержимого в файле `audit.log`, поэтому для формирования только нужного лога его желательно зачистить, и перезапустить рассматриваемый сервис. Запускаем формирование модуля и его включение, включени происходит через файл с расширением `.pp`   
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
Проблема невозможности добавления зоны на стенде https://github.com/mbfx/otus-linux-adm/blob/master/selinux_dns_problems/ , заключается в типе контекста для файлов содержащих записи зон, на это указывает анализ файла `/var/log/audit/audit.log`   
```
type=AVC msg=audit(1589369529.013:2012): avc:  denied  { create } for  pid=7288 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0
```
а также  
```
[root@ns01 vagrant]# ls -Z /etc/named/dynamic/
-rw-rw----. named named system_u:object_r:etc_t:s0       named.ddns.lab
-rw-rw----. named named system_u:object_r:etc_t:s0       named.ddns.lab.view1
```
Причем просто рекомендация `sealert` выполнить `semanage fcontext -a -t FILE_TYPE 'named.ddns.lab.view1.jnl'` не поможет,как я понял это из-за типа etc_t, в котором нет нужных разрешений, поэтому предварительно меняем тип контекста через `chcon` и после запускаем `semanage fcontext`   
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
В результате с клиентской машины пройдет пинг до сервера по днс имени `ping www.ddns.lab`  
![](https://github.com/dbudakov/12.SELinux/blob/master/images/2/ddns.png)  
решение оформлено в виде дополнительных задач для ansible при деплое стенда, решение можно проверить проверив доступность узла "www.ddns.lab" по ДНС имени, с клиентской машины.  
