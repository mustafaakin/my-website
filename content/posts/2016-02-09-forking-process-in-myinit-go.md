---
title: Writing my own init with Go - Part 2
date: 2016-02-09
---

I am working on myinit.go. Init process is no good if it cannot spawn other processes. I will use the other processes for critical functionality, i.e getting an IP address for future interactivity over SSH or maybe HTTP, or possibly Web Sockets. Because, why not? 

Anyways, I modified `myinit.go` to following and put it in the `/init` folder to have a nicer structure. It basically spawns a new process, and prints it output. The modified init code is as follows:

```go
package main 

import (
    "log"
    "os"
    "os/exec"
	"io"
)


func execute(filepath string) error {
    // Prepare running a command!
       
    cmd := exec.Command(filepath)    
    sout, _ := cmd.StdoutPipe()
    serr, _ := cmd.StderrPipe()
                
    // Start and get the output
    err := cmd.Start()
    if err != nil {
        log.Println("Error starting the program", err)
    }
    
    go io.Copy(os.Stdout, sout)
    go io.Copy(os.Stdout, serr)    
    
    err = cmd.Wait()
    if err != nil {
        log.Println("Error waiting program")
    }
    
    log.Println("Your program exited!")
    return nil
}


func main(){
    log.SetOutput(os.Stderr)
    
    log.Println("Hello world from Go!")
    log.Println("Gopher is alive!")
    
    log.Println("Executing some binary!")
    
    execute("/init/test/mybinary")    
    // Wait forever
        
    for {
        
    }    
}
```

And my `/init/test/mybinary.go` is just a println to stderr. 

```go
package main 

import (
    "log"
    "os"
)

func main(){
    log.SetOutput(os.Stderr)
    log.Println("Hello world, stderr")
    
    log.SetOutput(os.Stdout)
    log.Println("Hello world, stdout")
}
```

I upgraded to Ubuntu 16.04 having 4.4 Kernel, so outputs might change a little. For instance, in 3.19 I could not write to stdout, but now I can write to both stdout & stderr as well. 

The output is following: 

```text
[    6.441931] EXT4-fs (sda1): mounted filesystem with ordered data mode. Opts: (null)
done.
Begin: Running /scripts/local-bottom ... done.
Begin: Running /scripts/init-bottom ... done.
2016/02/08 21:48:50 Hello world from Go!
2016/02/08 21:48:50 Gopher is alive!
2016/02/08 21:48:50 Executing some binary!
2016/02/08 21:48:50 Hello world, stderr
2016/02/08 21:48:50 Hello world, stdout
2016/02/08 21:48:50 Your program exited!

```

Copying the stdout and stderr with go is less then ideal, we should be able to just create files for each and pass their file descriptiors. At least that's what I think.

Sometimes sync does not sync, I might be using it wrong. After a little research, I found out that `sudo blockdev --flushbufs /dev/nbd0p1` works better.

I modified the code so that the stdout and stderr is redirected to files. Also, the kernel paremeters must have the `rw` flag or you have to remount the /dev/sda1 as rw, but changing kernel parameters are much more easier.

```bash
$ kvm -m 1G -nographic -kernel vmlinuz-4.4.0-2-generic -initrd initrd.img-4.4.0-2-generic -append "console=ttyS0 root=/dev/sda1 rw init=/init/myinit" -hda mydisk.img 
```

```go
func listFiles(filepath string) {
    files, _ := ioutil.ReadDir(filepath)
    for _, f := range files {
        log.Println(f.Name())
    }    
}

func execute(filepath string) error {
    cmd := exec.Command(filepath)   
    
    fout, err := os.Create("/init/test/stdout")
    if err != nil {
        log.Fatal(err)
    }
    ferr, err := os.Create("/init/test/stderr")
    if err != nil {
        log.Fatal(err)
    }
    
    cmd.Stdout = fout
    cmd.Stderr = ferr
    defer fout.Close()
    defer ferr.Close()
    ....


func main(){
	....
    execute("/init/test/mybinary")    
    
    listFiles("/init/test")
    sout, _ := ioutil.ReadFile("/init/test/stdout")
    serr, _ := ioutil.ReadFile("/init/test/stderr")
    
    log.Println("Stdout of program", string(sout))
    log.Println("Stderr of program", string(serr))
    ....

```

