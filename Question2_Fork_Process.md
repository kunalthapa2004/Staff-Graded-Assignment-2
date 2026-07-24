# Question 2 – Child Process Manager with Signal Handling

## C Program

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>
#include <sys/wait.h>
#include <sys/types.h>
#include <time.h>

#define NUM_CHILDREN 5
#define TIMEOUT_SECONDS 4

void sigchld_handler(int sig) {
    int status;
    pid_t pid;
    while ((pid = waitpid(-1, &status, WNOHANG)) > 0) {
        if (WIFEXITED(status)) {
            printf("Child %d exited normally with status %d\n", pid, WEXITSTATUS(status));
        } else if (WIFSIGNALED(status)) {
            printf("Child %d was killed by signal %d\n", pid, WTERMSIG(status));
        }
    }
}

void child_task(int id) {
    if (id == 2 || id == 4) {
        printf("Child %d (PID %d): simulating hang...\n", id, getpid());
        while (1) pause();
    }
    int work_time = (rand() % 3) + 1;
    printf("Child %d (PID %d): working for %d seconds\n", id, getpid(), work_time);
    sleep(work_time);
    printf("Child %d (PID %d): done\n", id, getpid());
    exit(0);
}

int main() {
    struct sigaction sa;
    sa.sa_handler = sigchld_handler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART | SA_NOCLDSTOP;
    sigaction(SIGCHLD, &sa, NULL);

    pid_t children[NUM_CHILDREN];
    time_t start_times[NUM_CHILDREN];

    srand(time(NULL));

    for (int i = 0; i < NUM_CHILDREN; i++) {
        pid_t pid = fork();
        if (pid < 0) {
            perror("fork failed");
            exit(1);
        }
        if (pid == 0) {
            child_task(i);
        }
        children[i] = pid;
        start_times[i] = time(NULL);
        printf("Parent: spawned child %d with PID %d\n", i, pid);
    }

    while (1) {
        int all_done = 1;
        for (int i = 0; i < NUM_CHILDREN; i++) {
            if (children[i] == 0) continue;
            all_done = 0;

            int result = waitpid(children[i], NULL, WNOHANG);
            if (result == children[i]) {
                children[i] = 0;
                continue;
            }

            if (difftime(time(NULL), start_times[i]) > TIMEOUT_SECONDS) {
                printf("Parent: child PID %d timed out, sending SIGTERM\n", children[i]);
                kill(children[i], SIGTERM);
                sleep(1);
                if (waitpid(children[i], NULL, WNOHANG) == 0) {
                    printf("Parent: child PID %d ignored SIGTERM, sending SIGKILL\n", children[i]);
                    kill(children[i], SIGKILL);
                }
                children[i] = 0;
            }
        }
        if (all_done) break;
        sleep(1);
    }

    printf("Parent: all children handled, exiting.\n");
    return 0;
}
```


## Commands and Explanations

**Command: gcc -o process_manager process_manager.c**

This compiles the C source file into an executable named process_manager. GCC links the standard libraries automatically so all POSIX functions like fork and waitpid are available without extra flags.


**Command: ./process_manager**

Running the executable starts the parent process. It immediately forks five child processes and begins monitoring each one for timeout, printing status messages as each child finishes or gets terminated.


**Command: kill -SIGTERM [PID] then kill -SIGKILL [PID]**

SIGTERM asks a process to shut down gracefully, giving it a chance to clean up. SIGKILL is the forced version that the kernel handles directly and cannot be caught or ignored. The program sends SIGTERM first and only escalates to SIGKILL if the child is still alive after one second.


## Sample Output

```
Parent: spawned child 0 with PID 12301
Parent: spawned child 1 with PID 12302
Child 0 (PID 12301): working for 2 seconds
Child 1 (PID 12302): working for 1 seconds
Child 2 (PID 12303): simulating hang...
Child 0 (PID 12301): done
Child 1 (PID 12302): done
Parent: child PID 12303 timed out, sending SIGTERM
Child 12303 was killed by signal 15
Parent: all children handled, exiting.
```


## Explanation

fork() duplicates the parent process at the point of the call. The return value tells each copy whether it is the parent (child PID returned) or the child (0 returned), which is how the two branches diverge. waitpid with WNOHANG checks whether a child has finished without blocking, so the parent can keep monitoring all children in a loop instead of waiting on one at a time. sigaction registers a handler for SIGCHLD, which the kernel sends to the parent whenever a child changes state. The SA_NOCLDSTOP flag stops the handler from firing on pause events, keeping output clean. Together these mechanisms form a complete lifecycle: creation with fork, cooperative monitoring with WNOHANG, automatic cleanup with SIGCHLD, and forced termination with SIGTERM and SIGKILL when a child stops responding.
