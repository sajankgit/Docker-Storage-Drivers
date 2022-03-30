# Docker-Storage-Drivers

To run an application inside a container, we require lot of OS dependencies and libraries. Ideally, we would need to have a copy of all these dependencies and libraries stored separately for each container process. However, this is very inefficient way of storage and container launch time would also increase.

The popularity of docker wouldn’t be this much if there were no storage drivers.

Docker Storage Drivers figured out how to avoid making copies of dependencies and libraries of each and every container process and share it among the containers running in the host. This way storage became efficient and optimised.

Docker Storage drivers use the following two mechanisms to achieve this —

- UnionFileSystem
- Copy-On-Write


## Overlay2

The default storage driver that is used in docker is overlay2. The It uses a file system called overlayFS which is modern version of UnionFileSystem.
In OverlayFS, files and directories are considered as layers. 

It then allows a merged view of these layers to get a unified view.

![image](https://user-images.githubusercontent.com/37524392/160756139-002e9a25-0dc3-4079-82f9-35660cf89dd4.png)


These layers can be in different filesystem or in volumes.

The default storage path of docker in the host is `/var/lib/docker`.

```
/var/lib/docker# ls -al
total 60
drwx--x--x 15 root root 4096 Mar 24 06:43 .
drwxr-xr-x 25 root root 4096 Mar 24 06:43 ..
drwx------  2 root root 4096 Mar 24 06:43 builder
drwx------  4 root root 4096 Mar 24 06:43 buildkit
drwx------  3 root root 4096 Mar 24 06:43 containerd
drwx-----x  2 root root 4096 Mar 24 06:43 containers
drwx------  3 root root 4096 Mar 24 06:43 image
drwxr-x---  3 root root 4096 Mar 24 06:43 network
drwx-----x  3 root root 4096 Mar 30 04:32 overlay2
drwx------  4 root root 4096 Mar 24 06:43 plugins
drwx------  2 root root 4096 Mar 24 06:43 runtimes
drwx------  2 root root 4096 Mar 24 06:43 swarm
drwx------  2 root root 4096 Mar 30 04:30 tmp
drwx------  2 root root 4096 Mar 24 06:43 trust
drwx-----x  2 root root 4096 Mar 24 06:43 volumes
```

Currently, `overlay` directory is empty.

Now let’s create a docker image from the below docker file and see what is happening inside `overlay` directory.

```
# cat Dockerfile
FROM nginx
ADD file1 .
```


```
docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
docker1             latest              330fe7039ef5        6 minutes ago       142MB
```

In `overlay` directory,

```
/var/lib/docker/overlay2# ls
313409b1ca97db0b92e71002068ec1e3b592b4bb0ff78f106acc81ada1381426
60c068f1df995aa58aebee7c288b77499d21ac062a365ea5e8fde3dfaa45be5a
63211217ee91c4133bcf8309c3511b22e91ac6b599710312300e1723e31255c0
928c6ea4b46b57b1a3bb6d9c07c3aed2328e246ecde2f513b3846624a1e0a09f
b3a4bef8f673f015158686a25d2c3c9159b8fc5ac271c9b205ce77d1dc937807
ed6da4e32c6ccc4f6d4efa4ff27337c481be8716eada22a5235157458e8cf589
ffa9d2fd80f1978cd85efc4e52231b815a73cad7381ba9da63cadcfd5d4c1f9b
```

Here the first directory `313409b1ca97db0b92e71002068ec1e3b592b4bb0ff78f106acc81ada1381426` is created by the second command in the dockerfile. Rest are from `nginx`.

Now let’s create another image modifying just the second line.

```
# cat Dockerfile
FROM nginx
ADD file2 .
# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
docker2             latest              18e14e874b0c        3 minutes ago       142MB
docker1             latest              330fe7039ef5        8 minutes ago       142MB
```

```
# ls
313409b1ca97db0b92e71002068ec1e3b592b4bb0ff78f106acc81ada1381426
60c068f1df995aa58aebee7c288b77499d21ac062a365ea5e8fde3dfaa45be5a
63211217ee91c4133bcf8309c3511b22e91ac6b599710312300e1723e31255c0
928c6ea4b46b57b1a3bb6d9c07c3aed2328e246ecde2f513b3846624a1e0a09f
b3a4bef8f673f015158686a25d2c3c9159b8fc5ac271c9b205ce77d1dc937807
ed6da4e32c6ccc4f6d4efa4ff27337c481be8716eada22a5235157458e8cf589
eee4ce4aaf3c928018732ee036c855b2b11a17472d0315e706404d51fc7d85c7
ffa9d2fd80f1978cd85efc4e52231b815a73cad7381ba9da63cadcfd5d4c1f9b
```

Here you can just see an extra directory `eee4ce4aaf3c928018732ee036c855b2b11a17472d0315e706404d51fc7d85c7`. Rest is shared between both the images.

## Copy-On-Write

When we create a new container from an image, we add a new & thin writable layer on top of the underlying stack of layers present in the base docker image. 

All changes made to the running container, such as creating new files, modifying existing files or deleting files, are written to this thin writable container layer.

Because each container has its own thin writable container layer, and all changes are stored in this container layer, this means that multiple containers can share access to the same underlying image and yet have their own data state.

At some point, if any one process wants to modify or write to the data, only then does the operating system make a copy of the data for that process to use. Only the process that needs to write has access to the data copy. All other processes continue to use the original data.

Docker makes use of copy-on-write technology with both images and containers. This CoW strategy optimizes both image disk space usage and the performance of container start times.
ref: https://medium.com/@BeNitinAgarwal/docker-containers-filesystem-demystified-b6ed8112a04a

![image](https://user-images.githubusercontent.com/37524392/160756628-c7249bb5-2a46-4163-9289-b1b1d7c3ba3c.png)


## CONCLUSION

Hope this article helps you guys to understand how docker utilizes unionFS and CoW to optimize the storage and container start time. These played a major role in making docker so popular.

Enjoy you day !
