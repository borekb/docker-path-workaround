# Path conversion workaround for Docker in Git Bash / MSYS2 on Windows

> **UPDATE 03/2019**: I no longer use Windows and there have been some recent developments like ConPTY shipping in Windows 10 Fall 2018 update (1809) so I'm not entirely sure if the info below is still useful or valid. Feel free to open issues / send pull requests. 

> **UPDATE 07/2018**: I switched from Git Bash to MSYS2 recently which should be very similar, if not the same, but there some subtle differences which made me realize this is more tricky than I thought and that I don't 100% understand what is going on. If someone can help, please let me know in the comments.

Invoking `docker` in MSYS2 shell or Git Bash typically fails with complains about paths, for example:

```
$ docker run --rm -it ubuntu /bin/bash
stat C:/Program Files/Git/usr/bin/bash: no such file or directory
```

or

```
docker run -v "$PWD":/var/www/html php:7-apache
# (complains about C:/... path)
```

Note how the Linux path is prepended with `C:/Program Files/Git` which obviously breaks some of the commands.

This happens because MSYS2 shells (which includes Git Bash) translate Linux paths to Windows paths whenever a native Windows binary is called. This is generally a good thing, making it possible to run e.g. `notepad /c/some/file.txt` from Git Bash.

But with Docker, you typically want Linux paths, or, actually, it depends. This simple example is already very tricky:

```
docker run --rm -it -v "$PWD":/tmp/mounted ubuntu /bin/bash
```

- There is volume mapping. Because Docker doesn't support relative paths here, `"$PWD"` or `` `pwd` `` needs to be used. This might resolve to something like `/c/some/path` which might get translated to `C:\\some\\path`. Now it depends whether `docker.exe` supports this or not (it seems it does on Windows).
- The `-t` option might be problematic in mintty and other terminals, yielding an error like _"the input device is not a TTY.  If you are using mintty, try prefixing the command with 'winpty'"_.
- `/bin/bash` should definitely not be translated to `C:/Program Files/Git/usr/bin/bash`, that is plain wrong.

As a base for the workaround, create a small `docker` script (no extension) somewhere in your PATH, and make sure this script is higher-priority than the path of `docker.exe`. Docker is quite aggressive and puts itself very high in the list, the safest way is to become no. 1 **system path** (not user path) to beat it. Here is an example from my computer:

![image](https://user-images.githubusercontent.com/101152/39303507-b980631a-4956-11e8-8374-5182385a15a1.png)

Verify with this:

```
$ type -a docker
docker is /c/Users/Borek/OneDrive/Programs/cmder/bin/docker
docker is /c/Program Files/Docker/Docker/Resources/bin/docker
```

Your script must be first!

Now, put this into your script:

```
#!/bin/bash
winpty "docker.exe" "$@"
```

> ❗ Until https://github.com/Alexpux/MSYS2-packages/issues/411 is resolved, make sure you use winpty binaries from https://github.com/rprichard/winpty/releases, _not_ from `pacman -S winpty`.

That should be it. Try running this:

```
docker run --rm -it -v "$PWD":/tmp/mounted ubuntu /bin/bash
```

Inside the container session, `ls /tmp/mounted` should list your local PWD directory.

If that doesn't work, you can try either the `MSYS_NO_PATHCONV` or `MSYS2_ARG_CONV_EXCL` environment variables. For example, I used this in the past:

```
# Git Bash shell:
(export MSYS_NO_PATHCONV=1; "docker.exe" "$@")

# MSYS2 mingw64 shell:
(export MSYS2_ARG_CONV_EXCL="*"; winpty "docker.exe" "$@")
```

However, I no longer seem to need it. I don't fully understand why.

For `docker-compose`, do the same.

Hope this helps.

---

_This has been originally published [as this Gist](https://gist.github.com/borekb/cb1536a3685ca6fc0ad9a028e6a959e3)._
