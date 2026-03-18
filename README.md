import architecture from '../../../img/csapp/shell_lab/architecture.jpg';
import job_state_machine from '../../../img/csapp/shell_lab/job_state_machine.jpg';
import execution_flow from '../../../img/csapp/shell_lab/execution_flow.jpg';

# Unix Shell

## Overview
I implemented a simple Unix shell in C to understand how shells manage processes, handle signals, and implement job control. This project deepened my understanding of:

- How shells parse and execute commands- The role of signals in process control
- How job control allows users to manage multiple processes
- The interaction between the shell and the kernel when executing commands and handling signals 

--- 

### Key Takeaways
- Implemented a basic shell that supports job control and signal handling
- Learned how to use fork, exec, wait, and signal handling in C
- Gained insight into how shells manage foreground and background processes
- Understood the importance of blocking signals to avoid race conditions
- Developed intuition for how the shell interacts with the kernel to manage processes and signals   

---

## Diagrams
### Architecture
Purpose: View of the system's structure and components
<img src={architecture} alt="CS:APP architecture" style={{maxWidth: '300px', width: '100%', height: 'auto'}} />

---

### Job State Machine
Purpose: View the life cycle of a job, and how different signals may move the job to a different state (Running, Stopped, Terminated)
<img src={job_state_machine} alt="CS:APP job state machine" style={{maxWidth: '700px', width: '100%', height: 'auto'}} />

---

### Execution Flow
Purpose: View of a command flowing through the shell to be executed as a builtin or child process(excluding signal handling)
<img src={execution_flow} alt="CS:APP execution flow" style={{maxWidth: '700px', width: '100%', height: 'auto'}} />

---

## Implementations
### eval
Functionality: Main routine that parses and interprets the command line
#### Design Choices
- Why use execvp instead of execve?
    - execve requires the full path while execvp uses the relative path + execvp inherits the environment variables automatically.
- Why must the forked child not have the same process group id as the parent process?
    - The shell is the parent process, therefore, the shell and the forked child share a process group id. This means that if the shell forwards a signal to the process group id of a forked child, then it would send it to itself as well. Not intended functionality. If forward SIGINT -> shell would close.
- What is the purpose of blocking and unblocking SIGCHLD? 
    - Blocking signals is required to not have race conditions between processes accessing the shared jobs. 
    - A simple race condition that is unable to occur because of the SIGCHLD signal blocking is the child process being forked and terminating before the parent process adds the job to the job list. The parent tries to delete a job that does not exist, then has a job that will never be deleted.
- Why is SIGCHLD unblocked after waitfg instead of after addjob?
    - Because waitfg is using a sigsuspend() that will unblock all signals temporarily, the process should enter this function with SIGCHLD blocked or else it would receive a SIGCHLD signal after the while loop condition and never return from sigsuspend().
```c
void eval(char *cmdline)
{
    char *argv[MAXARGS];
    int is_bg = parseline(cmdline, argv);

    if (!builtin_cmd(argv)) {
        pid_t pid;
        sigset_t mask, prev_mask;

        /* Block SIGCHLD to avoid race between fork/addjob and the SIGCHLD handler */
        sigemptyset(&mask);
        sigaddset(&mask, SIGCHLD);
        sigprocmask(SIG_BLOCK, &mask, &prev_mask);

        if ((pid = fork()) == 0) {
            /* Child: place in new process group so shell and jobs have different PGIDs */
            setpgid(0, 0);

            /* Restore signal mask inherited from parent */
            sigprocmask(SIG_SETMASK, &prev_mask, NULL);

            /* Use execvp to search PATH like a normal shell */
            execvp(argv[0], argv);

            /* If execvp returns, there was an error */
            printf("%s: Command not found\n", argv[0]);
            exit(EXIT_FAILURE);
        }

        /* Parent Shell Process */
        int state = is_bg ? BG : FG;
        addjob(jobs, pid, state, cmdline);
        int jid = pid2jid(pid);

        if (!is_bg)
            waitfg(pid);

        /* Unblock SIGCHLD now that job is added */
        sigprocmask(SIG_SETMASK, &prev_mask, NULL);

        if (is_bg)
            printf("[%d] (%d) %s", jid, pid, cmdline);
    }

    return;
}
```

