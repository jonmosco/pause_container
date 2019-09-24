### Understanding Kubernetes pods with the pause container

This repo contains the files necessary to understand what a pod is
in Kubernetes, using Docker and the Google pause container.

If you prefer depth-first style of learning, really want the low-level
details of how things work, then this repo is for you.

Background

Pods

Pods in Kubernetes (k8s from here on out to save me from all the typing), are
a group of one or more containers.  They share the IPC, and UTS namespaces.

In k8s 1.7+, shared PID namespace is enabled if the cluster is running
Docker 1.13.1+, but has been reverted in 1.8, thus each container in a pod
will have to handle signals properly.

Your Kubernetes server must be at or later than version v1.10

As of 1.16, Process Namespace Sharing is a beta feature that is enabled by
default. It may be disabled by setting --feature-gates=PodShareProcessNamespace=false

### Namespaces

From namespaces(7):

```
DESCRIPTION
       A  namespace  wraps  a global system resource in an abstraction that makes it appear to the processes within
       the namespace that they have their own isolated instance of the global  resource.   Changes  to  the  global
       resource  are  visible to other processes that are members of the namespace, but are invisible to other pro-
       cesses.  One use of namespaces is to implement containers.
```

### Namespace sharing

The following namespaces will be described to detail how pods are created:

* network
* IPC
* PID
* UTS

### Network namespace

```
--net=container:name
```

### IPC Namespace

```
--ipc=container:name
```

The causes the container to join the specified containers IPC namespace.
IPC (POSIX/SysV IPC) namespace provides separation of named shared
memory segments, semaphores and message queues.

They can even use IPC or send each other signals like HUP or TERM
(With shared PID namespaces in Kubernetes 1.7, Docker >=1.13)

### UTS Namespace

When namespace sharing is used, each container has the same hostname as the
first container (the pause container in this cause) and the same IP.

### PID Namespace

Each container by default gets its own process namespace.

While in k8s PID namespace sharing is not enabled, examples are included here
for sake of brevity.

This causes the container to join the specified containers PID namespace:

```
--pid=container:name
```

PID namespaces can not see into their peers namespace.  New PID namespace
starts at 1, and has representation in all the parent namespace, and can
be monitored as such.  In this example, pause assumes PID 1, and nginx,
and ghost all become children of pause.

Inside the nginx container:

```
root@88abb5cfad6f:/# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 14:44 ?        00:00:00 /pause
root         7     0  0 14:44 ?        00:00:00 nginx: master process nginx -g daemon off;
nginx       13     7  0 14:44 ?        00:00:00 nginx: worker process
1000        14     0  0 14:44 ?        00:00:02 node current/index.js
root        92     0  0 19:50 pts/0    00:00:00 /bin/bash
root       395    92  0 19:51 pts/0    00:00:00 ps -ef
```

Inside the ghost container:

```
root@88abb5cfad6f:/var/lib/ghost# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 14:44 ?        00:00:00 /pause
root         7     0  0 14:44 ?        00:00:00 nginx: master process nginx -g daemon off;
systemd+    13     7  0 14:44 ?        00:00:00 nginx: worker process
node        14     0  0 14:44 ?        00:00:02 node current/index.js
root       396     0  0 19:51 pts/0    00:00:00 /bin/bash
root       403   396  0 19:52 pts/0    00:00:00 ps -ef
root@88abb5cfad6f:/var/lib/ghost#
```

When tailing the logs for pause, we can watch the process termination.
Running docker stop will send a SIGTERM, and after a grace period, it
sends SIGKILL:

```
docker logs -f 88abb5cfad6f


shutting down, got signal: Terminated
```

On a Kubernetes cluster, the containers that make up this pod all share the same IP address:

```
worker01# docker exec -ti <container_id> ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
3: eth0@if11: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    link/ether 0a:58:0a:02:07:0c brd ff:ff:ff:ff:ff:ff
    inet 10.2.7.12/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::384f:f6ff:fe26:5953/64 scope link
       valid_lft forever preferred_lft forever

worker01 ~ # docker exec -ti <container_id> ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
3: eth0@if11: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue state UP
    link/ether 0a:58:0a:02:07:0c brd ff:ff:ff:ff:ff:ff
    inet 10.2.7.12/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::384f:f6ff:fe26:5953/64 scope link
       valid_lft forever preferred_lft forever

worker01 ~ # docker exec -ti <container_id> ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
3: eth0@if11: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue
    link/ether 0a:58:0a:02:07:0c brd ff:ff:ff:ff:ff:ff
    inet 10.2.7.12/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::384f:f6ff:fe26:5953/64 scope link
       valid_lft forever preferred_lft forever
pltecworker02 ~ #
```

## The pause() system call

The pause() system call puts a process to sleep until it receives a signal
that either is handled or terminates the process.  pause() returns only
if a signal is received, in which case the signal is handled, and pause()
returns -1 and sets errno to EINTR.  If the kernel raises an ignored
signal, the process does not wake up.  It performs only two actions:

1. It puts the process in the interruptible sleep state
2. It calls schedule() to invoke the Linux process scheduler to find
   another process to run.  As the process is not actually waiting for anything,
   the kernel will not wake up unless it receives a signal.

## The pause container

The pause container provides a very simple program that assumes the role of
of PID 1.  It sleeps and watches for zombie processes.

The code for this is pause.c and is in this repo.

The sigreap() function will wait for any child process:

```
static void sigreap(int signo) {
  while (waitpid(-1, NULL, WNOHANG) > 0);
}
```

waitpid with arg pid=-1 means whatever can be reaped then WNOHANG means do not wait
so it will reap all child processes (any that are zombies) then return -1 when there
are none left, then return you can bind that signal handler on delivery of certain signals
e.g SIGCHLD sigaction(SIGCHLD, sigreap, ...)
the means, change the action upon reception of a SIGCHLD signal from the default to running sigreap()

## Creating a pod outside k8s

Creating the pod with docker:

```
$ docker run -d --name pause -p 8080:80 gcr.io/google_containers/pause-amd64:3.0
$ docker run -d --name nginx -v `pwd`/nginx.conf:/etc/nginx/nginx.conf --net=container:pause --ipc=container:pause --pid=container:pause nginx
$ docker run -d --name ghost --net=container:pause --ipc=container:pause --pid=container:pause ghost
```

Creating the pod with docker compose:

```
$ docker-compose up -d
```

Using the Makefile:

```
$ make pod
$ make stop
$ make clean
```

