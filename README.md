# process-manager

# Overview

This is a process manager capable of simulating allocating memory to processes and scheduling them for execution. In the first phase, a scheduling algorithm will allocate CPU to a process. In the second phase, a memory allocation algorithm will be used to allocate memory to processes before the scheduling takes place. Process scheduling is based on a scheduling algorithm with the assumption of a single CPU.

# Process Manager

The manage runs in cycles, which occurs after one quantum. This simulation time increases by the length of the quantum every cycle. At any given cycle, the simulation time is equal to the number of completed cycles multiplied by the length of the quantum: $T_s = N \cdot Q$, where $T_s$ is the current simulation time, $N$ is the number of cycles elapsed, and $Q$ is the length of the quantum. $Q$ is arbitrary, but I chose to limit it to an integer value between 1 and 3 inclusive.

At the start of each cycle, the manager carries out the tasks in sequence:

1. Identify if the currently running process (if any) has been completed. If so, it should be terminated and memory deallocated before subsequent scheduling tasks are performed.
2. Identify all processes submitted to the system since the last cycle occurred and add them to the input queue in the order they appear in the process file. A process is considered to have been submitted if its arrival time is less than or equal to than current simulation time $T_s$.
3. Move processes from the input queue to the ready queue upon successful memory allocation. The maximum simulated memory is 2048 MB. The ready queue holds processes that are ready to run. The manager will use one of the methods to allocate memory to processes:
	- Infinite memory: Assume memory requirements of the arrived processes are always immediately satisfied. All arrived processes will automatically enter a READY state and be placed at the end of the ready queue in the same order as they appear in the input queue
	- Allocate memory to processes in the input queue according to a memory allocation algorithm. Processes for which memory allocation is successful enter a READY state and are placed at the end of the ready queue in order of memory allocation to await CPU time.
4. Determine the process (if any) to run in this cycle depending on the scheduling algorithm.

## Process Lifecycle and States

1. Processes are submitted to the manager via an input file. The processes are read into a data structure to determine which processes should be added to the input queue based on their arrival time and current simulation time.
2. The process is in a READY state after it has arrived (arrival time $<=$ simulation time) and memory has been successfully allocated to it.
3. Once a process is READY, it can be considered by a process scheduling algorithm as a candidate to enter the RUNNING state - being allocated to the CPU
4. Once a process is in the RUNNING state it runs on the CPU for at least one quantum. If the scheduling algorithm decides to allocate CPU time to a different process, the RUNNING process is suspended and placed back in to the ready queue, and the state switched from RUNNING to READY
5. The total time a process gets to run on the CPU (RUNNING state) must be equal to or greater than its service time. The service time of a process is the amount of CPU time (not wall time) it requires before completion
6. After the service time of a process has elapsed, the process moves to the FINISHED state, and terminates and has its memory deallocated.


## Non pre-emptive process scheduling

This uses the Shortest Job First algorithm to select the next process to run among the ready processes. The process with the shortest service time is chosen among all ready processes, and the process is allowed to run for the entire duration of its service time without getting suspended or terminated.

After one quantum has elapsed:

- If the process has finished execution, the process is terminated and moves to the FINISHED state.
- If the process requires more CPU time, the process runs for another quantum.

## Pre-emptive process scheduling

This uses a round robin strategy and the process at the head of the ready queue is chosen to run for one quantum. After it has run for one quantum, it is suspended and placed at the end of the ready queue.

After one quantum has elapsed:

- If the process has completed its execution, the process is terminated and moves to the FINISHED state.
- If the process requires more CPU time and there are other READY processes, the process is suspended and enters the READY state again to await more CPU time.
- If the process requires more CPU time and there are no other READY processes, the process can continue to run for another quantum.

## Memory allocation

If memory is required to allocate processes, the manager simulates the allocation of memory before the scheduling phase. The manager only keeps track and reports simulated memory addresses.

At each cycle, the program attempts to allocate memory to all the processes in the input queue and move successfully allocated processes to the ready queue. Processes for which memory allocation cannot be currently met should remain in the input queue.

Processes whose memory allocation has succeeded are considered READY to run. 

My implementation uses the Best Fit algorithm with doubly linked lists. Adjacent empty holes are merged after a process finishes running.

