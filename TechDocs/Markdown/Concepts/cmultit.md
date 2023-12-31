## B Threads and Semaphores

One of the most impressive features of GEOS is its ability to perform 
several tasks simultaneously, even on the least powerful PCs. Of course, the 
PC has only one processor and can execute only one instruction at a time. 
GEOS keeps track of the various tasks, or threads, that are underway, and 
by switching from one thread to another many times per second creates the 
illusion that the PC is actually doing all the jobs at the same time.

This chapter covers

+ the basic concepts of multitasking,
+ the GEOS multitasking scheme,
+ the steps to creating a multi-threaded application, and
+ the use of semaphores to synchronize threads and avoid deadlock.

You will need to know the information in this appendix if you will be 
writing a multi-threaded application. The information in the appendix is 
not essential if your application will be single-threaded or if you will use 
the standard GEOS dual-thread architecture. For a dual-thread 
application, be careful not to send a message with **@call** from an object run 
by a user interface thread to an object run by any other thread of the 
application.

### B.1 Multitasking Goals

The GEOS multitasking system is one of the most sophisticated available 
for PCs today. It was designed with the latest available technology and was 
created to serve the following two primary goals:

+ Fast Response  
The GEOS multitasking system was designed with a strong emphasis 
on rapid response to user input. Prompt, visible reaction to user action 
is the single most important factor contributing to the perception of 
speed. For example, if a user changes an element in a spreadsheet 
(requiring the whole spreadsheet to be recomputed) and then pulls 
down a menu, he wants the menu to appear right away. If the menu 
does not appear until the computation is finished, the system will seem 
sluggish. If the computation takes a few seconds or longer, the user 
may wonder whether the system has crashed altogether.

+ Ease of Programming  
Making the programmer's job easy is another motive behind the design 
of GEOS multitasking. Ideally, the programmer should not need to be 
aware that his program will run in a multitasking environment. The 
program should proceed as though it were the only one running on the 
system. Besides certain "good citizen" rules, application programs are 
isolated from the multitasking environment. On the other hand, if a 
programmer wants to take advantage of the multitasking capability of 
GEOS (by designing a program to perform more than one task 
concurrently) the system is designed to make this as simple and 
efficient as possible.

### B.2 Two Models of Multitasking

Many operating systems provide the ability to carry out multiple tasks at 
the same time. While each operating system has its own unique way of 
managing this, there are two fundamental types of multitasking: 
cooperative (used by Microsoft Windows), and preemptive (used by GEOS).

#### B.2.1 Cooperative Multitasking

A cooperative multitasking system, as the name suggests, is one in which 
the various programs cooperate. They agree to share the system and its 
resources. Each program running under a system has complete control 
while it is actually running. Every so often, when it reaches a convenient 
place, it calls a special system routine (called a context-switch routine) to 
see if any other program has work do to. If so, that program takes control 
of the system until it in turn reaches a convenient stopping point and 
passes control on to the next waiting program. If several programs are 
ready to run, they wait in a queue so that each one gets a chance to run 
before the first program runs again. (It is also possible to implement a 
cooperative multitasking system where some programs have a higher 
priority than others. In this case, a more complicated algorithm might be 
used by the context-switch routine to determine which program gets to run 
next.)

Smooth operation of a cooperative multitasking system requires that all 
programs be written to call the context-switch routine frequently. When 
large calculations are being performed, programmers tend to find this 
requirement inconvenient. Writing well-behaved programs (i.e., programs 
that do not keep control of the processor for too long at a stretch) is 
especially difficult because most cooperative multitasking systems impose 
restrictions on when a context switch can take place.

#### B.2.2 Preemptive Multitasking

In a preemptive multitasking system, programs do not have to relinquish 
control of the system voluntarily. Instead of calling a context-switch 
routine, the program is written as though it were going to run continuously 
from start to finish. The hardware generates a timer interrupt a number of 
times each second, and that interrupt triggers the kernel's context-switch 
mechanism.

The context switch can also be triggered by other interrupts. For example, 
if the user moves the mouse in GEOS, the mouse will generate an interrupt. 
GEOS responds by marking the input thread runnable; the thread will then 
run after the interrupt is complete. This is how GEOS achieves its 
extraordinary response times to user input.