---

### builtin_cmd
Functionality: Recognizes and interprets the built-in commands: quit, fg, bg, and jobs
```c
int builtin_cmd(char **argv)
{
    if (!strcmp(argv[0], "quit")) {
        exit(0);
    }

    if (!strcmp(argv[0], "jobs")) {
        listjobs(jobs);
        return 1;
    }

    if (!strcmp(argv[0], "bg") || !strcmp(argv[0], "fg")) {
        do_bgfg(argv);
        return 1;
    }

    return 0;      /* not a builtin command */
}
```

---

### do_bgfg
Functionality: Implements the bg and fg built-in commands
#### Design Choices
- Why does SIGCHLD get blocked before getjobjid/getjobpid?
    - getjobjid/getjobpid are accessing the jobs array that is shared between processes. Since do_bgfg is using the job it finds to change the state, this creates a potential race condition between the time it finds the job and changes the state. The job variable is a struct job_t *, so if our do_bgfg function grabs a job, but this job is deleted after finishing, then do_bgfg attempts to change the state member of a NULL object. This ends up with a null dereference, and most likely with crash the program. 
- Why is SIGCHLD unblocked after waitfg?
    - Because waitfg is using a sigsuspend() that will unblock all signals temporarily, the process should enter this function with SIGCHLD blocked or else it would receive a SIGCHLD signal after the while loop condition and never return from sigsuspend().
- Why is val checked against INT_MIN and INT_MAX?
    - strtol() reads a long int, but getjobpid() and getjobjid() can only pass an argument of size int to find that associated id. Therefore, we must see if the long int returned can be safely converted to an int.
```c
void do_bgfg(char **argv)
{
    struct job_t *job = NULL;
    char *endptr;
    long val;
    bool is_jid;

    if (argv[1] == NULL) {
        printf("%s command requires PID or %%jobid argument\n", argv[0]);
        return;
    }

    is_jid = (argv[1][0] == '%');

    errno = 0;
    val = strtol(is_jid ? argv[1] + 1 : argv[1], &endptr, 10);

    if (errno == ERANGE || *endptr != '\0') {
        printf("fg: argument must be a PID or %%jobid\n");
        return;
    }

    if (val < INT_MIN || val > INT_MAX) {
        printf("Argument out of range\n");
        return;
    }
    int id = (int)val;

    sigset_t mask, prev_mask;
    sigemptyset(&mask);
    sigaddset(&mask, SIGCHLD);
    sigprocmask(SIG_BLOCK, &mask, &prev_mask);

    if (is_jid) {
        job = getjobjid(jobs, id);
    } else {
        job = getjobpid(jobs, id);
    }

    if (job == NULL) {
        if (is_jid)
            printf("[%d]: No such job\n", id);
        else
            printf("(%d): No such process\n", id);

        sigprocmask(SIG_SETMASK, &prev_mask, NULL);
        return;
    }

    job->state = !strcmp(argv[0], "bg") ? BG : FG;

    int pid = job->pid;

    kill(-pid, SIGCONT);

    if (!strcmp(argv[0], "fg"))
        waitfg(pid);

    sigprocmask(SIG_SETMASK, &prev_mask, NULL);

    return;
}
```

---

### waitfg
Functionality: Waits for a foreground job to complete
#### Design Choices
- How does do_bgfg and eval help waitfg() avoid race conditions?
    - Any time that waitfg() is entered, the SIGCHLD signal is blocked. This allows the function to not have any race conditions between the condition that is evaluated, and the sigsuspend that waits for the associated foreground process to finish.
