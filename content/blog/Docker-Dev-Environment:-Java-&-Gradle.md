---
date: 2022-04-12T18:48:00Z
title: "Docker Dev Environment: Java & Gradle"
draft: false
image:
   hero: audio-vaporwave-fence.png
   thumb:
author: blitterated
tags: ["Docker", "Java", "Gradle", "Software Engineering"]
categories: ["Software"]
---
# Create a Java and Gradle dev environment

Creating a development environment in Docker requires that we juggle two contexts. One is the Docker side, the run time and dependency management for the project. The other is the project itself, the source code.

Here we're going to take an iterative approach to building these up. Then at the end, we should have a complete set of files for the Docker development environment, as well as a scaffolded project directory. All the appropriate mappings between the two will be setup as well.

## First, build a Java and Gradle dev image

This first part avoids creating any files. Everything is done in heredocs. Because I'm a maniac. It also helps me separate the Docker Dev context from the Project Dev context. Later all of this will be broken out into its own project with a Dockerfile and supporting files.

### Build a dev image based on dde that installs java and gradle, and configures the environment.

This uses a heredoc approach discussed in [dockerfile best practices](Heredoc idea from https://docs.docker.com/develop/develop-images/dockerfile_best-practices/).

The quotes ("") around `EOF` are important. We don't want string substitution to occur before `docker build` begins interpreting the text stream. Otherwise, we just get empty values.

The dde image will already have basic linux CLI utils and s6 installed already.

The latest [OpenJDK is available here](https://jdk.java.net/archive/).

The latest [Gradle is available here](https://gradle.org/releases/).

```sh
docker build -t java_gradle -f- . <<"EOB"
FROM dde

ARG JINSTALLER=openjdk-17.0.2_linux-x64_bin.tar.gz
ARG JDK_VER=17.0.2
ARG JDK_PATH=/opt/jdk-$JDK_VER

ARG GRADLE_VER=7.4

# Download Java
ADD https://download.java.net/java/GA/jdk${JDK_VER}/dfd4a8d0985749f896bed50d7138ee7f/8/GPL/${JINSTALLER} /tmp/

# Install Java
RUN tar -zvxf /tmp/${JINSTALLER} -C /opt

# Download Gradle
ADD https://services.gradle.org/distributions/gradle-${GRADLE_VER}-bin.zip /tmp/

# Install Gradle
RUN unzip -d /opt /tmp/gradle-${GRADLE_VER}-bin.zip

# Add Java and Gradle to PATH
ENV PATH="${PATH}:${JDK_PATH}/bin:${JDK_PATH}/db/bin:${JDK_PATH}/jre/bin:/opt/gradle-${GRADLE_VER}/bin"

# Environment Vars
ENV J2SDKDIR=${JDK_PATH} \
    J2REDIR=${JDK_PATH} \
    JAVA_HOME=${JDK_PATH} \
    DERBY_HOME=${JDK_PATH}/db

# Don't override ENTRYPOINT
EOB
```

### Spin up a container and poke it with a pointed stick

We'll toss in `with-contenv` so we can see the environment variables we expect.

```sh
docker run -it --rm java_gradle with-contenv /bin/bash
```

Check that our settings took.

```sh
env | grep jdk
```

```
DERBY_HOME=/opt/jdk-17.0.2/db
JAVA_HOME=/opt/jdk-17.0.2
J2REDIR=/opt/jdk-17.0.2
J2SDKDIR=/opt/jdk-17.0.2
PATH=/command:/usr/bin:/bin:/opt/jdk-17.0.2/bin:/opt/jdk-17.0.2/db/bin:/opt/jdk-17.0.2/jre/bin:/opt/gradle-7.4/bin
```

Check that installs went to the right places.

```sh
ls /opt
```

```
gradle-7.4  jdk-17.0.2
```

Check that our installs run.


```sh
java --version
```

```
openjdk version "17.0.2" 2022-01-18
OpenJDK Runtime Environment (build 17.0.2+8-86)
OpenJDK 64-Bit Server VM (build 17.0.2+8-86, mixed mode, sharing)
```

```sh
gradle --version
```

```
Welcome to Gradle 7.4!
...
```

That's enough testing. Shut down the container.

### Expand the dev image to configure Gradle as an s6-rc.d service.

We're just going to cram the necessary files into the script below as heredocs. Heredocs in a heredoc.

```sh
docker build -t java_gradle -f- . <<"EOB"
# syntax=docker/dockerfile:1.3-labs
FROM dde

ARG JINSTALLER=openjdk-17.0.2_linux-x64_bin.tar.gz
ARG JDK_VER=17.0.2
ARG JDK_PATH=/opt/jdk-$JDK_VER

ARG GRADLE_VER=7.4

# Download Java
ADD https://download.java.net/java/GA/jdk${JDK_VER}/dfd4a8d0985749f896bed50d7138ee7f/8/GPL/${JINSTALLER} /tmp/

# Install Java
RUN tar -zvxf /tmp/${JINSTALLER} -C /opt

# Download Gradle
ADD https://services.gradle.org/distributions/gradle-${GRADLE_VER}-bin.zip /tmp/

# Install Gradle
RUN unzip -d /opt /tmp/gradle-${GRADLE_VER}-bin.zip

# Add Java and Gradle to PATH
ENV PATH="${PATH}:${JDK_PATH}/bin:${JDK_PATH}/db/bin:${JDK_PATH}/jre/bin:/opt/gradle-${GRADLE_VER}/bin"

# Environment Vars
ENV J2SDKDIR=${JDK_PATH} \
    J2REDIR=${JDK_PATH} \
    JAVA_HOME=${JDK_PATH} \
    DERBY_HOME=${JDK_PATH}/db

# Setup the files needed to run Gradle as a daemon under s6
ARG S6_SVC_DIR=/etc/s6-overlay/s6-rc.d
ARG GR_SVC_DIR=${S6_SVC_DIR}/gradle

RUN mkdir -p ${GR_SVC_DIR}

RUN cat <<EOF > ${GR_SVC_DIR}/type
longrun
EOF

RUN cat <<EOF > ${GR_SVC_DIR}/run
#!/command/with-contenv execlineb
/opt/gradle-${GRADLE_VER}/bin/gradle --foreground
EOF

RUN chmod +x ${GR_SVC_DIR}/run

RUN cat <<EOF > ${GR_SVC_DIR}/finish
#!/command/with-contenv execlineb
/opt/gradle-${GRADLE_VER}/bin/gradle --stop
EOF

RUN chmod +x ${GR_SVC_DIR}/finish

RUN touch ${S6_SVC_DIR}/user/contents.d/gradle

# Don't override ENTRYPOINT
EOB
```

### Spin up another container and poke it with a pointed stick

```sh
docker run -it --rm java_gradle with-contenv /bin/bash
```

Right off the bat:

```
Welcome to Gradle 7.4!

Here are the highlights of this release:
 - Aggregated test and JaCoCo reports
 - Marking additional test source directories as tests in IntelliJ
 - Support for Adoptium JDKs in Java toolchains

For more details see https://docs.gradle.org/7.4/release-notes.html

Daemon server started.
```

Check it out with a status switch.

```sh
gradle --status
```

```
   PID STATUS   INFO
    35 IDLE     7.4
```

Look a little closer with `ps`.

```sh
ps x | grep gradle
```

```
   25 ?        S      0:00 s6-supervise gradle
   35 ?        Ssl    0:02 /opt/jdk-17.0.2/bin/java -Xmx64m -Xms64m -Dorg.gradle.appname=gradle -classpath /opt/gradle-7.4/lib/gradle-launcher-7.4.jar org.gradle.launcher.GradleMain --foreground
  164 pts/0    S+     0:00 grep gradle
```

Looks like we're good to go. Shut down the container.

## Next, create a project directory and gin up some source code.

### Create a project directory:

```sh
mkdir dde_gradle
cd dde_gradle
```

### Prepare to gin up some source code.

This is where it gets a little tricky. We're going to be following [Gradle's Building Java Applications Sample](https://docs.gradle.org/current/samples/sample_building_java_applications.html), which nearly immediately starts with the `gradle init` scaffolding command. Before we can do that we need to:

1. Start up a container with a bind mount to our source code directory.
2. Figure out a way to run `gradle init` in the mapped directory inside the container.

####  Start up a container with a bind mount to our source code directory

This is easy enough. We'll just mount the current working directory into a folder in the container called `/project`.

Start up a new container in the background, and give it a name so it's easy to reference from the shell.

```sh
docker run -d --rm -v "$(pwd)":/project -w /project --name jgdemo java_gradle
```

Let's test it out by `docker exec`ing a shell into the container daemon and checking `gradle --status`.

```sh
docker exec -it jgdemo /bin/bash
```

Thus began an hour long rabbit hole exploring why the paths don't work with s6-overlay and `docker exec`.

Details are available in `s6-overlay,-docker-exec,-and-PATH.md`. 

`docker exec` just jams a new process into the container by changing its cgroup and namespace, and none of the typical methods to load an environment get called (PAM, Bash, s6, etc.). This means that `/command` never gets added to `PATH` which breaks `with-contenv`, so we can't even use that to get the PATH setting we expect to be able to reach our executable.

```
root@061ffe08fe7b:/project# gradle --status
bash: gradle: command not found
root@061ffe08fe7b:/project# with-contenv gradle --status
bash: with-contenv: command not found
root@061ffe08fe7b:/project# /command/with-contenv gradle --status
execlineb: fatal: unable to exec ifelse: No such file or directory
root@061ffe08fe7b:/project# /opt/gradle-7.4/bin/gradle --status
   PID STATUS   INFO
    35 IDLE     7.4
root@061ffe08fe7b:/project# echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

There's two approaches available here if we're sticking with `docker exec`. I haven't looked at `sshd` too much, but it seems like that reads it's own environment files for it's environment configuration.

1. use a `.bashrc`. This assumes that `docker exec` will be used directly. If so, we get an interactive/non-login environment. If instead we interpose a `bash -c` command, we then get a non-interactive/non-login environment
2. install the s6-overlay symlinks tarball so all the commands in `/command` are available at `/usr/bin`.

More time passed, and other ideas were given consideration. I decided to write a little utility to handle working with Docker Dev Environment derived images. It's called `dde`, and is available [here](https://github.com/blitterated/docker_dev_env_cli). 

While working on the `dde` utility, s6 3.1.0.1 arrived. I went ahead and rebuilt the base Docker Dev Environment image with this version. With it, some of the environment issues went away. I can see my variables and `PATH`s, but there's still no `/command` added to `PATH`.

That being said, the above experiment works now.

```sh
docker exec -it jgdemo /bin/bash
```

```sh
gradle --status
```

Here's the output.

```
   PID STATUS   INFO
    33 IDLE     7.4
```

It also works with a direct command.

```sh
docker exec jgdemo gradle --status
```

And it even works with a non-login/non-interactive shell.

```sh
docker exec jgdemo bash -c 'gradle --status'
```

I'm happy it works because this is the route I chose to add `/command` back into `PATH`.

```sh
docker exec jgdemo bash -c 'PATH=/command:${PATH} with-contenv gradle --status'
```

####  Second try: Start up a container with a bind mount to our source code directory

The goals are still:

1. Start up a container with a bind mount to our source code directory.
2. Figure out a way to run `gradle init` in the mapped directory inside the container.

Start over with a fresh container.

```sh
docker run -d --rm -v "$(pwd)":/project -w /project --name jgdemo java_gradle
```

Test it out by `docker exec`ing `gradle --status`.

```sh
docker exec jgdemo gradle --status
```

Which looks good.

```
   PID STATUS   INFO
    33 IDLE     7.4
```

Now we can begin to follow the [Gradle Java Applications Sample](https://docs.gradle.org/current/samples/sample_building_java_applications.html) tutorial.

### Init a gradle project in the container and across the bind mount.

First connect to the container and open a command prompt

```sh
docker exec -it jgdemo /bin/bash
```

Follow along with the tutorial's [Init Task](https://docs.gradle.org/current/samples/sample_building_java_applications.html#run_the_init_task) step. It may be slightly different. My particular session is shown below.

```
root@d255eaaf8d01:/project# gradle init
```

### Annnnnnddd... there's another weird behavior.

```
Starting a Gradle Daemon, 1 incompatible Daemon could not be reused, use --status for details
```

Using `gradle -status` doesn't help much other than showing that there are two Gradle daemons running.

```sh
gradle --status
```

```
   PID STATUS   INFO
    33 IDLE     7.4
   230 BUSY     7.4
```

Using `ps` tells an interesting story though.

```sh
ps auxf
```

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root       187  0.0  0.0   4116  3420 pts/0    Ss   18:28   0:00 /bin/bash
root       385  0.0  0.0   5900  2832 pts/0    R+   18:34   0:00  \_ ps auxf
root         1  0.0  0.0    208    64 ?        Ss   18:24   0:00 /package/admin/s6/command/s6-svscan -d4 -- /run/service
root        16  0.0  0.0    212    68 ?        S    18:24   0:00 s6-supervise s6-linux-init-shutdownd
root        17  0.0  0.0    204     4 ?        Ss   18:24   0:00  \_ /package/admin/s6-linux-init/command/s6-linux-init-shutdownd -c /run/s6/basedir -g 3000 -C -B
root        23  0.0  0.0    212    64 ?        S    18:24   0:00 s6-supervise s6rc-oneshot-runner
root        35  0.0  0.0    188     4 ?        Ss   18:24   0:00  \_ /package/admin/s6/command/s6-ipcserverd -1 -- /package/admin/s6/command/s6-ipcserver-access -v0 -E -l0 -i data/rules -- /package/admin/s6/command/s6-sudod -t 30000 -- /package/admin/s6-rc/command/s6-rc-oneshot-run -l ../.. --
root        24  0.0  0.0    212    64 ?        S    18:24   0:00 s6-supervise s6rc-fdholder
root        25  0.0  0.0    212    68 ?        S    18:24   0:00 s6-supervise gradle
root        34  1.4  1.3 3454184 112832 ?      Ssl  18:24   0:07  \_ /opt/jdk-17.0.2/bin/java -Xmx64m -Xms64m -Dorg.gradle.appname=gradle -classpath /opt/gradle-7.4/lib/gradle-launcher-7.4.jar org.gradle.launcher.GradleMain --foreground
root       230  7.6  3.9 3342516 322468 ?      Ssl  18:29   0:22 /opt/jdk-17.0.2/bin/java --add-opens java.base/java.util=ALL-UNNAMED --add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.lang.invoke=ALL-UNNAMED --add-opens java.base/java.util=ALL-UNNAMED --add-opens java.prefs/java.util.prefs=ALL-UNNAMED --add-opens java.prefs/java.util.prefs=ALL-UNNAMED --add-opens java.base/java.nio.charset=ALL-UNNAMED --add-opens java.base/java.net=ALL-UNNAMED --add-opens java.base/java.util.concurrent.atomic=ALL-UNNAMED -XX:MaxMetaspaceSize=256m -XX:+HeapDumpOnOutOfMemoryError -Xms256m -Xmx512m -Dfile.encoding=US-ASCII -Duser.country=US -Duser.language=en -Duser.variant -cp /opt/gradle-7.4/lib/gradle-launcher-7.4.jar org.gradle.launcher.daemon.bootstrap.GradleDaemon 7.4
```

The way the daemons are started are very different. I originally started the Gradle daemon in the s6 `run` script with `foreground`. This is because s6 monitors the `run` script. When that script exits, `s6` believes the process has died and must be restarted. Starting the daemon with `gradle --daemon` exits immediately. This results in `s6` creating a lot of gradle daemons.

I went back and experimented with the first `java_gradle` image we created, since it doesn't start the Gradle daemon automatically. I started a container, started a Gradle daemon, and ran an `init`, and everything worked and looked as expected.

1. Starting a daemon.

  ```
root@188b4a15a621:/project# gradle --daemon
Starting a Gradle Daemon (subsequent builds will be faster)

> Task :help

Welcome to Gradle 7.4.

Directory '/project' does not contain a Gradle build.

To create a new build in this directory, run gradle init

For more detail on the 'init' task, see https://docs.gradle.org/7.4/userguide/build_init_plugin.html

For more detail on creating a Gradle build, see https://docs.gradle.org/7.4/userguide/tutorial_using_tasks.html

To see a list of command-line options, run gradle --help

For more detail on using Gradle, see https://docs.gradle.org/7.4/userguide/command_line_interface.html

For troubleshooting, visit https://help.gradle.org

BUILD SUCCESSFUL in 5s
1 actionable task: 1 executed
```

1. Checking the processes. Note the daemon is PID 133.

  ```
root@188b4a15a621:/project# ps auxf
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root        55  0.0  0.0   4116  3496 pts/0    Ss   18:56   0:00 /bin/bash
root       194  0.0  0.0   5900  2828 pts/0    R+   18:58   0:00  \_ ps auxf
root         1  0.0  0.0    208    64 ?        Ss   18:56   0:00 /package/admin/s6/command/s6-svscan -d4 -- /run/service
root        15  0.0  0.0    212    60 ?        S    18:56   0:00 s6-supervise s6-linux-init-shutdownd
root        16  0.0  0.0    204     4 ?        Ss   18:56   0:00  \_ /package/admin/s6-linux-init/command/s6-linux-init-shutdownd -c /run/s6/basedir -g 3000 -C -B
root        22  0.0  0.0    212    64 ?        S    18:56   0:00 s6-supervise s6rc-oneshot-runner
root        29  0.0  0.0    188     4 ?        Ss   18:56   0:00  \_ /package/admin/s6/command/s6-ipcserverd -1 -- /package/admin/s6/command/s6-ipcserver-access -v0 -E -l0 -i data/rules -- /package/admin/s6/command/s6-sudod -t 30000 -- /package/admin/s6-rc/command/s6-rc-oneshot-run -l ../.. --
root        23  0.0  0.0    212    60 ?        S    18:56   0:00 s6-supervise s6rc-fdholder
root       133 19.6  3.1 3329944 252920 ?      Ssl  18:57   0:10 /opt/jdk-17.0.2/bin/java --add-opens java.base/java.util=ALL-UNNAMED --add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.lang.invoke=ALL-UNNAMED --add-opens java.base/java.util=ALL-UNNAMED --add-opens java.prefs/java.util.prefs=ALL-UNNAMED --add-opens java.prefs/java.util.prefs=ALL-UNNAMED --add-opens java.base/java.nio.charset=ALL-UNNAMED --add-opens java.base/java.net=ALL-UNNAMED --add-opens java.base/java.util.concurrent.atomic=ALL-UNNAMED -XX:MaxMetaspaceSize=256m -XX:+HeapDumpOnOutOfMemoryError -Xms256m -Xmx512m -Dfile.encoding=US-ASCII -Duser.country=US -Duser.language=en -Duser.variant -cp /opt/gradle-7.4/lib/gradle-launcher-7.4.jar org.gradle.launcher.daemon.bootstrap.GradleDaemon 7.4
```

1. Gradle init ran fine. No notice about creating a new daemon.

  ```
root@188b4a15a621:/project# gradle init
...
...
...
BUILD SUCCESSFUL in 31s
2 actionable tasks: 2 executed
```

1. Double checking the processes.

  ```
root@188b4a15a621:/project# ps auxf
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root        55  0.0  0.0   4116  3500 pts/0    Ss   18:56   0:00 /bin/bash
root       261  0.0  0.0   5900  2876 pts/0    R+   19:00   0:00  \_ ps auxf
root         1  0.0  0.0    208    64 ?        Ss   18:56   0:00 /package/admin/s6/command/s6-svscan -d4 -- /run/service
root        15  0.0  0.0    212    60 ?        S    18:56   0:00 s6-supervise s6-linux-init-shutdownd
root        16  0.0  0.0    204     4 ?        Ss   18:56   0:00  \_ /package/admin/s6-linux-init/command/s6-linux-init-shutdownd -c /run/s6/basedir -g 3000 -C -B
root        22  0.0  0.0    212    64 ?        S    18:56   0:00 s6-supervise s6rc-oneshot-runner
root        29  0.0  0.0    188     4 ?        Ss   18:56   0:00  \_ /package/admin/s6/command/s6-ipcserverd -1 -- /package/admin/s6/command/s6-ipcserver-access -v0 -E -l0 -i data/rules -- /package/admin/s6/command/s6-sudod -t 30000 -- /package/admin/s6-rc/command/s6-rc-oneshot-run -l ../.. --
root        23  0.0  0.0    212    60 ?        S    18:56   0:00 s6-supervise s6rc-fdholder
root       133 11.3  4.1 3342384 335812 ?      Ssl  18:57   0:16 /opt/jdk-17.0.2/bin/java --add-opens java.base/java.util=ALL-UNNAMED --add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.lang.invoke=ALL-UNNAMED --add-opens java.base/java.util=ALL-UNNAMED --add-opens java.prefs/java.util.prefs=ALL-UNNAMED --add-opens java.prefs/java.util.prefs=ALL-UNNAMED --add-opens java.base/java.nio.charset=ALL-UNNAMED --add-opens java.base/java.net=ALL-UNNAMED --add-opens java.base/java.util.concurrent.atomic=ALL-UNNAMED -XX:MaxMetaspaceSize=256m -XX:+HeapDumpOnOutOfMemoryError -Xms256m -Xmx512m -Dfile.encoding=US-ASCII -Duser.country=US -Duser.language=en -Duser.variant -cp /opt/gradle-7.4/lib/gradle-launcher-7.4.jar org.gradle.launcher.daemon.bootstrap.GradleDaemon 7.4
```

1. Check the gradle status, and there's still only one daemon process.

  ```
root@188b4a15a621:/project# gradle --status
   PID STATUS   INFO
   133 IDLE     7.4
```

#### Playing around with daemon options

My first thought was to run `gradle --daemon` in the `run` script, use `ps` to glean the PID, and then have the `run` script `wait` on the created process.

Then I found an interesting [Stack Overflow post](https://stackoverflow.com/questions/58419074/gradle-daemon-not-reused-despite-same-java-gradle-version) that related to my issue. 

As seen in some of the runs above, the JVM arguments to `gradle --foreground` are much different than `gradle --daemon`. According to the Stack Overflow post, those arguments are why `gradle` thinks the `--foreground` daemon isn't compatible.

As an experiment, I killed all the running Gradle daemons, started `gradle --foreground` in the background (heh heh), and then ran a build with the `--info` flag. My results matched what was stated in the Stack Overflow post.

```
root@188b4a15a621:/project# gradle --stop
Stopping Daemon(s)
1 Daemon stopped

root@188b4a15a621:/project# gradle --foreground &
[1] 848

root@188b4a15a621:/project# gradle build --info
Initialized native services in: /root/.gradle/native
Initialized jansi services in: /root/.gradle/native
Found daemon DaemonInfo{pid=848, address=[5ff01f8e-6a50-4443-b938-8640c0a8673e port:44819, addresses:[/127.0.0.1]], state=Idle, lastBusy=1649534057093, context=DefaultDaemonContext[uid=bfb496fe-406b-4b50-9f35-2b9b14165b67,javaHome=/opt/jdk-17.0.2,daemonRegistryDir=/root/.gradle/daemon,pid=848,idleTimeout=10800000,priority=NORMAL,daemonOpts=-Xms64m,-Xmx64m,-Dfile.encoding=US-ASCII,-Duser.country=US,-Duser.language=en,-Duser.variant]} however its context does not match the desired criteria.
At least one daemon option is different.
Wanted: DefaultDaemonContext[uid=null,javaHome=/opt/jdk-17.0.2,daemonRegistryDir=/root/.gradle/daemon,pid=885,idleTimeout=null,priority=NORMAL,daemonOpts=--add-opens,java.base/java.util=ALL-UNNAMED,--add-opens,java.base/java.lang=ALL-UNNAMED,--add-opens,java.base/java.lang.invoke=ALL-UNNAMED,--add-opens,java.base/java.util=ALL-UNNAMED,--add-opens,java.prefs/java.util.prefs=ALL-UNNAMED,--add-opens,java.prefs/java.util.prefs=ALL-UNNAMED,--add-opens,java.base/java.nio.charset=ALL-UNNAMED,--add-opens,java.base/java.net=ALL-UNNAMED,--add-opens,java.base/java.util.concurrent.atomic=ALL-UNNAMED,-XX:MaxMetaspaceSize=256m,-XX:+HeapDumpOnOutOfMemoryError,-Xms256m,-Xmx512m,-Dfile.encoding=US-ASCII,-Duser.country=US,-Duser.language=en,-Duser.variant]
Actual: DefaultDaemonContext[uid=bfb496fe-406b-4b50-9f35-2b9b14165b67,javaHome=/opt/jdk-17.0.2,daemonRegistryDir=/root/.gradle/daemon,pid=848,idleTimeout=10800000,priority=NORMAL,daemonOpts=-Xms64m,-Xmx64m,-Dfile.encoding=US-ASCII,-Duser.country=US,-Duser.language=en,-Duser.variant]

  Looking for a different daemon...
...
...
...
```

The post goes on to state that the `WANTED` options could be initialized in `GRADLE_OPTS`, and then the daemon should match. Sounds worth a try.

First make sure no daemons are running.

```
root@188b4a15a621:/project# gradle --stop

> Connecting to Daemon
Received command: Stop[id=e05f2124-e982-4c47-b0c0-a0b0f238ac0a].
Stopping Daemon(s) immediately stop command received
2 Daemons stopped
[1]+  Done                    gradle --info --foreground

root@188b4a15a621:/project# gradle --status
No Gradle daemons are running.
   PID STATUS   INFO
   133 STOPPED  (stop command received)
   539 STOPPED  (stop command received)
   730 STOPPED  (stop command received)
   848 STOPPED  (stop command received)
   920 STOPPED  (stop command received)
```

Then set values for `GRADLE_OPTS`.

```sh
GRADLE_OPTS="--add-opens java.base/java.util=ALL-UNNAMED --add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.lang.invoke=ALL-UNNAMED --add-opens java.base/java.util=ALL-UNNAMED --add-opens java.prefs/java.util.prefs=ALL-UNNAMED --add-opens java.prefs/java.util.prefs=ALL-UNNAMED --add-opens java.base/java.nio.charset=ALL-UNNAMED --add-opens java.base/java.net=ALL-UNNAMED --add-opens java.base/java.util.concurrent.atomic=ALL-UNNAMED -XX:MaxMetaspaceSize=256m -XX:+HeapDumpOnOutOfMemoryError -Xms256m -Xmx512m -Dfile.encoding=US-ASCII -Duser.country=US -Duser.language=en -Duser.variant"
```

Start a foreground daemon in the background. I love it.

```sh
gradle --foreground &
```

```
[1] 1078
Daemon server started.
```

Run a build.

```sh
gradle build
```

Well, setting `GRADLE_OPTS` in the shell didn't work, but putting it at the front of the `gradle --foreground &` command did.

```sh
GRADLE_OPTS="--add-opens java.base/java.util=ALL-UNNAMED --add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.lang.invoke=ALL-UNNAMED --add-opens java.base/java.util=ALL-UNNAMED --add-opens java.prefs/java.util.prefs=ALL-UNNAMED --add-opens java.prefs/java.util.prefs=ALL-UNNAMED --add-opens java.base/java.nio.charset=ALL-UNNAMED --add-opens java.base/java.net=ALL-UNNAMED --add-opens java.base/java.util.concurrent.atomic=ALL-UNNAMED -XX:MaxMetaspaceSize=256m -XX:+HeapDumpOnOutOfMemoryError -Xms256m -Xmx512m -Dfile.encoding=US-ASCII -Duser.country=US -Duser.language=en -Duser.variant" gradle --foreground &
```

The `daemonOpts` still don't match enough. They're so close, and only off by one char each because of parsing by `gradle`.

```
Wanted: DefaultDaemonContext[uid=null                                ,javaHome=/opt/jdk-17.0.2,daemonRegistryDir=/root/.gradle/daemon,pid=1580,idleTimeout=null    ,priority=NORMAL,daemonOpts=--add-opens,java.base/java.util=ALL-UNNAMED,--add-opens,java.base/java.lang=ALL-UNNAMED,--add-opens,java.base/java.lang.invoke=ALL-UNNAMED,--add-opens,java.base/java.util=ALL-UNNAMED,--add-opens,java.prefs/java.util.prefs=ALL-UNNAMED,--add-opens,java.prefs/java.util.prefs=ALL-UNNAMED,--add-opens,java.base/java.nio.charset=ALL-UNNAMED,--add-opens,java.base/java.net=ALL-UNNAMED,--add-opens,java.base/java.util.concurrent.atomic=ALL-UNNAMED,-XX:MaxMetaspaceSize=256m,-XX:+HeapDumpOnOutOfMemoryError,-Xms256m,-Xmx512m,-Dfile.encoding=US-ASCII,-Duser.country=US,-Duser.language=en,-Duser.variant]
Actual: DefaultDaemonContext[uid=445c5561-3851-4d4e-a151-389b3908344f,javaHome=/opt/jdk-17.0.2,daemonRegistryDir=/root/.gradle/daemon,pid=1514,idleTimeout=10800000,priority=NORMAL,daemonOpts=--add-opens=java.base/java.util=ALL-UNNAMED,--add-opens=java.base/java.lang=ALL-UNNAMED,--add-opens=java.base/java.lang.invoke=ALL-UNNAMED,--add-opens=java.base/java.util=ALL-UNNAMED,--add-opens=java.prefs/java.util.prefs=ALL-UNNAMED,--add-opens=java.prefs/java.util.prefs=ALL-UNNAMED,--add-opens=java.base/java.nio.charset=ALL-UNNAMED,--add-opens=java.base/java.net=ALL-UNNAMED,--add-opens=java.base/java.util.concurrent.atomic=ALL-UNNAMED,-XX:MaxMetaspaceSize=256m,-XX:+HeapDumpOnOutOfMemoryError,-Xms256m,-Xmx512m,-Dfile.encoding=US-ASCII,-Duser.country=US,-Duser.language=en,-Duser.variant]
```

Look closely at the first daemon option for Wanted and Actual.

```
--add-opens,java.base/java.util=ALL-UNNAMED,
--add-opens=java.base/java.util=ALL-UNNAMED,
```

To continue down this path I'd need to build gradle and follow what's going on by stepping through the code. For now I'm going to try the alternate approach of running `gradle --daemon` and capturing it's PID.

#### Run the gradle daemon, grab the PID, and wait

First, let's run the Gradle daemon and see if we can grab it's PID.

One of the cleaner options I found to get a PID is `pgrep` it's in the `procps` package for Ubuntu.

```sh
apt install procps
```

```
Reading package lists... Done
Building dependency tree
Reading state information... Done
procps is already the newest version (2:3.3.16-1ubuntu2.3).
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
```

Now start the daemon.

```sh
gradle --daemon
```

Once it's up and running us `pgrep` to find the process id. The `-f` flag allows us to specify grepping against the full command line. This way we can use "GradleDaemon" instead of just "java".

```sh
pgrep -f GradleDaemon
```

```
697
```

Next, let's try running a gradle daemon with a short, 20 second timeout. That'll give us time for a build and then some status checks. For this we need a `gradle.properties` file in the project root that defines the timeout period. I wish there was a way to do this on the command line.

```sh
cat <<EOF > gradle.properties
org.gradle.daemon.idletimeout=20000
EOF
```

Check the status to be sure we have a clean slate.

```sh
gradle --status
```

```
...
No Gradle daemons are running.
...
```

Now start a daemon.

```sh
gradle --daemon
```

```
Starting a Gradle Daemon (subsequent builds will be faster)
...

BUILD SUCCESSFUL in 6s
1 actionable task: 1 executed
```

Let's check the status again quickly before the timeout fires.

```sh
gradle --status
```

```
   PID STATUS   INFO
   136 IDLE     7.4
```

And then check again after 20 seconds or more has passed.

```sh
gradle --status
```

```
No Gradle daemons are running.
```

Finally, let's create a shell script that
1. starts the daemon
2. captures the PID
3. waits on the process

```sh
#/bin/sh
cat <<'EOF' > dstart.sh
echo Starting daemon
echo
gradle --daemon

echo Capturing PID
GDPID=$(pgrep -f GradleDaemon)
echo $GDPID
echo

echo Waiting on daemon timeout...
echo
wait $GDPID

echo DONE!
EOF

chmod +x dstart.sh
```

Now run it.

```sh
./dstart.sh
```

```
Starting daemon

Starting a Gradle Daemon, 2 stopped Daemons could not be reused, use --status for details

> Task :help

Welcome to Gradle 7.4.

To run a build, run gradle <task> ...

To see a list of available tasks, run gradle tasks

To see more detail about a task, run gradle help --task <task>

To see a list of command-line options, run gradle --help

For more detail on using Gradle, see https://docs.gradle.org/7.4/userguide/command_line_interface.html

For troubleshooting, visit https://help.gradle.org

BUILD SUCCESSFUL in 5s
1 actionable task: 1 executed
Capturing PID
969

Waiting on daemon timeout...

./dstart.sh: line 12: wait: pid 969 is not a child of this shell
```

:rage_face:

#### Back to dumpster diving in Gradle's source code

Current Gradle source requires JDK 11 to build completely. JDK 17 won't cut it.

[Download JDK 11.0.2 for Linux here](https://download.java.net/java/GA/jdk11/13/GPL/openjdk-11.0.1_linux-x64_bin.tar.gz)

[Download Gradle here](https://gradle.org/next-steps/?version=7.4.2&format=all)

Create an image

```sh
docker build -t jdk11gradle -f- . <<"EOB"
FROM dde

ARG JINSTALLER=openjdk-11.0.2_linux-x64_bin.tar.gz
ARG JDK_VER=11.0.2
ARG JDK_PATH=/opt/jdk-11.0.2

ARG GINSTALLER=gradle-7.4.2-all.zip
ARG GRADLE_VER=7.4.2

# Copy installation archives
COPY $JINSTALLER /tmp
COPY $GINSTALLER /tmp

# Install Java
RUN tar -zvxf /tmp/${JINSTALLER} -C /opt

# Install Gradle
RUN unzip -d /opt /tmp/${GINSTALLER}

# Add Java and Gradle to PATH
ENV PATH="${PATH}:${JDK_PATH}/bin:${JDK_PATH}/db/bin:${JDK_PATH}/jre/bin:/opt/gradle-${GRADLE_VER}/bin"

# Environment Vars
ENV J2SDKDIR=${JDK_PATH} \
    J2REDIR=${JDK_PATH} \
    JAVA_HOME=${JDK_PATH} \
    DERBY_HOME=${JDK_PATH}/db

# Don't override ENTRYPOINT
EOB
```

Run it, test it, and bail out.

```sh
docker run -it --rm --name gjdk11 jdk11gradle /bin/bash
```

```sh
java -version
gradle --status
```

```sh
exit
```

Checkout Gradle's source code and build it with the image.

```sh
git clone git@github.com:gradle/gradle.git
```

```sh
cd gradle
```

Run the image with a bind mount to this directory.

```sh
docker run -it --rm -v "$(pwd)":/project -w /project --name gjdk11 jdk11gradle /bin/bash
```

Build Gradle with Gradle

```sh
gradle build
```

It blows up with a failed dependency download for a Kotlin plugin.

```
* What went wrong:
Error resolving plugin [id: 'com.gradle.enterprise', version: '3.9']
> A problem occurred configuring project ':build-logic-settings:build-logic-settings-plugin'.
   > Dependency verification failed for configuration ':build-logic-settings:build-logic-settings-plugin:classpath'
     One artifact failed verification: gradle-kotlin-dsl-plugins-2.1.7.jar (org.gradle.kotlin:gradle-kotlin-dsl-plugins:2.1.7) from repository Gradle Central Plugin Repository
     If the artifacts are trustworthy, you will need to update the gradle/verification-metadata.xml file by following the instructions at https://docs.gradle.org/7.4.2/userguide/dependency_verification.html#sec:troubleshooting-verification
```

Let's go see how to [troubleshoot verification](https://docs.gradle.org/7.4.2/userguide/dependency_verification.html#sec:troubleshooting-verification).

Looks like verification of metadata and signatures can be disabled for dependencies, but it also has to be done for multiple subprojects. If I disable it at the top level, many more fail at the subproject levels. I'm not really interested in troubleshoot further because I think I've learned how Gradle was designed to operate.

Gradle wasn't designed to be run as a system level, generic daemon handling build requests from multiple projects. It was designed to be available to build a specific project. It think this drives a few things I've run into.

1. There's no separate way to start a daemon without doing a build.
2. Many settings like `gradle.properties` are found at the project level.
3. If the daemonOptions don't match, `gradle` will start a new daemon.

So at this point, I think it's fine to just let it be. I'll be removing gradle daemon start up from s6, and just letting it happen when a build is run.

The next section describes what happens when we contine with the tutorial. It can be done with any version of the docker image in this article.

### Initialize a project with Gradle

Make sure an image is running.

```sh
docker run -d --rm -v "$(pwd)":/project -w /project --name jgdemo java_gradle
docker exec -it jgdemo /bin/bash
```

Follow along with the tutorial's [Init Task](https://docs.gradle.org/current/samples/sample_building_java_applications.html#run_the_init_task) step. It may be slightly different. My particular session is shown below.

```
root@d255eaaf8d01:/project# gradle init

Select type of project to generate:
  1: basic
  2: application
  3: library
  4: Gradle plugin
Enter selection (default: basic) [1..4] 2

Select implementation language:
  1: C++
  2: Groovy
  3: Java
  4: Kotlin
  5: Scala
  6: Swift
Enter selection (default: Java) [1..6] 3

Split functionality across multiple subprojects?:
  1: no - only one application project
  2: yes - application and library projects
Enter selection (default: no - only one application project) [1..2] 1

Select build script DSL:
  1: Groovy
  2: Kotlin
Enter selection (default: Groovy) [1..2] 1

Generate build using new APIs and behavior (some features may change in the next minor release)? (default: no) [yes, no] no
Select test framework:
  1: JUnit 4
  2: TestNG
  3: Spock
  4: JUnit Jupiter
Enter selection (default: JUnit Jupiter) [1..4] 1

Project name (default: project): demo
Source package (default: demo): demo

> Task :init
Get more help with your project: https://docs.gradle.org/7.4/samples/sample_building_java_applications.html

BUILD SUCCESSFUL in 1m 57s
2 actionable tasks: 2 executed
```

### Double check the bind mount

Run `tree` in the container.

```
root@d255eaaf8d01:/project# tree
.
|-- app
|   |-- build.gradle
|   `-- src
|       |-- main
|       |   |-- java
|       |   |   `-- demo
|       |   |       `-- App.java
|       |   `-- resources
|       `-- test
|           |-- java
|           |   `-- demo
|           |       `-- AppTest.java
|           `-- resources
|-- gradle
|   `-- wrapper
|       |-- gradle-wrapper.jar
|       `-- gradle-wrapper.properties
|-- gradlew
|-- gradlew.bat
`-- settings.gradle

12 directories, 8 files
```

Now exit out of the container and run `tree` on the host.

```
λ ~/src/dde_gradle: tree
.
├── app
│   ├── build.gradle
│   └── src
│       ├── main
│       │   ├── java
│       │   │   └── demo
│       │   │       └── App.java
│       │   └── resources
│       └── test
│           ├── java
│           │   └── demo
│           │       └── AppTest.java
│           └── resources
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
└── settings.gradle

12 directories, 8 files
```

### Run the application

Reconnect to the container.

```sh
docker exec -it jgdemo /bin/bash
```

Run the project. It may need to download dependencies.

```sh
./gradle run
```

```
Downloading https://services.gradle.org/distributions/gradle-7.4-bin.zip
...........10%...........20%...........30%...........40%...........50%...........60%...........70%...........80%...........90%...........100%

> Task :app:run
Hello World!

BUILD SUCCESSFUL in 11s
2 actionable tasks: 2 executed
```

That was an interesting journey with lots of plot twists. Until next time.
