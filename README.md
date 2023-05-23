# Linux-Kernel
A summarization of Linux Kernel Development by Robert Love. Leave a ⭐ if you found this helpful!


<!-- Output copied to clipboard! -->

<!-- You have some errors, warnings, or alerts. If you are using reckless mode, turn it off to see inline alerts.
* ERRORs: 0
* WARNINGs: 0
* ALERTS: 8 -->


    EVERYTHING YOU'LL NEED TO KNOW ABOUT THE LINUX KERNEL



* When an application executes a system call, we say that the kernel is executing on behalf of the application. Furthermore, the application is said to be executing a system call in kernel-space, and the kernel is running in process context. This relationship— that applications call into the kernel via the system call interface—is the fundamental manner in which applications get work done. 
* When hardware wants to communicate with the system, it issues an interrupt that literally interrupts the processor, which in turn interrupts the kernel.A number identifies interrupts and the kernel uses this number to execute a specific interrupt handler to process and respond to the interrupt.
* The kernel notes the interrupt number of the incoming interrupt and executes the correct interrupt handler.The interrupt handler processes the keyboard data and lets the keyboard controller know it is ready for more data
* In many operating systems, including Linux, the interrupt handlers do not run in a process context. Instead, they run in a special interrupt context that is not associated with any process.
* Microkernels, on the other hand, are not implemented as a single large process. Instead, the functionality of the kernel is broken down into separate processes, usually called servers.
* microkernels communicate via message passing: An interprocess communication (IPC) mechanism is built into the system, and the various servers communicate with and invoke “services” from each other by sending messages over the IPC mechanism. The separation of the various servers prevents a failure in one server from bringing down another.
* Consequently, all practical microkernel-based systems now place most or all the servers in kernel-space, to remove the overhead of frequent context switches and potentially enable direct function invocation.
* Linux is a monolithic kernel
* As an example, on an x86 system using grub, you would copy arch/i386/boot/bzImage to /boot, name it something like vmlinuzversion , and edit /boot/grub/grub.conf, adding a new entry for the new kernel. Systems using LILO to boot would instead edit /etc/lilo.conf and then rerun lilo.
* The kernel does not have access to printf(), but it does provide printk(), which works pretty much the same as its more familiar cousin.The printk() function copies the formatted string into the kernel log buffer, which is normally read by the syslog program. Usage is similar to printf():
* A process is a program (object code stored on some media) in the midst of execution.
* .They also include a set of resources such as open files and pending signals, internal kernel data, processor state, a memory address space with one or more memory mappings, one or more threads of execution, and a data section containing global variables.
* Threads of execution, often shortened to threads, are the objects of activity within the process. Each thread includes a unique program counter, process stack, and set of processor registers.The kernel schedules individual threads, not processes.
* To Linux, a thread is just a special kind of process.
* On modern operating systems, processes provide two virtualizations: a virtualized processor and virtual memory.The virtual processor gives the process the illusion that it alone monopolizes the system, despite possibly sharing the processor among hundreds of other processes. Chapter 4,“Process Scheduling,” discusses this virtualization.Virtual memory lets the process allocate and manage memory as if it alone owned all the memory in the system.
* a process is an active program and related resources.
* A process begins its life when, not surprisingly, it is created. In Linux, this occurs by means of the fork() system call, which creates a new process by duplicating an existing one.The process that calls fork() is the parent, whereas the new process is the child.The parent resumes execution and the child starts execution at the same place: where the call to fork() returns.The fork() system call returns from the kernel twice: once in the parent process and again in the newborn child. Often, immediately after a fork it is desirable to execute a new, different program.The exec() family of function calls creates a new address space and loads a new program into it. In contemporary Linux kernels, fork() is actually implemented via the clone() system call, which is discussed in a following section. Finally, a program exits via the exit() system call.This function terminates the process and frees all its resources.A parent process can inquire about the status of a terminated child via the wait4()1 system call, which enables a process to wait for the termination of a specific process.When a process exits, it is placed into a special zombie state that represents terminated processes until the parent calls wait() or waitpid().
* The kernel stores the list of processes in a circular doubly linked list called the task list.2 Each element in the task list is a process descriptor of the type struct task_struct, which is defined in <linux/sched.h>.The process descriptor contains all the information about a specific process.
* The task_struct structure is allocated via the slab allocator
* calculate the location of the process descriptor via the stack pointer
* With the process descriptor now dynamically created via the slab allocator, a new structure, struct thread_info, was created that again lives at the bottom of the stack (for stacks that grow down) and at the top of the stack (for stacks that grow up).3 See Figure 3.2. The thread_info structure is defined on x86 in <asm/thread_info.h> 

    The thread_info structure is defined on x86 in <asm/thread_info.h> as


    struct thread_info {


     struct task_struct    	     *task;


     struct exec_domain         *exec_domain;


     __u32           		     flags;


     __u32                 	     status;


     __u32                 	     cpu;


     int                   		     preempt_count;


    mm_segment_t                addr_limit;


    struct restart_block          restart_block; 


    void                  		    *sysenter_return; 


    int                 		     uaccess_err;


    };


     