- A total of 2048 MB is available to allocate to user processes.
- The memory requirement of each process is known in advance and static.
- A processes memory is allocated in its entirety and in a contiguous block. The allocation remains for the duration of the process' runtime.
- Once a process terminates, its memory is freed and merged into adjacent holes.

# Usage

The arguments can be passed in any order but must be passed correctly and only once.

`./allocate -f <filename> -s (SJF | RR) -m (infinite | best-fit) -q (1 | 2 | 3)`

- `-f filename` specifies a valid relative or absolute path to the input file describing the processes.
- `-s scheduler` where scheduler is one of {SJF, RR}.
- `-m memory-strategy` where memory-strategy is one of {infinite, best-fit}.
- `-q quantum` where quantum is one of {1, 2, 3}.

Example: `./allocate -f processes.txt -s RR -m best-fit -q 3`

This program simulates the execution of processes in the `processes.txt` using Round Robin  for scheduling and Best Fit for memory allocation, with a quantum of 3 seconds.

Given `processes.txt` with the following information:

```
0 P4 30 16
29 P2 40 64
99 P1 20 32
```

The program should simulate the execution of 3 processes where process P4 arrives at time 0, needs 30 seconds of CPU time to finish, and requires 16 MB of memory; process P2 arrives at time 29, needs 40 seconds of time to complete and requires 64 MB of memory, etc.

The expected output is:

```
0,READY,process_name=P1,assigned_at=0
0,RUNNING,process_name=P1,remaining_time=50
12,READY,process_name=P2,assigned_at=8
12,RUNNING,process_name=P2,remaining_time=30
15,RUNNING,process_name=P1,remaining_time=38
18,RUNNING,process_name=P2,remaining_time=27
21,RUNNING,process_name=P1,remaining_time=35
24,RUNNING,process_name=P2,remaining_time=24
27,RUNNING,process_name=P1,remaining_time=32
30,RUNNING,process_name=P2,remaining_time=21
33,RUNNING,process_name=P1,remaining_time=29
36,RUNNING,process_name=P2,remaining_time=18
39,RUNNING,process_name=P1,remaining_time=26
42,READY,process_name=P5,assigned_at=16
42,RUNNING,process_name=P2,remaining_time=15
45,RUNNING,process_name=P5,remaining_time=80
48,RUNNING,process_name=P1,remaining_time=23
51,READY,process_name=P4,assigned_at=24
51,RUNNING,process_name=P2,remaining_time=12
54,RUNNING,process_name=P5,remaining_time=77
57,RUNNING,process_name=P4,remaining_time=10
60,RUNNING,process_name=P1,remaining_time=20
63,RUNNING,process_name=P2,remaining_time=9
66,RUNNING,process_name=P5,remaining_time=74
69,RUNNING,process_name=P4,remaining_time=7
72,RUNNING,process_name=P1,remaining_time=17
75,RUNNING,process_name=P2,remaining_time=6
78,RUNNING,process_name=P5,remaining_time=71
81,RUNNING,process_name=P4,remaining_time=4
84,RUNNING,process_name=P1,remaining_time=14
87,RUNNING,process_name=P2,remaining_time=3
90,FINISHED,process_name=P2,proc_remaining=3
90,RUNNING,process_name=P5,remaining_time=68
93,RUNNING,process_name=P4,remaining_time=1
96,FINISHED,process_name=P4,proc_remaining=2
96,RUNNING,process_name=P1,remaining_time=11
99,RUNNING,process_name=P5,remaining_time=65
102,RUNNING,process_name=P1,remaining_time=8
105,RUNNING,process_name=P5,remaining_time=62
108,RUNNING,process_name=P1,remaining_time=5
111,RUNNING,process_name=P5,remaining_time=59
114,RUNNING,process_name=P1,remaining_time=2
117,FINISHED,process_name=P1,proc_remaining=1
117,RUNNING,process_name=P5,remaining_time=56
174,FINISHED,process_name=P5,proc_remaining=0
Turnaround time 95
Time overhead 4.60 2.82
Makespan 174
```

When the simulation is completed, the performance statistics is printed.

- Turnaround time: average time between the time when the process is completed and when it arrived.
- Time overhead: maximum and average time overhead when running a process. Overhead time is defined as the turnaround time of the process divided by its service time.
- Makespan: the simulation time of when the simulated ended.

