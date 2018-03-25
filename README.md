**shell-command-lock - Atomic locking for shell commands.**

https://github.com/jakeogh/shell-command-lock

Prevent identical commands from executing concurrently.
A command is the combination of the program and it's arguments: $0 $*

Requires: sh, sha1sum

**Theory:**

 1. Generate unique and reproducible string from $0 $* (the script file name and all it's arguments). sha1sum($0 $*) is used.
 2. Obtain an atomic lock using the filesystem. Exit on failure (since the same command is already running).
 3. Write $$ (the current PID) to the lockfile. This is not critical, but it is nice to have.
 4. Delete the lockfile.

**Install:**

Place in $PATH.

```
$ sudo ln -s -r shell-command-lock /usr/bin/shell-command-lock
```

**Use:**

insert:
```
. /usr/bin/shell-command-lock
```
_before_ the critical section in the parent script. The lock is removed when the parent script terminates via the trap.

This script should have no effect on the parent script other than locking. The variable names are set readonly to prevent collisions with names in the parent script. The set commands are done in subshells so we don't need to save and restore state.

**Design notes:**

- Attempts to be strictly POSIX compliant (no extensions).
- Does not depend on bash specific features.
- Redirection using noclobber is the atomic locking primitive instead of mkdir because it's faster.

**Benchmarks (mkdir vs noclobber):**
```
$ time for x in {1..24000} ; do /bin/mkdir lock ; /bin/rmdir lock ; done
$ time for x in {1..24000} ; do set -o noclobber; :> lock ; /usr/bin/unlink lock ; done
```

**Bugs: (unavoidable?)**

- IMPORTANT: If the trap is re-defined in the parent script, then that trap will need to handle deleting the lock.
- The lockfile is orphaned if a exit signal happens after the lock is obtained and before the trap is set.

**More information:**

 - man flock
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

