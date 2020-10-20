**Вариант 1. Развертывание кластера с помощью ClusterControl**

* Установила docker на виртуальную машину согласно инструкции с официального сайта.
* Скачала образ ClusterControl

      docker pull severalnines/clustercontrol
     
* Запустила образ, пробросив порты для доступа к UI

      docker run -d --name clustercontrol -p 5000:80 -p 5001:443 -p 9500:9500 -p 9501:9501 severalnines/clustercontrol 
      
*  ClusterControl Web UI доступен по адресу http://146.148.46.31:5000/clustercontrol

* Скопировала в контейнер ssh-ключ для доступа к GCP

      docker cp ~/.ssh/id_rsa f9c5323e26df:/tmp
      
* Создала 4 виртуальные машины для разворачивания кластера:
  * postgres-8m - master
  * postgres-8s1 - slave1
  * postgres-8s2 - slave2
  * postgres-8p - proxy
  
* На ClusterControl Web UI в разделе Deploy создала кластер, заполнив необходимые параметры:
  * имя пользователя для доступа к серверам: rsa-key-20200623
  * путь к ssh-ключу: /tmp/id_rsa
  * имя кластера: postgres-hw8
  * версию postgresql: 11
  * имя пользователя и пароль для мониторигна postgresql: admin
  * адреса серверов для master, slave-1 и slave-2
  
* После развертывания кластера проверила работоспособность: подключилась к мастеру, создала базу и таблицу в ней

* Добавила к кластеру баллансировщик нагрузки.

* На контейнере с ClusterControl выполнила команды для создания пользователя и базы данных:

      s9s cluster --create-account --cluster-id=1 --account=admin1:******@34.121.114.17
      s9s cluster --create-database --cluster-id=1 --account=admin1:******@34.121.114.17 --db-name=demo_hand
      
* Подключилась к кластеру через баллансировщик нагрузки, создала таблицу, вставила в таблицу данные
    
      psql -U admin1 -h 34.121.114.17 -p 5433 -d demo_hand
      
* Создала для сервисного аккаунта GCP ключ

* На ClusterControl Web UI в разделе Deploy in the Cloud создала кластер, указав необходимые параметры

* На контейнере с ClusterControl выполнила команды для создания пользователя и базы данных: 

      s9s cluster --create-account --cluster-id=2 --account=admin:******@34.73.218.14
      s9s cluster --create-database --cluster-id=2 --account=admin:******@34.73.218.14 --db-name=demo_gcp
      
* Подключилась к кластеру через баллансировщик нагрузки, создала таблицу

      psql -U admin -h 4.73.218.14 -p 5433 -d demo_gcp
      
      
