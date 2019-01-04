
The net has plenty[1] of questions that boil down to "my app has color output, but the color is lost when I pipe or redirect it".

Glossig over the details, in general it's because the app detects that it's not outputing to a terminal directly and does not output the ASCII color codes.

The solution is normally to ask the app to output the ASCII color codes anyway. This is spp specific.

colorpipe aims to be a general solution to the problem, independent of app support.

--Usage--

```
./colorpipe /usr/bin/tree | head
```


--Details--

```
# this is an old clone of https://github.com/gentoo/gentoo
$ cd gentoo

$ time /usr/bin/git --no-pager log | head
commit 6fd47a431fc14b9408099905b36630fdb8ff73b9
Author: Kent Fredric <kentnl@gentoo.org>
Date:   Sun May 21 07:54:45 2017 +1200

    dev-perl/Devel-Cover: Bump to version 1.250.0 re bug #615730
    
    Upstream:
    - Fix for 5.25.x op_sibling
    - Drop support for perl <5.10
    - Fix compile with MSVC

real    0m0.005s
user    0m0.003s
sys     0m0.001s

# bummer no color. ah well, some apps have flags to override their tty output detection,
# like ls: "/bin/ls -al --color=always | head" outputs the first 10 lines in color.
# I am sure git has a similar option.
# What about a general solution? The expect package has unbuffer, which fools the app into
# thinking it's outputting to the terminal

$ time /usr/bin/unbuffer /usr/bin/git --no-pager log | head
commit 6fd47a431fc14b9408099905b36630fdb8ff73b9 (HEAD -> x11-wm/qtile, upstream/master, origin/master, origin/HEAD, master)
Author: Kent Fredric <kentnl@gentoo.org>
Date:   Sun May 21 07:54:45 2017 +1200

    dev-perl/Devel-Cover: Bump to version 1.250.0 re bug #615730
    
    Upstream:
    - Fix for 5.25.x op_sibling
    - Drop support for perl <5.10
    - Fix compile with MSVC

real    0m5.257s
user    0m5.866s
sys     0m4.581s

# cool, that worked, it's in color... but it took a long time... 5.257s vs 0.005.
# So it's waiting for git to exit before returning.
# time to ask #bash
# #bash koala_man:
#    if you don't care that the command runs for a while, you can just do
#    head < <(unbuffer /usr/bin/git --no-pager log)

$ time head < <(/usr/bin/unbuffer /usr/bin/git --no-pager log)
commit 6fd47a431fc14b9408099905b36630fdb8ff73b9 (HEAD -> x11-wm/qtile, upstream/master, origin/master, origin/HEAD, master)
Author: Kent Fredric <kentnl@gentoo.org>
Date:   Sun May 21 07:54:45 2017 +1200

    dev-perl/Devel-Cover: Bump to version 1.250.0 re bug #615730
    
    Upstream:
    - Fix for 5.25.x op_sibling
    - Drop support for perl <5.10
    - Fix compile with MSVC

real    0m0.029s
user    0m0.001s
sys     0m0.000s

# nice, it has color, and it's fast. but... we know git kept running in the background. double cheking:

$ head < <(time /usr/bin/unbuffer /usr/bin/git --no-pager log)
commit 6fd47a431fc14b9408099905b36630fdb8ff73b9 (HEAD -> x11-wm/qtile, upstream/master, origin/master, origin/HEAD, master)
Author: Kent Fredric <kentnl@gentoo.org>
Date:   Sun May 21 07:54:45 2017 +1200

    dev-perl/Devel-Cover: Bump to version 1.250.0 re bug #615730
    
    Upstream:
    - Fix for 5.25.x op_sibling
    - Drop support for perl <5.10
    - Fix compile with MSVC
$ 
real    0m5.054s
user    0m5.858s
sys     0m4.282s


# Yep. So not a good option for a general solution.
# The command could just keep running forever because we wanted the first 10 lines.
# ask #tcl
# #tcl aspect:
#    I think the simplest script to dwyw is something like: trap exit SIGPIPE; spawn -noecho {*}$argv; expect

$ echo -e '''#!/usr/bin/env tclsh'''"\npackage require Expect\n"'''trap exit SIGPIPE; spawn -noecho {*}$argv; expect''' > colorpipe && chmod +x colorpipe

$ cat colorpipe
#!/usr/bin/env tclsh
package require Expect
trap exit SIGPIPE; spawn -noecho {*}$argv; expect

$ time ./nuunbuffer /usr/bin/git --no-pager log | head
commit 6fd47a431fc14b9408099905b36630fdb8ff73b9 (HEAD -> x11-wm/qtile, upstream/master, origin/master, origin/HEAD, master)
Author: Kent Fredric <kentnl@gentoo.org>
Date:   Sun May 21 07:54:45 2017 +1200

    dev-perl/Devel-Cover: Bump to version 1.250.0 re bug #615730
    
    Upstream:
    - Fix for 5.25.x op_sibling
    - Drop support for perl <5.10
    - Fix compile with MSVC

real    0m0.021s
user    0m0.008s
sys     0m0.010s

# Perfect. it's in color, it's fast and it does not leave a process running.
# Lets try it on /usr/bin/tree (over the main gentoo ebuild repo)

$ time /usr/bin/tree | wc -l
126149

real    0m0.656s
user    0m0.390s
sys     0m0.281s

$ /usr/bin/tree
.
├── app-accessibility
│   ├── SphinxTrain
│   │   ├── Manifest
│   │   ├── SphinxTrain-0.9.1-r1.ebuild
│   │   ├── SphinxTrain-1.0.8.ebuild
│   │   ├── files
│   │   │   ├── gcc.patch
│   │   │   └── gcc34.patch
│   │   └── metadata.xml


^C <snip>

# by default, tree outputs in color and it did here.

$ time /usr/bin/tree | head
.
├── app-accessibility
│   ├── SphinxTrain
│   │   ├── Manifest
│   │   ├── SphinxTrain-0.9.1-r1.ebuild
│   │   ├── SphinxTrain-1.0.8.ebuild
│   │   ├── files
│   │   │   ├── gcc.patch
│   │   │   └── gcc34.patch
│   │   └── metadata.xml

real    0m0.003s
user    0m0.000s
sys     0m0.003s

# no color. ok try unbuffer

$ time /usr/bin/unbuffer /usr/bin/tree | head
.
├── app-accessibility
│   ├── SphinxTrain
│   │   ├── Manifest
│   │   ├── SphinxTrain-0.9.1-r1.ebuild
│   │   ├── SphinxTrain-1.0.8.ebuild
│   │   ├── files
│   │   │   ├── gcc.patch
│   │   │   └── gcc34.patch
│   │   └── metadata.xml

real    0m1.388s
user    0m1.391s
sys     0m1.247s

# That worked, its in color, but had the expected long delay.
# now colorpipe:

$ time ./colorpipe /usr/bin/tree | head
.
├── app-accessibility
│   ├── SphinxTrain
│   │   ├── Manifest
│   │   ├── SphinxTrain-0.9.1-r1.ebuild
│   │   ├── SphinxTrain-1.0.8.ebuild
│   │   ├── files
│   │   │   ├── gcc.patch
│   │   │   └── gcc34.patch
│   │   └── metadata.xml

real    0m0.020s
user    0m0.008s
sys     0m0.010s

# seems like a general solution to the "force color through the pipe" problem, independent of app support.
```

[1]
* <https://unix.stackexchange.com/questions/19317/can-less-retain-colored-output>
* <https://bbs.archlinux.org/viewtopic.php?id=67982>
* <https://superuser.com/questions/417957/is-there-any-way-to-keep-text-passed-to-head-tail-less-etc-to-be-colored>
* <https://superuser.com/questions/385768/less-emulate-a-tty-to-preserve-piped-color-output/385799>
* <https://stackoverflow.com/questions/2327191/preserve-colouring-after-piping-grep-to-grep>
* <https://stackoverflow.com/questions/1312922/detect-if-stdin-is-a-terminal-or-pipe/7601564>


