---
layout: post
published: true
title: springboot mysql on openshift minishift sandbox 
date: 2020-03-20 04:28
subtitle: 
summary: springboot mysql on openshift minishift sandbox
---

## Minishift

```bash
mkdir apptest ;\
cd apptest ;\
git clone https://github.com/swapanc/minishift_jdg_springboot_kafka_example ;\
cd minishift_jdg_springboot_kafka_example ;\
mvn clean install ;\  
cd src/main/docker-files   
minishift start   
minishift pc-env | Invoke-Expression   
minishift docker-env | Invoke-Expression   
oc login   
docker login -u developer -p $(oc whoami -t) $(minishift openshift registry)   
docker build -t springboot_mysql -f ./dockerfile_springboot_mysql .   
docker images   
docker tag springboot_mysql $(minishift openshift registry)/myproject/springboot_mysql   
docker push $(minishift openshift registry)/myproject/springboot_mysql   
docker pull openshift/mysql-56-centos7   
oc new-app -e MYSQL_USER=root -e MYSQL_PASSWORD=root -e MYSQL_DATABASE=test openshift/mysql-56-centos7   
oc get pods   
oc rsh mysql-56-centos7-1-nvth9   
mysql -u root   
CREATE USER 'root'@'%' IDENTIFIED BY 'root';   
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;   
FLUSH PRIVILEGES;   
exit   
exit   
oc get svc     
oc new-app -e spring_datasource_url=jdbc:mysql://172.30.103.248:3306/test mysql-56-centos7 --name=swapapp     
oc get pods     
logs -f springbootmysql-1-5ngv4     
oc get svc     
oc expose svc springbootmysql     
oc get route     
   
curl -v http://swapapp-myproject.192.168.1.204.nip.io/demo/all      
output:     
[   
{"name":"UBUNTU 17.10 LTS","lastaudit":1502409600000,"id":1},   
{"name":"RHEL 7","lastaudit":1500595200000,"id":2},      
{"name":"Solaris 11","lastaudit":1502582400000,"id":3}   
]   
   
   
curl http://swapapp-myproject.192.168.1.204.nip.io/demo/add?name=SpringBootMysqlTest   
3Saved   
   
   
curl http://swapapp-myproject.192.168.1.204.nip.io/demo/all   
[   
{"name":"UBUNTU 17.10 LTS","lastaudit":1502409600000,"id":1},   
{"name":"RHEL 7","lastaudit":1500595200000,"id":2},      
{"name":"Solaris 11","lastaudit":1502582400000,"id":3},      
{"name":"SpringBootMysqlTest","lastaudit":1548892800000,"id":4}      
]   
```   
