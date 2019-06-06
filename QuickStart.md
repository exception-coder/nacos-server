```bash
$ git clone https://github.com/exception-coder/nacos-server.git
$ cd nacos-server/nacos-docker-demo/nacos
$ docker-compose up
$  docker network create nacos_default
$ docker network connect nacos_default mysql5.7
```