With preemptive multitasking, each program can have the illusion that it 
is running continuously and has complete control of the system. It also 
enables the system to interact quickly with the user even when 
applications are busily computing new results.

For example, a spreadsheet program can keep running until the timer 
interrupt causes a context switch. Other programs, including the one 
responsible for drawing menus, then get their turns to run. If a user clicks 
on a pull-down menu, the menu will appear. When the spreadsheet 
program regains control of the system, it can carry on from where it was 
interrupted, blissfully unaware that any of this has taken place.

While preemptive multitasking makes most things simpler for the user 
and application programmer, there are a few important issues to consider 
in writing programs for a preemptive multitasking system such as GEOS. 
When the context switches are controlled by a timer interrupt, they can 
occur between any two instructions. If a program is interrupted while it is 
updating a data structure, that data structure may be left in an 
inconsistent state while another thread is running. If the data structure is 
not accessed by any other process running on the system, there is no 
problem: the update will be completed when the program resumes. 
However, some data structures (including system resources) may be 
accessed by more than one program. It is important that two updates to the 
same data do not happen at the same time.

This problem is analogous to one often experienced by network users. If a 
text file is being edited at the same time by two different users and they 
both save their changes to the file, whoever saves first will have his version 
overwritten by the other. Many systems have a means of locking a file while 
you are editing it; no one else can begin editing the file while you have it 
locked. A preemptive multitasking system must have a similar locking 
scheme to prevent two accesses to the same data structure from happening 
at the same time. The locking mechanism should be as transparent as 
possible to the programmer. For example, the locking and unlocking of 
system resources should happen automatically so that application 
programmers need not concern themselves with it.

This is exactly how GEOS coordinates its resources, as you shall see in the 
following sections.

### B.3 GEOS Multitasking

GEOS implements a preemptive multitasking scheme. Application 
programs are required to follow certain "good citizen" rules and are 
otherwise given the illusion that they are alone on the system. The 
applications themselves are, for the most part, isolated from the 
multitasking environment.

#### B.3.1 GEOS Threads

The various units that take turns running in the system are called 
"threads" in GEOS. Threads can have different priorities; a thread that has 
a higher priority (indicated by a lower priority number) will generally get 
more processor time.

##### B.3.1.1 Keeping Track of Threads

In order to switch among threads, the system needs to keep track of certain 
things about each one. For each thread, the system keeps track of priority, 
the most recent values of the registers, and flags. The priorities are used to 
determine which thread is going to run next. When the thread is run, the 
appropriate registers are reset to the values they had the last time the 
thread was stopped. This allows the thread to resume execution as though 
it had never been interrupted. 

Like other things in GEOS, each thread has a handle, a sixteen-bit value 
which programs use to refer to the thread. When calling thread-related 
routines (e.g., to set the thread's priority), programs use the handle to 
specify the thread.

##### B.3.1.2 Event-Driven and Procedural Threads

GEOS uses two different types of threads. The two types differ only in the 
way they run when their turn comes and in the way they are created. The 
discussions about priority, context switches, and synchronization 
elsewhere in this chapter apply equally to both types.

The first type of GEOS thread is an "event-driven" thread. An event-driven 
thread normally executes code for one or more objects. Each event-driven 
thread has an event queue; when a message is sent to any object created by 
the thread, the message is placed in the thread's event queue. The thread 
processes each event in the order received by executing the appropriate 
message handler from the object's class definition.

Messages can also be sent to the thread itself rather than to an object 
created by the thread. When the thread is created, it is assigned a 
class-normally a subclass of **ProcessClass** (for non-application threads) 
or **GenProcessClass** (for application threads)-to determine the handlers 
to use when messages are sent directly to the thread. In this sense, the 
thread can be considered an instance of the given class.

The second type of thread is procedural. Rather than running handlers for 
messages in an object-oriented scheme, it simply executes procedural code. 
The system does not provide an event queue for a procedural thread, and 
messages cannot be sent to such a thread.

#### B.3.2 Context Switches

Context switches are triggered in two ways under GEOS. The first is a timer 
or other hardware interrupt. The second occurs when the thread reaches a 
point where it cannot continue right away, such as when the thread exits 
or when it attempts to access a locked resource.

The PC hardware generates a timer interrupt sixty times per second. The 
time between timer interrupts is called a tick. Each thread is allowed to run 
for a specified number of ticks before the timer interrupt routine will 
transfer control to a different thread. This number of ticks is the same for 
all threads; it is called a time slice.

When a thread begins its turn, GEOS sets a counter to the number of ticks 
in a time slice. At each timer interrupt, GEOS decrements the counter. If 
the counter has not reached zero, control is immediately returned to the 
running thread. When the counter reaches zero, GEOS checks to see if some 
other thread has reached a higher priority than the current thread. If so, 
the current thread is placed in the system's list of runnable threads (called 
the run queue), and the highest priority thread begins running. Otherwise 
the current thread gets to run for another time slice.

