
https://github.com/jakeogh/commandlock

## commandlock - Advisory locking for processes launching

Prevent identical commands with identical arguments from executing concurrently.


**Theory:**

 1. Generate unique and reproducible string from `$0 $*` (the script name and all it's arguments). `sha1sum($0 $*)` is used.
 2. Obtain an atomic* lock using the filesystem. Exit on failure (since the same command is already running).
 3. Execute code, any attempts to execute the same command should fail.
 4. Release the lock by deleting the lockfile (this is done with a `trap`).
 5. Exit.

`* filesystem and kernel dependent`


**Files:**
- `lock`: bash-specific shell script to launch a process with `commandlock`
- `commandlock`: advisory locking code, only used by sourcing, see `lock` for example
- `commandlock-test`: test script


**General Install:**

Place in `$PATH`

```
    git clone https://github.com/jakeogh/commandlock
    sudo cp commandlock/commandlock /usr/bin/commandlock
    sudo cp commandlock/lock /usr/bin/lock
```

**Gentoo Install:**
```
   su
   layman -o https://raw.githubusercontent.com/jakeogh/jakeogh/master/jakeogh.xml -f -a jakeogh
   echo "source /var/lib/layman/make.conf" >> /etc/portage/make.conf
   layman -S
   emerge commandlock
```

###Usage:


Dont `factor` twice:
```
$ lock time factor `printf "%0.s7" {1..29}` & lock time factor `printf "%0.s7" {1..29}`
[1] 15549
ERROR: [15549] /usr/bin/lock time factor 77777777777777777777777777777 Locking failed: /dev/shm/commandlock_185c131569abe83db71f5af0d51e3cdba09c0f85
ERROR: [15549] /usr/bin/lock time factor 77777777777777777777777777777 is/was already running. Exiting (1).
77777777777777777777777777777: 7 3191 16763 43037 62003 77843839397
0.00user 0.00system 0:00.00elapsed 95%CPU (0avgtext+0avgdata 2064maxresident)k
0inputs+0outputs (0major+94minor)pagefaults 0swaps
[1]+  Exit 1                  lock time factor `printf "%0.s7" {1..29}`
```


Make sure a `wget` job is unique:
```
$ lock wget http://www.wrh.noaa.gov/images/twc/granalyst/kemx_cr_0.jpg
```


Example deliberately attempting to run the same command at the same time, like the `factor` example:
```
$ lock curl https://www.wrh.noaa.gov/images/twc/granalyst/kemx_cr_0.jpg -o /dev/null & lock curl https://www.wrh.noaa.gov/images/twc/granalyst/kemx_cr_0.jpg -o /dev/null
[1] 26842
ERROR: [26842] /usr/bin/lock curl https://www.wrh.noaa.gov/images/twc/granalyst/kemx_cr_0.jpg -o /dev/null Locking failed: /dev/shm/commandlock_cb201bdd3d9970730fa58471d8f554db966694d3
ERROR: [26842] /usr/bin/lock curl https://www.wrh.noaa.gov/images/twc/granalyst/kemx_cr_0.jpg -o /dev/null is/was already running. Exiting (1).
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  698k  100  698k    0     0   334k      0  0:00:02  0:00:02 --:--:--  334k
[1]+  Exit 1                  lock curl https://www.wrh.noaa.gov/images/twc/granalyst/kemx_cr_0.jpg -o /dev/null

```


or to use it to only lock a specific section of a script, insert:
```
. commandlock || exit 1
```
_before_ the critical section in the parent script. The lock is removed via the trap when the parent script terminates.


Sourcing `commandlock` should have no effect on the parent script other than locking.
- The variable names are set readonly to prevent collisions with names in the parent script.
- The set commands are done in subshells so there is no state to save and restore.

**Design notes:**

- Attempts to be POSIX compliant.
- Locking code in `commandlock` does not depend on bash-specific features.
- Redirection using `-o noclobber` is the atomic locking primitive instead of `mkdir` because it's faster.

**Benchmark mkdir vs noclobber:**
```
$ time for x in {1..24000} ; do /bin/mkdir lock ; /bin/rmdir lock ; done
$ time for x in {1..24000} ; do set -o noclobber; :> lock ; /usr/bin/unlink lock ; done
```

**Bugs: (unavoidable?)**

- If the trap is re-defined in the parent script, that trap will need to handle deleting the lock.
- The lockfile is orphaned if an exit signal happens after the lock is obtained and before the trap is set.

**Requires:**

 - sh
 - sha1sum

**More info:**

 - `man flock`
 - http://www.davidpashley.com/articles/writing-robust-shell-scripts.html
 - http://wiki.bash-hackers.org/howto/mutex
 - http://mywiki.wooledge.org/BashFAQ/045
 - http://wiki.grzegorz.wierzowiecki.pl/code:mutex-in-bash
 - http://code.google.com/p/pylockfile/
 - https://github.com/skx/sysadmin-util/blob/master/with-lock
 - https://github.com/jaysoffian/dotlock
 - http://sysadvent.blogspot.com/2008/12/day-9-lock-file-practices.html
 - http://apenwarr.ca/log/?m=201012#13
 - https://news.ycombinator.com/item?id=2000349
 - https://github.com/WoLpH/portalocker
 - https://news.ycombinator.com/item?id=22212338
