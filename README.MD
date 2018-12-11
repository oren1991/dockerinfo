# Docker info

## Install docker

### Creating docker network based on nginx-proxy
This way containers can be reached via virtual hostnames and nginx will handle which container send the request to based on the virtual host setup

```console
foo@bar:~$ docker network create nginx-proxy #this is the name of the networdk docker would use
```
