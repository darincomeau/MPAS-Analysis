Support Parallel Tasks
======================

<h2>
Xylar Asay-Davis <br>
date: 2017/02/22 <br>
</h2>
<h3> Summary </h3>

Currently, the full analysis suite includes 7 tasks, 5 for the ocean and 2 for sea ice.
The number of tasks is expected to grow over time.  Task parallelism in some
form is needed to allow as many tasks as desired to be run simultaneiously.
Successful completion of this design will mean that the analysis suite produces
identical results to the current `develop` branch but that several analysis
tasks (a number selected by the user) run simultaneously.

<h3> Requirements </h3>

<h4> Requirement: Tasks run simultaneously <br>
Date last modified: 2017/02/22 <br>
Contributors: Xylar Asay-Davis
</h4>
There must be a mechanism for running more than one analysis task simultaneously.

<h4> Requirement: Select maximum number of tasks <br>
Date last modified: 2017/02/22 <br>
Contributors: Xylar Asay-Davis
</h4>
There must be a mechanism for the user to select the maximum number of tasks
to run simultaneously.  This might be necessary to control the number of processors
or the amount of memory used on a given machine or (in the case of running
analysis on login nodes) to be nice to other users on a shared resource.

<h4> Requirement: Lock files written by multiple tasks <br>
Date last modified: 2017/03/10 <br>
Contributors: Xylar Asay-Davis
</h4>
There must be a mechanism for locking files (during either reading or writing) if
they can be written by multiple tasks.  This is necessary to prevent cases where
multiple tasks write to the same file simultaneously or one task reads from a file
at the same time another is writing.

<h4> Consideration: Task parallelism should work on either login or compute nodes <br>
Date last modified: 2017/02/22 <br>
Contributors: Xylar Asay-Davis
</h4>
On some systems, care needs to be taken that scripts run on the compute nodes
rather than on the management node(s).  For example, on Edison and Cori, the
`aprun` command is required to ensure that scripts run on compute nodes.

<h4> Consideration: There may need to be a way to limit the memory used by a task <br>
Date last modified: 2017/02/22 <br>
Contributors: Phillip J. Wolfram, Xylar Asay-Davis
</h4>
It may be that `xarray-dask` with subprocess (or similar) may need to be some
initialization of xarray corresponding to reduced memory available.  For example,
with 10 processes on a node, `xarray` / `dask` should be initialized to use only
1/10th of the memory and CPUs per task. `xarray-dask` may require special
initialization for efficiency and to avoid crashes.

<h3> Algorithmic Formulations </h3>

<h4> Design solution: Tasks run simultaneously <br>
Date last modified: 2017/02/23 <br>
Contributors: Xylar Asay-Davis
</h4>

I propose to have a config option, `parallelTaskCount`, that is the number of concurrent
tasks that are to be performed. If this flag is set to a number greater than 1, analysis
tasks will run concurrently.  To accomplish this, I propose to use `subprocess.call` or
one of its variants within `run_analysis.py`to call itself but with only one task at a
time.  Thus, if `run_analysis.py` gets called with only a single task (whether directly
from the command line or through `subprocess.call`), it would execute that task without
spawning additional subprocesses.

This approach would require having a method for creating a list of individual tasks
to be performed, launching `parallelTaskCount` of those tasks, and then waiting for
them to complete, launching additional tasks as previous tasks complete.  The approach
would also require individual log files for each task, each stored in the log directory
(already a config option).

<h4> Design solution: Select maximum number of tasks <br>
Date last modified: 2017/02/23 <br>
Contributors: Xylar Asay-Davis
</h4>

This is accomplished with the `parallelTaskCount` flag above.  A value of
`parallelTaskCount = 1` would indicate serial execution, though likely still
via launching subprocesses for each task.

The command `subprocess.Popen` allows enough flexibility that it will be possible
to launch several jobs, andthen to farm out additional jobs as each returns. It should
be possible to use a combination of `os.kill(pid, 0)`, which checks if a
process is running, and `os.waitpid(-1,0)`, which waits for any subprocess to finish,
to accomplish launching several processes and waiting until the first one finishes
before launching the next task, or in pseudo-code:
```
processes = launchTasks(taskNames[0:taskCount])
remainingTasks = taskNames[taskCount:]
while len(processes) > 0:
    process = waitForTask(processes)
    processes.pop(process)
    if len(remainingTasks) > 0:
        process = launchTasks(remainingTasks[0])
        proceses.append(process)
        remainingTasks = remainingTasks[1:]
```

Output from the main `run_analysis.py` task will list which analysis tasks were run
and which completed successfully.  The full analysis will exit with an error if one
task fails, but only after attempting to run all desired analysis tasks.  This allows
the failure of one analysis task not to interrupt execution of other analyses.

