# docker-issues
A list of issues I have found using Docker on Windows &amp; Mac with
workarounds/fixes for each.

Issue list:

- [io.timeout error connecting to the Docker daemon](#01-io-timeout)
- [Certificate issue after every restart of the Boot2Docker VM](#02-cert-issue)
- [Unable to mount local path to volume on container](#03-unable-to-mount)

## Issues that occur on both Windows and Mac OS X

The following issues can occur on either OS.

###<a name="01-io-timeout"></a> io.timeout error connecting to the Docker daemon

This is a weird one as it seems to occur for folks that have Cisco
AnyConnect installed on their machines for accessing corporate VPNs. For me
I have AnyConnect installed on both my PC and Mac but I only hit this issue
on my Mac.

#### Reason

Seems to be related to Cisco AnyConnect messing with the routing for the host
only network configured on the Boot2docker VM. The host only network is used
by the Docker client to connect to the daemon running on the VM.

References:
- https://github.com/boot2docker/boot2docker/issues/392
- http://stackoverflow.com/questions/26686358/docker-cant-connect-to-boot2docker-because-of-tcp-timeout

#### Solution

I managed to resolve the issue on my Mac by executing the following command
in a terminal:

```
$ sudo route -nv add -net 192.168.59 -interface vboxnet1
```

Note that the name of the host only interface on your machine might be
different so check with the VirtualBox UI first.

References:
- http://stackoverflow.com/a/26804653


###<a name="02-cert-issue"></a> Certificate issue after every restart of the Boot2Docker VM

After upgrading to Docker 1.7 I ran into an issue where I received the following error each time I tried to connect to the Docker daemon:

```
An error occurred trying to connect: Get
https://192.168.59.103:2376/v1.19/containers/json: x509: certificate is valid
for 127.0.0.1, 10.0.2.15, not 192.168.59.103
```

#### Reason

Seems to be something to do with eth1 not being up on the VM when the
SSL certificates are created.

References:
- https://github.com/boot2docker/boot2docker/issues/938

#### Solution

The fix that I found to be successful is documented here: https://gist.github.com/garthk/d5a17007c277aa5c76de

The solution consists of the following steps:

1. SSH into the Boot2docker VM:

  ```
  $ boot2docker ssh
  ```

2. Open the /var/lib/boot2docker/profile file and add the script snippet below:

  ```
  $ sudo vi /var/lib/boot2docker/profile
  ```

  Script to add:
  ```bash
  wait4eth1() {
    CNT=0
    until ip a show eth1 | grep -q UP
    do
      [ $((CNT++)) -gt 60 ] && break || sleep 1
    done
    sleep 1
  }
  wait4eth1
  ```

3. Save the file, exit SSH and restart the Boot2docker VM:

  ```
  $ boot2docker down
  $ boot2docker up
  ```

After restarting you will be able to connect to the VM every time it is
started.


## Issues with Docker using Git or Windows (msysgit)

The following issues are specific to using Docker from the Git for Windows bash
prompt.

###<a name="03-unable-to-mount"></a> Unable to mount local path to volume on container

I get this issue with Git for Windows 1.9.5.msysgit.1 and it manifests itself by
returning an error when I attempt to mount a local folder under my c:\Users
folder to a container.

For example, the following command:
```
$ docker run -it --rm -v /c/Users/me:/var/tmp -w /var/tmp ruby
```

Will result in an error that looks like this:

```
invalid value "c:\\Users\\me;C:\\Program Files (x86)\\Git\\var\\tmp" for
 flag -v: \Users\me;C:\Program Files (x86)\Git\var\tmp is not an absolute pa
th
See 'c:\Program Files\Boot2Docker for Windows\docker.exe run --help'.
```

#### Reason

The error is caused by how msysgit parses relative paths that start with a
forward slash '/'.

References:
- https://github.com/docker/docker/issues/12590

#### Solution

The solution that I used was to prefix all paths used in calls to docker run
with an extra forward slash, i.e. in -v and -w switches.

So ```-v /c/Users/me:/var/tmp``` becomes ```-v //c/Users/me:/var/tmp```.
When using ```$(pwd)``` to explode the current working directory then
```-v $(pwd):/var/tmp``` becomes ```-v /$(pwd):/var/tmp```.

The -w switch is also affected so ```-w /var/tmp``` becomes ```-w //var/tmp```.

As I write bash scripts that execute Docker commands that must run on both
Windows and Mac machines I use the following snippet to detect whether the
extra forward slash is required:

```bash
OS=$(uname)
if [[ $OS == MINGW* ]]; then
  # Using msysgit
  # -v /$(pwd)
else
  # Not using msysgit
  # -v $(pwd)
fi
```

References:
- https://github.com/docker/docker/issues/12590#issuecomment-96767796
