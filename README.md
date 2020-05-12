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
Далее зачистим и проанализируем `audit.log`, при помощи `sealert`, после зачистки ОБЯЗАТЕЛЬНО вновь перезапусть `nginx` чтобы было что анализировать. 
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

