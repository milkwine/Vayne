# NAME

Vayne - Distribute task queue

# SYNOPSIS

    use Vayne;
    use Vayne::Callback;
    use Vayne::Tracker;

    my @workload = <>; chomp @workload;
    my $tracker = Vayne::Tracker->new();
    my $step = Vayne->task('foo');
    my $taskid = $tracker->add_task(
         'region:region-first',
         {
             name   => 'foo',
             step   => $step,
             expire => 90,
         },
         @workload
    );
    my $call = Vayne::Callback->new();
    my $stat = $call->wait($taskid);

# GETTING STARTED

    # First time only
    > vayne-init -d $HOME/vayne

    # Setup configurations in $HOME/vayne/conf (zookeeper mongodb redis)

    # Add our first region, then the region info will upload to zk server.
    > vayne-ctrl --set --region region-first --server redisserver:port --password redispasswd

    # Define our first task, $HOME/vayne/task/foo

      # check server ssh server
      - name: 'check ssh'                     #step's name
        worker: tcp                           #step's worker
        param:                                #step's parameters
          port: 22
          input: ''
          check: '^SSH-2.0-OpenSSH'

      - name: 'foo'
        worker: dump
        param:
          bar: baz

      - name: 'only suc'
        need:
          - 'check ssh': 1
        worker: dump
        param:
          - array
          - key1: value1
            key2: value2

      # tracke the job result
      - name: 'tracker'
        worker: track

    # Switch the server you run workers to our first region.
    > vayne-ctrl --switch --region region-first

    # Run workers.
    > $HOME/vayne/worker/tcp &
    > $HOME/vayne/worker/dump &
    > $HOME/vayne/worker/tracker &

    # Run task tracker.
    > vayne-tracker &

    # Submit our task by CLI.
    > echo '127.0.0.1'|vayne-task --add --name foo --expire 60 --strategy region:region-first
    # or
    > vayne-task --add --name foo --expire 60 --strategy region:region-first < server_list

    # Query our task through taskid by CLI.
    > vayne-task --taskid 9789F5E6-2644-11E6-A6F0-AF9AF8F9E07F --query

    # Or use Vayne lib in your program like SYNOPSIS.

# DESCRIPTION

Vayne is a distribute task queue with many feature.

## FEATURE

- Logical Region with Flexible Spawning Strategy

    Has the concept of logical region.
    You can spawn task into different region with strategy.
    Spawning strategy can be easily write.

- Custome Task Flow with Reusable Worker

    Worker is a process can do some specific stuffs.
    Step have a step's name, a worker's name and parameters.
    You can define custome task flow by constructing any steps.

