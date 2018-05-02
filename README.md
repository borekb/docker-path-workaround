Git Bash is an awesome shell that comes with [Git for Windows](https://gitforwindows.org/) but Docker and Docker Compose don't work well in it due to path conversions, see e.g. [this issue](https://github.com/docker/toolbox/issues/673). To confirm that you have the problem, try to run this:

```
$ docker run --rm -it ubuntu /bin/bash
```

If you get this error, you're impacted:

```
stat C:/Program Files/Git/usr/bin/bash: no such file or directory
```

Note how the Linux path is prepended with `C:/Program Files/Git` which is of course completely wrong from the container's point of view. Another common examples are volume mappings.

The key to a solution is the  `MSYS_NO_PATHCONV` variable. Create two small scripts, `docker` and `docker-compose`, and put them in a location that has **higher-priority than Docker's path**. Actually, Docker puts itself very high in the list, you have to become no. 1 **system path** (not user PATH) to beat it. Here is an example from my computer:

![image](https://user-images.githubusercontent.com/101152/39303507-b980631a-4956-11e8-8374-5182385a15a1.png)

The contents of the `docker` proxy script is this:

```sh
#!/bin/bash

(export MSYS_NO_PATHCONV=1; "docker.exe" "$@")
```

Similarly for `docker-compose`.

One final important note is that **you have to do this after every Docker update**: it will simply put itself back at the top.

Hope this helps.