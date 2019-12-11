---
layout: spec
---

EECS 485 Project 4: Map Reduce
==============================
{: .primer-spec-toc-ignore }

Due: 8pm on March 27th, 2019. This is a group project to be completed in groups of two to three.


# Changelog


# Introduction
In this project, you will implement a MapReduce server in Python. This will be a single machine, multi-process, multi-threaded server that will execute user-submitted MapReduce jobs. It will run each job to completion, handling failures along the way, and write the output of the job to a given directory. Once you have completed this project, you will be able to run any MapReduce job on your machine, using a MapReduce implementation you wrote!

There are two primary modules in this project: the `Master`, which will listen for MapReduce jobs, manage the jobs, distribute work amongst the workers, and handle faults.  `Worker` modules register themselves with the master, and then await commands, performing map, reduce or sorting (grouping) tasks based on instructions given by the master.

You will not write map reduce programs, but rather the MapReduce server. We have provided several sample map/reduce programs that you can use to test your MapReduce server.

Refer to the [Python processes, threads and sockets tutorial](python-processes-threads-sockets.html) for background and examples.

**Table of Contents**
- [Setup](#setup)
- [Run the MapReduce Server](#run-the-mapreduce-server)
- [MapReduce Server Specification](#mapreduce-server-specification)
- [Walk-through example](#walk-through-example)
- [Testing](#testing)
- [Submitting and grading](#submitting-and-grading)
- [Further Reading](#further-reading)


# Setup

### Group registration
Register your group on the [Autograder](https://autograder.io).

### Project folder
Create a folder for this project ([instructions](https://eecs485staff.github.io/p1-insta485-static/setup_os.html#create-a-folder)).  Your folder location might be different.

```console
$ pwd
/Users/awdeorio/src/eecs485/p4-mapreduce
```

### Version control
Set up version control using the [Version control tutorial](https://eecs485staff.github.io/p1-insta485-static/setup_git.html).  You might also take a second look at the [Version control for a team](https://eecs280staff.github.io/p1-stats/setup_git.html#version-control-for-a-team) tutorial.

After you're done, you should have a local repository with a "clean" status and your local repository should be connected to a remote GitLab repository.
```console
$ pwd
/Users/awdeorio/src/eecs485/p4-mapreduce
$ git status
On branch master
Your branch is up-to-date with 'origin/master'.

nothing to commit, working tree clean
$ git remote -v
origin	https://gitlab.eecs.umich.edu/awdeorio/p4-mapreduce.git (fetch)
origin	https://gitlab.eecs.umich.edu/awdeorio/p4-mapreduce.git (push)
```

You should have a `.gitignore` file ([instructions](setup_git.html#add-a-gitignore-file)).
```console
$ pwd
/Users/awdeorio/src/eecs485/p4-mapreduce
$ head .gitignore
This is a sample .gitignore file that's useful for EECS 485 projects.
...
```

### Python virtual environment
Create a Python virtual environment using the Project 1 [Python Virtual Environment Tutorial](https://eecs485staff.github.io/p1-insta485-static/setup_virtual_env.html).

Check that you have a Python virtual environment, and that it's activated (remember `source env/bin/activate`).
```console
$ pwd
/Users/awdeorio/src/eecs485/p4-mapreduce
$ ls -d env
env
$ echo $VIRTUAL_ENV
/Users/awdeorio/src/eecs485/p4-mapreduce/env
```

### Starter files
Download and unpack the starter files.
```console
$ pwd
/Users/awdeorio/src/eecs485/p4-mapreduce
$ wget https://eecs485staff.github.io/p4-mapreduce/starter_files.tar.gz
$ tar -xvzf starter_files.tar.gz
```

Move the starter files to your project directory and remove the original `starter_files/` directory and tarball.
``` console
$ pwd
/Users/awdeorio/src/eecs485/p4-mapreduce
$ mv starter_files/* .
$ rm -rf starter_files starter_files.tar.gz
```

You should see these files.
``` console
$ tree
.
├── mapreduce
│   ├── __init__.py
│   ├── master
│   │   ├── __init__.py
│   │   └── __main__.py
│   ├── submit.py
│   ├── utils.py
│   └── worker
│       ├── __init__.py
│       └── __main__.py
├── setup.py
├── tests
│   ├── correct
│   │   ├── grep_correct.txt
│   │   └── word_count_correct.txt
│   ├── exec
│   │   ├── grep_map.py
│   │   ├── grep_reduce.py
│   │   ├── wc_map.sh
│   │   ├── wc_map_slow.sh
│   │   ├── wc_reduce.sh
│   │   └── wc_reduce_slow.sh
│   ├── input
│   │   ├── file01
            ...
│   │   └── file08
│   ├── input_small
│   │   ├── file01
│   │   └── file02
│   ├── testdata
        ...
│   ├── test_worker_8.py
│   └── utils.py
└── VERSION
```

Here's a brief description of each of the starter files.

| `VERSION` | Version of the starter files |
| `mapreduce` | mapreduce Python package skeleton files |
| `mapreduce/master/` | mapreduce master skeleton module, implement this |
| `mapreduce/worker/` | mapreduce worker skeleton module, implement this |
| `mapreduce/submit.py` | Provided code to submit a new MapReduce job |
| `mapreduce/utils.py` | Code shared between master and worker |
| `setup.py` | mapreduce Python package configuration |
| `tests/` | Public unit tests |
| `tests/exec/` | Sample mapreduce programs, all use stdin and stdout |
| `tests/correct/` | Sample mapreduce program correct output |
| `tests/input/` | Sample mapreduce program input |
| `tests/input_small/` | Sample mapreduce program input for fast testing |
| `testdata/` | Files used by our public tests |

Before making any changes to the clean starter files, it's a good idea to make a commit to your Git repository.

### Libraries
Complete the [Python processes, threads and sockets tutorial](python-processes-threads-sockets.html).

Here are some quick links to the libraries we used in our instructor implementation.
- [Python Multithreading](https://docs.python.org/3/library/threading.html)
- [Python Sockets](https://docs.python.org/3/library/socket.html)
- [Python SH Module](https://amoffat.github.io/sh/)
- [Python JSON Library](https://docs.python.org/3/library/json.html)
- [Python Logging facility](https://docs.python.org/3/library/logging.html)
  - We've also provided [sample logging code](example.html#logging)


# Run the MapReduce server
You will write a `mapreduce` Python package includes `master` and `worker` modules.  Launch a master with the command line entry point `mapreduce-master` and a worker with `mapreduce-worker`.  We've also provided `mapreduce-submit` to send a new job to the master.

### Start a master and workers
The starter code will run out of the box, it just won't do anything. The master and the worker run as seperate processes, so you will have to start them up separately. This will start up a master which will listen on port 6000 using TCP. Then, we start up two workers, and tell them that they should communicate with the master on port 6000, and then tell them which port to listen on. The ampersand (`&`) means to start the process in the background.
```console
$ mapreduce-master 6000 &
$ mapreduce-worker 6000 6001 &
$ mapreduce-worker 6000 6002 &
```

See your processes running in the background.  Note: use `pgrep -lf` on OSX and `pgrep -af` on GNU/Linux systems.
```console
$ pgrep -lf mapreduce-worker
15364 /usr/local/Cellar/python3/3.6.3/Frameworks/Python.framework/Versions/3.6/Resources/Python.app/Contents/MacOS/Python /Users/awdeorio/src/eecs485/p4-mapreduce/env/bin/mapreduce-worker 6000 6001
15365 /usr/local/Cellar/python3/3.6.3/Frameworks/Python.framework/Versions/3.6/Resources/Python.app/Contents/MacOS/Python /Users/awdeorio/src/eecs485/p4-mapreduce/env/bin/mapreduce-worker 6000 6002
$ pgrep -lf mapreduce-master
15353 /usr/local/Cellar/python3/3.6.3/Frameworks/Python.framework/Versions/3.6/Resources/Python.app/Contents/MacOS/Python /Users/awdeorio/src/eecs485/p4-mapreduce/env/bin/mapreduce-master 6000
```

Stop your processes.
```console
$ pkill -f mapreduce-master
$ pkill -f mapreduce-worker
$ pgrep -lf mapreduce-worker  # no output, because no processes
$ pgrep -lf mapreduce-master  # no output, because no processes
```

### Submit a MapReduce job
Lastly, we have also provided `mapreduce/submit.py`. It sends a job to the Master's main TCP socket.  You can specify the job using command line arguments.
```console
$ mapreduce-submit --help
Usage: mapreduce-submit [OPTIONS]

  Top level command line interface.

Options:
  -p, --port INTEGER      Master port number, default = 6000
  -i, --input DIRECTORY   Input directory, default=./tests/input
  -o, --output DIRECTORY  Output directory, default=./output
  -m, --mapper FILE       Mapper executable, default=./tests/exec/wc_map.sh
  -r, --reducer FILE      Reducer executable,
                          default=./tests/exec/wc_reduce.sh
  --nmappers INTEGER      Number of mappers, default=4
  --nreducers INTEGER     Number of reducers, default=1
  --help                  Show this message and exit.
```

Here's how to run a job.  Later, we'll simplify starting the server using a shell script.  Right now we expect the job to fail because Master and Worker are not implemented.
```console
$ pgrep -f mapreduce-master  # check if you already started it
$ pgrep -f mapreduce-worker  # check if you already started it
$ mapreduce-master 6000 &
$ mapreduce-worker 6000 6001 &
$ mapreduce-worker 6000 6002 &
$ mapreduce-submit --mapper tests/exec/wc_map.sh --reducer tests/exec/wc_reduce.sh
```

### Init script
The MapReduce server is an example of a *service* (or *daemon*), a program that runs in the background.  We'll write an *init script* to start, stop and check on the map reduce master and worker processes.  It should be a shell script named `bin/mapreduce`.  Print the messages in the following examples.

Be sure to follow the shell script best practices ([Tutorial](https://eecs485staff.github.io/p1-insta485-static/#utility-scripts)).


### Start server
{: .primer-spec-toc-ignore }
Exit 1 if a master or worker is already running.  Otherwise, execute the following commands.
```bash
mapreduce-master 6000 &
sleep 2
mapreduce-worker 6000 6001 &
mapreduce-worker 6000 6002 &
```

Example
```console
$ ./bin/mapreduce start
starting mapreduce ...
```

Example: accidentally start server when it's already running.
```console
$ ./bin/mapreduce start
Error: mapreduce-master is already running
```

### Stop server
{: .primer-spec-toc-ignore }
Execute the following commands.  Notice that `|| true` will prevent a failed "nice" shutdown message from causing the script to exit early.  Also notice that we automatically figure out the correct option for Netcat (`nc`).
```bash
if nc -V 2>&1 | grep -q GNU; then
  NC="nc -c"  # Linux (GNU)
else
  NC="nc"     # macOS (BSD)
fi
echo '{"message_type": "shutdown"}' | $NC localhost 6000 || true
sleep 2  # give the master time to receive signal and send to workers
```

Then, check if a master or worker is still running and kill the process.  The following example is for the master.
```bash
if pgrep -lf mapreduce-master; then
  echo "killing mapreduce master ..."
  pkill -f mapreduce-master
fi
```

Example 1, server responds to shutdown message.
```console
$ ./bin/mapreduce stop
stopping mapreduce ...
```

Example 2, server doesn't respond to shutdown message and process is killed.
```console
./bin/mapreduce stop
stopping mapreduce ...
killing mapreduce master ...
killing mapreduce worker ...
```

### Server status
{: .primer-spec-toc-ignore }
Example
```console
$ ./bin/mapreduce start
starting mapreduce ...
$ ./bin/mapreduce status
master running
workers running
$ ./bin/mapreduce stop
stopping mapreduce ...
$ ./bin/mapreduce status
master not running
workers not running
```

### Restart server
{: .primer-spec-toc-ignore }
Example
```console
$ ./bin/mapreduce restart
stopping mapreduce ...
starting mapreduce ...
```


# MapReduce server specification
Here, we describe the functionality of the MapReduce server. The fun part is that we are only defining the functionality and the communication spec, the implementation is entirely up to you. You must follow our exact specifications below, and the Master and the Worker should work independently (i.e. do not add any more data or dependencies between the two classes). Remember that the Master/Workers are listening on TCP/UDP sockets for all incoming messages. **Note**: To test your server, we will will only be checking for the messages we listed below. You should *not* rely on any communication other than the messages listed below.

As soon as the Master/Worker receives a message on its main TCP socket, it should handle that message to completion before continuing to listen on the TCP socket. In this spec, let's say every message is handled in a function called `handle_msg`. When the message returns and ends execution, the Master will continue listening in an infinite while loop for new messages. Each TCP message should be communicated using a new TCP connection. *Note:* All communication in this project will be strings formatted using JSON; sockets receive strings but your thread must parse it into JSON.

We put [Master/Worker] before the subsections below to identify which class should handle the given functionality.

### Code organization
Your code will go inside the `mapreduce/master` and `mapreduce/worker` packages, where you will define the two classes (we got you started in `mapreduce/master/__main__.py` and ``mapreduce/worker/__main__.py``). Since we are using Python packages, you may create new files as you see fit inside each package. We have also provided a `utils.py` inside `mapreduce/` which you can use to house code common to both Worker and Master. We will only define the communication specs for the Master and the Worker, but the actual implementation of the classes is entirely up to you.

### Master overview
The Master should accept only one command line argument.

`port_number` : The primary TCP port that the Master should listen on.

On startup, the Master should do the following:
- Create a new folder `tmp`. This is where we will store all intermediate files used by the MapReduce server.
  - Hint: `os.makedirs()` is an easier way to setup your `tmp` directories and can prevent some of the errors that functions like `os.mkdir()` might cause.
  - Hint: use `os.path.join()` on file paths to prevent `fileNotFound` errors on the autograder.
  - If `tmp` already exists, keep it
  - Delete any old mapreduce job folders in `tmp`.  HINT: see the Python [glob](https://docs.python.org/3/library/glob.html) module and use `"tmp/job-*"`.
- Create a new thread, which will listen for UDP heartbeat messages from the workers. This should listen on (`port_number - 1`)
- Create any additional threads or setup you think you may need. Another thread for fault tolerance could be helpful.
- Create a new TCP socket on the given `port_number` and call the `listen()` function. Note: only one `listen()` thread should remain open for the whole lifetime of the master.
- Wait for incoming messages!  Ignore invalid messages, including those that fail JSON decoding.

### Worker overview
The Worker should accept two command line arguments.

`master_port`: The TCP socket that the Master is actively listening on (same as the `port_number` in the Master constructor)

`worker_port`: The TCP socket that this worker should listen on to receive instructions from the master

On initialization, each Worker should do a similar sequence of actions as the Master:

- Get the process ID of the Worker. This will be the Worker's unique ID, which it should then use to register with the master.
- Create a new TCP socket on the given `worker_port` and call the `listen()` function. Note: only one `listen()` thread should remain open for the whole lifetime of the worker.
- Send the `register` message to the Master
- Upon receiving the `register_ack` message, create a new thread which will be responsible for sending heartbeat messages to the Master.

**NOTE:** The Master should safely ignore any heartbeat messages from a Worker before that Worker successfully registers with the Master.

### Shutdown [master + worker]
Because all of of our tests require shutdown to function properly, it should be implemented first. The Master can receive a special message to initiate server shutdown. The shutdown message will be of the following form and will be received on the main TCP socket:
```json
{
  "message_type": "shutdown"
}
```

The Master should forward this message to all of the Workers that have registered with it. The Workers, upon receiving the shutdown message, should terminate as soon as possible. If the Worker is already in the middle of executing a task (as described below), it is okay for it to complete that task before being able to handle the shutdown message as both these happen inside a single thread. 

After forwarding the message to all Workers, the Master should terminate itself.

At this point, you should be able to pass `test_master_0`, `test_worker_0`.  Another shutdown test is `test_integration_0`, but you'll need to implement worker registration first.

### Worker registration [master + worker]
The Master should keep track of all Workers at any given time so that the work is only distributed among the ready Workers. Workers can be in the following states:
- `ready`: Worker is ready to accept work
- `busy`: Worker is performing a task
- `dead`: Worker has failed to ping for some amount of time

The Master must listen for registration messages from Workers. Once a Worker is ready to listen for instructions, it should send a message like this to the Master
```json
{
  "message_type" : "register",
  "worker_host" : string,
  "worker_port" : int,
  "worker_pid" : int
}
```

The Master will then respond with a message acknowledging the Worker has registered, formatted like this. After this message has been received, the Worker should start sending heartbeats. More on this later.
```json
{
  "message_type": "register_ack",
  "worker_host": string,
  "worker_port": int,
  "worker_pid" : int
}
```

After the first Worker registers with the Master, the Master should check the job queue (described later) if it has any work it can assign to the Worker (because a job could have arrived at the Master before any Workers registered). If the Master is already executing a map/group/reduce, it can wait until the next stage or wait until completion of the complete current task to assign the Worker any tasks.

At this point, you should be able to pass `test_master_1` and `test_worker_1` and `test_integration_0`.

### New job request [master]
In the event of a new job, the Master will receive the following message on its main TCP socket:
```json
{
  "message_type": "new_master_job",
  "input_directory": string,
  "output_directory": string,
  "mapper_executable": string,
  "reducer_executable": string,
  "num_mappers" : int,
  "num_reducers" : int
}
```

In response to a job request, the Master will create a set of new directories where all of the temporary files for the job will go, of the form `tmp/job-{id}`, where id is the current job counter (starting at 0 just like all counters). The directory structure will resemble this example(you should create 4 new folders for each job):
```json
tmp
  job-0/
    mapper-output/
    grouper-output/
    reducer-output/
  job-1/
    mapper-output/
    grouper-output/
    reducer-output/
```

Remember, each MapReduce job occurs in 3 stages: Mapping, Grouping, Reducing. Workers will do the mapping and reducing using the given executable files independently, but the Master and Workers will have to cooperate to do the Grouping Stage. After the directories are setup, the Master should check if there are any Workers ready to work and check whether the MapReduce server is currently executing a job. If the server is busy, or there are no available Workers, the job should be added to an internal queue (described next) and end the function execution. If there are workers and the server is not busy, than the Master can begin job execution.

At this point, you should be able to pass `test_master_2`. 

### Job queue [master]
If a Master receives a new job while it is already executing one or when there were no ready workers, it should accept the job, create the directories, and store the job in an internal queue until the current one has finished. **Note that this means that the current job's map, group, and reduce tasks must be complete before the next job's Mapping Stage can begin.** As soon as a job finishes, the Master should process the next pending job if there is one (and if there are ready Workers) by starting its Mapping Stage. For simplicity, in this project, your MapReduce server will only execute one MapReduce task at any time.

As noted earlier, when you see the first Worker register to work, you should check the job queue for pending jobs.

### Input partitioning [master]
To start off the Mapping Stage, the Master should scan the input directory and partition the input files in 'X' parts (where 'X' is the number of map tasks specified in the incoming job). After partitioning the input, the Master needs to let each Worker know what work it is responsible for. Each Worker could get zero, one, or many such tasks. The Master will send a JSON message of the following form to each Worker (on each Worker's specific TCP socket), letting them know that they have work to do:
```json
{
  "message_type": "new_worker_job",
  "input_files": [list of strings],
  "executable": string,
  "output_directory": string
  "worker_pid": int
}
```

Consider the case where there are 2 Workers available, 5 input files and 4 map tasks specified. The master should create 4 tasks, 3 with one input file each and 1 with 2 input files. It would then attempt to balance these tasks among all the workers. In this case, it would send 2 map tasks to each worker. The master does not need to wait for a done message before it assigns more tasks to a Worker, a Worker should be able to handle multiple tasks at the same time.

### Mapping [workers]
When a worker receives this new task message, its `handle_msg` will start execution of the given executable over the specified input file, while directing the output to the given `output_directory` (one output file per input file and you should run the executable on each input file). The input is passed to the executable through standard in and is outputted to a specific file. The output file names should be the same as the input file (overwrite file if it already exists). The `output_directory` in the Mapping Stage will always be the mapper-output folder (i.e. `tmp/job-{id}/mapper-output/`).

For example, the Master should specify the input file is `data/input/file_001.txt` and the output file `tmp/job-0/mapper-output/file_001.txt`

Hint: See the command line package `sh` listed in the [Libraries](#libraries) section. See `sh.Command(...)`, and the `_in` and `_out` arguments in order to funnel the input and output easily.

The Worker should be agnostic to map or reduce tasks. Regardless of the type of operation, the Worker is responsible for running the specified executable over the input files one by one, and piping to the output directory for each input file. Once a Worker has finished its task, it should send a TCP message to the Master's main socket of the form:
```json
{
  "message_type": "status",
  "output_files" : [list of strings],
  "status": "finished",
  "worker_pid": int
}
```
At this point, you should be able to pass `test_worker_3`, `test_worker_4`, `test_worker_5`. 

### Grouping [master + workers]
Once all of the mappers have finished, the Master will start the Grouping Stage. This should begin right after the LAST Worker finishes the Mapping Stage (i.e. you will get a finished message from the last Worker and the `handle_msg` handling that message will continue this Grouping Stage).

To start the Grouping Stage, the Master looks at all of the files created by the mappers, and assigns Workers to sort and merge the files. Sorting in the  Grouping Stage should happen by **line** not by key. If there are more files than Workers, the Master should attempt to balance the files evenly among them. If there are fewer files than Workers, it is okay if some Workers sit idle during this stage. Each Worker will be responsible for merging some number of files into one larger file. The Master will then take these files, merge them into one larger file, and then partition that file into the correct number of files for the reducers. The messages sent to the Workers should look like this:
```json
{
  "message_type": "new_sort_job",
  "input_files": [list of strings],
  "output_file": string,
  "worker_pid": int
}
```

Once the Worker has finished, it should send back a message formatted as follows:
```json
{
  "message_type": "status",
  "output_file" : string,
  "status": "finished"
  "worker_pid": int
}
```

Once the Master has received all of the output files from the Grouping Stage, it should read through those files and group them into one large sorted file. You shouldn't assume that all of the data from the Grouping Stage can fit into program memory at once. However, because all of the individual files are sorted alphabetically, we can read each file line by line and use the merge function from merge sort  to create the single large sorted file. The `merge` function from Python's `heapq` library will be helpful in doing this.

Use the following naming convention for the output of the Grouping Stage workers: `grouped0`, `grouped1` ... etc.  This help avoid duplicate or clobbered files when a worker fails.

Use the following naming convention for the output of the Grouping Stage master, which will be the inputs for the Reducing Stage.  File should be named `reduceX`, where `X` is the reduce task number. If there are 4 reduce tasks specified, the master should create `reduce1`, `reduce2`, `reduce3`, `reduce4` in the grouper output directory.


### Reducing [workers]
To the Worker, this is the same as the Mapping Stage - it doesn't need to know if it is running a map or reduce task. The Worker just runs the executable it is told to run - the Master is responsible for making sure it tells the Worker to run the correct map or reduce executable. The `output_directory` in the Reducing Stage will always be the `reducer-output` folder. Again, use the same output file name as the input file.

Once a Worker has finished its task, it should send a TCP message to the Master's main socket of the form:
```json
{
  "message_type": "status",
  "output_files" : [list of strings],
  "status": "finished"
  "worker_pid": int
}
```

### Wrapping up [master]
As soon as the master has received the last "finished" message for the reduce tasks for a given job, the Master should move the output files from the `reducer-output` directory to the final output directory specified by the original job creation message (The value specified by the `output_directory` key). In the final output directory, the files should be renamed `outputfilex`, where `x` is the final output file number. If there are 4 final output files, the master should rename them `outputfile1, outputfile2, outputfile3, outputfile4`.  Create the output directory if it doesn't already exist. Check the job queue for the next available job, or go back to listening for jobs if there isn't one currently in the job queue.

### Fault tolerance and heartbeats [master + worker]
Workers can die at any time and may not finish tasks that you send them. Your Master must accommodate for this. If a Worker misses more than 5 pings in a row, you should assume that it has died, and assign whatever work it was responsible for to another Worker machine.

Each Worker will have a heartbeat thread to send updates to Master via UDP. The messages should look like this, and should be sent every 2 seconds:
```json
{
  "message_type": "heartbeat",
  "worker_pid": int
}
```

If a Worker dies before completing **all** the tasks assigned to it, then **all** of those tasks (completed or not) should be redistributed to live Workers. At each point of the execution (mapping, grouping, reducing) the Master should attempt to evenly distribute work among available Workers. If a Worker dies while it is executing a task, the Master will have to assign that task to another Worker. You should mark the failed Worker as dead, but do not remove it from the Master's internal data structures. This is due to constraints on the Python dictionary data structure. It can result in an error when keys are modified while iterating over the dictionary. For more info on this, please refer to this [link](https://docs.python.org/3/c-api/dict.html).

Your Master should attempt to maximize concurrency, but avoid duplication.  In other words, don't send the same task to different Workers until you know that the Worker who was previously assigned that task has died.

At this point, you should be able to pass `test_master_3`, `test_master_4`, `test_worker_2`, `test_integration_1`,`test_integration_2`, and `test_integration_3`. 

# Walk-through example

See a [complete example here](example.html).


# Testing
To aid in writing test cases we have included a IntegrationManager class which is similar to the manager the autograder will use to test your submissions. You can find this in the starter file `tests/integration_manager.py`.

In addition, we have provided a simple word count map and reduce example. You can use these executables, as well as the sample data provided, and compare your server's output with the result obtained by running:
```console
$ cat tests/input/* | ./tests/exec/wc_map.sh | sort | \
    ./tests/exec/wc_reduce.sh > correct.txt
```

This will generate a file called `correct.txt` with the final answers, and they must match your server's output, as follows:
```console
$ cat tmp/job-{id}/reducer-output/* | sort > output.txt
$ diff output.txt correct.txt
```

Note that these executables can be in any language - your server should not limit us to running map and reduce jobs written in python3! To help you test this, we have also provided you with a word count solution written in a shell script (see section below).

Note that the autograder will watch your worker and master only for the messages we specified above. Your code should have no other dependency besides the communication spec, and the messages sent in your system must match those listed in this spec exactly.

Run the public unit tests.  Add the `-vs` flag to pytest to show output, which includes logging messages.
```console
$ pwd
/Users/awdeorio/src/eecs485/p4-mapreduce
$ pytest -vs
```


### Test for busy waiting
A solution that busy-waits may pass on your development machine and fail on the autograder due to a timeout.  Your laptop is probably much more powerful than the restricted autograder environment, so you might not notice the performance problem locally.  See the [Processes, Threads and Sockets in Python Tutorial](python-processes-threads-sockets.html#busy-waiting) for an explanation of busy-waiting.

To detect busy waiting, time a master without any workers.  After a few seconds, kill it by pressing `Control`-`C` several times.  Ignore any errors or exceptions.  We can tell that this solution busy-waits because the user time is similar to the real time.
```console
$ time mapreduce-master 6000
INFO:root:Starting master:6000
...

real	0m4.475s
user	0m4.429s
sys	0m0.039s
```

This example does not busy wait.  Notice that the user time is small compared to the real time.
```console
$ time mapreduce-master 6000
INFO:root:Starting master:6000
...

real	0m3.530s
user	0m0.275s
sys	0m0.036s
```

### Testing fault tolerance
This section will help you verify fault tolerance.  The general idea is to kill a worker while it's running an intentionally slow MapReduce job.

We have provided an intentionally slow MapReduce job in `tests/exec/wc_map_slow.sh` and `tests/exec/wc_reduce_slow.sh`.  These executables use `sleep` statements to simulate a slow running task.  You may want to increase the sleep time.
```console
$ grep -B1 sleep tests/exec/wc_*slow.sh
tests/exec/wc_map_slow.sh-# Simulate a long running job
tests/exec/wc_map_slow.sh:sleep 3
--
tests/exec/wc_reduce_slow.sh-# Simulate a long running job
tests/exec/wc_reduce_slow.sh:sleep 3
```

First, start one MapReduce Master and two Workers.  Wait for them to start up.
```console
$ pkill -f mapreduce-           # Kill any stale Master or Worker
$ mapreduce-master 6000 &       # Start Master
$ mapreduce-worker 6000 6001 &  # Start Worker 0
$ mapreduce-worker 6000 6002 &  # Start Worker 1
```

Submit an intentionally slow MapReduce job and wait for the mappers to begin executing.
```console
$ mapreduce-submit \
    --input ./tests/input_small \
    --output ./output \
    --mapper ./tests/exec/wc_map_slow.sh \
    --reducer ./tests/exec/wc_reduce_slow.sh \
    --nmappers 2 \
    --nreducers 2
```

Kill one of the workers while it is executing a map or reduce job.  Quickly use `pgrep` to find the PID of a worker, and then use `kill` on its PID. You can use `pgrep` again to check that you actually killed the worker.
```console
$ pgrep -af mapreduce-worker  # Linux
$ pgrep -lf mapreduce-worker  # macOS
77811 /usr/local/Cellar/python/3.7.2_1/Frameworks/Python.framework/Versions/3.7/Resources/Python.app/Contents/MacOS/Python /Users/awdeorio/src/eecs485/p4-mapreduce/env/bin/mapreduce-worker 6000 6001
77893 /usr/local/Cellar/python/3.7.2_1/Frameworks/Python.framework/Versions/3.7/Resources/Python.app/Contents/MacOS/Python /Users/awdeorio/src/eecs485/p4-mapreduce/env/bin/mapreduce-worker 6000 6002
$ kill 77811
$ pgrep -lf mapreduce-worker
77893 /usr/local/Cellar/python/3.7.2_1/Frameworks/Python.framework/Versions/3.7/Resources/Python.app/Contents/MacOS/Python /Users/awdeorio/src/eecs485/p4-mapreduce/env/bin/mapreduce-worker 6000 6002
```

Here's a way to kill one worker with one line.
```console
$ pgrep -af mapreduce-worker | head -n1 | awk '{print $1}' | xargs kill  # Linux
$ pgrep -lf mapreduce-worker | head -n1 | awk '{print $1}' | xargs kill  # macOS
[2]-  Terminated: 15          mapreduce-worker 6000 6001
```

Finally, verify the correct behavior.  Read to logging messages from your Master and Worker to consider:
- How many mapping tasks should the first Worker have received?
- How many mapping tasks should the second Worker have received?
- How many sorting and reducing tasks should the first and the second Worker receive?

**Pro-tip:**  Script this test, adding `sleep` commands in between commands to give them time for startup and TCP communication.


### Code style
As in previous projects, all Python code should contain no errors or warnings from `pycodestyle`, `pydocstyle`, and `pylint`.  Run pylint like this:
```console
$ pylint --disable=no-value-for-parameter mapreduce
```

You may not use any external dependencies aside from what is provided in `setup.py`.

# Submitting and grading
One team member should register your group on the autograder using the *create new invitation* feature.

Submit a tarball to the autograder, which is linked from [https://eecs485.org](https://eecs485.org).  Include the `--disable-copyfile` flag only on macOS.
```console
$ tar \
  --disable-copyfile \
  --exclude '*__pycache__*' \
  --exclude '*tmp*' \
  -czvf submit.tar.gz \
  setup.py \
  bin \
  mapreduce
```

### Rubric
This is an approximate rubric.

| Deliverable                              | Value |
| -----------------------------------------| ----- |
| Public unit tests                        | 60%   |
| Hidden unit tests run after the deadline | 40%   |

# Further reading
[Google's original MapReduce
paper](https://static.googleusercontent.com/media/research.google.com/en//archive/mapreduce-osdi04.pdf)


<!-- Prevent this doc from being picked up by search engines -->
<meta name="robots" content="noindex">
