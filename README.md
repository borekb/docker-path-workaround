Git Bash is an awesome shell that comes with [Git for Windows](https://gitforwindows.org/) but Docker and Docker Compose don't work well in it due to path conversions, see e.g. [this issue](https://github.com/docker/toolbox/issues/673). Same with MSYS2 shells.

To confirm that you have the problem, run this:

```
$ docker run --rm -it ubuntu /bin/bash
```

If you get this error, you're impacted:

```
stat C:/Program Files/Git/usr/bin/bash: no such file or directory
```

Note how the Linux path is prepended with `C:/Program Files/Git` which is of course completely wrong from the container's point of view. Another common example is volume mapping, e.g.:

```
docker run -v "$PWD":/var/www/html php:7-apache
```

This commonly fails as well.

The key to a solution is to set either the `MSYS_NO_PATHCONV` variable for Git Bash shell, or `MSYS2_ARG_CONV_EXCL` for MSYS shells.

Create two small scripts, `docker` and `docker-compose` (no extension) in some location that has **higher-priority than Docker's path**. Docker is quite aggressive and puts itself very high in the list, the safest way is to become no. 1 **system path** (not user path) to beat it. Here is an example from my computer:

![image](https://user-images.githubusercontent.com/101152/39303507-b980631a-4956-11e8-8374-5182385a15a1.png)

The contents of the `docker` proxy script should look like this:

```sh
#!/bin/bash

if [[ -v MSYS2_PATH_TYPE ]]; then
    # !!! Until https://github.com/Alexpux/MSYS2-packages/issues/411 is resolved,
    # !!! make sure you use winpty binaries from https://github.com/rprichard/winpty/releases
    # !!! *not* from `pacman -S winpty`.
    (export MSYS2_ARG_CONV_EXCL="*"; winpty "docker.exe" "$@")
else
    (export MSYS_NO_PATHCONV=1; "docker.exe" "$@")
fi
```

Similarly for `docker-compose`.

Hope this helps.