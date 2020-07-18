# Docker notes
Notes, tips and issues that may be useful for docker development

  * [Dockerfile](#dockerfile)
   * [Errors in ADD pass unnoticed](#errors-in-add-pass-unnoticed)
   * [Problems with the `syntax` parser directive](#problems-with-the-syntax-parser-directive)
   * [ARG values in multistage builds](#arg-values-in-multistage-builds)
   * [Preserve changes after declaring VOLUME](#preserve-changes-after-declaring-volume)
   * [Missing CMD and ENTRYPOINT](#missing-cmd-and-entrypoint)
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

The values defined by the ARG instruction are only available when building an image but their scope depends on where they are declared within the dockerfile [[2]],[3]]. For example, consider [this dockerfile](dockerfiles/ARG_scope/Dockerfile_ARG_scope_test_01) reproduced below
```
ARG version=latest
ARG last_image=alpine
ARG p1=p1
FROM alpine:$version as a1
ARG q1=q1
ARG f1=/home/${p1:-mp1}${q1:-mq1}
RUN touch "$f1"

ARG p2=p2
FROM alpine:$version as a2
ARG q2=q2
ARG f2=/home/${p1:-mp1}${q1:-mq1}${p2:-mp2}${q2:-mq2}
RUN touch "$f2"

ARG p3=p3
FROM $last_image:$version
COPY --from=a1 "$f1" "${f1:-/home/mf1}"
COPY --from=a2 "$f2" "${f2:-/home/mf2}"
ARG q3=q3
ARG f3=/home/${p1:-mp1}${q1:-mq1}${p2:-mp2}${q2:-mq2}${p3:-mp3}${q3:-mq3}
RUN touch "$f3"

ENTRYPOINT ["/bin/sh"]
```
This dockerfile contains three FORM instructions and three build stages. In addition, it defines the variables `version` and `last_image` before the first FROM. Finally, it defines three variables `p1`, `p2`, and `p3` which pretend to be outside the build stages, and `q1`, `q2` and `q3` which are within the build stages.

With this dockerfile, the command
```bash
docker build -t ARG_scope .
```
yields an image with the file `/home/mp1mq1mp2mq2mp3q3`, thereby indicating that none of `p1`, `q1`, `p2`, `q2`, and `p3` are available within the third build stage. This was actually expected and consistent with [[2]] and [[3]]. We can try and make them available by mmodifying the third stage of the dockerfile following [[2]], e.g. by inserting
```
ARG p1
ARG p2
ARG p3
ARG q1
ARG q2
```
before `ARG q3=q3` (see [this dockerfile](dockerfiles/ARG_scope/Dockerfile_ARG_scope_test_02)). However, the image produced contains the file `/home/p1mq1mp2mq2mp3q3`, thereby indicating that `p1` is available but not the others.

The variables `version` and `last_image` are accessible from all FROM instructions. This can be verified by running the command
```bash
docker build --build-arg last_image=hello -t ARG_scope .
```
and noticing that the build fails because the image `hello:latest` cannot be located. Further, modifying the `last_image` variable before the last FROM as below (see [this dockerfile](dockerfiles/ARG_scope/Dockerfile_ARG_scope_test_03))
```
ARG last_image=hello
FROM  $last_image:$version
```
does not have the same effect, indicating that, once set, variables defined before the first FROM cannot be modified dynamically for subsequent FROM instructions.

In conclusion, it seems to me that:

+ ARG variables defined before the first FROM 
  + are available for all FROMs.
  + cannot be modified so that different FROMs see different values.
  + can be made available within each build stage following [[2]].
+ ARG variables defined between FROM instructions
  + only belong to that build stage.
  + cannot be made available to other build stages.
  
### Preserve changes after declaring VOLUME

Changes after declaring a volume are said to be discarded [[4]], but that actually depends on how the change is made. Consider [this dockerfile](dockerfiles/VOLUME/Dockerfile_VOLUME_test_01)
```
FROM alpine
RUN mkdir /testA
RUN printf "hello\n" > /testA/hello
RUN printf "hello\n" > /testA/hello_again
VOLUME /testA
# This one does not change the content of the data
RUN printf "hello\n" > /testA/hello
# This one does change the content of the data
RUN printf "hello again\n" > /testA/hello_again
# This one adds a new file
RUN printf "hello yet again\n" > /testA/hello_yet_again
```
which tests the effect of modifying a volume after declaring it

+ by overwriting a file with the same content.
+ by overwriting a file with different content.
+ by generating a new file.

In line with [[4]], the resulting docker image contains none of the modifications.

Now, consider [this dockerfile](dockerfiles/VOLUME/Dockerfile_VOLUME_test_02)
```
FROM alpine
RUN mkdir /testA
COPY hello /testA/
COPY hello_again /testA/
VOLUME /testA
# This one does not change the content of the data
RUN printf "hello\n" > /testA/hello
# This one does change the content of the data
RUN printf "hello again\n" > /testA/hello_again
# This one adds a new file
RUN printf "hello yet again\n" > /testA/hello_yet_again
```
which replaces the RUN instructions before VOLUME with COPY instructions, which retrieve the files from the building context. The files are subsequently modified analogously to the previous dockerfile. Once again, in line with [[4]], the resulting docker image contains none of the modifications.

However, consider [this other dockerfile](dockerfiles/VOLUME/Dockerfile_VOLUME_test_03)
```
FROM alpine
RUN mkdir /testA
RUN printf "hello\n" > /testA/hello
RUN printf "hello again\n" > /testA/hello_again
VOLUME /testA
# This one does not change the content of the data
COPY hello /testA/
# This one does change the content of the data
COPY hello_again /testA/
# This one adds a new file
COPY hello_yet_again /testA/
```
which preserves the RUN instructions before VOLUME but replaces the ones after it with COPY instructions. Seemingly in contradiction with [[4]], the resulting docker image contains not the ogirinal files created before VOLUME, but all the files created after it by the COPY instructions. The result is the same if all the RUN instructions are replaced by COPY instructions.

### Missing CMD and ENTRYPOINT

As mentioned in [[5]], dockerfiles must have at least a CMD or ENTRYPOINT instruction. However, they need not be explicitly declared nor point to an actual executable for the build to succeed. Consider [this dockerfile](dockerfile/CMD/Dockerfile_CMD_test_01)
```
FROM scratch
COPY hello /test/
CMD [""]
```
which has an empty string for command, or [this dockerfile](dockerfile/CMD/Dockerfile_CMD_test_02)
```
FROM scratch
COPY hello /test/
ENTRYPOINT [""]
```
which has an empty string for entry point, or [this dockerfile](dockerfile/CMD/Dockerfile_CMD_test_03)
```
FROM alpine
COPY hello /test/
```
which has neither CMD nor ENTRYPOINT. All of them will successfully produce an image containing the specified file. This can be verified by running
```
docker export $(docker create <imageid>) | tar t
```

The first two images cannot be run with `docker run [-it] <imageid>`. However, the last one can be run despite having neither CMD nor ENTRYPOINT instructions. This is because the CMD instruction is inheretid from the `alpine` image. Hence, CMD and ENTRYPOINT need not explicitly appear in a dockerfile.


## References
[1]: https://docs.docker.com/engine/reference/builder/#parser-directives
[2]: https://docs.docker.com/engine/reference/builder/#understand-how-arg-and-from-interact
[3]: https://vsupalov.com/docker-arg-env-variable-guide/
[4]: https://docs.docker.com/engine/reference/builder/#notes-about-specifying-volumes
[5]: https://docs.docker.com/engine/reference/builder/#understand-how-cmd-and-entrypoint-interact



1. [Dockerfile reference: Parser directives](https://docs.docker.com/engine/reference/builder/#parser-directives)
2. [Dockerfile reference: How ARG and FROM interact](https://docs.docker.com/engine/reference/builder/#understand-how-arg-and-from-interact)
3. https://vsupalov.com/docker-arg-env-variable-guide/
4. [Notes about specifying volumes](https://docs.docker.com/engine/reference/builder/#notes-about-specifying-volumes)
5. [Understand how CMD and ENTRYPOINT interact](https://docs.docker.com/engine/reference/builder/#understand-how-cmd-and-entrypoint-interact)
