---
title: Writing my own init with Go - Part 3, Packaging/Configs
date: 2016-02-12
---

When this post gained much traction in HackerNews and r/golang, I was both suprised and happy. The general interest made me more motivated to do more and write more. And also some good discussions happend on [HackerNews thread](https://news.ycombinator.com/item?id=11064694) that was & will be helpful to me. I decided to package myinit in Go, and rename it to [gonit](https://github.com/mustafaakin/gonit) Contributions and ideas are welcome through [email](mailto:mustafa91@gmail.com) and in the issues of the GitHub repo. 

Earlier, I was exploring how to properly spawn processes. Upon discussion and exploring further, I found out that, I had to dig deeper. There is a nice function in `syscall` package, `func ForkExec(argv0 string, argv []string, attr *ProcAttr) (pid int, err error)` that forks and execs a given process with given arguments and additional `ProcAttr` in which you can define environment and open files. It handles most of the stuff, even the user/group namespaces. Of course, that function alone does not expose all raw functionality, but whatever you need can be possibly added via extending it with additional syscalls.

```go
func setupSignalHandler() {
	c := make(chan os.Signal, 1)
	signal.Notify(c, os.Interrupt,
		syscall.SIGCHLD,
		syscall.SIGTERM,
		syscall.SIGKILL)

	go func() {
		for sig := range c {
			log.Printf("captured %v", sig)
		}
	}()
}

...

fstdin, err := os.Create(stdinpath)
fstdout, err := os.Create(stdoutpath)
fstderr, err := os.Create(stderrpath)

procAttr := &syscall.ProcAttr{
	Dir:   "/",
	Env:   []string{"MYVAR=345"},
	Files: []uintptr{fstdin.Fd(), fstdout.Fd(), fstderr.Fd()},
	Sys:   nil,
}

pid, err := syscall.ForkExec(filepath, nil, procAttr)
```

I now spawn the services as in the above. I still write their outputs to file. By providing some sensible structure in `procAttr.Sys`, we can make the kernel allocate a tty for it, or make it use one from `/dev/pts`, I guess. I am still looking into that stuff for proper handling. I am catching SIGCHLD, but not so certain yet how to handle them properly. In the following order, sources of Sysvinit, Upstart, SystemD and Suckless Init is helping a lot. 

On the other hand, I converted the build, deploy and run pipeline in Makefile. Also, I used a better logger, the logger we deserve, logrus. It helps me to do the following type of logging, which is much better than regular string messages, or printf based messages.

```golang
log.WithFields(log.Fields{
	"service": filepath,
	"pid":     pid,
}).Info("Started service succesfully")
```

A simple output is now as follows:

```text
# make start_vm 
rm -Rf /tmp/gonit-build 
mkdir -p /tmp/gonit-build/init
mkdir -p /tmp/gonit-build/services
COMPILE init
go build -o /tmp/gonit-build/init/gonit github.com/mustafaakin/gonit/cmd/init
COMPILE services
CGO_ENABLED=0 go build -o /tmp/gonit-build/services/networking github.com/mustafaakin/gonit/cmd/networking
go build -o /tmp/gonit-build/services/terminals github.com/mustafaakin/gonit/cmd/terminals
PACKAGE package
# Folders, for better readability
mkdir -p /tmp/gonit-disk/init
mkdir -p /tmp/gonit-disk/init/services
# Copy required bins
cp /tmp/gonit-build/init/gonit /tmp/gonit-disk/init/gonit
cp /tmp/gonit-build/services/networking /tmp/gonit-disk/init/services/networking
cp /tmp/gonit-build/services/terminals  /tmp/gonit-disk/init/services/terminals
# Copy configs
cp config.yml /tmp/gonit-disk/config.yml
sync
blockdev --flushbufs /dev/nbd0p1
blockdev --flushbufs /dev/nbd0
sync
sleep 2 
RUN start_vm
# TODO: The following sync and flushbufs needs to be changed, we will just disable cache on our image but it seems problematic on rbd?
kvm -m 1G -nographic -kernel /boot/vmlinuz-4.4.0-2-generic -initrd /boot/initrd.img-4.4.0-2-generic -append "console=ttyS0 root=/dev/sda1 rw init=/init/gonit quiet" -hda ~/gonit-disk.img
/dev/sda1: recovering journal
/dev/sda1: clean, 28/65536 files, 10462/261888 blocks
INFO[0000] The gonit is booting. Go is now in control.  
INFO[0000] Starting services                             title=Gonit based OS version=0.1
INFO[0000] Starting service                              service=networking
INFO[0000] Started service succesfully                   pid=188 service=/init/services/networking
INFO[0000] Waiting for 3 seconds                        
INFO[0000] captured child exited                        
INFO[0003] Service ended.                                service=networking stderr=I will do networking and stuff later
lo  65536
ens3 52:54:00:12:34:56 1500
 stdout=WAT
INFO[0003] Starting service                              service=terminals
INFO[0003] Started service succesfully                   pid=191 service=/init/services/terminals
INFO[0003] Waiting for 3 seconds                        
INFO[0003] captured child exited                        
INFO[0006] Service ended.                                service=terminals stderr=I will arrange terminals and stuff
 stdout=
```

I sometimes still have issues with syncing the disk image, sometimes it gets corrupt just before booting the image and init or some other tools fails.

As you can see, we have some dummy services. They are defined in a .yaml file, that will be subject to change. We also need the dependencies, start order, waiting them properly and handling crashes and respawns. As they write their output to files, I just used sleep 3 and read the contents of the files, and logged them. In reality, they can be running continously, or just once/twice etc. I still have to decide how should the services be. The init might also watch the `config.yaml` for changes. Also, rightnow we just get sigchld, we should also wait for killed/died children so they don't turn into zombies. 

```yaml
version: 0.1
title: Gonit based OS
services:
  - name: networking
    description: Configures the networking
  - name: terminals
    description: Configures the TTYs for user input
```

One more interesting thing, although I made `networking` and `terminals` as dummy programs, In `networking` I have used the `net` package, causing my binaries to not be completely static. Because, net uses some glibc bindings, they cannot be statically included. When you just `go build` you will see the dependencies with `ldd` as follows:

```bash
$ go build -o /tmp/net cmd/networking/main.go && ldd /tmp/net
	linux-vdso.so.1 =>  (0x00007fffcc18b000)
	libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f1aac3b7000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f1aabfed000)
	/lib64/ld-linux-x86-64.so.2 (0x0000562d6de4a000)
```

To overcome this issue, you have to use `CGO_ENABLED=0` before `go build` and it still works as expected both on my host machine and the VM (as seen in above). 

```bash
$ CGO_ENABLED=0 go build -o /tmp/net cmd/networking/main.go && ldd /tmp/net
	not a dynamic executable
$ /tmp/net
I will do networking and stuff later
WATlo  65536
enp3s0 18:67:b0:2c:50:5e 1500
wlp2s0 c8:f7:33:e7:b8:55 1500
docker0 56:84:7a:fe:97:99 1500
virbr0 52:54:00:91:fc:ef 1500
virbr0-nic 52:54:00:91:fc:ef 1500
veth43bfe73 4a:df:b0:98:2a:f1 1500
```

As you see, there are still lots of stuff to do. But first, I have to decide how the system file layout should be. Standard layout is too standard, but I don't actually like it. Actually, it is not really related to init, but in near future, I will use my gonit to build a little useless distro that no one will probably use. It will also have a package manager, so deciding a file layout right now, or starting to think it would be a good idea. Yeah, I know who am I to change a file layout, but If you read this blog post series, this is just an experiment. Right now, I have the following structure in my mind:

```bash
/kernel  # boot images and initrds
/system
	/gonit # the init binary
	/config.yaml
	/services
		/networking
			/bin
			/logs
			config.yaml
		/terminals
			/bin
			/logs
			config.yaml

/programs   # still thinking about it, cowardly refusing to name them apps
	/myprogram1 
	/myprogram2

/cache # The cache if ever needed
	/services
	/programs

/libs # for compiled libs
	/32
	/64
	/multiarch # am I doing them right?  

# I want to change them if possible, but too many programs will probably depend on them statically
/sys
/fs
/proc
/run
```

To join me in this experiment, I would like you to visit the [github.com/mustafaakin/gonit](https://github.com/mustafaakin/gonit) , as I said earlier, any input is welcome. For my next post, I will look into networking in more detail. I know it is not init related, but I am trying to provide all the crucial utilities that will be base in my distro.