# Docker notes
Notes, tips and issues that may be useful for docker development


## Dockerfile

### Errors in ADD pass unnoticed

It seems that errors in the ADD command pass unnoticed unless they affect other subsequent commands during the image creating. Specifically, executing the command
```
docker build --no-cache=true -t test -f Dockerfile .
```
with the following Dockerfile
```
FROM scratch
ADD alpine-minirootfs-3.12.0-x86_64.tar.gz `
RUN apk update
```
produces the output
```
Sending build context to Docker daemon  74.69MB
Step 1/3 : FROM scratch
 ---> 
Step 2/3 : ADD alpine-minirootfs-3.12.0-x86_64.tar.gz `
 ---> 07914b5a41f8
Step 3/3 : RUN apk update
 ---> Running in 2187e1651eba
OCI runtime create failed: container_linux.go:349: starting container process caused "exec: \"/bin/sh\": stat /bin/sh: no such file or directory": unknown
```
Note that the error occurred during the RUN step (3/3), as opposed to during the ADD step (2/3). Further, it mentions that `/bin/sh` is missing, but fails to notice that the cause is the use of the backstick instead of the slash in the ADD step.

Without the RUN step, the error would have passed unnoticed, and `docker build` would have reported that the image was successfully created, as shown below
```
Sending build context to Docker daemon  74.69MB
Step 1/2 : FROM scratch
 ---> 
Step 2/2 : ADD alpine-minirootfs-3.12.0-x86_64.tar.gz `
 ---> de09dc664ee5
Successfully built de09dc664ee5
Successfully tagged test:latest
```
but not with the results that one would have expected should the backstick be replaced with a slash.

The specific error above stems from a typo while experimenting with different escape characters in a casual environment, and it is hopefully rare enough to care. However, what is distressing is the fact that the error messages, which informative about what was the problem during the build, are uninformative on their origin within the Dockerfile. In fact, searching for explanations on this error message led to quite many different alternatives (e.g. see [here](https://github.com/moby/moby/issues/31702)), none of which was the one happening in this case.