In a future PR, this work can be expanded to include checking if the appropriate
analysis member (AM) was turned on during the run and skipping any analysis tasks that
depend on that AM if not (Issue #58).

<h4> Design solution: Lock files written by multiple tasks <br>
Date last modified: 2017/03/10 <br>
Contributors: Xylar Asay-Davis
</h4>

Teh design solution is based on the process lock in the fasteners package:
http://fasteners.readthedocs.io/en/latest/examples.html#interprocess-locks

Currently, only mapping files should be written by multiple tasks, requiring locks.

The algorithm consists of 2 changes.  First, I removed the option `overwriteMappingFiles`,
which is now always `False`---if a mapping file exists, it is not overwritten.  This
was necessary because now only one task will write a given mapping file if it doesn't
already exist and the other tasks will wait for it to be written.  Then, all tasks
know there is a valid mapping file that they can read without having to lock the file.

The second change was to add a lock around the subprocess call to `ESMF_RegridWeightGen`
that make sure only one process generates the mapping file.  Each process attempts to
acquire the lock and checks if the mapping file already exists once it acquires the
lock.  If not, it generates the mapping file and releases the lock.  If so, it just
releases the lock and moves on.  Thus, only the first process to acquire the lock
generates the mapping file and the others wait until it is finished.

<h4> Design solution: Task parallelism should work on either login or compute nodes <br>
Date last modified: 2017/02/23 <br>
Contributors: Xylar Asay-Davis
</h4>
For the time being, I propose to address only task parallelism on the login nodes and to
extend the parallelism to work robustly on compute nodes as a separate project.
Nevertheless, I will seek to implement this design in a way that should be conducive to
this later extension.  Likely what will be required is a robust way of adding a prefix
to the commandline (e.g. `aprun -np 1`) when calling subprocesses.  Adding such a prefix
should be relatively simple.

<h4> Design solution: There may need to be a way to limit the memory used by a task <br>
Date last modified: 2017/02/23 <br>
Contributors: Xylar Asay-Davis
</h4>
I am not very familiar with `dask` within `xarray` and I do not intend to address this
consideration directly in this project.  However, on my brief investigation, it seems like
the proper way to handle this may be to have a `chunk` config option either for all tasks
or for individual tasks that can be used to control the size of data in memory. I think
such an approach can be investigated in parallel to this project.  An intermediate solution
for situations where memory is limited would be to set `parallelTaskCount` to a small number.


<h3> Design and Implementation </h3>

This design has been implemented in the test branch https://github.com/xylar/MPAS-Analysis/tree/parallel_tasks

<h4> Implementation: Tasks run simultaneously <br>
Date last modified: 2017/03/10 <br>
Contributors: Xylar Asay-Davis
</h4>

Tasks can now run in parallel.  This has been implemented in these 4 functions within `run_analysis.py`:
```python
def run_parallel_tasks(config, analyses, configFiles, taskCount):
    # {{{
    """
    Run this script once each for several parallel tasks.

    Author: Xylar Asay-Davis
    Last Modified: 03/08/2017
    """

    taskNames = [analysisModule.get_task_name(**kwargs) for
                 analysisModule, kwargs in analyses]

    taskCount = min(taskCount, len(taskNames))

    (processes, logs) = launch_tasks(taskNames[0:taskCount], config,
                                     configFiles)
    remainingTasks = taskNames[taskCount:]
    while len(processes) > 0:
        (taskName, process) = wait_for_task(processes)
        if process.returncode == 0:
            print "Task {} has finished successfully.".format(taskName)
        else:
            print "ERROR in task {}.  See log file {} for details".format(
                taskName, logs[taskName].name)
        logs[taskName].close()
        # remove the process from the process dictionary (no need to bother)
        processes.pop(taskName)

        if len(remainingTasks) > 0:
            (process, log) = launch_tasks(remainingTasks[0:1], config,
                                          configFiles)
            # merge the new process and log into these dictionaries
            processes.update(process)
            logs.update(log)
            remainingTasks = remainingTasks[1:]
    # }}}


def launch_tasks(taskNames, config, configFiles):  # {{{
    """
    Launch one or more tasks

    Author: Xylar Asay-Davis
    Last Modified: 03/08/2017
    """
    thisFile = os.path.realpath(__file__)

    logsDirectory = build_config_full_path(config, 'output',
                                           'logsSubdirectory')
    make_directories(logsDirectory)

    commandPrefix = config.getWithDefault('execute', 'commandPrefix',
                                          default='')
    if commandPrefix == '':
        commandPrefix = []
    else:
        commandPrefix = commandPrefix.split(' ')

    processes = {}
    logs = {}
    for taskName in taskNames:
        args = commandPrefix + [thisFile, '--generate', taskName] + configFiles

        logFileName = '{}/{}.log'.format(logsDirectory, taskName)

        # write the command to the log file
        logFile = open(logFileName, 'w')
        logFile.write('Command: {}\n'.format(' '.join(args)))
        # make sure the command gets written before the rest of the log
        logFile.flush()
        print 'Running {}'.format(taskName)
        process = subprocess.Popen(args, stdout=logFile,
                                   stderr=subprocess.STDOUT)
        processes[taskName] = process
        logs[taskName] = logFile

    return (processes, logs)  # }}}


def wait_for_task(processes):  # {{{
    """
    Wait for the next process to finish and check its status.  Returns both the
    task name and the process that finished.

    Author: Xylar Asay-Davis
    Last Modified: 03/08/2017
    """

    # first, check if any process has already finished
    for taskName, process in processes.iteritems():  # python 2.7!
        if(not is_running(process)):
            return (taskName, process)

    # No process has already finished, so wait for the next one
    (pid, status) = os.waitpid(-1, 0)
    for taskName, process in processes.iteritems():
        if pid == process.pid:
            process.returncode = status
            # since we used waitpid, this won't happen automatically
            return (taskName, process)  # }}}


def is_running(process):  # {{{
    """
    Returns whether a given process is currently running

    Author: Xylar Asay-Davis
    Last Modified: 03/08/2017
    """

    try:
        os.kill(process.pid, 0)
    except OSError:
        return False
    else:
        return True  # }}}
```

<h4> Implementation: Select maximum number of tasks <br>
Date last modified: 2017/03/10 <br>
Contributors: Xylar Asay-Davis
</h4>

There is a configuration option, `parallelTaskCount`, which defaults to 1, meaning tasks run in serial:
```
[execute]
## options related to executing parallel tasks

# the number of parallel tasks (1 means tasks run in serial, the default)
parallelTaskCount = 8
```

<h4> Implementation: Lock files written by multiple tasks <br>
Date last modified: 2017/03/10 <br>
Contributors: Xylar Asay-Davis
</h4>

Here is the code for locking the mapping file within `shared.interpolation.interpolate:
```python
import fasteners
...
# lock the weights file in case it is being written by another process
with fasteners.InterProcessLock(_get_lock_path(outWeightFileName)):
    # make sure another process didn't already create the mapping file in
    # the meantime
    if not os.path.exists(outWeightFileName):
        # make sure any output is flushed before we add output from the
        # subprocess
        subprocess.check_call(args)
```

<h4> Implementation: Task parallelism should work on either login or compute nodes <br>
Date last modified: 2017/03/10 <br>
Contributors: Xylar Asay-Davis
</h4>

I have included a config option `commandPrefix` that *should* be able to be used to
run the analysis on compute nodes.  If the command prefix is empty, the code should run
as normal on the compute nodes.

```
[execute]
## options related to executing parallel tasks

# the number of parallel tasks (1 means tasks run in serial, the default)
parallelTaskCount = 1

# Prefix on the commnd line before a parallel task (e.g. 'srun -n 1 python')
# Default is no prefix (run_analysis.py is executed directly)
commandPrefix = srun -n 1
```

<h4> Implementation: There may need to be a way to limit the memory used by a task <br>
Date last modified: 2017/03/10 <br>
Contributors: Xylar Asay-Davis
</h4>

As mentioned above, I have not addressed this consideration in this project.  Currently,
the suggested approach would be to limit `parallelTaskCount` to a number of tasks that
does not cause memory problems.  More sophisticated approaches could be explored in the
future.

<h3> Testing </h3>

<h4> Testing and Validation: Tasks run simultaneously <br>
Date last modified: 2017/03/10 <br>
Contributors: Xylar Asay-Davis
</h4>

So far, I have tested extensively on my laptop (`parallelTaskCount = 1`, `2`, `4` and `8`)
with the expected results. Later, I will test on Edison and Wolf as well.

<h4> Testing and Validation: Select maximum number of tasks <br>
Date last modified: 2017/03/10 <br>
Contributors: Xylar Asay-Davis
</h4>

Same as above.

<h4> Implementation: Lock files written by multiple tasks <br>
Date last modified: 2017/03/10 <br>
Contributors: Xylar Asay-Davis
</h4>

I ran multiple climatology map tasks at the same time and verified from the log files
that only one created each mapping file.  Others must have waited for that file to be
written or they would have crashed almost immediately when they tried to read the
mapping file during remapping operations.  So I'm confident the code is working as
intended.

<h4> Testing and Validation: Task parallelism should work on either login or compute nodes <br>
Date last modified: 2017/03/10<br>
Contributors: Xylar Asay-Davis
</h4>

On Edison and Wolf, I will test running the analysis with parallel tasks both on login nodes
and by submitting a job to run on the compute nodes (using the appropriate `commandPrefix`).

<h4> Testing and Validation: There may need to be a way to limit the memory used by a task <br>
Date last modified: 2017/03/10 <br>
Contributors: Xylar Asay-Davis
</h4>

Assuming no crashes in my testing on compute nodes with all tasks running in parallel, I will
leave this consideration for investigation in the future.

