```python
#!/usr/bin/env python3
from dataclasses import dataclass
from enum import Enum
from typing import List, Dict, Optional, Set
from tabulate import tabulate
import argparse
import time

class ExecutionModel(Enum):
    REQUEST_PER_THREAD = "rpt"
    REACTIVE = "reactive"

class TaskState(Enum):
    READY = "Ready"
    RUNNING = "Running"
    IO = "IO"
    COMPLETED = "Completed"

class TaskType(Enum):
    IO = "io"
    COMPUTE = "compute"

@dataclass
class Task:
    id: int
    task_type: TaskType
    compute_time: int
    io_time: int = 0
    state: TaskState = TaskState.READY
    thread_id: Optional[int] = None
    compute_time_remaining: int = 0
    io_time_remaining: int = 0
    io_completion_time: Optional[int] = None
    core: Optional[int] = None
    is_first_compute_phase: bool = True  # Track which compute phase we're in

    def __init__(self, task_id: str, task_type: str, thread_id: Optional[str] = None):
        self.id = task_id
        self.task_type = TaskType(task_type)
        self.thread_id = int(thread_id) if thread_id else None
        self.state = TaskState.READY
        self.io_completion_time = None
        self.core = None
        self.is_first_compute_phase = True

        if self.task_type == TaskType.IO:
            # For IO tasks: 1s compute + 4s IO + 1s compute
            self.compute_time = 1  # Per-phase compute time
            self.io_time = 4
            self.compute_time_remaining = 1  # Start with first phase
            self.io_time_remaining = 4
        else:
            # For compute tasks: 6s compute
            self.compute_time = 6
            self.io_time = 0
            self.compute_time_remaining = 6
            self.io_time_remaining = 0

    def start_io_phase(self):
        """Transition from compute to IO phase"""
        if self.task_type == TaskType.IO and self.is_first_compute_phase:
            self.is_first_compute_phase = False
            self.compute_time_remaining = 1  # Set up second compute phase
            return True
        return False

    def finish_compute(self):
        """Check if we should transition to IO or complete"""
        if self.task_type == TaskType.IO:
            if self.is_first_compute_phase:
                return "io"  # Go to IO phase
            return "complete"  # Second phase done, complete task
        return "complete"  # Compute task done

    @classmethod
    def create_task(cls, task_id: str, task_type: str, thread_id: Optional[str] = None) -> 'Task':
        return cls(task_id, task_type, thread_id)

    def get_status(self) -> str:
        thread_str = f"t{self.thread_id}" if self.thread_id is not None else "None"
        core_str = f"Core{self.core + 1}" if self.core is not None else "None"
        return (f"Task {self.id}: State={self.state.value}, Thread={thread_str}, Core={core_str}, "
                f"Compute Left={self.compute_time_remaining}, IO Left={self.io_time_remaining}")

class Scheduler:
    def __init__(self, num_cores: int, execution_model: ExecutionModel):
        self.num_cores = num_cores
        self.execution_model = execution_model
        self.tasks: List[Task] = []
        self.io_queue: List[Task] = []
        self.current_time = 0
        self.available_cores = set(range(num_cores))
        self.next_thread_id = 1
        self.cpu_time = 0
        self.core_assignments = {i: None for i in range(num_cores)}
        # Track history of assignments for context switch calculation
        self.core_thread_history = []  # List of (time, core, thread_id)
        self.thread_task_history = []  # List of (time, thread_id, task_id)

        if execution_model == ExecutionModel.REACTIVE:
            self.max_thread_id = num_cores
            self.thread_assignments = {i+1: None for i in range(num_cores)}
        else:
            self.max_thread_id = None
            self.thread_assignments = None

    def add_task(self, compute_time: int, io_time: int):
        task_id = len(self.tasks) + 1
        thread_id = None
        if self.execution_model == ExecutionModel.REQUEST_PER_THREAD:
            thread_id = str(self.next_thread_id)
            self.next_thread_id += 1
        task = Task.create_task(str(task_id), 'io' if io_time > 0 else 'compute', thread_id)
        self.tasks.append(task)

    def start_io(self, task: Task):
        if task.state != TaskState.IO:
            task.state = TaskState.IO
            task.io_completion_time = self.current_time + task.io_time_remaining
            if task.core is not None:
                # Record thread release for both modes
                self.core_thread_history.append((self.current_time, task.core, None))
                self.available_cores.add(task.core)
                self.core_assignments[task.core] = None
                task.core = None
            if task not in self.io_queue:
                self.io_queue.append(task)
            # Record thread release for REACTIVE mode
            if self.execution_model == ExecutionModel.REACTIVE and task.thread_id is not None:
                self.thread_task_history.append((self.current_time, task.thread_id, None))
                self.thread_assignments[task.thread_id] = None
                task.thread_id = None

    def check_io_completion(self):
        completed_io = []
        for task in self.io_queue:
            if task.io_completion_time <= self.current_time:
                task.io_time_remaining = 0
                task.state = TaskState.READY
                completed_io.append(task)

        for task in completed_io:
            self.io_queue.remove(task)

    def assign_cores(self):
        available_cores = []
        for core in range(self.num_cores):
            if self.core_assignments[core] is None:
                available_cores.append(core)

        if not available_cores:
            return

        ready_tasks = [task for task in self.tasks if task.state == TaskState.READY]
        if not ready_tasks:
            return

        for core in available_cores:
            if not ready_tasks:
                break

            task = ready_tasks.pop(0)
            self.core_assignments[core] = task
            task.state = TaskState.RUNNING
            task.core = core

            if self.execution_model == ExecutionModel.REACTIVE:
                # In REACTIVE mode, thread is based on core
                thread_id = core + 1
                self.thread_assignments[thread_id] = task.id
                task.thread_id = thread_id
                # Record new assignments
                self.thread_task_history.append((self.current_time, thread_id, task.id))
                self.core_thread_history.append((self.current_time, core, thread_id))
            else:
                # In RPT mode, task already has its thread, record core taking that thread
                self.core_thread_history.append((self.current_time, core, int(task.thread_id)))

    def calculate_context_switches(self):
        cpu_switches = 0
        thread_switches = 0

        if self.execution_model == ExecutionModel.REQUEST_PER_THREAD:
            # For RPT model, count all core-to-thread changes
            core_thread_map = {}  # Keep track of last thread for each core
            for time, core, thread_id in self.core_thread_history:
                if core not in core_thread_map:
                    core_thread_map[core] = None  # Initialize with no thread

                if core_thread_map[core] != thread_id:
                    # Count switch if either:
                    # 1. Going from no thread to a thread (None -> t1)
                    # 2. Going from one thread to another (t1 -> t3)
                    # 3. Going from a thread to no thread (t1 -> None)
                    cpu_switches += 1
                    core_thread_map[core] = thread_id
        else:
            # For REACTIVE model, each core has a fixed thread ID (core + 1)
            # Only count when a core gets its first thread and when it releases its last thread
            core_first_thread = {}  # First thread seen for each core
            core_last_thread = {}   # Last thread seen for each core

            # Find first and last thread assignments for each core
            for time, core, thread_id in self.core_thread_history:
                if thread_id is not None:
                    if core not in core_first_thread:
                        core_first_thread[core] = thread_id
                    core_last_thread[core] = thread_id

            # Count switches: one for getting first thread, one for releasing last thread
            for core in core_first_thread:
                cpu_switches += 2  # One for start, one for end

        # Calculate thread context switches (thread-to-task changes)
        thread_task_map = {}  # Keep track of last task for each thread
        for time, thread_id, task_id in self.thread_task_history:
            if thread_id not in thread_task_map:
                thread_task_map[thread_id] = task_id
            elif thread_task_map[thread_id] != task_id:
                thread_switches += 1
                thread_task_map[thread_id] = task_id

        return cpu_switches, thread_switches

    def run(self):
        print(f"\nStarting simulation with {self.num_cores} cores and {len(self.tasks)} tasks")
        print(f"Execution Model: {self.execution_model.name}")
        print("\nTask Types:")
        print("IO     : 1s compute + 4s IO + 1s compute")
        print("Compute: 6s compute")

        def get_table_data():
            headers = ["Task ID", "State", "Thread", "Core", "Compute Left", "IO Left"]
            rows = []
            for task in self.tasks:
                thread_str = "-"
                core_str = "-"

                # For running tasks in REACTIVE mode, look up thread and core
                if task.state == TaskState.RUNNING:
                    if task.thread_id is not None:
                        thread_str = f"t{task.thread_id}"
                    elif self.execution_model == ExecutionModel.REACTIVE:
                        for thread_id, task_id in self.thread_assignments.items():
                            if task_id == task.id:
                                thread_str = f"t{thread_id}"
                                break

                    if task.core is not None:
                        core_str = f"Core{task.core + 1}"
                    else:
                        # Look up core from core_assignments
                        for core, assigned_task in self.core_assignments.items():
                            if assigned_task and assigned_task.id == task.id:
                                core_str = f"Core{core + 1}"
                                break

                # For tasks in IO, calculate remaining IO time
                io_remaining = task.io_time_remaining
                if task.state == TaskState.IO and task.io_completion_time is not None:
                    io_remaining = max(0, task.io_completion_time - self.current_time)

                rows.append([
                    f"Task {task.id}",
                    task.state.value,
                    thread_str,
                    core_str,
                    task.compute_time_remaining,
                    io_remaining
                ])
            return headers, rows

        print("\nInitial state:")
        headers, rows = get_table_data()
        print(tabulate(rows, headers=headers, tablefmt="grid"))

        while not all(task.state == TaskState.COMPLETED for task in self.tasks):
            self.current_time += 1
            print(f"\nTime: {self.current_time}")

            # Process running tasks
            for task in self.tasks:
                if task.state == TaskState.RUNNING:
                    task.compute_time_remaining -= 1
                    self.cpu_time += 1

                    if task.compute_time_remaining <= 0:
                        next_state = task.finish_compute()
                        if next_state == "io":
                            task.start_io_phase()  # Set up second compute phase
                            self.start_io(task)
                        else:  # complete
                            task.state = TaskState.COMPLETED
                            if task.core is not None:
                                self.available_cores.add(task.core)
                                self.core_assignments[task.core] = None
                                task.core = None

            # Check IO completion
            self.check_io_completion()

            # Assign available cores
            self.assign_cores()

            # Print current state in table format
            headers, rows = get_table_data()
            print(tabulate(rows, headers=headers, tablefmt="grid"))

            # Break if no progress is being made
            if not any(t.state in [TaskState.RUNNING, TaskState.IO] for t in self.tasks) and \
               any(t.state != TaskState.COMPLETED for t in self.tasks):
                print("\nDeadlock detected - no tasks can make progress!")
                break

        total_time = self.current_time
        cpu_utilization = (self.cpu_time / ((total_time - 1) * self.num_cores)) * 100 if total_time > 1 else 0
        cpu_switches, thread_switches = self.calculate_context_switches()

        print(f"\nSimulation completed in {total_time} seconds")
        print(f"Average CPU utilization: {cpu_utilization:.1f}%")
        print(f"CPU context switches: {cpu_switches}")
        print(f"Thread context switches: {thread_switches}")

def main():
    parser = argparse.ArgumentParser(description='Task Scheduler Simulator')
    parser.add_argument('--cores', type=int, default=2,
                      help='Number of CPU cores (default: 2)')
    parser.add_argument('--tasks', type=int, default=4,
                      help='Number of tasks to simulate (default: 4)')
    parser.add_argument('--type', type=str, choices=['io', 'compute'], default='io',
                      help='Type of tasks to simulate: "io" (1s compute + 4s IO + 1s compute) or "compute" (6s compute)')
    parser.add_argument('--model', type=str, choices=['rpt', 'reactive'], default='rpt',
                      help='Execution model: "rpt" (Request Per Thread) or "reactive"')
    args = parser.parse_args()

    scheduler = Scheduler(num_cores=args.cores, execution_model=ExecutionModel(args.model))

    # Add tasks based on type
    for _ in range(args.tasks):
        if args.type == 'io':
            scheduler.add_task(compute_time=2, io_time=3)
        else:  # compute
            scheduler.add_task(compute_time=6, io_time=0)

    scheduler.run()

if __name__ == "__main__":
    main()
```


Commands to run : 

```shell
python3 io_task_simulator.py --cores 2 --tasks 4 --type io --model reactive
```
