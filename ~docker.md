```
ONTAINER_USER=$1                                                                  
                                                                                   
if [ -z "$CONTAINER_USER" ]; then                                                  
    CONTAINER_USER=$USER                                                           
fi                                                                                 
                                                                                   
my-docker openbmc/ubuntu2404:latest $CONTAINER_USER-$RANDOM $CONTAINER_USER   
```

```
#!/bin/bash                                                                        
                                                                                   
IMAGE_NAME=$1                                                                      
CONTAINER_NAME=$2                                                                  
CONTAINER_USER=$3                                                                  
                                                                                   
if [ -z "$IMAGE_NAME" ] || [ -z "$CONTAINER_NAME" ]; then                          
    echo [usage] my-docker \$image \$container [user]                            
    exit                                                                           
fi                                                                                 
                                                                                   
if [ -z "$CONTAINER_USER" ]; then                                                  
    CONTAINER_USER=$USER                                                           
fi                                                                                 
                                                                                   
docker run \                                                                       
    --cap-add=sys_admin \                                                          
    --cap-add=sys_nice \                                                           
    --net=host \                                                                   
    --hostname=docker \                                                            
    --name $CONTAINER_NAME \                                                       
    -v /home:/home \                                                               
    -v /etc/passwd:/etc/passwd \                                                   
    -v /etc/shadow:/etc/shadow \                                                   
    -v /etc/group:/etc/group \                                                     
    --rm \                                                                         
    -w $PWD \                                                                      
    -it \                                                                          
    $IMAGE_NAME \                                                                  
    /bin/bash -c "su $CONTAINER_USER"                             


# -t: pseudo terminal
# -v: mount
# -u: user and group
# -d: detach
```

- docker file

```
FROM ubuntu:24.04                                                                  
                                                                                   
ENV DEBIAN_FRONTEND noninteractive                                                 
                                                                                   
RUN apt-get update && apt-get install -yy \                                        
  build-essential \                                                                
  chrpath \                                                                        
  cpio \                                                                           
  debianutils \                                                                    
  diffstat \                                                                       
  file \                                                                           
  gawk \                                                                           
  git \                                                                            
  iputils-ping \                                                                   
  libdata-dumper-simple-perl \                                                     
  liblz4-tool \                                                                    
  libsdl1.2-dev \                                                                  
  libthread-queue-any-perl \                                                       
  locales \                                                                        
  python3 \                                                                        
  socat \                                                                          
  subversion \                                                                     
  texinfo \                                                                        
  vim \                                                                            
  wget \                                                                           
  zstd \                                                                           
  sudo                                                                             
                                                                                   
# Set the locale                                                                   
RUN locale-gen en_US.UTF-8                                                         
ENV LANG en_US.UTF-8                                                               
ENV LANGUAGE en_US:en                                                              
ENV LC_ALL en_US.UTF-8                                            
```