And then the output becomes:

```text
2016/02/08 22:07:58 Hello world from Go!
2016/02/08 22:07:58 Gopher is alive!
2016/02/08 22:07:58 Executing some binary!
2016/02/08 22:07:58 Your program exited!
2016/02/08 22:07:58 mybinary
2016/02/08 22:07:58 mybinary.go
2016/02/08 22:07:58 stderr
2016/02/08 22:07:58 stdout
2016/02/08 22:07:58 Stdout of program 2016/02/08 22:07:58 Hello world, stdout

2016/02/08 22:07:58 Stderr of program 2016/02/08 22:07:58 Hello world, stderr
```

This has been a fun ride, I can spawn processes, see and save their outputs. However, I have no idea what will happen for long running commands. Because, unlike using `dup2`, I do not duplicate the actual file descriptors, but it uses `io.Copy` inside, and one or more Go routine is probably spawned in the background because docs for `os.Exec` states:

```text
	// Stdout and Stderr specify the process's standard output and error.
	//
	// If either is nil, Run connects the corresponding file descriptor
	// to the null device (os.DevNull).
	//
	// If Stdout and Stderr are the same writer, at most one
	// goroutine at a time will call Write.
	Stdout io.Writer
	Stderr io.Writer
```

and I see several Go routines for `io.Copy` in `stdin()` and `writerDescriptor()` functions inside it. Therefore, better solution would be to avoid Go's higher level functions such as `os.Command()` and use the `ForkExec(argv0 string, argv []string, attr *ProcAttr) (pid int, err error)` that takes the following structs as a parameter besides program name and parameters:

```go
type ProcAttr struct {
        Dir   string    // Current working directory.
        Env   []string  // Environment.
        Files []uintptr // File descriptors.
        Sys   *SysProcAttr
}

type SysProcAttr struct {
        Chroot      string         // Chroot.
        Credential  *Credential    // Credential.
        Ptrace      bool           // Enable tracing.
        Setsid      bool           // Create session.
        Setpgid     bool           // Set process group ID to Pgid, or, if Pgid == 0, to new pid.
        Setctty     bool           // Set controlling terminal to fd Ctty (only meaningful if Setsid is set)
        Noctty      bool           // Detach fd 0 from controlling terminal
        Ctty        int            // Controlling TTY fd
        Foreground  bool           // Place child's process group in foreground. (Implies Setpgid. Uses Ctty as fd of controlling TTY)
        Pgid        int            // Child's process group ID if Setpgid.
        Pdeathsig   Signal         // Signal that the process will get when its parent dies (Linux only)
        Cloneflags  uintptr        // Flags for clone calls (Linux only)
        UidMappings []SysProcIDMap // User ID mappings for user namespaces.
        GidMappings []SysProcIDMap // Group ID mappings for user namespaces.
        // GidMappingsEnableSetgroups enabling setgroups syscall.
        // If false, then setgroups syscall will be disabled for the child process.
        // This parameter is no-op if GidMappings == nil. Otherwise for unprivileged
        // users this should be set to false for mappings work.
        GidMappingsEnableSetgroups bool
}
```

As you see this struct has many features from users to tty's settings and even uid-gid mappings! I am still reading how to handle tty stuff properly. This was just an experiment to see how can we spawn dumb processes that spawns and dies. As I mentioned earlier, I will use those processes to offload some functionality from init, such as getting an IP from DCHP and setting it on interfaces. Therefore, my init can be simpler. 