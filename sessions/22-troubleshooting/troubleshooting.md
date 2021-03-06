layout: true
class: inverse, middle, large

---
class: special
# Galaxy Troubleshooting

slides by @martenson, @natefoo

.footnote[\#usegalaxy / @galaxyproject]

---
class: larger

## Please Interrupt!

We like your questions.

---
class: special, middle
# Everything always goes wrong

---
# Database migration

In case you see database migration errors during startup you can:

* Check DB table `migrate_version` column `version`.
* Check folder `lib/galaxy/model/migrate/versions/` - the latest migration should match the DB.
* Clean the `*.pyc` files to make sure there is no remnant from other code revisions.
  * Something like `$ find lib -name "*.pyc" -delete` should work.

---
# Mako templates

In case you see page inconsistencies or template errors in logs.

* Clean the `*.pyc` files to make sure there is no remnant from other code revisions.
  * Something like `$ find database/compiled_templates name "*.pyc" -delete`.

---
# Client build

In case your javascript/css local changes are not visible.

* JavaScript files are located in `client/galaxy/scripts`.
* CSS files are in `client/galaxy/style/`.
* Check `client/README.md` for details regarding client build.
* Run `$ make client` from root folder.
  * You can also check the `Makefile` in root for other useful targets.

???
* This should only matter when you modify some of the Galaxy's static assets.
* Does not affect javascript or style in mako templates.

---
class: normal
# Missing tools

* Restart Galaxy and watch the log for
```shell
Loaded tool id: toolshed.g2.bx.psu.edu/repos/iuc/sickle/sickle/1.33, version: 1.33 into tool panel....
```

* Check `integrated_tool_panel.xml` for
```xml
<tool id="toolshed.g2.bx.psu.edu/repos/iuc/sickle/sickle/1.33" />
```

* If it is TS tool check `config/shed_tool_conf.xml` for
```xml
<tool file="toolshed.g2.bx.psu.edu/repos/iuc/sickle/43e081d32f90/sickle/sickle.xml" guid="toolshed.g2.bx.psu.edu/repos/iuc/sickle/sickle/1.33">
...
</tool>
```

---
class: smaller
# Jobs aren't running (always gray)

Check the Galaxy server log for errors. Successful job lifecycle:

```console
galaxy.jobs DEBUG 2016-11-03 20:11:54,897 (2) Working directory for job is: /home/galaxyguest/galaxy/database/jobs_directory/000/2
galaxy.jobs.handler DEBUG 2016-11-03 20:11:54,903 (2) Dispatching to local runner
galaxy.jobs DEBUG 2016-11-03 20:11:54,939 (2) Persisting job destination (destination id: local:///)
galaxy.jobs.runners DEBUG 2016-11-03 20:11:55,205 Job [2] queued (301.934 ms)
galaxy.jobs.handler INFO 2016-11-03 20:11:55,238 (2) Job dispatched
galaxy.jobs.command_factory INFO 2016-11-03 20:11:55,702 Built script [/home/galaxyguest/galaxy/database/jobs_directory/000/2/tool_script.sh] for ...
galaxy.jobs.runners DEBUG 2016-11-03 20:11:56,395 (2) command is: mkdir -p working; cd working; ...
galaxy.jobs.runners.local DEBUG 2016-11-03 20:11:56,410 (2) executing job script: /home/galaxyguest/galaxy/database/jobs_directory/000/2/galaxy_2.sh
galaxy.jobs DEBUG 2016-11-03 20:11:56,424 (2) Persisting job destination (destination id: local:///)
galaxy.jobs.runners.local DEBUG 2016-11-03 20:11:59,665 execution finished: /home/user/galaxy/database/jobs_directory/000/2/galaxy_2.sh
galaxy.model.metadata DEBUG 2016-11-03 20:11:59,821 loading metadata from file for: HistoryDatasetAssociation 2
galaxy.jobs INFO 2016-11-03 20:12:00,577 Collecting metrics for Job 2
galaxy.jobs DEBUG 2016-11-03 20:12:00,872 job 2 ended (finish() executed in (1206.709 ms))
galaxy.model.metadata DEBUG 2016-11-03 20:12:00,891 Cleaning up external metadata files
```

---
# Jobs aren't running -  gray (!)

Not yet picked up by the Galaxy job handler subsystem

Solutions:
- If single or multiprocess
  - Ensure that handler ID(s) in `job_conf.xml` match `--server-name`(s)
- If multiprocess, check to ensure that your job handler(s) are running
  - If yes, restart handler(s)

---
# Jobs aren't running - gray clock

A handler has seen this job

Verify concurrency limits unmet in `job_conf.xml`:
- Local jobs: value of plugin `workers` attribute (default: 4)
- Entire `<limits>` section

Exceeding disk quota also pauses jobs, but users are notified of this.

---
# Jobs aren't running - gray clock

Check database for:
- State
- Destination id assigned?
- Jobs owned by user in non-terminal state if limits in use

---
# Jobs aren't finishing - yellow

The tool/dependency is no longer executing but the job is still "running"

Solutions:
- Check cluster (advanced) for job
- Check last log message for job
- Setting/collecting metadata can be slow: wait
- Check handlers as with gray (!)

---
# Quota not being updated for a certain user

In the Admin/Users menu you can run `Recalculate Disk Usage` on a certain user.

There is also a command line version: `scripts/set_user_disk_usage.py`

![Recalculate Disk Usage Option](images/admin_recalc_usage.png)

---
class: special
# Tool errors

Tools can fail for a variety of reasons, some valid, some invalid.

Some made up examples follow.

---
# Tool errors - stderr

Tool stderr contains:
```shell
Warning: File /galaxy/job/working/foo.tmp created in the future!
```

The system running the job has an incorrect clock, this does not affect the correctness of results.

Solutions:
- Fix the clock
- If the wrapped tool uses proper exit codes, use `<tool profile="16.04">` to ignore stderr

---
# Tool errors - stderr

Tool stderr contains:
```shell
Warning: Discarded 10000 lines of /path/to/input/dataset_1.dat because they looked funny
```

Maybe a problem, maybe not.

Solutions:
- Check tool input(s) and parameters
- Verify input is not corrupt
- If behavior is expected and the tool uses proper exit codes, use `<tool profile="16.04">` to ignore stderr

---
# Tool errors - memory errors

Tool stderr contains one of:
```shell
MemoryError                 # Python
what():  std::bad_alloc     # C++
Segmentation Fault          # C - but could be other problems too
Killed                      # Linux OOM Killer
```

Solutions:
- Change input sizes or params
  - Map/reduce?
- Decrease the amount of memory the tool needs
- Increase the amount of memory available to the job
  - Stop other jobs
  - Add memory
  - Cluster (advanced)
- Cross your fingers and rerun the job

---
# Tool errors - system errors

Tool stderr contains:
```shell
open(): /path/to/input/dataset_1.dat: No such file or directory
```

Verify that `dataset_1.dat` exists.

Solution:
- Fix the filesystem error (NFS?) and rerun the job

---
# Tool errors - dependency problems

Tool stderr contains:
```shell
sh: command not found: samtools
```

Solutions:
- If this is the upload tool and the missing command is really `samtools`
  - install `samtools` on `$PATH` or `<tool_dependencies_dir>/samtools/default`
- Else
  - Verify that tool dependencies are properly installed
  - Verify that `tool_dependency_dir` is accessible

---
# Tool errors - dependency problems

Tool stderr contains:
```shell
foo: /lib/x86_64-linux-gnu/libc.so.6: version 'GLIBC_2.23' not found (required by foo)
```

`foo` was compiled against Glibc 2.23 but Glibc < 2.23 is installed.

Solutions:
- Verify that tool dependencies were properly installed
- Recompile `foo` on the "oldest" system on which it might run

---
# Tool errors - dependency problems

Tool stderr contains:
```shell
foo: error while loading shared libraries: libhitch.so.42: cannot open shared object file: No such file or directory
```

`foo` was compiled against `libhitch.so.42` but it's not on the runtime linker path

Solutions:
- Verify that tool dependencies were properly installed
- Modify dependency's `env.sh` to set `$LD_LIBRARY_PATH` as appropriate
- Install `libhitch` on target system
- Recompile `foo` with `-Wl,-rpath=/path/to/libhitch/dir`
- Recompile `foo` without `-lhitch` (if possible)

---
# Tool errors - Empty green history item

1. The tool is not correctly detecting error conditions: stderr, exit code?
2. The tool correctly produced an empty dataset for the given params, inputs

Solutions:
1. Fix the tool wrapper to detect errors
2. User education

---
# One last word on tool errors

All Devteam/IUC tools have tests

Use these tests to verify that the tool works in the basic case

If yes:
- Input/parameter problem
- Tool wrapper bug
- Tool bug
- Resource problem (sysadmin problem)
If no:
- Sysadmin problem

---
# Galaxy UI is slow

- Investigate load
  - Web server(s)
  - Database server
- Investigate memory usage, swapping
- Investigate iowait
  - Use iostat
- Use `gdb` (demo!)

---
# Galaxy UI is slow

- Use heartbeat (demo!)

---
# Local or Network FS slow/down

- Use `iostat` (demo!)
- Use `time dd` (demo!)

Solutions:
- Don't put the Galaxy server in that FS. Local distribution, CVMFS, ??
- Get a better network FS

---
# NFS caching errors

Galaxy gets "No such file or directory" for files that exist. NFS attribute caching is to blame. Set:

```ini
retry_job_output_collection = 5
```

[retry_job_output_collection](https://github.com/galaxyproject/galaxy/blob/dev/lib/galaxy/jobs/__init__.py#L1229)

---
# Hung server processes

- Use `uwsgitop` (demo!)

- `uwsgitop` state busy?
  -  5      12.7    3453    59962   1       13      0       **busy**    89ms    0       0       536.0M  1       0       450m    09:35:36
- Process state D?
  - g2main    3440 17.2  5.2 1696480 863536 ?      **D**    Nov10 167:46 /cvmfs/main.galaxyproject.org/venv/bin/uwsgi --ini /srv/galaxy/main/config/uwsgi.ini

Kernel uninterruptable sleep - normal unless stuck. Probably IO. Check filesystems, disks.

---
# Hung server processes

- Investigate `/proc/pid` especially `fd`
- Use `strace` (demo!)

---
# Undead processes

```
bind(): Address already in use [core/socket.c line 769]
```

Use `lsof -i :<port>` (demo!)

---
# pkill considered useful

```console
$ sudo pkill -INT -u galaxy uwsgi
```

---
# Remember ALL the log files

All may be relevant:
- uWSGI log
- handler logs
- Pulsar logs
- nginx access, error logs
- syslog/messages
- authlog
- browser console log

---
# Data Manager loading failure

![dm404-1.png](images/dm404-1.png)

---
# Data manager loading failure

![dm404-2.png](images/dm404-2.png)

---
# Data manager loading failure

![dm404-3.png](images/dm404-3.png)

---
# Data manager loading failure

Solution: Add to `/etc/apache2/sites-available/000-galaxy.conf`:
```apache
AllowEncodedSlashes on
```

---
# Database problems

Slow queries, high load, etc.

- Use `EXPLAIN ANALYZE` (demo!)
- Use `VACUUM ANALYZE`

Increase `shared_buffers`. 2GB on Main (16GB of memory on VM)

---
# error: [Errno 32] Broken pipe

This error is not indicative of any kind of failure. It just means that the client closed a connection before the server finished sending a response.

---
# Installation failures - Tool dependencies

```
utils.c:33:18: fatal error: zlib.h: No such file or directory
compilation terminated.
make: *** [utils.o] Error 1
```

Find `zlib.h` on [packages.ubuntu.com](http://packages.ubuntu.com/)

---
# Installation failures - Python dependencies

```
psutil/_psutil_linux.c:12:20: fatal error: Python.h: No such file or directory
compilation terminated.
error: command 'x86_64-linux-gnu-gcc' failed with exit status 1
```

Find `Python.h` on [packages.ubuntu.com](http://packages.ubuntu.com/)

---
# uWSGI errors

```
uwsgi: unrecognized option '--ini-paste'
```

Make sure you know which uWSGI you're running.

`/usr/bin/python` needs the `uwsgi-plugins-python` package and requires:

```console
$ uwsgi --plugin python
```

---
# Job failures

"Unable to run job due to a misconfiguration of the Galaxy job running system.  Please contact a site administrator."

There is a traceback in the Galaxy log. Go find it.

---
# Node failures

```console
$ sinfo -Nel
Fri Nov 11 09:50:01 2016
NODELIST   NODES PARTITION       STATE CPUS    S:C:T MEMORY TMP_DISK WEIGHT FEATURES REASON
localhost      1    debug*        down    2    2:1:1      1        0      1   (null) Node unexpectedly rebooted
```

Figure out why it rebooted and:

```console
$ sudo scontrol update nodename=localhost state=resume
```

---
class: special, middle
# Any troubling Galaxy situations you have?

---
# Where to get help

- [Galaxy Biostar](https://biostar.usegalaxy.org/)
- [galaxy-dev Mailing list](http://dev.list.galaxyproject.org/)
- [IRC](https://wiki.galaxyproject.org/Support/IRC): \#galaxyproject on Freenode
- [Wiki: Support](https://wiki.galaxyproject.org/Support)