* The system identifies processes by a unique process identification value or PID
* The PID is a numerical value represented by the opaque type4 pid_t, which is typically an int. Because of backward compatibility with earlier Unix and Linux versions, however, the default maximum value is only 32,768 (that of a short int), although the value optionally can be increased as high as four million (this is controlled in <linux/threads.h>.The kernel stores this value as pid inside each process descriptor.
* Although 32,768 might be sufficient for a desktop system, large servers may require many more processes
* If the system is willing to break compatibility with old applications, the administrator may increase the maximum value via /proc/sys/kernel/pid_max. Inside the kernel, tasks are typically referenced directly by a pointer to their task_struct structure. In fact, most kernel code that deals with processes works directly with struct task_struct. Consequently, it is useful to be able to quickly look up the process descriptor of the currently executing task, which is done via the current macro. This macro must be independently implemented by each architecture. Some architectures save a pointer to the task_struct structure of the currently running process in a register, enabling for efficient access. Other architectures, such as x86 (which has few registers to waste), make use of the fact that struct thread_info is stored on the kernel stack to calculate the location of thread_info and subsequently the task_struct. On x86, current is calculated by masking out the 13 least-significant bits of the stack pointer to obtain the thread_info structure.This is done by the current_thread_info() function
* Finally, current dereferences the task member of thread_info to return the task_struct:
* current_thread_info()->task;
* The state field of the process descriptor describes the current condition of the process
* n TASK_RUNNING—The process is runnable; it is either currently running or on a runqueue waiting to run (runqueues are discussed in Chapter 4).This is the only possible state for a process executing in user-space; it can also apply to a process in kernel-space that is actively running. n TASK_INTERRUPTIBLE—The process is sleeping (that is, it is blocked), waiting for some condition to exist.When this condition exists, the kernel sets the process’s state to TASK_RUNNING.The process also awakes prematurely and becomes runnable if it receives a signal.


![alt_text](images/image1.png "image_tooltip")




* TASK_UNINTERRUPTIBLE—This state is identical to TASK_INTERRUPTIBLE except that it does not wake up and become runnable if it receives a signal
* __TASK_TRACED—The process is being traced by another process, such as a debugger, via ptrace. __TASK_STOPPED—Process execution has stopped; the task is not running nor is it eligible to run.This occurs if the task receives the SIGSTOP, SIGTSTP, SIGTTIN,or SIGTTOU signal or if it receives any signal while it is being debugged.
* Kernel code often needs to change a process’s state.The preferred mechanism is using
* set_task_state(task, state);
* The method set_current_state(state) is synonymous to set_task_state(current, state). See <linux/sched.h> for the implementation of these and related functions.
* Normal program execution occurs in user-space.When a program executes a system call (see Chapter 5,“System Calls”) or triggers an exception, it enters kernel-space.At this point, the kernel is said to be “executing on behalf of the process” and is in process context.When in process context, the current macro is valid.
* el.A process can begin executing in kernel-space only through one of these interfaces—all access to the kernel is through these interfaces
* All processes are descendants of the init process, whose PID is one.The kernel starts init in the last step of the boot process.The init process, in turn, reads the system initscripts and executes more programs, eventually completing the boot process.
* Each task_struct has a pointer to the parent’s task_struct, named parent, and a list of children, named children
* you can follow the process hierarchy from any one process in the system to any other.
* This is easy because the task list is a circular, doubly linked list
* Process creation in Unix is unique. Most operating systems implement a spawn mechanism to create a new process in a new address space, read in an executable, and begin executing it.
* The first, fork(), creates a child process that is a copy of the current task. It differs from the parent only in its PID (which is unique), its PPID (parent’s PID, which is set to the original process), and certain resources and statistics, such as pending signals, which are not inherited.The second function, exec(), loads a new executable into the address space and begins executing it.The combination of fork()followed by exec()is similar to the single function most operating systems provide.
* Traditionally, upon fork(), all resources owned by the parent are duplicated and the copy is given to the child.This approach is naive and inefficient in that it copies much data that might otherwise be shared
* In Linux, fork() is implemented through the use of copy-on-write pages. Copy-on-write (or COW) is a technique to delay or altogether prevent copying of the data. Rather than duplicate the process address space, the parent and the child can share a single copy. The data, however, is marked in such a way that if it is written to, a duplicate is made and each process receives a unique copy.
* if exec() is called immediately after fork()—they never need to be copied. The only overhead incurred by fork() is the duplication of the parent’s page tables and the creation of a unique process descriptor for the child
* Linux implements fork() via the clone() system call. 
* .The clone() system call, in turn, calls do_fork(). The bulk of the work in forking is handled by do_fork(), which is defined in kernel/fork.c.This function calls copy_process() and then starts the process running. The interesting work is done by copy_process(): 
* It calls dup_task_struct(), which creates a new kernel stack, thread_info structure, and task_struct for the new process.The new values are identical to those of the current task.At this point, the child and parent process descriptors are identical. 
* It then checks that the new child will not exceed the resource limits on the number of processes for the current user. 
* The child needs to differentiate itself from its parent.Various members of the process descriptor are cleared or set to initial values. Members of the process descriptor not inherited are primarily statistically information.The bulk of the values in task_struct remain unchanged. 
* The child’s state is set to TASK_UNINTERRUPTIBLE to ensure that it does not yet run. 
* copy_process() calls copy_flags() to update the flags member of the task_struct.The PF_SUPERPRIV flag, which denotes whether a task used superuser privileges, is cleared.The PF_FORKNOEXEC flag, which denotes a process that has not called exec(), is set. 
* It calls alloc_pid() to assign an available PID to the new task. 
* Depending on the flags passed to clone(), copy_process() either duplicates or shares open files, filesystem information, signal handlers, process address space, and namespace.These resources are typically shared between threads in a given process; otherwise they are unique and thus copied here. 
* Finally, copy_process() cleans up and returns to the caller a pointer to the new child.
* Back in do_fork(), if copy_process() returns successfully, the new child is woken up and run. Deliberately, the kernel runs the child process first.8 In the common case of the child simply calling exec() immediately, this eliminates any copy-on-write overhead that would occur if the parent ran first and began writing to the address space.
* The vfork()system call has the same effect as fork(), except that the page table entries of the parent process are not copied. Instead, the child executes as the sole thread in the parent’s address space, and the parent is blocked until the child either calls exec() or exits.
* Because the semantics of vfork() are tricky (what, for example, happens if the exec() fails?),
* The vfork() system call is implemented via a special flag to the clone() system call: 1. In copy_process(), the task_struct member vfork_done is set to NULL. 2. In do_fork(), if the special flag was given, vfork_done is pointed at a specific address. 3. After the child is first run, the parent—instead of returning—waits for the child to signal it through the vfork_done pointer. 4. In the mm_release() function, which is used when a task exits a memory address space, vfork_done is checked to see whether it is NULL. If it is not, the parent is signaled. 5. Back in do_fork(), the parent wakes up and returns.
* Linux implements all threads as standard processes
* Instead, a thread is merely a process that shares certain resources with other processes. Each thread has a unique task_struct and appears to the kernel as a normal process— threads just happen to share resources, such as an address space, with other processes.
* Linux, threads are simply a manner of sharing resources between processes (which are already quite lightweight)
* Threads are created the same as normal tasks, with the exception that the clone() system call is passed flags corresponding to the specific resources to be shared
* The previous code results in behavior identical to a normal fork(), except that the address space, filesystem resources, file descriptors, and signal handlers are shared. In other words, the new task and its parent are what are popularly called threads.

    
![alt_text](images/image2.png "image_tooltip")


* It is often useful for the kernel to perform some operations in the background.The kernel accomplishes this via kernel threads—standard processes that exist solely in kernelspace.The significant difference between kernel threads and normal processes is that kernel threads do not have an address space. (Their mm pointer, which points at their address space, is NULL.) They operate only in kernel-space and do not context switch into user-space. Kernel threads, however, are schedulable and preemptable, the same as normal processes. Linux delegates several tasks to kernel threads, most notably the flush tasks and the ksoftirqd task.You can see the kernel threads on your Linux system by running the command ps -ef.There are a lot of them! Kernel threads are created on system boot by other kernel threads. Indeed, a kernel thread can be created only by another kernel thread.The kernel handles this automatically by forking all new kernel threads off of the kthreaded kernel process.
* When started, a kernel thread continues to exist until it calls do_exit() or another part of the kernel calls kthread_stop()
* It is sad, but eventually processes must die.When a process terminates, the kernel releases the resources owned by the process and notifies the child’s parent of its demise. Generally, process destruction is self-induced. It occurs when the process calls the exit() system call
* process receives a signal or exception it cannot handle or ignore. Regardless of how a process terminates, the bulk of the work is handled by do_exit(), defined in kernel/exit.c, which completes a number of chores: 1. It sets the PF_EXITING flag in the flags member of the task_struct. 2. It calls del_timer_sync() to remove any kernel timers. Upon return, it is guaranteed that no timer is queued and that no timer handler is running. 3. If BSD process accounting is enabled, do_exit() calls acct_update_integrals() to write out accounting information. 4. It calls exit_mm() to release the mm_struct held by this process. If no other process is using this address space—that it, if the address space is not shared—the kernel then destroys it. 5. It calls exit_sem(). If the process is queued waiting for an IPC semaphore, it is dequeued here. 6. It then calls exit_files() and exit_fs() to decrement the usage count of objects related to file descriptors and filesystem data, respectively. If either usage counts reach zero, the object is no longer in use by any process, and it is destroyed. 7. It sets the task’s exit code, stored in the exit_code member of the task_struct,to the code provided by exit() or whatever kernel mechanism forced the termination.The exit code is stored here for optional retrieval by the parent. 8. It calls exit_notify() to send signals to the task’s parent, reparents any of the task’s children to another thread in their thread group or the init process, and sets the task’s exit state, stored in exit_state in the task_struct structure, to EXIT_ZOMBIE. 9. do_exit() calls schedule() to switch to a new process (see Chapter 4). Because the process is now not schedulable, this is the last code the task will ever execute. do_exit() never returns.
* .After the parent has obtained information on its terminated child, or signified to the kernel that it does not care, the child’s task_struct is deallocated. The wait() family of functions are implemented via a single (and complicated) system call, wait4().The standard behavior is to suspend execution of the calling task until one of its children exits, at which time the function returns with the PID of the exited child. Additionally, a pointer is provided to the function that on return holds the exit code of the terminated child. When it is time to finally deallocate the process descriptor, release_task() is invoked. It does the following: 1. It calls __exit_signal(), which calls __unhash_process(), which in turns calls detach_pid() to remove the process from the pidhash and remove the process from the task list. 2. __exit_signal() releases any remaining resources used by the now dead process and finalizes statistics and bookkeeping. 3. If the task was the last member of a thread group, and the leader is a zombie, then release_task() notifies the zombie leader’s parent. 4. release_task() calls put_task_struct() to free the pages containing the process’s kernel stack and thread_info structure and deallocate the slab cache containing the task_struct.
* The Dilemma of the Parentless Task If a parent exits before its children, some mechanism must exist to reparent any child tasks to a new process, or else parentless terminated processes would forever remain zombies, wasting system memory.The solution is to reparent a task’s children on exit to either another process in the current thread group or, if that fails, the init process. do_exit() calls exit_notify(), which calls forget_original_parent(), which, in turn, calls find_new_reaper() to perform the reparenting
* This code attempts to find and return another task in the process’s thread group. If another task is not in the thread group, it finds and returns the init process. Now that a suitable new parent for the children is found, each child needs to be located and reparented to reaper:
* With the process successfully reparented, there is no risk of stray zombie processes.The init process routinely calls wait() on its children, cleaning up any zombies assigned to it.
* To best utilize processor time, assuming there are runnable processes, a process should always be running. If there are more runnable processes than processors in a system
* In preemptive multitasking, the scheduler decides when a process is to cease running and a new process is to begin running
* The time a process runs before it is preempted is usually predetermined, and it is called the timeslice of the process. The timeslice, in effect, gives each runnable process a slice of the processor’s time
* the timeslice is dynamically calculated as a function of process behavior and configurable system policy
* Conversely, in cooperative multitasking, a process does not stop running until it voluntary decides to do so.The act of a process voluntarily suspending itself is called yielding.
* The scheduler cannot make global decisions regarding how long processes run; processes can monopolize the processor for longer than the user desires; and a hung process that never yields can potentially bring down the entire system
* By introducing a constant-time algorithm for timeslice calculation and per-processor runqueues, it rectified the design limitations of the earlier scheduler.
* O(1) scheduler had several pathological failures related to scheduling latency-sensitive applications.These applications, called interactive processes, include any application with which the user interacts.Thus, although the O(1) scheduler was ideal for large server workloads—which lack interactive processes—it performed below par on desktop systems, where interactive applications are the raison d’être
* Policy is the behavior of the scheduler that determines what runs when
* Processes can be classified as either I/O-bound or processor-bound.The former is characterized as a process that spends much of its time submitting and waiting on I/O requests. Consequently, such a process is runnable for only short durations, because it eventually blocks waiting on more I/O. (Here, by I/O, we mean any type of blockable resource, such as keyboard input or network I/O, and not just disk I/O.) Most graphical user interface (GUI) applications, for example, are I/O-bound, even if they never read from or write to the disk, because they spend most of their time waiting on user interaction via the keyboard and mouse. Conversely, processor-bound processes spend much of their time executing code.They tend to run until they are preempted because they do not block on I/O requests very often. Because they are not I/O-driven, however, system response does not dictate that the scheduler run them often.A scheduler policy for processor-bound processes, therefore, tends to run such processes less frequently but for longer durations.The ultimate example of a processor-bound process is one executing an infinite loop. More palatable examples include programs that perform a lot of mathematical calculations, such as sshkeygen or MATLAB. Of course, these classifications are not mutually exclusive. Processes can exhibit both behaviors simultaneously:The X Window server, for example, is both processor and I/Ointense. Other processes can be I/O-bound but dive into periods of intense processor action.A good example of this is a word processor, which normally sits waiting for key presses but at any moment might peg the processor in a rabid fit of spell checking or macro calculation. The scheduling policy in a system must attempt to satisfy two conflicting goals: fast process response time (low latency) and maximal system utilization (high throughput).
* The scheduler policy in Unix systems tends to explicitly favor I/O-bound processes, thus providing good process response time. Linux, aiming to provide good interactive response and desktop performance, optimizes for process response (low latency), thus favoring I/O-bound processes over processor-bound processors
* A common type of scheduling algorithm is priority-based scheduling.The goal is to rank processes based on their worth and need for processor time.The general idea, which isn’t exactly implemented on Linux, is that processes with a higher priority run before those with a lower priority, whereas processes with the same priority are scheduled round-robin (one after the next, repeating). On some systems, processes with a higher priority also receive a longer timeslice.The runnable process with timeslice remaining and the highest priority always runs. Both the user and the system can set a process’s priority to influence the scheduling behavior of the system. The Linux kernel implements two separate priority ranges.The first is the nice value, a number from –20 to +19 with a default of 0. Larger nice values correspond to a lower priority—you are being “nice” to the other processes on the system. Processes with a lower nice value (higher priority) receive a larger proportion of the system’s processor compared to processes with a higher nice value (lower priority).
* Mac OS X, the nice value is a control over the absolute timeslice allotted to a process; in Linux, it is a control over the proportion of timeslice
* The second range is the real-time priority.The values are configurable, but by default range from 0 to 99, inclusive. Opposite from nice values, higher real-time priority values correspond to a greater priority.All real-time processes are at a higher priority than normal processes; that is, the real-time priority and nice value are in disjoint value spaces. Linux implements real-time priorities in accordance with the relevant Unix standards, specifically POSIX.1b.All modern Unix systems implement a similar scheme.You can see a list of the processes on your system and their respective real-time priority (under the column marked RTPRIO) with the command ps -eo state,uid,pid,ppid,rtprio,time,comm. A value of “-” means the process is not real-time.
* the proportion of processor time that any process receives is determined only by the relative difference in niceness between it and the other runnable processes. The nice values, instead of yielding additive increases to timeslices, yield geometric differences.The absolute timeslice allotted any nice value is not an absolute number, but a given proportion of the processor
* we discuss four components of CFS:
* Time Accounting 
* Process Selection
* The Scheduler Entry Point
* Sleeping and Waking Up
* CFS uses the scheduler entity structure, struct sched_entity, defined in <linux/sched.h>, to keep track of process accounting:
* The scheduler entity structure is embedded in the process descriptor, struct task_stuct, as a member variable named se.
* The vruntime variable stores the virtual runtime of a process, which is the actual runtime (the amount of time spent running) normalized (or weighted) by the number of runnable processes.The virtual runtime’s units are nanoseconds and therefore vruntime is decoupled from the timer tick.The virtual runtime is used to help us approximate the “ideal multitasking processor” that CFS is modeling
* vruntime to account for how long a process has run and thus how much longer it ought to run. The function update_curr(), defined in kernel/sched_fair.c, manages this accounting
* update_curr() calculates the execution time of the current process and stores that value in delta_exec. It then passes that runtime to __update_curr(), which weights the time by the number of runnable processes.The current process’s vruntime is then incremented by the weighted value
* In this manner, vruntime is an accurate measure of the runtime of a given process and an indicator of what process should run next.
* CFS attempts to balance a process’s virtual runtime with a simple rule:When CFS is deciding what process to run next, it picks the process with the smallest vruntime
* CFS uses a red-black tree to manage the list of runnable processes and efficiently find the process with the smallest vruntime.A red-black tree, called an rbtree in Linux, is a type of self-balancing binary search tree 
* red-black trees are a data structure that store nodes of arbitrary data, identified by a specific key, and that they enable efficient search for a given key. (Specifically, obtaining a node identified by a given key is logarithmic in time as a function of total nodes in the tree.) t red-black trees are a data structure that store nodes of arbitrary data, identified by a specific key, and that they enable efficient search for a given key. (Specifically, obtaining a node identified by a given key is logarithmic in time as a function of total nodes in the tree.)
* the process that CFS wants to run next, which is the process with the smallest vruntime, is the leftmost node in the tree.That is, if you follow the tree from the root down through the left child, and continue moving to the left until you reach a leaf node, you find the process with the smallest vruntime.
* “run the process represented by the leftmost node in the rbtree.”The function that performs this selection is __pick_next_entity(), defined in kernel/sched_fair.c
* Note that __pick_next_entity() does not actually traverse the tree to find the leftmost node, because the value is cached by rb_leftmost
* —it is even easier to cache the leftmost node.The return value from this function is the process that CFS next runs
* .Adding processes to the tree is performed by enqueue_entity():
* As with adding a process to the red-black tree, the real work is performed by a helper function, __dequeue_entity():
* The main entry point into the process schedule is the function schedule(), defined in kernel/sched.c
* .The only important part of the function—which is otherwise too uninteresting to reproduce here—is its invocation of pick_next_task(), also defined in kernel/sched.c.The pick_next_task() function goes through each scheduler class, starting with the highest priority, and selects the highest priority process in the highest priority class
* there is a small hack to quickly select the next CFS-provided process if the number of runnable processes is equal to the number of CFS runnable processes (which suggests that all runnable processes are provided by CFS). The core of the function is the for() loop, which iterates over each class in priority order, starting with the highest priority class. Each class implements the pick_next_task() function, which returns a pointer to its next runnable process or, if there is not one, NULL.The first class to return a non-NULL value has selected the next runnable process. CFS’s implementation of pick_next_task() calls pick_next_entity(), which in turn calls the __pick_next_entity()
* sleeping (blocked) are in a special nonrunnable state
* A task sleeps for a number of reasons, but always while it is waiting for some event.
* .A common reason to sleep is file I/O
* :The task marks itself as sleeping, puts itself on a wait queue, removes itself from the red-black tree of runnable, and calls schedule() to select a new process to execute.Waking back up is the inverse:The task is set as runnable, removed from the wait queue, and added back to the red-black tree.
* As discussed in the previous chapter, two states are associated with sleeping, TASK_INTERRUPTIBLE and TASK_UNINTERRUPTIBLE.They differ only in that tasks in the TASK_UNINTERRUPTIBLE state ignore signals, whereas tasks in the TASK_INTERRUPTIBLE state wake up prematurely and respond to a signal if one is issued. Both types of sleeping tasks sit on a wait queue, waiting for an event to occur, and are not runnable.
* Wait Queues Sleeping is handled via wait queues.A wait queue is a simple list of processes waiting for an event to occur.Wait queues are represented in the kernel by wake_queue_head_t.Wait queues are created statically via DECLARE_WAITQUEUE() or dynamically via init_waitqueue_head(). Processes put themselves on a wait queue and mark themselves not runnable.When the event associated with the wait queue occurs, the processes on the queue are awakened. It is important to implement sleeping and waking correctly, to avoid race conditions.
* The task performs the following steps to add itself to a wait queue:
* Creates a wait queue entry via the macro DEFINE_WAIT().
* Adds itself to a wait queue via add_wait_queue().This wait queue awakens the process when the condition for which it is waiting occurs. Of course, there needs to be code elsewhere that calls wake_up() on the queue when the event actually does occur.
* Calls prepare_to_wait() to change the process state to either TASK_INTERRUPTIBLE or TASK_UNINTERRUPTIBLE.This function also adds the task back to the wait queue if necessary, which is needed on subsequent iterations of the loop.
* If the state is set to TASK_INTERRUPTIBLE, a signal wakes the process up.This is called a spurious wake up (a wake-up not caused by the occurrence of the event). So check and handle signals.
* When the task awakens, it again checks whether the condition is true. If it is, it exits the loop. Otherwise, it again calls schedule() and repeats.
* Now that the condition is true, the task sets itself to TASK_RUNNING and removes itself from the wait queue via finish_wait().
* Waking Up 
* Waking is handled via wake_up(), which wakes up all the tasks waiting on the given wait queue. It calls try_to_wake_up(), which sets the task’s state to TASK_RUNNING, calls enqueue_task() to add the task to the red-black tree, and sets need_resched if the awakened task’s priority is higher than the priority of the current task.The code that causes the event to occur typically calls wake_up() itself. For example, when data arrives from the hard disk, the VFS calls wake_up() on the wait queue that holds the processes waiting for the data.

    
![alt_text](images/image3.png "image_tooltip")


* Context switching, the switching from one runnable task to another, is handled by the context_switch()function defined in kernel/sched.c. It is called by schedule() when a new process has been selected to run. It does two basic jobs: 
* Calls switch_mm(), which is declared in <asm/mmu_context.h>, to switch the virtual memory mapping from the previous process’s to that of the new process.
* Calls switch_to(), declared in <asm/system.h>, to switch the processor state from the previous process’s to the current’s.This involves saving and restoring stack information and the processor registers and any other architecture-specific state that must be managed and restored on a per-process basis. 
* The kernel, however, must know when to call schedule(). If it called schedule() only when code explicitly did so, user-space programs could run indefinitely. Instead, the kernel provides the need_resched flag to signify whether a reschedule should be performed (see Table 4.1).This flag is set by scheduler_tick() when a process should be preempted, and by try_to_wake_up() when a process that has a higher priority than the currently running process is awakened.The kernel checks the flag, sees that it is set, and calls schedule() to switch to a new process.The flag is a message to the kernel that the scheduler should be invoked as soon as possible because another process deserves to run.
* Upon returning to user-space or returning from an interrupt, the need_resched flag is checked. If it is set, the kernel invokes the scheduler before continuing. The flag is per-process, and not simply global, because it is faster to access a value in the process descriptor (because of the speed of current and high probability of it being cache hot)
* User preemption occurs when the kernel is about to return to user-space, need_resched is set, and therefore, the scheduler is invoked
* In short, user preemption can occur
* n When returning to user-space from a system call n When returning to user-space from an interrupt handler
* In nonpreemptive kernels, kernel code runs until completion
* In the 2.6 kernel, however, the Linux kernel became preemptive: It is now possible to preempt a task at any point, so long as the kernel is in a state in which it is safe to reschedule
* The kernel can preempt a task running in the kernel so long as it does not hold a lock
* When the counter is zero, the kernel is preemptible. Upon return from interrupt, if returning to kernel-space, the kernel checks the values of need_resched and preempt_count. If need_resched is set and preempt_count is zero, then a more important task is runnable, and it is safe to preempt.Thus, the scheduler is invoked. If preempt_count is nonzero, a lock is held, and it is unsafe to reschedule.
* At that time, the unlock code checks whether need_resched is set. If so, the scheduler is invoked
* Kernel preemption can occur 
* When an interrupt handler exits, before returning to kernel-space 
* When kernel code becomes preemptible again 
* If a task in the kernel explicitly calls schedule() 
* If a task in the kernel blocks (which results in a call to schedule())
* Linux provides two real-time scheduling policies, SCHED_FIFO and SCHED_RR.The normal, not real-time scheduling policy is SCHED_NORMAL.Via the scheduling classes framework, these real-time policies are managed not by the Completely Fair Scheduler, but by a special real-time scheduler, defined in kernel/sched_rt.c.The rest of this section discusses the real-time scheduling policies and algorithm. SCHED_FIFO implements a simple first-in, first-out scheduling algorithm without timeslices.A runnable SCHED_FIFO task is always scheduled over any SCHED_NORMAL tasks. When a SCHED_FIFO task becomes runnable, it continues to run until it blocks or explicitly yields the processor; it has no timeslice and can run indefinitely. Only a higher priority SCHED_FIFO or SCHED_RR task can preempt a SCHED_FIFO task.Two or more SCHED_FIFO tasks at the same priority run round-robin, but again only yielding the processor when they explicitly choose to do so. If a SCHED_FIFO task is runnable, all tasks at a lower priority cannot run until it becomes unrunnable. SCHED_RR is identical to SCHED_FIFO except that each process can run only until it exhausts a predetermined timeslice.That is, SCHED_RR is SCHED_FIFO with timeslices—it is a real-time, round-robin scheduling algorithm.When a SCHED_RR task exhausts its timeslice, any other real-time processes at its priority are scheduled round-robin.
* .The kernel does not calculate dynamic priority values for real-time tasks.This ensures that a real-time process at a given priority always preempts a process at a lower priority.
* Soft realtime refers to the notion that the kernel tries to schedule applications within timing deadlines, but the kernel does not promise to always achieve these goals. Conversely, hard real-time systems are guaranteed to meet any scheduling requirements within certain limits.
* the default real-time priority range is zero to 99.This priority space is shared with the nice values of SCHED_NORMAL tasks:They use the space from MAX_RT_PRIO to (MAX_RT_PRIO + 40). By default, this means the –20 to +19 nice range maps directly onto the priority space from 100 to 139.
* 
![alt_text](images/image4.png "image_tooltip")

* The sched_setscheduler() and sched_getscheduler() system calls set and get a given process’s scheduling policy and real-time priority, respectively.Their implementation, like most system calls, involves a lot of argument checking, setup, and cleanup.The important work, however, is merely to read or write the policy and rt_priority values in the process’s task_struct. The sched_setparam() and sched_getparam() system calls set and get a process’s real-time priority.These calls merely encode rt_priority in a special sched_param structure.
* Only root can provide a negative value, thereby lowering the nice value and increasing the priority.The nice() function calls the kernel’s set_user_nice() function, which sets the static_prio and prio values in the task’s task_struct as appropriate.
* The Linux scheduler enforces hard processor affinity
* This task must remain on this subset of the available processors no matter what.”This hard affinity is stored as a bitmask in the task’s task_struct as cpus_allowed
* By default, all bits are set and, therefore, a process is potentially runnable on any processor
* The kernel enforces hard affinity in a simple manner. First, when a process is initially created, it inherits its parent’s affinity mask. Because the parent is running on an allowed processor, the child thus runs on an allowed processor. Second, when a processor’s affinity is changed, the kernel uses the migration threads to push the task onto a legal processor. Finally, the load balancer pulls tasks to only an allowed processor.
* A large number of runnable processes, scalability concerns, trade-offs between latency and throughput, and the demands of various workloads make a one-size-fits-all algorithm hard to achieve.The Linux kernel’s new CFS process scheduler, however, comes close to appeasing all parties and providing an optimal solution for most use cases with good scalability through a novel, interesting approach.
* System calls provide a layer between the hardware and user-space processes.This layer serves three primary purposes. First, it provides an abstracted hardware interface for userspace.
* Finally, a single common layer between user-space and the rest of the system allows for the virtualized system provided to processes
* Conversely, the kernel is concerned only with the system calls; what library calls and applications make use of the system calls is not of the kernel’s concern.
* System calls also provide a return value of type long4 that signifies success or error—usually, although not always, a negative return value denotes an error.A return value of zero is usually (but again not always) a sign of success.The C library, when a system call returns an error, writes a special error code into the global errno variable.
* Let’s look at how system calls are defined. First, note the asmlinkage modifier on the function definition.This is a directive to tell the compiler to look only on the stack for this function’s arguments.This is a required modifier for all system calls. Second, the function returns a long. For compatibility between 32- and 64-bit systems, system calls defined to return an int in user-space return a long in the kernel.Third, note that the getpid() system call is defined as sys_getpid() in the kernel.This is the naming convention taken with all system calls in Linux: System call bar() is implemented in the kernel as function sys_bar().
* The kernel keeps a list of all registered system calls in the system call table, stored in sys_call_table.This table is architecture; on x86-64 it is defined in arch/i386/kernel/syscall_64.c.This table assigns each valid syscall to a unique syscall number.
* System calls in Linux are faster than in many other operating systems.This is partly because of Linux’s fast context switch times; entering and exiting the kernel is a streamlined and simple affair.The other factor is the simplicity of the system call handler and the individual system calls themselves.
* It is not possible for user-space applications to execute kernel code directly
* If applications could directly read and write to the kernel’s address space, system security and stability would be nonexistent
* The mechanism to signal the kernel is a software interrupt: Incur an exception, and the system will switch to kernel mode and execute the exception handler
* The system call handler is the aptly named function system_call().
* Denoting the Correct System Call
* Simply entering kernel-space alone is not sufficient because multiple system calls exist, all of which enter the kernel in the same manner.Thus, the system call number must be passed into the kernel. On x86, the syscall number is fed to the kernel via the eax register. Before causing the trap into the kernel, user-space sticks in eax the number corresponding to the desired system call.The system call handler then reads the value from eax.
* The system_call() function checks the validity of the given system call number by comparing it to NR_syscalls. If it is larger than or equal to NR_syscalls
* Because each element in the system call table is 64 bits (8 bytes), the kernel multiplies the given system call number by four to arrive at its location in the system call table
* 
![alt_text](images/image5.png "image_tooltip")

* Multiplexing syscalls (a single system call that does wildly different things depending on a flag argument) is discouraged in Linux
* The system call should have a clean and simple interface with the smallest number of arguments possible
* Many system calls provide a flag argument to address forward compatibility.
* but to enable new functionality and options without breaking backward compatibility or needing to add a new system call.
* Remember the Unix motto:“Provide mechanism, not policy.”
* e I/O syscalls must check whether the file descriptor is valid. Processrelated functions must check whether the provided PID is valid
* One of the most important checks is the validity of any pointers that the user provides
* Before following a pointer into user-space, the system must ensure that n The pointer points to a region of memory in user-space. Processes must not be able to trick the kernel into reading data in kernel-space on their behalf. n The pointer points to a region of memory in the process’s address space.The process must not be able to trick the kernel into reading someone else’s data. n If reading, the memory is marked readable. If writing, the memory is marked writable. If executing, the memory is marked executable.The process must not be able to bypass memory access restrictions.
* For writing into user-space, the method copy_to_user()
* For reading from user-space, the method copy_from_user() is analogous to copy_to_user().The function reads from the second parameter into the first parameter the number of bytes specified in the third parameter. 
* A call to capable() with a valid capabilities flag returns nonzero if the caller holds the specified capability and zero otherwise. For example, capable(CAP_SYS_NICE) checks whether the caller has the ability to modify nice values of other processes. By default, the superuser possesses all capabilities and nonroot possesses none. For example, here is the reboot() system call. Note how its first step is ensuring that the calling process has the CAP_SYS_REBOOT.
* The kernel is in process context during the execution of a system call.The current pointer points to the current task, which is the process that issued the syscall. In process context, the kernel is capable of sleeping (for example, if the system call blocks on a call or explicitly calls schedule()) and is fully preemptible.These two points are important. First, the capability to sleep means that system calls can make use of the majority of the kernel’s functionality
* When the system call returns, control continues in system_call(), which ultimately switches to user-space and continues the execution of the user process.
* After the system call is written, it is trivial to register it as an official system call: 1. Add an entry to the end of the system call table.This needs to be done for each architecture that supports the system call (which, for most calls, is all the architectures).The position of the syscall in the table, starting at zero, is its system call number. For example, the tenth entry in the list is assigned syscall number nine. 2. For each supported architecture, define the syscall number in <asm/unistd.h>. 3. Compile the syscall into the kernel image (as opposed to compiling as a module). This can be as simple as putting the system call in a relevant file in kernel/, such as sys.c, which is home to miscellaneous system calls.
* Look at these steps in more detail with a fictional system call, foo(). First, we want to add sys_foo() to the system call table.
* Finally, the actual foo() system call is implemented. Because the system call must be compiled into the core kernel image in all configurations, in this example we define it in kernel/sys.c.You should put it wherever the function is most relevant; for example, if the function is related to scheduling, you could define it in kernel/sched.c.
* Thankfully, Linux provides a set of macros for wrapping access to system calls. It sets up the register contents and issues the trap instructions.These macros are named _syscall n (), where n is between 0 and 6
* The pros of implementing a new interface as a syscall are as follows:
* System calls are simple to implement and easy to use.
* System call performance on Linux is fast. 
* The cons:
* You need a syscall number, which needs to be officially assigned to you. 
* After the system call is in a stable series kernel, it is written in stone.The interface cannot change without breaking user-space applications. 
* Each architecture needs to separately register the system call and support it. 
* System calls are not easily used from scripts and cannot be accessed directly from the filesystem.
* Because you need an assigned syscall number, it is hard to maintain and use a system call outside of the master kernel tree. 
* For simple exchanges of information, a system call is overkill. 
* The alternatives: 
* Implement a device node and read() and write() to it. Use ioctl() to manipulate specific settings or retrieve specific information.
* Memory allocation inside the kernel is not as easy as memory allocation outside the kernel
* the kernel cannot easily deal with memory allocation errors, and the kernel often cannot sleep
* The kernel treats physical pages as the basic unit of memory management.Although the processor’s smallest addressable unit is a byte or a word, the memory management unit (MMU, the hardware that manages memory and performs virtual to physical address translations) typically deals in pages.Therefore, the MMU maintains the system’s page tables with page-sized granularity (hence their name). In terms of virtual memory, pages are the smallest unit that matters.
* Most 32-bit architectures have 4KB pages, whereas most 64-bit architectures have 8KB pages
* The kernel represents every physical page on the system with a struct page structure. This structure is defined in <linux/mm_types.h>
* .The flags field stores the status of the page. Such flags include whether the page is dirty or whether it is locked in memory.
* .The flag values are defined in <linux/page-flags.h>.
* The _count field stores the usage count of the page—that is, how many references there are to this page.
* A page may be used by the page cache (in which case the mapping field points to the address_space object associated with this page)
* The virtual field is the page’s virtual address. Normally, this is simply the address of the page in virtual memory. Some memory (called high memory) is not permanently mapped in the kernel’s address space.
* page structure is associated with physical pages, not virtual pages.
* The kernel uses this data structure to describe the associated physical page.The data structure’s goal is to describe physical memory, not the data contained therein. 
* The kernel uses this structure to keep track of all the pages in the system,
* If a page is not free, the kernel needs to know who owns the page. Possible owners include user-space processes, dynamically allocated kernel data, static kernel code, the page cache, and so on. Developers are often surprised that an instance of this structure is allocated for each physical page in the system.They think,“What a lot of memory wasted!” Let’s look at just how bad (or good) the space consumption is from all these pages.Assume struct page consumes 40 bytes of memory, the system has 8KB physical pages, and the system has 4GB of physical memory. In that case, there are about 524,288 pages and page structures on the system.The page structures consume 20MB: perhaps a surprisingly large number in absolute terms, but only a small fraction of a percent relative to the system’s 4GB—not too high a cost for managing all the system’s physical pages.
* Because of hardware limitations, the kernel cannot treat all pages as identical. Some pages, because of their physical address in memory, cannot be used for certain tasks
* Some hardware devices can perform DMA (direct memory access) to only certain memory addresses. 
* Some architectures can physically addressing larger amounts of memory than they can virtually address. Consequently, some memory is not permanently mapped into the kernel address space.
* Because of these constraints, Linux has four primary memory zones:
* ZONE_DMA—This zone contains pages that can undergo DMA. 
* ZONE_DMA32—Like ZOME_DMA, this zone contains pages that can undergo DMA. Unlike ZONE_DMA, these pages are accessible only by 32-bit devices. On some architectures, this zone is a larger subset of memory. 
* ZONE_NORMAL—This zone contains normal, regularly mapped, pages. 
* ZONE_HIGHMEM—This zone contains “high memory,” which are pages not permanently mapped into the kernel’s address space.
* These zones, and two other, less notable ones, are defined in <linux/mmzone.h>. The actual use and layout of the memory zones is architecture-dependent. 
* As a counterexample, on the x86 architecture, ISA devices cannot perform DMA into the full 32-bit address space1 because ISA devices can access only the first 16MB of physical memory. Consequently, ZONE_DMA on x86 consists of all memory in the range 0MB–16MB.
* ZONE_HIGHMEM works in the same regard.What an architecture can and cannot directly map varies. On 32-bit x86 systems, ZONE_HIGHMEM is all memory above the physical 896MB mark. On other architectures, ZONE_HIGHMEM is empty because all memory is directly mapped.The memory contained in ZONE_HIGHMEM is called high memory.2 The rest of the system’s memory is called low memory. 
* ZONE_NORMAL tends to be whatever is left over after the previous two zones claim their requisite shares. On x86, for example, ZONE_NORMAL is all physical memory from 16MB to 896MB.
* Linux partitions the system’s pages into zones to have a pooling in place to satisfy allocations as needed. For example, having a ZONE_DMA pool gives the kernel the capability to satisfy memory allocations needed for DMA. If such memory is needed, the kernel can simply pull the required number of pages from ZONE_DMA. Note that the zones do not have any physical relevance but are simply logical groupings used by the kernel to keep track of pages. 
* Although some allocations may require pages from a particular zone, other allocations may pull from multiple zones. For example, although an allocation for DMA-able memory must originate from ZONE_DMA, a normal allocation can come from ZONE_DMA or ZONE_NORMAL but not both; allocations cannot cross zone boundaries 
* For example, a 64-bit architecture such as Intel’s x86-64 can fully map and handle 64-bits of memory.Thus, x86-64 has no ZONE_HIGHMEM and all physical memory is contained within ZONE_DMA and ZONE_NORMAL. Each zone is represented by struct zone, which is defined in <linux/mmzone.h>:
* The lock field is a spin lock that protects the structure from concurrent access. Note that it protects just the structure and not all the pages that reside in the zone
* The watermark array holds the minimum, low, and high watermarks for this zone.The kernel uses watermarks to set benchmarks for suitable per-zone memory consumption
* The name field is, unsurprisingly, a NULL-terminated string representing the name of this zone.The kernel initializes this value during boot in mm/page_alloc.c, and the three zones are given the names DMA, Normal, and HighMem.
* All these interfaces allocate memory with page-sized granularity and are declared in <linux/gfp.h>.
* This allocates 2order (that is, 1 << order) contiguous physical pages and returns a pointer to the first page’s page structure; on error it returns NULL.We look at the gfp_t type and gfp_mask parameter in a later section.You can convert a given page to its logical address with the function
* void * page_address(struct page *page)
* If you need only one page, two functions are implemented as wrappers to save you a bit of typing:
* struct page * alloc_page(gfp_t gfp_mask) unsigned long __get_free_page(gfp_t gfp_mask)
* This function works the same as __get_free_page(), except that the allocated page is then zero-filled—every bit of every byte is unset.
* All data must be zeroed or otherwise cleaned before it is returned to userspace to ensure system security is not compromised.
* A family of functions enables you to free allocated pages when you no longer need them:
* void __free_pages(struct page *page, unsigned int order) 
* void free_pages(unsigned long addr, unsigned int order)
* void free_page(unsigned long addr) 
* You must be careful to free only pages you allocate. Passing the wrong struct page or address, or the incorrect order, can result in corruption. Remember, the kernel trusts itself. Unlike with user-space, the kernel will happily hang itself if you ask it.
* The kmalloc() function’s operation is similar to that of user-space’s familiar malloc() routine, with the exception of the additional flags parameter.The kmalloc() function is a simple interface for obtaining kernel memory in byte-sized chunks. If you need whole pages, the previously discussed interfaces might be a better choice. For most kernel allocations, however, kmalloc() is the preferred interface. The function is declared in <linux/slab.h>:
* The function returns a pointer to a region of memory that is at least size bytes in length
* Flags are represented by the gfp_t type, which is defined in <linux/types.h> as an unsigned int.
* The flags are broken up into three categories: action modifiers, zone modifiers, and types.
* Zone modifiers specify from where to allocate memory
* Zone modifiers specify from which of these zones to allocate.Type flags specify a combination of action and zone modifiers as needed by a certain type of memory allocation.Type flags simplify the specification of multiple modifiers; instead of providing a combination of action and zone modifiers, you can specify just one type flag.The GFP_KERNEL is a type flag, which is used for code in process context inside the kernel
* 
![alt_text](images/image6.png "image_tooltip")

* This call instructs the page allocator (ultimately alloc_pages()) that the allocation can block, perform I/O, and perform filesystem operations, if needed
* Zone modifiers specify from which memory zone the allocation should originate.
* There are only three zone modifiers because there are only three zones other than ZONE_NORMAL
* .The __GFP_DMA flag forces the kernel to satisfy the request from ZONE_DMA.This flag says, I absolutely must have memory into which I can perform DMA. Conversely, the __GFP_HIGHMEM flag instructs the allocator to satisfy the request from either ZONE_NORMAL or (preferentially) ZONE_HIGHMEM
* You cannot specify __GFP_HIGHMEM to either __get_free_pages() or kmalloc(). Because these both return a logical address, and not a page structure.
* Only alloc_pages() can allocate high memory.
* The vast majority of allocations in the kernel use the GFP_KERNEL flag
* Conversely, the GFP_KERNEL allocation can put the caller to sleep to swap inactive pages to disk, flush dirty pages to disk, and so on.
* In the vast majority of the code that you write, you use either GFP_KERNEL or GFP_ATOMIC
* 
![alt_text](images/image7.png "image_tooltip")

* The counterpart to kmalloc() is kfree(), which is declared in <linux/slab.h>:
* The kfree() method frees a block of memory previously allocated with kmalloc(). Do not call this function on memory not previously allocated with kmalloc(), or on memory that has already been freed. Doing so is a bug, resulting in bad behavior such as freeing memory belonging to another part of the kernel
* In this example, an interrupt handler wants to allocate a buffer to hold incoming data.The preprocessor macro BUF_SIZE is the size in bytes of this desired buffer, which is presumably larger than just a couple of bytes.
* vmalloc()
* The vmalloc() function works in a similar fashion to kmalloc(), except it allocates memory that is only virtually contiguous and not necessarily physically contiguous.This is how a user-space allocation function works:The pages returned by malloc() are contiguous within the virtual address space of the processor, but there is no guarantee that they are actually contiguous in physical RAM.The kmalloc() function guarantees that the pages are physically contiguous (and virtually contiguous).The vmalloc() function ensures only that the pages are contiguous within the virtual address space. It does this by allocating potentially noncontiguous chunks of physical memory and “fixing up” the page tables to map the memory into a contiguous chunk of the logical address space. For the most part, only hardware devices require physically contiguous memory allocations. On many architectures, hardware devices live on the other side of the memory management unit and, thus, do not understand virtual addresses. Consequently, any regions of memory that hardware devices work with must exist as a physically contiguous block and not merely a virtually contiguous one. Blocks of memory used only by software—for example, process-related buffers—are fine using memory that is only virtually contiguous.
* pages obtained via vmalloc() must be mapped by their individual pages (because they are not physically contiguous), which results in much greater TLB4 thrashing than you see when directly mapped memory is used. Because of these concerns, vmalloc() is used only when absolutely necessary—typically, to obtain large regions of memory
* The function returns a pointer to at least size bytes of virtually contiguous memory.
* void vfree(const void *addr) This function frees the block of memory beginning at addr that was previously allocated via vmalloc().
* .To facilitate frequent allocations and deallocations of data, programmers often introduce free lists.A free list contains a block of available, already allocated, data structures. When code requires a new instance of a data structure, it can grab one of the structures off the free list rather than allocate the sufficient amount of memory and set it up for the data structure. Later, when the data structure is no longer needed, it is returned to the free list instead of deallocated. In this sense, the free list acts as an object cache, caching a frequently used type of object. 
* One of the main problems with free lists in the kernel is that there exists no global control.When available memory is low, there is no way for the kernel to communicate to every free list that it should shrink the sizes of its cache to free up memory.The kernel has no understanding of the random free lists at all.To remedy this, and to consolidate code, the Linux kernel provides the slab layer (also called the slab allocator).The slab layer acts as a generic data structure-caching layer
* The slab layer attempts to leverage several basic tenets: 
* Frequently used data structures tend to be allocated and freed often, so cache them. 
* Frequent allocation and deallocation can result in memory fragmentation (the inability to find large contiguous chunks of available memory).To prevent this, the cached free lists are arranged contiguously. Because freed data structures return to the free list, there is no resulting fragmentation.
* The free list provides improved performance during frequent allocation and deallocation because a freed object can be immediately returned to the next allocation.
* If the allocator is aware of concepts such as object size, page size, and total cache size, it can make more intelligent decisions. 
* If part of the cache is made per-processor (separate and unique to each processor on the system), allocations and frees can be performed without an SMP lock. 
* If the allocator is NUMA-aware, it can fulfill allocations from the same memory node as the requestor. 
* Stored objects can be colored to prevent multiple objects from mapping to the same cache lines.
    * The slab layer divides different objects into groups called caches, each of which stores a different type of object.There is one cache per object type. For example, one cache is for process descriptors (a free list of task_struct structures), whereas another cache is for inode objects (struct inode). Interestingly, the kmalloc() interface is built on top of the slab layer, using a family of general purpose caches. The caches are then divided into slabs (hence the name of this subsystem).The slabs are composed of one or more physically contiguous pages.Typically, slabs are composed of only a single page. Each cache may consist of multiple slabs
    * These structures are frequently created and destroyed, so it makes sense to manage them via the slab allocator. Thus, struct inode is allocated from the inode_cachep cache. (Such a naming convention is standard.) This cache is made up of one or more slabs—probably a lot of slabs because there are a lot of objects. Each slab contains as many struct inode objects as possible.When the kernel requests a new inode structure, the kernel returns a pointer to an already allocated, but unused structure from a partial slab or, if there is no partial slab, an empty slab.When the kernel is done using the inode object, the slab allocator marks the object as free
    * The descriptor is stored inside the slab if the total size of the slab is sufficiently small, or if internal slack space is sufficient to hold the descriptor. The slab allocator creates new slabs by interfacing with the low-level kernel page allocator via __get_free_pages():
    * This function uses __get_free_pages() to allocate memory sufficient to hold the cache.
    * Memory is then freed by kmem_freepages(), which calls free_pages()
    * The slab layer invokes the page allocation function only when there does not exist any partial or empty slabs in a given cache.The freeing function is called only when available memory grows low and the system is attempting to free memory, or when a cache is explicitly destroyed.
    * The first parameter is a string storing the name of the cache.The second parameter is the size of each element in the cache.The third parameter is the offset of the first object within a slab.This is done to ensure a particular alignment within the page. Normally, zero is sufficient, which results in the standard alignment.The flags parameter specifies optional settings controlling the cache’s behavior
* SLAB_HWCACHE_ALIGN
* SLAB_POISON
* SLAB_RED_
* SLAB_PANIC
* SLAB_CACHE_DMA
* ZONE_DMA
* To destroy a cache, call
* int kmem_cache_destroy(struct kmem_cache *cachep)
* .The caller of this function must ensure two conditions are true prior to invoking this function:
* After a cache is created, an object is obtained from the cache via
* void * kmem_cache_alloc(struct kmem_cache *cachep, gfp_t flags)
* After a task dies, if it has no children waiting on it, its process descriptor is freed and returned to the task_struct_cachep slab cache
* Because process descriptors are part of the core kernel and always needed, the task_struct_cachep cache is never destroyed
* If you frequently create many objects of the same type, consider using the slab cache
* In user-space, allocations such as some of the examples discussed thus far could have occurred on the stack because we knew the size of the allocation a priori. User-space is afforded the luxury of a large, dynamically growing stack, whereas the kernel has no such luxury—the kernel’s stack is small and fixed
* Early in the 2.6 kernel series, however, an option was introduced to move to single-page kernel stacks.When enabled, each process is given only a single page—4KB on 32-bit architectures and 8KB on 64-bit architectures.This was done for two reasons. First, it results in a page with less memory consumption per process. Second and most important is that as uptime increases, it becomes increasingly hard to find two physically contiguous unallocated pages. Physical memory becomes fragmented, and the resulting VM pressure from allocating a single new process is expensive.
* Now, each process’s entire call chain has to fit in its kernel stack.
* When single page stacks are enabled, interrupt handlers are given their own stacks. In any case, unbounded recursion and alloca() are obviously not allowed.
* In any given function, you must keep stack usage to a minimum.There is no hard and fast rule, but you should keep the sum of all local (that is, automatic) variables in a particular function to a maximum of a couple hundred bytes. Performing a large static allocation on the stack, such as of a large array or structure, is dangerous
* the kernel does not make any effort to manage the stack, when the stack overflows, the excess data simply spills into whatever exists at the tail end of the stack.The first thing to eat it is the thread_info structure
* At best, the machine will crash when the stack overflows.At worst, the overflow will silently corrupt data
* On the x86 architecture, all physical memory beyond the 896MB mark is high memory
* the kernel provides temporary mappings (which are also called atomic mappings).These are a set of reserved mappings that can hold a temporary mapping.The kernel can atomically map a high memory page into one of these reserved mappings.
* Setting up a temporary mapping is done via
    * void *kmap_atomic(struct page *page, enum km_type type)
* The mapping is undone via
    * void kunmap_atomic(void *kvaddr, enum km_type type)
* Modern SMP-capable operating systems use per-CPU data—data that is unique to a given processor—extensively.Typically, per-CPU data is stored in an array. Each item in the array corresponds to a possible processor on the system.The current processor number indexes this array, which is how the 2.4 kernel handles per-CPU data.
* Note that no lock is required because this data is unique to the current processor. If no processor touches this data except the current, no concurrency concerns exist, and the current processor can safely access the data without lock
* Kernel preemption is the only concern with per-CPU data. Kernel preemption poses two problems, listed here:
* If your code is preempted and reschedules on another processor, the cpu variable is no longer valid because it points to the wrong processor. (In general, code cannot sleep after obtaining the current processor.) 
* If another task preempts your code, it can concurrently access my_percpu on the same processor, which is a race condition.
* The 2.6 kernel introduced a new interface, known as percpu, for creating and manipulating per-CPU data.
* This new interface, however, grew out of the needs for a simpler and more powerful method for manipulating per-CPU data on large symmetrical multiprocessing computers.
* The alloc_percpu() macro allocates one instance of an object of the given type for every processor on the system
* The get_cpu_var()macro returns a pointer to the specific instance of the current processor’s data
* There are several benefits to using per-CPU data.The first is the reduction in locking requirements
* Per-CPU data greatly reduces cache invalidation.This occurs as processors try to keep their caches in sync. If one processor manipulates data held in another processor’s cache, that processor must flush or otherwise update its cache. Constant cache invalidation is called thrashing the cache and wreaks havoc on system performance.The use of per-CPU data keeps cache effects to a minimum because processors ideally access only their own data.The percpu interface cache-aligns all data to ensure that accessing one processor’s data does not bring in another processor’s data on the same cache line. Consequently, the use of per-CPU data often removes (or at least minimizes) the need for locking.The only safety requirement for the use of per-CPU data is disabling kernel preemption, which is much cheaper than locking, and the interface does so automatically. Per-CPU data can safely be used from either interrupt or process context
* No one is currently required to use the new per-CPU interface. Doing things manually (with an array as originally discussed) is fine, as long as you disable kernel preemption 
* If you do decide to use per-CPU data in your kernel code, consider the new interface. One caveat against its use is that it is not backward compatible with earlier kernels.
* If you need contiguous physical pages, use one of the low-level page allocators or kmalloc().This is the standard manner of allocating memory from within the kernel
* GFP_ATOMIC and GFP_KERNEL. Specify the GFP_ATOMIC flag to perform a high priority allocation that will not sleep.This is a requirement of interrupt handlers and other pieces of code that cannot sleep. Code that can sleep, such as process context code that does not hold a spin lock, should use GFP_KERNEL
* If you want to allocate from high memory, use alloc_pages().The alloc_pages() function returns a struct page and not a pointer to a logical address. Because high memory might not be mapped, the only way to access it might be via the corresponding struct page structure.To obtain an actual pointer, use kmap() to map the high memory into the kernel’s logical address space.
* If you do not need physically contiguous pages—only virtually contiguous—use vmalloc(), although bear in mind the slight performance hit taken with vmalloc() over kmalloc().The vmalloc() function allocates kernel memory that is virtually contiguous but not, per se, physically contiguous. It performs this feat much as user-space allocations do, by mapping chunks of physical memory into a contiguous logical address space. 
* If you are creating and destroying many large data structures, consider setting up a slab cache.The slab layer maintains a per-processor object cache (a free list), which might greatly enhance object allocation and deallocation performance. Rather than frequently allocate and free memory, the slab layer stores a cache of already allocated objects for you. When you need a new chunk of memory to hold your data structure, the slab layer often does not need to allocate more memory and instead simply can return an object from the cache.
* Obtaining memory inside the kernel is not always easy because you must be careful to ensure that the allocation process respects certain kernel conditions, such as an inability to block or access the filesystem
* the virtual filesystem (VFS), the kernel subsystem responsible for managing filesystems and providing a unified, consistent file API to user-space applications
* The Virtual Filesystem (sometimes called the Virtual File Switch or more commonly simply the VFS) is the subsystem of the kernel that implements the file and filesystem-related interfaces provided to user-space programs.All filesystems rely on the VFS to enable them not only to coexist, but also to interoperate.This enables programs to use standard Unix system calls to read and write to different filesystems
* The VFS is the glue that enables system calls such as open(), read(), and write()to work regardless of the filesystem
* More so, the system calls work between these different filesystems and media— we can use standard system calls to copy or move files from one filesystem to another
* the kernel implements an abstraction layer around its low-level filesystem interface.This abstraction layer enables Linux to support different filesystems
* The abstraction layer works by defining the basic conceptual interfaces and data structures that all filesystems support
* To the VFS layer and the rest of the kernel, however, each filesystem looks the same.
* Historically, Unix has provided four basic filesystem-related abstractions: files, directory entries, inodes, and mount points. 
* A filesystem is a hierarchical storage of data adhering to a specific structure. Filesystems contain files, directories, and associated control information.Typical operations performed on filesystems are creation, deletion, and mounting. In Unix, filesystems are mounted at a specific mount point in a global hierarchy known as a namespace.1 This enables all mounted filesystems to appear as entries in a single tree. Contrast this single, unified tree with the behavior of DOS and Windows, which break the file namespace up into drive letters, such as C:.This breaks the namespace up among device and partition boundaries, “leaking” hardware details into the filesystem abstraction.As this delineation may be arbitrary and even confusing to the user, it is inferior to Linux’s unified namespace. 
* A file is an ordered string of bytes.The first byte marks the beginning of the file, and the last byte marks the end of the file.
* Unix filesystems implement these notions as part of their physical ondisk layout. For example, file information is stored as an inode in a separate block on the disk; directories are files; control information is stored centrally in a superblock.
* even if a filesystem does not support distinct inodes, it must assemble the inode data structure in memory as if it did. Or if a filesystem treats directories as a special object, to the VFS they must represent directories as mere files,
* Such filesystems still work, however, and the overhead is not unreasonable
* TheVFS is object-oriented.2 A family of data structures represents the common file model.These data structures are akin to objects
* Because the kernel is programmed strictly in C, without the benefit of a language directly supporting object-oriented paradigms, the data structures are represented as C structures.The structures contain both data and pointers to filesystem-implemented functions that operate on the data.
* The four primary object types of the VFS are
* The superblock object, which represents a specific mounted filesystem. 
* The inode object, which represents a specific file. 
* The dentry object, which represents a directory entry, which is a single component of a path.
* The file object, which represents an open file as associated with a process.
* An operations object is contained within each of these primary objects.These objects describe the methods that the kernel invokes against the primary objects:
* The super_operations object, which contains the methods that the kernel can invoke on a specific filesystem, such as write_inode() and sync_fs() 
* The inode_operations object, which contains the methods that the kernel can invoke on a specific file, such as create() and link() 
* The dentry_operations object, which contains the methods that the kernel can invoke on a specific directory entry, such as d_compare() and d_delete() 
* The file_operations object, which contains the methods that a process can invoke on an open file, such as read() and write()
* The operations objects are implemented as a structure of pointers to functions that operate on the parent object
* Again, note that objects refer to structures—not explicit class types, such as those in C++ or Java.These structures, however, represent specific instances of an object, their associated data, and methods to operate on themselves.They are very much objects. 
* The VFS loves structures, and it is comprised of a couple more than the primary objects previously discussed. Each registered filesystem is represented by a file_system_type structure.This object describes the filesystem and its capabilities. Furthermore, each mount point is represented by the vfsmount structure.This structure contains information about the mount point, such as its location and mount flags. 
* Finally, two per-process structures describe the filesystem and files associated with a process
* The superblock object is implemented by each filesystem and is used to store information describing that specific filesystem.This object usually corresponds to the filesystem superblock or the filesystem control block, which is stored in a special sector on disk (hence the object’s name). Filesystems that are not disk-based (a virtual memory–based filesystem, such as sysfs, for example) generate the superblock on-the-fly and store it in memory. 
* The superblock object is represented by struct super_block and defined in <linux/fs.h>
* The most important item in the superblock object is s_op, which is a pointer to the superblock operations table.The superblock operations table is represented by struct super_operations and is defined in <linux/fs.h>
* Each item in this structure is a pointer to a function that operates on a superblock object.The superblock operations perform low-level operations on the filesystem and its inodes. When a filesystem needs to perform an operation on its superblock, it follows the pointers from its superblock object to the desired method. For example, if a filesystem wanted to write to its superblock, it would invoke
* sb->s_op->write_super(sb);
* In this call, sb is a pointer to the filesystem’s superblock. Following that pointer into s_op yields the superblock operations table and ultimately the desired write_super() function, which is then invoked. Note how the write_super() call must be passed a superblock, despite the method being associated with one.This is because of the lack of object-oriented support in C.
* Let’s take a look at some of the superblock operations that are specified by super_operations:
*  struct inode * alloc_inode(struct super_block *sb)
    *  Creates and initializes a new inode object under the given superblock.
* void destroy_inode(struct inode *inode) 
    * Deallocates the given inode.
* void dirty_inode(struct inode *inode)
    *  Invoked by the VFS when an inode is dirtied (modified). Journaling filesystems such as ext3 and ext4 use this function to perform journal updates.
* void write_inode(struct inode *inode, int wait) 
    * Writes the given inode to disk.The wait parameter specifies whether the operation should be synchronous.
* void drop_inode(struct inode *inode)
    * Called by the VFS when the last reference to an inode is dropped. Normal Unix filesystems do not define this function, in which case the VFS simply deletes the inode.
* void delete_inode(struct inode *inode)
    *  Deletes the given inode from the disk.
*  void put_super(struct super_block *sb) 
    * Called by the VFS on unmount to release the given superblock object.The caller must hold the s_lock lock.
*  void write_super(struct super_block *sb) 
    * Updates the on-disk superblock with the specified superblock.The VFS uses this function to synchronize a modified in-memory superblock with the disk.The caller must hold the s_lock lock.
*  int sync_fs(struct super_block *sb, int wait) 
    * Synchronizes filesystem metadata with the on-disk filesystem.The wait parameter specifies whether the operation is synchronous.
*  void write_super_lockfs(struct super_block *sb) 
    * Prevents changes to the filesystem, and then updates the on-disk superblock with the specified superblock. It is currently used by LVM (the LogicalVolume Manager).
*  void unlockfs(struct super_block *sb) 
    * Unlocks the filesystem against changes as done by write_super_lockfs().
*  int statfs(struct super_block *sb, struct statfs *statfs) 
    * Called by the VFS to obtain filesystem statistics.The statistics related to the given filesystem are placed in statfs.
*  int remount_fs(struct super_block *sb, int *flags, char *data)
    * Called by the VFS when the filesystem is remounted with new mount options.The caller must hold the s_lock lock.
*  void clear_inode(struct inode *inode) 
    * Called by the VFS to release the inode and clear any pages containing related data.
*  void umount_begin(struct super_block *sb) 
    * Called by the VFS to interrupt a mount operation. It is used by network filesystems, such as NFS.
* All these functions are invoked by the VFS, in process context.All except dirty_inode() may all block if needed.
* The inode object represents all the information needed by the kernel to manipulate a file or directory. For Unix-style filesystems, this information is simply read from the on-disk inode. If a filesystem does not have inodes, however, the filesystem must obtain the information from wherever it is stored on the disk. Filesystems without inodes generally store file-specific information as part of the file; unlike Unix-style filesystems, they do not separate file data from its control information.
* An inode represents each file on a filesystem, but the inode object is constructed in memory only as files are accessed.This includes special files, such as device files or pipes. Consequently, some of the entries in struct inode are related to these special files. For example, the i_pipe entry points to a named pipe data structure, i_bdev points to a block device structure, and i_cdev points to a character device structure.These three pointers are stored in a union because a given inode can represent only one of these (or none of them) at a time.
* As with the superblock operations, the inode_operations member is important. It describes the filesystem’s implemented functions that the VFS can invoke on an inode.
* The following interfaces constitute the various functions that the VFS may perform, or ask a specific filesystem to perform, on a given inode:
*  int create(struct inode *dir, struct dentry *dentry, int mode) TheVFS calls this function from the creat() and open() system calls to create a new inode associated with the given dentry object with the specified initial access mode.
* struct dentry * lookup(struct inode *dir, struct dentry *dentry) This function searches a directory for an inode corresponding to a filename specified in the given dentry.
* int link(struct dentry *old_dentry, struct inode *dir, struct dentry *dentry) Invoked by the link() system call to create a hard link of the file old_dentry in the directory dir with the new filename dentry.
* int unlink(struct inode *dir, struct dentry *dentry) Called from the unlink() system call to remove the inode specified by the directory entry dentry from the directory dir.
* int symlink(struct inode *dir, struct dentry *dentry, const char *symname) Called from the symlink() system call to create a symbolic link named symname to the file represented by dentry in the directory dir.
* int mkdir(struct inode *dir, struct dentry *dentry, int mode) Called from the mkdir() system call to create a new directory with the given initial mode.
* int rmdir(struct inode *dir, struct dentry *dentry) Called by the rmdir() system call to remove the directory referenced by dentry from the directory dir.
* int mknod(struct inode *dir, struct dentry *dentry, int mode, dev_t rdev) Called by the mknod() system call to create a special file (device file, named pipe, or socket).The file is referenced by the device rdev and the directory entry dentry in the directory dir.The initial permissions are given via mode.
* int rename(struct inode *old_dir, struct dentry *old_dentry, struct inode *new_dir, struct dentry *new_dentry) Called by the VFS to move the file specified by old_dentry from the old_dir directory to the directory new_dir, with the filename specified by new_dentry.
* int readlink(struct dentry *dentry, char *buffer, int buflen) Called by the readlink() system call to copy at most buflen bytes of the full path associated with the symbolic link specified by dentry into the specified buffer.
*  int follow_link(struct dentry *dentry, struct nameidata *nd) Called by the VFS to translate a symbolic link to the inode to which it points.The link pointed at by dentry is translated, and the result is stored in the nameidata structure pointed at by nd.
* int put_link(struct dentry *dentry, struct nameidata *nd) Called by the VFS to clean up after a call to follow_link().
*  void truncate(struct inode *inode) Called by the VFS to modify the size of the given file. Before invocation, the inode’s i_size field must be set to the desired new size.
*  int permission(struct inode *inode, int mask) Checks whether the specified access mode is allowed for the file referenced by inode.This function returns zero if the access is allowed and a negative error code otherwise. Most filesystems set this field to NULL and use the generic VFS method, which simply compares the mode bits in the inode’s objects to the given mask. More complicated filesystems, such as those supporting access control lists (ACLs), have a specific permission() method.
*  int setattr(struct dentry *dentry, struct iattr *attr) Called from notify_change() to notify a “change event” after an inode has been modified.
*  int getattr(struct vfsmount *mnt, struct dentry *dentry, struct kstat *stat) Invoked by the VFS upon noticing that an inode needs to be refreshed from disk. Extended attributes allow the association of key/values pairs with files.
* int setxattr(struct dentry *dentry, const char *name, const void *value, size_t size, int flags) Used by the VFS to set the extended attribute name to the value value on the file referenced by dentry.
* ssize_t getxattr(struct dentry *dentry, const char *name, void *value, size_t size) Used by the VFS to copy into value the value of the extended attribute name for the specified file.
* ssize_t listxattr(struct dentry *dentry, char *list, size_t size) Copies the list of all attributes for the specified file into the buffer list.
* int removexattr(struct dentry *dentry, const char *name)
* As discussed, the VFS treats directories as a type of file. In the path /bin/vi, both bin and vi are files—bin being the special directory file and vi being a regular file.An inode object represents each of these components
* As discussed, the VFS treats directories as a type of file. In the path /bin/vi, both bin and vi are files—bin being the special directory file and vi being a regular file.An inode object represents each of these components
* The VFS creates it on-the-fly from a string representation of a pathname. Because the dentry object is not physically stored on the disk
* the kernel caches dentry objects in the dentry cache or, simply, the dcache.
*  Lists of “used” dentries linked off their associated inode via the i_dentry field of the inode object. Because a given inode can have multiple links, there might be multiple dentry objects; consequently, a list is used.
*  A doubly linked “least recently used” list of unused and negative dentry objects.The list is inserted at the head, such that entries toward the head of the list are newer than entries toward the tail.When the kernel must remove entries to reclaim memory, the entries are removed from the tail; those are the oldest and presumably have the least chance of being used in the near future. 
* A hash table and hashing function used to quickly resolve a given path into the associated dentry object.
* The hash table is represented by the dentry_hashtable array. Each element is a pointer to a list of dentries that hash to the same value.
* .The first object is used to describe a specific variant of a filesystem, such as ext3, ext4, or UDF.The second data structure describes a mounted instance of a filesystem.
* The get_sb() function reads the superblock from the disk and populates the superblock object when the filesystem is loaded.The remaining functions describe the filesystem’s properties
* Things get more interesting when the filesystem is actually mounted, at which point the vfsmount structure is created.This structure represents a specific instance of a filesystem—in other words, a mount point.
* The complicated part of maintaining the list of all mount points is the relation between the filesystem and all the other mount points.The various linked lists in vfsmount
* Each process on the system has its own list of open files, root filesystem, current working directory, mount points, and so on
* The array fd_array points to the list of open file objects. Because NR_OPEN_DEFAULT is equal to BITS_PER_LONG, which is 64 on a 64-bit architecture; this includes room for 64 file objects. If a process opens more than 64 file objects, the kernel allocates a new array and points the fdt pointer at it. In this fashion, access to a reasonable number of file objects is quick, taking place in a static array. If a process opens an abnormal number of files, the kernel can create a new array. If the majority of processes on a system opens more than 64 files, for optimum performance the administrator can increase the NR_OPEN_DEFAULT preprocessor macro to match.
* The list member specifies a doubly linked list of the mounted filesystems that make up the namespace. 
* The namespace structure works the other way around. By default, all processes share the same namespace. (That is, they all see the same filesystem hierarchy from the same mount table.) Only when the CLONE_NEWNS flag is specified during clone() is the process given a unique copy of the namespace structure. Because most processes do not provide this flag, all the processes inherit their parents’ namespaces
* In addition to managing its own memory, the kernel also has to manage the memory of user-space processes.This memory is called the process address space
* Linux is a virtual memory operating system
* An individual process’s view of memory is as if it alone has full access to the system’s physical memory. More important, the address space of even a single process can be much larger than physical memory.
* The process address space consists of the virtual memory addressable by a process and the addresses within the virtual memory that the process is allowed to use. Each process is given a flat 32- or 64-bit address space, with the size depending on the architecture.The term flat denotes that the address space exists in a single range.
* Some operating systems provide a segmented address space, with addresses existing not in a single linear range, but instead in multiple segments. Modern virtual memory operating systems generally have a flat memory model and not a segmented one. Normally, this flat address space is unique to each process.
* Both processes can have different data at the same address in their respective address spaces.Alternatively, processes can elect to share their address space with other processes.
* A memory address is a given value within the address space, such as 4021f000
* Memory areas can contain all sorts of goodies, such as
* A memory map of the executable file’s code, called the text section. 
* A memory map of the executable file’s initialized global variables, called the data section. 
* A memory map of the zero page (a page consisting of all zeros, used for purposes such as this) containing uninitialized global variables, called the bss section.
* A memory map of the zero page used for the process’s user-space stack. (Do not confuse this with the process’s kernel stack, which is separate and maintained and used by the kernel.)
* An additional text, data, and bss section for each shared library, such as the C library and dynamic linker, loaded into the process’s address space. 
* Any memory mapped files.
* Any shared memory segments. n Any anonymous memory mappings, such as those associated with malloc()
* All valid addresses in the process address space exist in exactly one area;memory areas do not overlap.As you can see,there is a separate memory area for each different chunk of memory in a running process:the stack,object code,global variables,mapped file,and so on
* The mm_users field is the number of processes using this address space. For example, if two threads share this address space, mm_users is equal to two.The mm_count field is the primary reference count for the mm_struct.All mm_users equate to one increment of mm_count.
* If nine threads shared an address space, mm_users would be nine, but again mm_count would be only one. Only when mm_users reaches zero.
* Having two counters enables the kernel to differentiate between the main usage counter (mm_count) and the number of processes using the address space (mm_users).
* The mmap and mm_rb fields are different data structures that contain the same thing: all the memory areas in this address space.The former stores them in a linked list, whereas the latter stores them in a red-black tree.A red-black tree is a type of binary tree; like all binary trees, searching for a given element is an O(log n) operation
* All of the mm_struct structures are strung together in a doubly linked list via the mmlist field.The initial element in the list is the init_mm memory descriptor, which describes the address space of the init process.The list is protected from concurrent access via the mmlist_lock, which is defined in kernel/fork.c.
* The memory descriptor associated with a given task is stored in the mm field of the task’s process descriptor. (The process descriptor is represented by the task_struct structure, defined in <linux/sched.h>.) Thus, current->mm is the current process’s memory descriptor.The copy_mm() function copies a parent’s memory descriptor to its child during fork().The mm_struct structure is allocated from the mm_cachep slab cache via the allocate_mm() macro in kernel/fork.c. Normally, each process receives a unique mm_struct and thus a unique process address space. Processes may elect to share their address spaces with their children by means of the CLONE_VM flag to clone().
* When the process associated with a specific address space exits, the exit_mm(), defined in kernel/exit.c, function is invoked.This function performs some housekeeping and updates some statistics. It then calls mmput(), which decrements the memory descriptor’s mm_users user counter. If the user count reaches zero, mmdrop() is called to decrement the mm_count usage counter. If that counter is finally zero, the free_mm() macro is invoked to return the mm_struct to the mm_cachep slab cache via kmem_cache_free(), because the memory descriptor does not have any users.
* Kernel threads do not have a process address space and therefore do not have an associated memory descriptor.Thus, the mm field of a kernel thread’s process descriptor is NULL. This is the definition of a kernel thread—processes that have no user context. This lack of an address space is fine because kernel threads do not ever access any userspace memory. (Whose would they access?) Because kernel threads do not have any pages in user-space, they do not deserve their own memory descriptor and page tables. (Page tables are discussed later in the chapter.) Despite this, kernel threads need some of the data, such as the page tables, even to access kernel memory.To provide kernel threads the needed data, without wasting memory on a memory descriptor and page tables, or wasting processor cycles to switch to a new address space whenever a kernel thread begins running, kernel threads use the memory descriptor of whatever task ran previously. Whenever a process is scheduled, the process address space referenced by the process’s mm field is loaded.The active_mm field in the process descriptor is then updated to refer to the new address space. Kernel threads do not have an address space and mm is NULL. Therefore, when a kernel thread is scheduled, the kernel notices that mm is NULL and keeps the previous process’s address space loaded.The kernel then updates the active_mm field of the kernel thread’s process descriptor to refer to the previous process’s memory descriptor.The kernel thread can then use the previous process’s page tables as needed. Because kernel threads do not access user-space memory, they make use of only the information in the address space pertaining to kernel memory, which is the same for all processes
* The memory area structure, vm_area_struct, represents memory areas. It is defined in <linux/mm_types.h>. In the Linux kernel, memory areas are often called virtual memory areas (abbreviated VMAs).
* The vm_area_struct structure describes a single memory area over a contiguous interval in a given address space.The kernel treats each memory area as a unique memory object. Each memory area possesses certain properties, such as permissions and a set of associated operations.
* Recall that each memory descriptor is associated with a unique interval in the process’s address space.The vm_start field is the initial (lowest) address in the interval, and the vm_end field is the first byte after the final (highest) address in the interval.That is, vm_start is the inclusive start, and vm_end is the exclusive end of the memory interval.
* The vm_mm field points to this VMA’s associated mm_struct. Note that each VMA is unique to the mm_struct with which it is associated.Therefore, even if two separate processes map the same file into their respective address spaces, each has a unique vm_area_struct to identify its unique memory area. Conversely, two threads that share an address space also share all the vm_area_struct structures therein.
* vm_flags contains information that relates to each page in the memory area, or the memory area as a whole, and not specific individual pages
* 
![alt_text](images/image8.png "image_tooltip")

* The VM_SHARED flag specifies whether the memory area contains a mapping that is shared among multiple processes. If the flag is set, it is intuitively called a shared mapping.If the flag is not set, only a single process can view this particular mapping, and it is called a private mapping.
* The VM_IO flag specifies that this memory area is a mapping of a device’s I/O space. This field is typically set by device drivers when mmap()
* The VM_SEQ_READ flag provides a hint to the kernel that the application is performing sequential (that is, linear and contiguous) reads in this mapping
* As discussed, memory areas are accessed via both the mmap and the mm_rb fields of the memory descriptor.These two data structures independently point to all the memory area objects associated with the memory descriptor. In fact, they both contain pointers to the same vm_area_struct structures, merely represented in different ways. The first field, mmap, links together all the memory area objects in a singly linked list. Each vm_area_struct structure is linked into the list via its vm_next field.The areas are sorted by ascending address.The first memory area is the vm_area_struct structure to which mmap points.The last structure points to NULL. The second field, mm_rb, links together all the memory area objects in a red-black tree. The root of the red-black tree is mm_rb, and each vm_area_struct structure in this address space is linked to the tree via its vm_rb field
* The pmap(1) utility displays a formatted listing of a process’s memory areas. It is a bit more readable than the / proc output, but it is the same information. It is found in newer versions of the procps package.
* The kernel often has to perform operations on a memory area, such as whether a given address exists in a given VMA.These operations are frequent and form the basis of the mmap() routine, which is covered in the next section.A handful of helper functions are defined to assist these jobs. These functions are all declared in <linux/mm.h>.
* This function searches the given address space for the first memory area whose vm_end field is greater than addr. In other words, this function finds the first memory area that contains addr or begins at an address greater than addr. If no such memory area exists, the function returns NULL. Otherwise, a pointer to the vm_area_struct structure is returned. Note that because the returned VMA may start at an address greater than addr, the given address does not necessarily lie inside the returned VMA.The result of the find_vma() function is cached in the mmap_cache field of the memory descriptor. Because of the probability of an operation on one VMA being followed by more operations on that same VMA, the cached results have a decent hit rate (about 30–40% in practice). Checking the cached result is quick. If the given address is not in the cache, you must search the memory areas associated with this memory descriptor for a match.This is done via the red-black tree.
* The initial check of mmap_cache tests whether the cached VMA contains the desired address. Note that simply checking whether the VMA’s vm_end field is bigger than addr would not ensure that this is the first such VMA that is larger than addr.Thus, for the cache to be useful here, the given addr must lie in the VMA
* The find_vma_prev() function works the same as find_vma(), but it also returns the last VMA before addr.The function is also defined in mm/mmap.c and declared in <linux/mm.h>:
* The find_vma_intersection()function returns the first VMA that overlaps a given address interval.The function is defined in <linux/mm.h>
* The first parameter is the address space to search, start_addr is the start of the interval, and end_addr is the end of the interval.
* The do_mmap()function is used by the kernel to create a new linear address interval. Saying that this function creates a new VMA is not technically correct, because if the created address interval is adjacent to an existing address interval, and if they share the same permissions, the two intervals are merged into one. If this is not possible, a new VMA is created. In any case, do_mmap() is the function used to add an address interval to a process’s address space—whether that means expanding an existing memory area or creating a new one.
* This function maps the file specified by file at offset offset for length len.The file parameter can be NULL and offset can be zero, in which case the mapping will not be backed by a file. In that case, this is called an anonymous mapping. If a file and offset are provided, the mapping is called a file-backed mapping.
* The flags parameter specifies flags that correspond to the remaining VMA flags. These flags specify the type and change the behavior of the mapping.They are also defined in <asm/mman.h>.
* If any of the parameters are invalid, do_mmap() returns a negative value. Otherwise, a suitable interval in virtual memory is located. If possible, the interval is merged with an adjacent memory area. Otherwise, a new vm_area_struct structure is allocated from the vm_area_cachep slab cache, and the new memory area is added to the address space’s linked list and red-black tree of memory areas via the vma_link() function. Next, the total_vm field in the memory descriptor is updated. Finally, the function returns the initial address of the newly created address interval. The do_mmap() functionality is exported to user-space via the mmap() system call.
* The original mmap() took an offset in bytes as the last parameter; the current mmap2() receives the offset in pages.This enables larger files with larger offsets to be mapped
* Both library calls use the mmap2() system call, with the original mmap() converting the offset from bytes to pages.
* The do_munmap() function removes an address interval from a specified process address space.The function is declared in <linux/mm.h>:
    * int do_munmap(struct mm_struct *mm, unsigned long start, size_t len) 
* The first parameter specifies the address space from which the interval starting at address start of length len bytes is removed. On success, zero is returned. Otherwise, a negative error code is returned. The munmap()system call is exported to user-space as a means to enable processes to remove address intervals from their address space.
* Although applications operate on virtual memory mapped to physical addresses, processors operate directly on those physical addresses. Consequently, when an application accesses a virtual memory address, it must first be converted to a physical address before the processor can resolve the request. Performing this lookup is done via page tables. Page tables work by splitting the virtual address into chunks. Each chunk is used as an index into a table.The table points to either another table or the associated physical page.
* In Linux, the page tables consist of three levels.The multiple levels enable a sparsely populated address space, even on 64-bit machines
* Using three levels is a sort of “greatest common denominator”—architectures with a less complicated implementation can simplify the kernel page tables as needed with compiler optimizations.
* The top-level page table is the page global directory (PGD), which consists of an array of pgd_t types. On most architectures, the pgd_t type is an unsigned long.The entries in the PGD point to entries in the second-level directory, the PMD. 
* The second-level page table is the page middle directory (PMD), which is an array of pmd_t types.The entries in the PMD point to entries in the PTE.
* The final level is called simply the page table and consists of page table entries of type pte_t. Page table entries point to physical pages.
* In most architectures, page table lookups are handled (at least to some degree) by hardware.
* The kernel must set things up, however, in such a way that the hardware is happy and can do its thing
* Each process has its own page tables (threads share them, of course).The pgd field of the memory descriptor points to the process’s page global directory. Manipulating and traversing page tables requires the page_table_lock
* Page table data structures are quite architecture-dependent and thus are defined in <asm/page.h>. Because nearly every access of a page in virtual memory must be resolved to its corresponding address in physical memory, the performance of the page tables is very critical.
* most processors implement a translation lookaside buffer, or simply TLB, which acts as a hardware cache of virtual-to-physical mappings.When accessing a virtual address, the processor first checks whether the mapping is cached in the TLB. If there is a hit, the physical address is immediately returned. Otherwise, if there is a miss, the page tables are consulted for the corresponding physical address
* Future possibilities include shared page tables with copy-on-write semantics. In that scheme, page tables would be shared between parent and child across a fork().When the parent or the child attempted to modify a particular page table entry, a copy would be created, and the two processes would no longer share that entry. Sharing page tables would remove the overhead of copying the page table entries on fork().
