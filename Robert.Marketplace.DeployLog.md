## 25.11.19 Статус Kafka Cluster из зеленого перекрасился в красный.
3 нода (136.243.8.29) перестала отвечать на connection со стороны 2 других брокеров. На машине CPU utilization доходил до 80%. После перезагрузки ноды CPU упал до 40% (DataDog metrics) и восстановился mount volume **/gluster**. Через 10 мин кластер ожил и **ISR Delta** & **Offline partitions** стали падать и загорелись зеленым. 
На 136.243.15.162 и 136.243.8.29 ,bkcz d bcnthbrt **patroni** из-за того что порт 5432 был занят оригинальным сервисом **postgresql**.

**Решение**: убить сервис postgres удалив все **/etc/systemd/system/postgres.service** файлы и **links** на них и вызвать команды как указано тут: https://superuser.com/questions/513159/how-to-remove-systemd-services


## 25.11.19 Сломан https://plon.market (при попытке открыть страницу выдает Access Denied)
Обнаружил, что на машине 136.243.15.162 и на машине 136.243.8.29 отмонтировался /gluster. Это приводило к тому, что 70% запросов либо выдавало ошибку, либо выдавало поломанную верстку. 

**Решение**: перезагрузил 136.243.8.29 машину, после чего на всех 3 нодах /gluster был восстановлен демоном автоматически.

**Рекомендация**: не держать код приложения в /gluster - собирать Docker контейнер и пихать весь контент по максимуму туда.

## 23.09.19  Настройка Portainer
[Инсталляция](https://portainer.readthedocs.io/en/stable/agent.html)
1. Возникла след. проблема: агенты принимают Portainer только в первый раз. Если его пересоздавать, то надо пересоздавать и агентов. Либо использовать ```AGENT_SECRET``` 

## 17.09.19  Настройка Postgres Cluster: Patroni + HAProxy
1. Взял за основу плейбуки [postgres](https://github.com/geerlingguy/ansible-role-postgresql/tree/master/tasks "https://github.com/geerlingguy/ansible-role-postgresql/tree/master/tasks"), [patroni](https://github.com/kostiantyn-nemchenko/ansible-role-patroni), роль haproxy накидал сам.
2. в bootstrap блоке настроек ```patroni``` указан пользователь replicator, но он не создается, хотя пароль для superuser применяется. **Решение:** сделал репликацию через ```superuser```
3. Мастер стартовал без проблем, реплики, запускаемые patroni, не стартовали, как оказалось (после изучения логов, для этого увеличил debugLEvel=DEBUG), реплика пытается переименовать ```data_dir```, например если ```data_dir=/var/lib/patroni```, то реплика пытается переименовать на этапе ```bootstrap``` каталог в ```data_dir=/var/lib/patroni_2019_xx_yy``` поскольку пользователь ```postgres``` не имеет доступа к корневому каталогу, происходила ошибка и реплики висели в состоянии ```uninitialized```. **Решение:** создал корневую ```var/lib/patroni``` с владельцем ```postgres``` а каталог с данными указал  ```var/lib/patroni/data```, после чего реплики стали бутстрапиться.
4. При создании кластера ```patroni``` лезет в ```Zookeeper``` и создает каталог вида ```<patroni_namespace>/<patroni_scope>```. Если удалять кластер руками (файлы данных и конфигураций) то надо руками и почистить zookeeper. **Решение:** ```zookeeper-shell <zoo-host:port> rmr /<patroni_namespace>``` 
5. В целях безопасности выдачу состояния haproxy забиндил на ```localhost```, с машины с Nginx проксирование ижет поверх SSH тунеля


## 13.09.19  Первичная настройка кластера
1. Для установки кластера KAFKA остановился на Confluent-Kafka решении, поскольку они дают мощный набор Ansible PlayBooks, сделал форк [https://github.com/Sirius-social/ansible-playbook-kafka](https://github.com/Sirius-social/ansible-playbook-kafka)
Выбор связан прежде всего с необходимостью проводить сложные настройки SASL_SSL, руками получится много дольше, кроме того  Confluent поставляет мощный инструмент мониторинга ControlCenter и проч приблуды из коробки.  
2. Скрипты из ```ansible-playbook-kafka``` не работают на Debian10 пришлось откатывать к Debian9
3. ```ansible``` которая устанавливалась из репы ставит python2.7 изза чего возникли проблемы с ansible task ```synchronize``` в блоках SSL Sync и SSL Generation. **Решение:** пришлось патчить на ```fetch + archive + unarchive + copy```  
4. Когда попытался поменять ```{{user}} {{group}}}``` в плейбуке ```kafka_install_playbook.yml```, сервисы не стартовали. **Решение:** пришлось вернуть родные значения ```cp-kafka confluent-kafka```
5. **Проблема из-за которой потерял 2 дня!!!:** Kafka брокеры при синхре через Zookeeper передают свои адреса (advertised listeners), если этот параметр явно не указан, то брокер адрес берет из ```hostname```. Это лечится либо явным указанием ```advertised listeners``` в ```/etc/kafka/server.properties``` либо указанием рабочего ```hostname``` через ```hostnamectl```, либо корректным hostname. **Решение:** Остановился на установке hostname вместо патчинга плейбуков.
