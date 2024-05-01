```
docker run \
    --cap-add=sys_admin \
    --cap-add=sys_nice \
    --net=host \
    --hostname=docker \
    --name my-container \
    -t \
    -v /home:/home \
    -v /etc/passwd:/etc/passwd:ro \
    -v /etc/group:/etc/group:ro \
    -u root:root \
    -d \
    openbmc/ubuntu:latest-qemuarm-x86_64 \


-t: pseudo terminal
-v: mount
-u: user and group
-d: detach
```

```
docker exec \                                                                   
    -w $PWD \                                                                   
    -u root \                                                                   
    -it \                                                                       
    my-container bash -c "su - $USER"

-w: working dir
-u: user and group
-it: interactive and pseudo terminal
```
