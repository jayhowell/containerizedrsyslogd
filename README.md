# Containerizing rsyslogd - and getting it working locally with the logger cmd

This article shows how to get the containerized version of rsyslogd to work inside of a container

**Some knowledge first**

1. Most loggers talk through a socket loaded in linux. The default logger socket is at /dev/log
2. rsyslogd can open up a socket at /dev/log so applicatons can send logs to the default loger.
3. journald was added in RHEL7 to make logging more performant.
4. rsyslog uses the plugin(called imjournal) in the rsyslog.conf file located in /etc/ to access the journal
5. prior to RHEL7, rsyslog opened up the default logger port on /dev/log
6. After RHEL7 journald had the reponsibility for opening up the defualt logger socket on /dev/log
7. logs being written to /dev/log --> journald --> rsyslog(to be distributed to whatever log aggrigator you want)
8. The logger command by default logs directly to /dev/log

**So what's going on with the container image @ registry.redhat.io/rhel8/rsyslog:8.8-5?**

* rsyslog is launched by a stand alone shell script, not by systemd
* There is NO journald being loaded at all, so no socket(/dev/log) gets created
* The /etc/rsyslog.conf file in the container is the default one the comes with RHEL that works with journald, which has the socket creation turned off.
    * You can tell by looking at the module declaration of imuxsock, where it says SysSock.Use="off"
    * If you change this argument to SysSock.Use="on, rsyslog will create a default log socket at /dev/log
* With SysSock.Use="off", if you use "logger 'log my message'", you'll find that the message does not get added to /var/log/messages, it just gets thrown in the bit bucket.
* So we have to do the following things
    * Mount a changed rsyslog.conf file into the container that has the following changes
        * SysSock.Use="on"
        * remove the module load for imjournal - if you don't turn this off it will throw warnings that it can't find the journal
    * Mount a permanent directory for /var/log/messages(I created a tmp/var/log directory in my home directory)
    * Run the pod as --privileged - you have to do this in order to run the system service

Here's how you execute

```
podman run -d --privileged --rm --name rsyslog --net=host --pid=host -v ~/tmp/rsyslog.conf:/etc/rsyslog.conf -v ~/tmp/var/log/:/var/log  registry.redhat.io/rhel8/rsyslog:8.8-5
```

Here's what my session looks like.

```
[jhowell@jhowell tmp]$ podman run -d --privileged --rm --name rsyslog --net=host --pid=host -v ~/tmp/rsyslog.conf:/etc/rsyslog.conf -v ~/tmp/var/log/:/var/log  registry.redhat.io/rhel8/rsyslog:8.8-5
8e991841911e4de2c7cbed89f6cd9c53a597445e346e4efbfcbc3ce6e214ae86
[jhowell@jhowell tmp]$ podman exec -itl /bin/bash
[root@jhowell /]# logger "Red Hat Rawks"
[root@jhowell /]# tail /var/log/messages
May 19 15:30:30 jhowell rsyslogd: [origin software="rsyslogd" swVersion="8.2102.0-13.el8" x-pid="27924" x-info="https://www.rsyslog.com"] start
May 19 15:32:03 jhowell root: Red Hat Rawks
```

Attached is the rsyslog.conf file I used. As I said, the only changes I made to the default conf file in RHEL was to turn on the socket and to delete the journal module.