- Simple Worker Interface with Good Performance

    [Vayne::Worker](https://metacpan.org/pod/Vayne::Worker) is using [Coro](https://metacpan.org/pod/Coro) which can provide excellent performance in network IO. 
    Worker has a simple interface to write, also you can use Coro::\* module to enhance worker performance.
    Whole system is combined with Message Queue. 
    You can get a better performance easily by increasing the worker counts while MQ is not the bottleneck.

## HOW IT WORKS

                                                                               +--------+
                                                                               |Worker A| x N
                                                                               +--------+
                                                         +------------+                        Workers may run on several servers
                                                         |            |        +--------+
                                                         |  Region A  |        |Worker B| x N
                                                         |            |        +--------+
                                                         +------------+
                                                                                 .....
    +-----------+                                                              +----------+
    |           |                                        +------------+        |JobTracker| x N
    | Task Conf |                                        |            |        +----------+
    |           |                                        |  Region B  |
    | +-------+ |                                        |            |
    | | step1 | |    +-----------+                       +------------+                                                 +-----------+          +-----------+
    | +-------+ |    |           |        Spawn Jobs                                           Save Job Information     |           | +------> |           |
    |           | +  | workloads |    +---------------->                                       +--------------------->  |  Mongodb  |          |TaskTracker|
    | +-------+ |    |           |       with Strategy                                          to Center Mongodb       |           | <------+ |           |
    | | step2 | |    +-----------+                       +------------+                                                 +-----------+          +-----------+
    | +-------+ |                                        |            |
    |    ...    |                                        |  Region C  |          .....                                       ^
    | +-------+ |                                        |            |                                                      |
    | | stepN | |                                        +------------+                                                      |
    | +-------+ |                                                                                                            |
    |           |                                                                                                            |
    +-----------+                                        +------------+                                                      |
                                                         |            |                                                      |
                                                         |  Region D  |                                                      |
                  |                                      |            |                                                      |
                  |                                      +------------+                                                      |
                  |                                                                                                          |
                  |                                                                                                          |
                  |                                                                                                          |
                  |                                                                                                          |
                  |                                Save Task Information to Center Mongodb                                   |
                  +----------------------------------------------------------------------------------------------------------+
    

### 0. Task Conf & Step

The Task Conf is combined with several ordered steps.
Each step have a step's name, a worker's name and parameters.
Workload will be prosessed step by step.

### 1. Spawn Task

Vayne support CLI and API to spawn a task.
A task contain numbers of jobs.
Task info will write to _task collection_ in mongodb first.
Jobs will be hashed into saperated region by strategy.
Then enqueue jobs to their region's redis queue named by first step of the job, and write to _job collection_ in mongodb.

### 2. Queue & Region

Like [Redis::JobQueue](https://metacpan.org/pod/Redis::JobQueue), Vayne use _redis_ for job queuing and job info caching.
The data structure is nearly the same as ["JobQueue data structure stored in Redis" in Redis::JobQueue](https://metacpan.org/pod/Redis::JobQueue#JobQueue-data-structure-stored-in-Redis).

Each **region** has a _queue(redis server)_. Both their infomation are saved on _zookeeper server_.

Each _real server_ which you want to run workers should belong to a **region**.

**Worker** will register its names under real server's **region** on _zookeeper server_ when it start.

Details see ["DATA STRUCTURE" in Vayne::Zk](https://metacpan.org/pod/Vayne::Zk#DATA-STRUCTURE).

### 2. Worker

When it start, worker register its names on _zookeeper server_.
Then generate some [Coro](https://metacpan.org/pod/Coro) threads below:

- Check the Registration

    Go die when the registration changed.
    Ex: Region info changed; Real Server switch to another region; Connection to zk failed.

    _ \* The worker will die very quickly when zookeeper server is not available. It may cause some problems. Should be careful. _

- Job Consumer

    BLPOP queues which worker registered, then put the job into [Coro::Channel](https://metacpan.org/pod/Coro::Channel)

- Worker

    Get **job** from [Coro::Channel](https://metacpan.org/pod/Coro::Channel), and do the stuff with it.
    Tag the _result_ and _status_ on the **job**.
    Put the **job** to update [Coro::Channel](https://metacpan.org/pod/Coro::Channel).

- Update Job Info

    Get **Job** from  update [Coro::Channel](https://metacpan.org/pod/Coro::Channel).
    Push the job to next queue according to the job's step.

`INT`, `TERM`, `HUP` signals will be catched.
Then graceful stop the worker.

### 4. JOB TRACKER

Job tracker is a special worker, it just send the job info dealed by previous workers to mongodb.
Usually 'tracker' should be the last step of a job.

### 5. TASK TRACKER

Script [vayne-tracker](https://metacpan.org/pod/vayne-tracker)
Loop
bla..

## BACKEND

Redis-3.2 [http://redis.io/](http://redis.io/)

Zookeeper-3.3.6 [http://zookeeper.apache.org/](http://zookeeper.apache.org/)

MongoDB-3.0.6 [https://www.mongodb.com/](https://www.mongodb.com/)

## DATA STRUCTURE

### Zookeeper

["DATA STRUCTURE" in Vayne::Zk](https://metacpan.org/pod/Vayne::Zk#DATA-STRUCTURE)

### Redis

Data Structure for job&queue is nearly the same as
["JobQueue data structure stored in Redis" in Redis::JobQueue](https://metacpan.org/pod/Redis::JobQueue#JobQueue-data-structure-stored-in-Redis)

### MongoDB

bla bla..

## HOW TO WRITE A WORKER

bla bla..

# AUTHOR

SiYu Zhao <zuyis@cpan.org>

# COPYRIGHT

Copyright 2016- SiYu Zhao

# LICENSE

This library is free software; you can redistribute it and/or modify
it under the same terms as Perl itself.

# SEE ALSO

[Redis::JobQueue](https://metacpan.org/pod/Redis::JobQueue)