Sometimes a thread will try to access a system resource (or shared data 
object) which is currently in use by another thread. When this happens, the 
thread must wait until the desired resource is available. The thread is 
placed on a queue, and a new thread is selected from the run queue. Every 
thread that is not currently running is either in the run queue (waiting to 
be executed) or in another queue waiting for a needed resource to become 
available. When the resource becomes available, the thread is moved to the 
run queue and is ready to be run again. This process is described in greater 
detail in [section B.5](#b5-synchronizing-threads).

#### B.3.3 Thread Scheduling

When there is a context switch, GEOS must determine which thread will be 
executed next. It does this by examining the run queue and selecting the 
most important thread to run. A thread's importance is reflected in its 
priority: the lower the priority number, the more important the thread.

##### B.3.3.1 Base Priority

Each thread gets a base priority, a value from zero to 255. Applications 
generally have a priority between 128 and 191. Threads that are critical to 
quick system response (such as user input threads that manage such 
things as pull-down menus and dialog boxes) are given lower numbers. 
Higher numbers can be given to less time-critical threads such as those 
used for print spooling and other background activities. To provide faster 
response to the user, GEOS temporarily reduces an application's base 
priority by 32 (giving it a higher priority) when the user is interacting with 
it. When the user switches to interact with another application, the first 
application's base priority returns to normal, and the new one gets a 
reduced base priority number.

##### B.3.3.2 Recent CPU Usage

GEOS keeps track of a thread's recent CPU usage with a number that varies 
from zero to 60. Starting at zero, the number is incremented at every timer 
interrupt while the thread is running. Once each second, the recent CPU 
usage is halved, so that as a thread's CPU usage recedes into the past, its 
recent CPU usage number will diminish. The resulting number reflects how 
much time the thread has had control of the CPU and how long ago this 
time was.

##### B.3.3.3 Current Priority

Once each second, the current priority of each thread is recomputed by 
adding the base priority to the recent CPU usage. The resulting number is 
the one used in selecting a thread from the run queue.

When it is time for a context switch, GEOS selects the thread with the 
lowest current priority number from the run queue. If there is a tie, the 
selection is arbitrary. However, because recent CPU usage counts against a 
thread, two threads of equal priority will not stay that way. One will run, 
and its recent CPU usage (and thus its current priority number) will be 
increased. The other thread will therefore get its chance to run.

#### B.3.4 Applications and Threads

There are two standard architectures for GEOS applications: single-thread 
and dual-thread. While the single-thread option is somewhat easier to 
program, there are distinct advantages to the dual-thread method.

In the dual-thread architecture, one thread manages the application's user 
interface while the other manages the rest of the application's 
functionality. Since both threads are event-driven, each has an event 
queue. Messages that are sent to user interface objects (resulting from 
mouse clicks, keyboard input, etc.) can be handled without waiting for 
other tasks in the application to be completed. This allows the application 
to respond to user input (by putting up menus, moving windows, and so on) 
without first completing the current non-user-interface task (which may 
involve a lot of computation). 

The dual-thread architecture, however, poses a problem of 
synchronization: One thread can get ahead of the other. Threads that count 
on each other must keep track of each other's progress in order to avoid 
this; when potential problems are identified, use semaphores to keep the 
threads in line (see [section B.5](#b5-synchronizing-threads)).

### B.4 Using Multiple Threads

It is possible for an application to create additional threads for a variety of 
purposes. For example, a terminal emulation program might have a thread 
whose sole purpose is to monitor the serial line for incoming characters. 
This might avert the danger of a serial input buffer or stream overflowing 
while the application is performing an involved task, such as loading a text 
file from disk, while not requiring a great deal of fixed memory for the serial 
driver's input buffer.

#### B.4.1 How GEOS Threads Are Created

GEOS threads can be created in three different ways. The first thread (or 
pair of threads, in the dual-thread architecture) for each application is 
created automatically when the application is launched. By calling the 
appropriate routines, the application can create additional threads to 
handle messages sent to certain objects or to run procedural code.

##### B.4.1.1 The Application's Primary Thread

The application's primary thread is created automatically by GEOS when 
the application is launched. (See ["Applications and Geodes", Chapter 6](cappl.md) 
for information on launching applications.) For example, if a user 
double-clicks on your application's icon in a GeoManager window, 
GeoManager calls the library routine **UserLoadApplication()**, specifying 
the geode file and certain other parameters. This calls the **GeodeLoad()** 
routine in the GEOS kernel.

If the program is written using the single-thread model, GEOS creates an 
event-driven thread to handle messages sent to any object in the program. 
If the program is written using the dual-thread model, GEOS creates one 
event-driven thread to handle messages sent to the program's user 
interface objects and another to handle messages sent to other objects in 
the program.

If the program requires more than two threads, the extra thread(s) must 
be allocated manually on startup and destroyed before the application exits 
completely.

##### B.4.1.2 Event-Driven Threads

ThreadAttachToQueue()

To create an event-driven thread (one that handles messages sent to 
certain objects), send a MSG_PROCESS_CREATE_EVENT_THREAD to your 
application's primary thread, passing as arguments the object class for the 
new thread (a sublass of **ProcessClass**) and the stack size for the new 
thread (1 K bytes is usually a good value, or around 3 K bytes for threads 
that will handle keyboard navigation or manage a text object). This 
message is detailed in ["System Classes", Chapter 1 of the Object Reference 
Book](../Objects/osyscla.md).

GEOS will create the new thread, give it an event queue, and send it a 
MSG_META_ATTACH. Initially, the thread will handle only messages sent 
to the thread itself. If the thread creates any new objects, however, it will 
handle messages sent to those objects as well. To control the behavior of the 
new thread, define a subclass of **ProcessClass** and a new handler for 
MSG_META_ATTACH. The new handler can create objects or perform 
whatever task is needed. Be sure to start your new handler with 
**@callsuper()** so that the predefined initializations are done as well.

If you have a thread that you want attached to a different event queue, you 
can use **ThreadAttachToQueue()**. This routine is not widely used except 
when applications are shutting down and objects need to continue handling 
messages while not returning anything. It's unlikely you will ever use this 
routine.

##### B.4.1.3 Threads That Run Procedural Code

ThreadCreate()

To create a thread to run procedural code, first load the initial function into 
fixed memory. Then call the system routine **ThreadCreate()**, passing the 
following arguments: The base priority for the new thread, an optional 
sixteen-bit argument to pass to the new thread, the entry point for the code, 
the amount of stack space GEOS should allocate for the new thread, and the 
owner of the new thread.

#### B.4.2 Managing Priority Values

ThreadGetInfo(), ThreadModify(), ThreadGetInfoType

You can ascertain and modify the priority of any thread in the system, 
given the thread's handle. The handle is provided by the routines that 
create threads, and it can be provided by one thread to another in a 
message. The following system routines relate to the priority of a thread:

**ThreadGetInfo()** returns information about a thread. When calling 
**ThreadGetInfo()**, pass the handle of the thread in question and a value 
of the type **ThreadGetInfoType** (see below). If zero is passed as the 
thread handle, **ThreadGetInfo()** returns information on whatever thread 
executed the call.

ThreadGetInfoType is an enumerated type with three possible values:

TGIT_PRIORITY_AND_USAGE  
This requests the base priority and recent CPU usage of a 
thread. (To determine the current priority, simply add the 
base priority to the recent CPU usage.)

TGIT_THREAD_HANDLE  
This requests the handle of the thread. Use this (with a 
thread handle of zero) to get the caller's own thread handle.

TGIT_QUEUE_HANDLE  
This requests the handle of the event queue for an 
event-driven thread. It returns a zero handle if the thread is 
not event-driven.

**ThreadModify()** changes the priority of a thread. The arguments to pass 
include the handle of the thread to modify (zero for the thread executing 
the call), a new base priority for the thread, and two flags: One that 
indicates whether to change the thread's base priority and one that 
indicates whether to reset the thread's recent CPU usage to zero. If the flag 
to change the thread's base priority is not set, the new base priority 
argument is ignored. In general, you should only lower a thread's priority 
(i.e., raise its base priority number). Applications that raise their own 
priority damage the performance of the system as a whole. Keep in mind 
that GEOS already favors the thread with which the user is interacting.

There are several pre-defined priority levels you can use to set a thread's 
priority. You may wish to use these when debugging to raise the priority of 
a potentially buggy thread for efficient debugging. These are listed below, 
each a different constant.

+ PRIORITY_TIME_CRITICAL  
Threads should not be set time-critical unless they must own the 
processor exclusively for a certain amount of time. Excluding other 
threads can have undesirable side effects.

+ PRIORITY_HIGH  

+ PRIORITY_UI  

+ PRIORITY_FOCUS  

+ PRIORITY_STANDARD  

+ PRIORITY_LOW  

+ PRIORITY_LOWEST  

#### B.4.3 Handling Errors in a Thread

ThreadHandleException(), ThreadException

Some threads in GEOS will want to handle certain errors in special ways. 
The errors a particular thread can intercept and handle are listed in an 
enumerated type called **ThreadException**, the elements of which are 
shown below:

+ TE_DIVIDE_BY_ZERO

+ TE_OVERFLOW

+ TE_BOUND

+ TE_FPU_EXCEPTION

+ TE_SINGLE_STEP

+ TE_BREAKPOINT

A thread can handle a particular exception by setting up a special handler 
routine and calling **ThreadHandleException()** when one of these 
exceptions occurs. This is useful if a number of objects are run by the same 
thread and all should handle a particular exception in the same way; the 
routine can be thread-specific rather than object-specific. 
**ThreadHandleException()** must be passed the thread's handle, the 
exception type, and a pointer to the handler routine's entry point.

#### B.4.4 When a Thread Is Finished

ThreadDestroy()

Whenever an application creates an additional thread with 
MSG_PROCESS_CREATE_EVENT_THREAD or **ThreadCreate()**, it must be 
sure that the thread exits when it is finished. Simply exiting the 
application may not eliminate any additional threads, and these threads 
can cause GEOS to hang when shutting down the system.

When a thread exits, it should first release any semaphores or thread locks 
it has locked and free any memory or other resources that are no longer 
needed. Resources in memory do not have to be freed in the same thread 
that allocated them, but you should be sure that they are freed before the 
application exits.

A procedural thread exits by calling **ThreadDestroy()** with two 
arguments: an error code and an optr. When the thread exits, it sends (as 
its last act) a MSG_PROCESS_NOTIFY_THREAD_EXIT to the application's 
primary thread and a MSG_META_ACK to the object descriptor passed. 
Each message has the error code as an argument. In designing a 
multi-threaded application, you can create methods for 
MSG_PROCESS_NOTIFY_THREAD_EXIT (in your primary thread's class) or 
MSG_META_ACK (in any class) for communication among threads, and you 
may use the error code for any data you choose. The convention is that an 
error code of zero represents successful completion of a thread's task.

An event-driven thread should not call **ThreadDestroy()** directly because 
its event queue must be removed from the system cleanly. Instead, send a 
MSG_META_DETACH to the thread, passing the same arguments as for 
**ThreadDestroy()**. The handler for MSG_META_DETACH in **MetaClass** 
cleanly removes the event queue and terminates the thread, sending the 
same messages as described above. You may write a special handler for 
MSG_META_DETACH when you subclass **ProcessClass**, but be sure to end 
the handler with **@callsuper()** so the thread exits properly.

### B.5 Synchronizing Threads

Because GEOS is a preemptive multitasking environment, it needs a way 
to prevent two threads from accessing system resources (or other shared 
data) simultaneously. Otherwise, data could be corrupted. GEOS uses 
semaphores to prevent to threads from performing conflicting operations at 
the same time.

#### B.5.1 Semaphores: The Concept

A semaphore is a data structure on which three basic operations are 
performed. These operations allow threads to avoid conflicting with other 
threads. Think of a semaphore as a flag which programs can set to indicate 
that some resource is locked. Anyone else who wants to use the resource 
must wait in line until whoever set the flag resets it. The three basic 
operations on a semaphore are initialize, set, and reset.

##### B.5.1.1 Initialize

The "initialize" operation creates a semaphore and gives it a name. In its 
initial state, the semaphore is "unlocked," meaning the first process that 
attempts to access it will succeed right away. A semaphore must be 
initialized before it can be used, although the initialization can be handled 
by the operating system so that it is transparent to the programmer.

##### B.5.1.2 Set (the "P" Operation)

The "P" operation is what a program performs in order to make sure it is 
allowed to proceed. For example, if the program is about to access shared 
data, it performs the "P" operation on the semaphore protecting that data 
to make sure no other program is accessing it.

If the semaphore is unlocked and a thread performs the "P" operation on it, 
the thread simply marks the semaphore locked and proceeds normally. If 
the semaphore is locked, a thread performing the "P" operation will block. 
This means the thread will stop running and will wait in the thread queue 
associated with the semaphore. When its turn to perform the protected 
operation arrives, the thread will proceed.

##### B.5.1.3 Reset (the "V" Operation)

When a thread has finished a protected operation, it performs the "V" 
operation to unlock the semaphore. If there are other threads in the queue 
for this semaphore, one of them is restarted and takes over, keeping the 
semaphore locked. Thus only one thread at a time runs the protected 
operation. If a thread performs the "V" operation on a semaphore and there 
are no other threads waiting in the semaphore's queue, the thread simply 
marks the semaphore unlocked and proceeds.

Programs must always reset a semaphore when they are done with it. If 
you fail to reset a semaphore, other threads may wait forever.

Because only one thread at a time is performing the protected operation 
and this thread is responsible for unlocking the semaphore, it is sometimes 
said to "have" the semaphore. The "P" and "V" operations are often referred 
to as "grabbing" and "releasing" a semaphore, respectively.

##### B.5.1.4 The Dreaded Deadlock Problem

When semaphores are not used carefully, they can cause programs to stop 
running entirely. Suppose Thread A tries to grab a semaphore which 
Thread B has locked. Thread A stops running until Thread B releases the 
semaphore. Thread B then tries to grab a second semaphore, which Thread 
A has previously locked. Thread B waits for Thread A to release the second 
semaphore, but Thread A is waiting for Thread B to release the first 
semaphore. Neither thread will ever wake up. This situation is called 
"deadlock."

To avoid deadlock, follow these guidelines:

+ When possible, avoid having a thread attempt to lock one semaphore 
while it already has another one locked.

+ When two or more semaphores may be locked by the same thread at the 
same time, they should always be used in the same order. Semaphores 
are often arranged in a hierarchy, with the coarsest (the one controlling 
access to the most resources) at the top and the finest at the bottom. 
Any thread grabbing multiple semaphores in this hierarchy must 
always grab from top to bottom; that is, no thread should grab a 
semaphore "above" one it already has locked.

+ In certain situations, a semaphore is used "between" two threads. One 
thread needs to wait until another performs a specific action. The first 
thread is said to "block" on the second. Of course, two threads must 
never block on each other. To ensure this situation never arises, only 
one of the threads should use the **@call** keyword when sending 
messages to the other; the other should always use **@send** and, when 
getting return information, have some sort of notification message sent 
in response.

+ When using the GEOS messaging system to send a message with **@call** 
to an object in another thread, the sending thread automatically blocks 
on the receiving thread. Since a number of user interface objects must 
be sent messages with **@call**, the application thread sometimes blocks 
on the user interface thread. To avoid deadlock, code that runs in the 
user interface thread must never send messages with **@call** to objects 
in the application thread. (This is a particular example of the above 
rule being implemented.)

#### B.5.2 Semaphores In GEOS

GEOS contains several system resources that are protected by semaphores. 
Since application programs can access these resources only through library 
routines, the programmer does not need to be aware of these semaphores; 
the required operations are performed by the library routines. For system 
resources (e.g. files, memory, handles), GEOS defines semaphores and 
provides special routines to set and reset them. The chapter that describes 
each system resource explains how to use the special semaphores that 
protect the resource.

The routines described in this section illustrate GEOS semaphores and can 
be used to create semaphores to protect resources defined within a 
multithreaded application. There are routines for each of the operations 
(initialization, P, and V), and there are special routines which simplify the 
use of a semaphore by multiple objects within the same thread.

Note that it is possible to create a semaphore with a starting value greater 
than one. That is, you can create a semaphore that will allow more than one 
thread to grab it at once. Typically, if only one thread may grab the 
semaphore, the thread is called a "mutual exclusion," or "mutex," 
semaphore, because it is normally used to make sure two threads don't 
mutually grab a particular resource.

Semaphores that can be grabbed by more than one thread are generally 
called "scheduling semaphores" because they allow easy manipulation of 
scheduled resources. The classic example of this is the "producer-consumer 
problem" wherein one thread produces buffers and another consumes 
them. Initially, no buffers exist, so the semaphore starts at zero. The 
consumer goes into a loop wherein it simply P's the semaphore (blocking 
until a buffer exists), takes the first buffer in the queue, processes the 
buffer, destroys the buffer, and then returns to the top of the loop. The 
producer, meanwhile, can produce any number of buffers, queue them, and 
V the semaphore once for each consumable buffer. The consumer thread 
will continue to process until all the buffers are consumed, and then it will 
block and wait for more buffers.

##### B.5.2.1 Operations on a Semaphore

ThreadAllocSem(), ThreadPSem(), ThreadPTimedSem(), 
ThreadVSem(), ThreadFreeSem()

To create a semaphore, simply call the routine **ThreadAllocSem()**, 
passing an initial value for the semaphore. This should normally be one to 
indicate the semaphore is unlocked. If you want the semaphore to be locked 
initially, pass an initial value of zero. In either case, the returned value will 
be the handle of the newly created semaphore. Use this handle with the 
routines described below.

Once a semaphore is created, a thread can lock it (i.e., perform the "P" 
operation) by calling the routine **ThreadPSem()**, passing the semaphore's 
handle as an argument. If the semaphore is unlocked, the thread locks it 
and proceeds; otherwise the thread waits in the semaphore's thread queue.

Another routine that performs the "P" operation is **ThreadPTimedSem()**. 
When calling this routine, pass as arguments the semaphore's handle and 
an integer representing a number of ticks. This integer is a time limit: If 
another thread has the semaphore locked and does not unlock it within the 
specified number of ticks, the routine will return with a flag indicating the 
lock was unsuccessful. Programs that use **ThreadPTimedSem()** must 
check this flag and must not perform the protected operation if it is set. The 
most common use of this routine is with a time limit of zero, meaning that 
the semaphore should be locked only if it is available right away-if it is 
not available, the thread will continue with some other action.

To release the semaphore (by performing the "V" operation), the thread 
calls **ThreadVSem()**, again passing the semaphore's handle. If there are 
other threads waiting for the semaphore, the one with the lowest current 
priority number takes over.

When a semaphore is no longer needed, it can be destroyed by calling 
**ThreadFreeSem()** with the semaphore's handle as an argument.

##### B.5.2.2 Operations on a Thread Lock

ThreadAllocThreadLock(), ThreadGrabThreadLock(), 
ThreadReleaseThreadLock(), ThreadFreeThreadLock()

At times it is convenient to have a program lock a semaphore that it has 
already locked. For example, one routine might lock a semaphore 
protecting a piece of memory and then call itself recursively. A thread lock 
is a semaphore that can be locked any number of times, as long as each lock 
is performed by the same thread. If another thread tries to grab the thread 
lock, it will wait until the first thread has performed the "V" operation once 
for each time it has run the "P" operation. It is possible to write reentrant 
routines using thread locks but not using regular semaphores.

A thread lock is initialized with the **ThreadAllocThreadLock()** routine, 
which takes no arguments. A thread lock is always created unlocked. To 
perform the "P" and "V" operation on a thread lock, use 
**ThreadGrabThreadLock()** and **ThreadReleaseThreadLock()**, 
respectively, and pass the semaphore's handle as an argument. These 
routines are analogous to **ThreadPSem()** and **ThreadVSem()** for 
semaphores. When a thread lock is no longer needed, it should be freed 
with a call to **ThreadFreeThreadLock()**.

[Machine Architecture](chardw.md) <-- &nbsp;&nbsp; [table of contents](../concepts.md) &nbsp;&nbsp; --> [Libraries](clibr.md)
