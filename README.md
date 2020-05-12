Для выполнения первой части задания приводим файл /etc/nginx/nginx.conf  
к следующему виду:  
![](https://github.com/dbudakov/11.SELinux/blob/master/images/main/main_nginx.conf.png)  
и раскомментим строку с нестандартым портом  
![](https://github.com/dbudakov/11.SELinux/blob/master/images/1.1/nginx.conf_1.png)  
Перезапустив `nginx`, он выпадет в ошибку  
![](https://github.com/dbudakov/11.SELinux/blob/master/images/1.1/restart_nginx_1.png)  
статус демона следующий:  
![](https://github.com/dbudakov/11.SELinux/blob/master/images/1.1/satus_nginx_1.png)  
Обращаем внимание на строку `nginx: [emerg] bind() to 0.0.0.0:5081 failed (13: Permission denied)`  
Далее зачистим и проанализируем audit.log, при помощи sealert, после зачисти ОБЯЗАТЕЛЬНО вновь перезапусть `nginx` чтобы было что анализировать.  
![](https://github.com/dbudakov/11.SELinux/blob/master/images/main/%3Eaudit.log.png)  
![](https://github.com/dbudakov/11.SELinux/blob/master/images/1.1/restart_nginx_1.png)  
![](https://github.com/dbudakov/11.SELinux/blob/master/images/main/sealert.png)  
