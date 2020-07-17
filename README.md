# Docker notes
Notes, tips and issues that may be useful for docker development

  * [Dockerfile](#dockerfile)
   * [Errors in ADD pass unnoticed](#errors-in-add-pass-unnoticed)
   * [Problems with the `syntax` parser directive](#problems-with-the-syntax-parser-directive)
   * [ARG values in multistage builds](#arg-values-in-multistage-builds)
  * [References](#references)

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

### Problems with the `syntax` parser directive

As stated in the dockerfile reference [[1]], the `syntax` parser directive allows one to specify the dockerfile builder to use. This feature is there said to be enabled only if the BuildKit backend is used. Unmentioned explicitly is the fact that, when the BuildKit backend is not used, the line containing the directive is parsed as a comment, thereby causing all subsequent parser directives to be parsed as comments as well.

To illustrate this, consider the following dockerfile
```
# syntax=docker/dockerfile
# escape=`
FROM scratch
ADD alpine-minirootfs-3.12.0-x86_64.tar.gz /
RUN echo "Hello world" `
    > /home/hello_world
```
which, after running 
```
docker build --no-cache=true -t alpine:test2 -f Dockerfile .
```
produces the following output
```
Sending build context to Docker daemon  74.69MB
Error response from daemon: Dockerfile parse error line 6: unknown instruction: >
```
showing that the `escape` parser directive has not been taken into account. I believe this is because, since BuildKit is not used, the `syntax` parser directive was interpreted as a comment. Hence, as mentioned in the dockerfile reference, docker stops interpreting any subsequent parser directives as such.

On the contrary, consider now the following Dockerfile
```
# escape=`
# syntax=docker/dockerfile
FROM scratch
ADD alpine-minirootfs-3.12.0-x86_64.tar.gz /
RUN echo "Hello world" `
    > /home/hello_world
```
which produces the following output after running the same build command
```
Sending build context to Docker daemon  74.69MB
Step 1/3 : FROM scratch
 ---> 
Step 2/3 : ADD alpine-minirootfs-3.12.0-x86_64.tar.gz /
 ---> 630b99176fe4
Step 3/3 : RUN echo "Hello world"     > /home/hello_world
 ---> Running in 3732c6d82e8a
Removing intermediate container 3732c6d82e8a
 ---> 87ba37bfb242
Successfully built 87ba37bfb242
Successfully tagged alpine:test2
```
showing that the `escape` parser directive was taken into account and now the backstick is taken as the escape character.

In conclusions, the order of the parser directives is important if the same dockerfile is intended to be portable across backends.

### ARG values in multistage builds

The values defined by the ARG instruction are said to be available when building an image but not when running it [[2]],[3]]. However, they are not available everywhere during the build unless special care is taken. 

As mention in [[3]], the ARG instruction defines default values that can be overwritten at build-time, e.g. by passing the values withAs mention in [[2]], ARG can appear before FROM

Under construction...

## References
[1]: https://docs.docker.com/engine/reference/builder/#parser-directives
[2]: https://docs.docker.com/engine/reference/builder/#understand-how-arg-and-from-interact
[3]: https://vsupalov.com/docker-arg-env-variable-guide/

1. [Dockerfile reference: Parser directives](https://docs.docker.com/engine/reference/builder/#parser-directives)
2. [Dockerfile reference: How ARG and FROM interact](https://docs.docker.com/engine/reference/builder/#understand-how-arg-and-from-interact)
3. https://vsupalov.com/docker-arg-env-variable-guide/
