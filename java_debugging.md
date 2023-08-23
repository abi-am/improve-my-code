# Debugging Java pipelines

Debugging applications are hard, Java applications even more so. In this document we will discuss how to do basic debugging of pipelines using Java applications.

Based on a true story.

---

## Background

A user reported that a pipeline is "not working", specifically it's not writing anything in logs and output directory/files.

## Debugging

First, in case of using Slurm, we need to find the pipeline's JOBID

```
$ squeue | tail -1
              6553      test rnavar_s   ubuntu  R      22:52      1 bio
```

Next we need to find the PID of the JOB.

```
$ pgrep -fa 6553
1048123 slurmstepd: [6553.batch]
1048135 /bin/bash /var/spool/slurm/slurmd/job06553/slurm_script
```

We can see that our job's Slurm process is `1048123` run by `slurmstepd`.

Depending on your pipeline, there will be one or more processes under `slurmstepd`, we can use `pstree` to find it/them.

```
$ pstree -ap 1048123
slurmstepd,1048123
  ├─slurm_script,1048135 /var/spool/slurm/slurmd/job06553/slurm_script
  │   └─java,1048136 --add-opens=java.base/java.lang=ALL-UNNAMED --add-opens=java.base/java.io=ALL-UNNAMED --add-opens=java.base/java.nio=ALL-UNNAMED --add-opens=java.base/java.net=ALL-UNNAMED ...
  │       ├─{java},1048181
  │       ├─{java},1048182
  │       ├─{java},1048183
  │       ├─{java},1048185
  │       ├─{java},1048187
  │       ├─{java},1048190
  │       ├─{java},1048195
  │       ├─{java},1048196
  │       ├─{java},1048197
  │       ├─{java},1048198
  │       ├─{java},1048199
  │       ├─{java},1048200
  │       ├─{java},1048201
  │       ├─{java},1048202
  │       ├─{java},1048204
  │       ├─{java},1048206
  │       ├─{java},1048265
  │       ├─{java},1048266
  │       ├─{java},1048267
  │       ├─{java},1048268
  │       ├─{java},1048269
  │       ├─{java},1048270
  │       ├─{java},1048271
  │       ├─{java},1048272
  │       ├─{java},1048273
  │       ├─{java},1048274
  │       ├─{java},1048275
  │       ├─{java},1048276
  │       ├─{java},1048277
  │       ├─{java},1048278
  │       ├─{java},1048279
  │       ├─{java},1048280
  │       ├─{java},1048281
  │       └─{java},1054018
  ├─{slurmstepd},1048125
  └─{slurmstepd},1048126
```

Turns out we have a single process, `java` with the PID `1048136`, which multiple threads. Usually we would debug applications using `strace`, but with Java, it would be better to use `jstack`

```
$ jstack -e 1048136
2023-08-23 16:16:14
Full thread dump OpenJDK 64-Bit Server VM (11.0.14+9-Ubuntu-0ubuntu2.20.04 mixed mode, sharing):

Threads class SMR info:
_java_thread_list=0x00007fd78c000c40, length=10, elements={
0x00007fd818017800, 0x00007fd818457000, 0x00007fd818459000, 0x00007fd818461800,
0x00007fd818463800, 0x00007fd818465800, 0x00007fd818467800, 0x00007fd818469800,
0x00007fd81849f800, 0x00007fd78c001000
}

"main" #1 prio=5 os_prio=0 cpu=1816.80ms elapsed=424.17s allocated=109M defined_classes=2032 tid=0x00007fd818017800 nid=0xffe75 runnable  [0x00007fd81ed9c000]
   java.lang.Thread.State: RUNNABLE
        at sun.nio.ch.FileDispatcherImpl.lock0(java.base@11.0.14/Native Method)
        at sun.nio.ch.FileDispatcherImpl.lock(java.base@11.0.14/FileDispatcherImpl.java:96)
        at sun.nio.ch.FileChannelImpl.tryLock(java.base@11.0.14/FileChannelImpl.java:1161)
        at java.nio.channels.FileChannel.tryLock(java.base@11.0.14/FileChannel.java:1165)
        at java_nio_channels_FileChannel$tryLock.call(Unknown Source)
        at org.codehaus.groovy.runtime.callsite.CallSiteArray.defaultCall(CallSiteArray.java:47)
        at org.codehaus.groovy.runtime.callsite.AbstractCallSite.call(AbstractCallSite.java:125)
        at org.codehaus.groovy.runtime.callsite.AbstractCallSite.call(AbstractCallSite.java:130)
        at nextflow.util.HistoryFile.withFileLock(HistoryFile.groovy:429)
        at jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(java.base@11.0.14/Native Method)
        at jdk.internal.reflect.NativeMethodAccessorImpl.invoke(java.base@11.0.14/NativeMethodAccessorImpl.java:62)
        at jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(java.base@11.0.14/DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(java.base@11.0.14/Method.java:566)
        at org.codehaus.groovy.runtime.callsite.PlainObjectMetaMethodSite.doInvoke(PlainObjectMetaMethodSite.java:43)
        at org.codehaus.groovy.runtime.callsite.PogoMetaMethodSite$PogoCachedMethodSiteNoUnwrapNoCoerce.invoke(PogoMetaMethodSite.java:193)
        at org.codehaus.groovy.runtime.callsite.PogoMetaMethodSite.callCurrent(PogoMetaMethodSite.java:61)
        at org.codehaus.groovy.runtime.callsite.CallSiteArray.defaultCallCurrent(CallSiteArray.java:51)
        at org.codehaus.groovy.runtime.callsite.AbstractCallSite.callCurrent(AbstractCallSite.java:171)
        at org.codehaus.groovy.runtime.callsite.AbstractCallSite.callCurrent(AbstractCallSite.java:185)
        at nextflow.util.HistoryFile.generateNextName(HistoryFile.groovy:462)
        at nextflow.cli.CmdRun.checkRunName(CmdRun.groovy:400)
        at nextflow.cli.CmdRun.run(CmdRun.groovy:282)
        at nextflow.cli.Launcher.run(Launcher.groovy:480)
        at nextflow.cli.Launcher.main(Launcher.groovy:639)

"Reference Handler" #2 daemon prio=10 os_prio=0 cpu=1.48ms elapsed=424.14s allocated=0B defined_classes=0 tid=0x00007fd818457000 nid=0xffe84 waiting on condition  [0x00007fd800257000]
   java.lang.Thread.State: RUNNABLE
[...]

"VM Thread" os_prio=0 cpu=57.83ms elapsed=424.14s tid=0x00007fd818454000 nid=0xffe83 runnable

"GC Thread#0" os_prio=0 cpu=25.12ms elapsed=424.17s tid=0x00007fd818032000 nid=0xffe76 runnable

"GC Thread#1" os_prio=0 cpu=31.47ms elapsed=419.99s tid=0x00007fd7a0039000 nid=0xffec9 runnable
[...]
"VM Periodic Task Thread" os_prio=0 cpu=611.96ms elapsed=423.75s tid=0x00007fd8184a3800 nid=0xffe8e waiting on condition

JNI global refs: 17, weak refs: 0

```

We can see that the stack is at [`sun.nio.ch.FileDispatcherImpl.lock0`](https://github.com/openjdk/jdk/blob/jdk-11%2B14/src/java.base/unix/classes/sun/nio/ch/FileDispatcherImpl.java#L167C75-L167C75), which is a filesystem lock syscall.

## Solution

Turns out the NFS export was mounted without a lock mechanism, hence why it was "stuck".

---

I hope this guide was helpful.

Good luck!