- Why is sigsuspend() used instead of sleep()?
    - Sleep() blocks the process. The reason that sigsuspend() is better is because it is an atomic unblock + wait.
```c
 21 void waitfg(pid_t pid)
 20 {
 19     sigset_t mask, prev_mask;
 18     sigemptyset(&mask);
 17
 16     while (fgpid(jobs) == pid)
 15         sigsuspend(&mask);
 14
 13     return;
 12 }
 11
```

---

### sigchld_handler
Functionality: Catches SIGCHILD signals
#### Design Choices
- Why are all signals blocked before entering the waiting loop?
    - Race conditions between delete job and changing the job state would occur if not for the signal blocks. If the shell process is in the middle of the sigchld_handler, and knows this process is going to be reaped, it does not want to give the potential chance for another signal to be received -> signal handler to be entered again, and try to modify the job being reaped.
```c
void sigchld_handler(int sig)
{
    int status;
    pid_t pid;
    sigset_t mask, prev_mask;

    /* Block all signals while reaping/altering job list */
    sigfillset(&mask);

    while ((pid = waitpid(-1, &status, WNOHANG | WUNTRACED)) > 0) {
        sigprocmask(SIG_BLOCK, &mask, &prev_mask);

        if (WIFEXITED(status)) {
            /* Normal exit: remove job */
            deletejob(jobs, pid);
        } else if (WIFSIGNALED(status)) {
            /* Terminated by signal: report and remove job */
            printf("Job [%d] (%d) terminated by signal %d\n", pid2jid(pid), pid, WTERMSIG(status));
            deletejob(jobs, pid);
        } else if (WIFSTOPPED(status)) {
            /* Stopped by signal: mark as stopped */
            struct job_t *job = getjobpid(jobs, pid);
            if (job != NULL) {
                job->state = ST;
                printf("Job [%d] (%d) stopped by signal %d\n", pid2jid(pid), pid, WSTOPSIG(status));
            }
        }

        sigprocmask(SIG_SETMASK, &prev_mask, NULL);
    }

    return;
}
```

---

### sigtstp_handler
Functionality: Catches SIGTSTP signals
#### Design Choices
- Why don't we need any signal blockers?
    - Although the function does look through the shared jobs, the only race condition would be the foreground process is reaped, and the kill() would still run, but sets errno to tell the system the process does not exist, and fails gracefully. Therefore, this race condition is harmless.
- Why do is SIGTSTP sent to the whole foreground process group instead of just the foreground process?
    - The foreground process group must receive the signal because the parent process was sent the signal, and the child processes should follow suit.
```c
void sigtstp_handler(int sig)
{
    pid_t fg_pid = fgpid(jobs);
    
    if (fg_pid > 0)
        kill(-fg_pid, SIGTSTP);

    return;
}
```

---

### sigint_handler
Functionality: Catches SIGINT(ctrl-c) signals
#### Design Choices
- Why don't we need any signal blockers?
    - Although the function does look through the shared jobs, the only race condition would be the foreground process is reaped, and the kill() would still run, but sets errno to tell the system the process does not exist, and fails gracefully. Therefore, this race condition is harmless.
- Why do is SIGINT sent to the whole foreground process group instead of just the foreground process?
    - The foreground process group must receive the signal because the parent process was sent the signal, and the child processes should follow suit.
```c
void sigint_handler(int sig)
{
    pid_t fg_pid = fgpid(jobs);

    if (fg_pid != 0)
        kill(-fg_pid, SIGINT);
    
    return;
}
```

---

### sigquit_handler
Functionality: Catches SIGQUIT(ctrl-z) signals
```c
void sigquit_handler(int sig)
{
    printf("Terminating after receipt of SIGQUIT signal\n");
    exit(1);
}
```

---

## Some Natural Pitfalls
- Blocking everything
- Not understanding the flow of kernel trap from syscall -> kernel sends signal to shell -> shell forwards process

---

## Why not just block everything?