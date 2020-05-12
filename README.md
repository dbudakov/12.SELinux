Для выполнения первой части задания приводим файл /etc/nginx/nginx.conf  
к следующему виду:  
![](https://github.com/dbudakov/11.SELinux/blob/master/images/main/main_nginx.conf.png)  
и раскомментим строку с нестандартым портом  
![](https://github.com/dbudakov/11.SELinux/blob/master/images/1.1/nginx.conf_1.png)  
Перезапустив `nginx`, он выпадет в ошибку  
![](https://github.com/dbudakov/11.SELinux/blob/master/images/1.1/restart_nginx_1.png)  
статус демона следующий:  
![](https://github.com/dbudakov/11.SELinux/blob/master/images/1.1/status%20nginx%201.png)  
Обращаем внимание на строку `nginx: [emerg] bind() to 0.0.0.0:5081 failed (13: Permission denied)`  
Далее зачистим и проанализируем audit.log, при помощи sealert, после зачисти ОБЯЗАТЕЛЬНО вновь перезапусть `nginx` чтобы было что анализировать.  
![](https://github.com/dbudakov/11.SELinux/blob/master/images/main/%3Eaudit.log.png)  
![](https://github.com/dbudakov/11.SELinux/blob/master/images/1.1/restart_nginx_1.png)  
![](https://github.com/dbudakov/11.SELinux/blob/master/images/main/sealert.png)  

В выводе мы увидим много информации и способы решения ошибки с запуском nginx а именно:  
```
Do
# semanage port -a -t PORT_TYPE -p tcp 5081
    where PORT_TYPE is one of the following: http_cache_port_t, http_port_t, jboss_management_port_t, jboss_messaging_port_t, ntop_port_t, puppet_port_t.
```
Команда добавляет указанный порт в тип указанного контекста, в выводе указаны каие типы можно расширить    
```
Do
setsebool -P nis_enabled 1
```
Команда разрешает использование нестандартных портов, по сути открывает всем сервисам такую возможноть, что сравнимо с отключением SELinux  
```
Do
allow this access for now by executing:
# ausearch -c 'nginx' --raw | audit2allow -M my-nginx
# semodule -i my-nginx.pp
```
Эти команды формируют модуль, на основе анализа лога audit.log, довольно дорогой способ, но локализует и решает ошибки с разрешениями
