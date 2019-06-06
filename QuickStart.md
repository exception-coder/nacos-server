```bash
# 临时关闭 SELinux 
$ sudo setenforce 0
$ git clone https://github.com/exception-coder/nacos-server.git
$ cd nacos-server/nacos-docker-demo/nacos
$ docker-compose up
$ docker network create nacos_default
$ docker network connect nacos_default mysql5.7
```

