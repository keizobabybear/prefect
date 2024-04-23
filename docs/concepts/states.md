---
description: Prefect states contain information about the status of a flow or task run.
tags:
    - orchestration
    - flow runs
    - task runs
    - states
    - status
    - state change hooks
    - triggers
search:
  boost: 2
---

# States

## Overview

States are rich objects that contain information about the status of a particular [task](/concepts/tasks) run or [flow](/concepts/flows/) run. While you don't need to know the details of the states to use Prefect, you can give your workflows superpowers by taking advantage of it.

At any moment, you can learn anything you need to know about a task or flow by examining its current state or the history of its states. For example, a state could tell you that a task:


- is scheduled to make a third run attempt in an hour

- succeeded and what data it produced

- was scheduled to run, but later cancelled

- used the cached result of a previous run instead of re-running

- failed because it timed out


By manipulating a relatively small number of task states, Prefect flows can harness the complexity that emerges in workflows.

!!! note "Only runs have states"
    Though we often refer to the "state" of a flow or a task, what we really mean is the state of a flow _run_ or a task _run_. Flows and tasks are templates that describe what a system does; only when we run the system does it also take on a state. So while we might refer to a task as "running" or being "successful", we really mean that a specific instance of the task is in that state.

The final state of the flow is determined by its return value. The following rules apply:

- If an exception is raised directly in the flow function, the flow run is marked as failed.
- If the flow does not return a value (or returns `None`), its state is determined by the states of all of the tasks and subflows within it.
  - If _any_ task run or subflow run failed, then the final flow run state is marked as `FAILED`.
  - If _any_ task run was cancelled, then the final flow run state is marked as `CANCELLED`.
- If a flow returns a manually created state, it is used as the state of the final flow run. This allows for manual determination of final state.
- If the flow run returns _any other object_, then it is marked as completed.

The following examples illustrate each of these cases:

### Raise an exception

If an exception is raised within the flow function, the flow is immediately marked as failed.

```python hl_lines="5"
from prefect import flow

@flow
def always_fails_flow():
    raise ValueError("This flow immediately fails")

if __name__ == "__main__":
    always_fails_flow()
```

Running this flow produces the following result:

<div class="terminal">
```bash
22:22:36.864 | INFO    | prefect.engine - Created flow run 'acrid-tuatara' for flow 'always-fails-flow'
22:22:36.864 | INFO    | Flow run 'acrid-tuatara' - Starting 'ConcurrentTaskRunner'; submitted tasks will be run concurrently...
22:22:37.060 | ERROR   | Flow run 'acrid-tuatara' - Encountered exception during execution:
Traceback (most recent call last):...
ValueError: This flow immediately fails
```
</div>

### Return `none`

A flow with no return statement is determined by the state of all of its task runs.

```python
from prefect import flow, task

@task
def always_fails_task():
    raise ValueError("I fail successfully")

@task
def always_succeeds_task():
    print("I'm fail safe!")
    return "success"

@flow
def always_fails_flow():
    always_fails_task.submit().result(raise_on_failure=False)
    always_succeeds_task()

if __name__ == "__main__":
    always_fails_flow()
```

Running this flow produces the following result:

<div class="terminal">
```bash
18:32:05.345 | INFO    | prefect.engine - Created flow run 'auburn-lionfish' for flow 'always-fails-flow'
18:32:05.346 | INFO    | Flow run 'auburn-lionfish' - Starting 'ConcurrentTaskRunner'; submitted tasks will be run concurrently...
18:32:05.582 | INFO    | Flow run 'auburn-lionfish' - Created task run 'always_fails_task-96e4be14-0' for task 'always_fails_task'
18:32:05.582 | INFO    | Flow run 'auburn-lionfish' - Submitted task run 'always_fails_task-96e4be14-0' for execution.
18:32:05.610 | ERROR   | Task run 'always_fails_task-96e4be14-0' - Encountered exception during execution:
Traceback (most recent call last):
  ...
ValueError: I fail successfully
18:32:05.638 | ERROR   | Task run 'always_fails_task-96e4be14-0' - Finished in state Failed('Task run encountered an exception.')
18:32:05.658 | INFO    | Flow run 'auburn-lionfish' - Created task run 'always_succeeds_task-9c27db32-0' for task 'always_succeeds_task'
18:32:05.659 | INFO    | Flow run 'auburn-lionfish' - Executing 'always_succeeds_task-9c27db32-0' immediately...
I'm fail safe!
18:32:05.703 | INFO    | Task run 'always_succeeds_task-9c27db32-0' - Finished in state Completed()
18:32:05.730 | ERROR   | Flow run 'auburn-lionfish' - Finished in state Failed('1/2 states failed.')
Traceback (most recent call last):
  ...
ValueError: I fail successfully
```
</div>

### Return a future

If a flow returns one or more futures, the final state is determined based on the underlying states.

```python hl_lines="15"
from prefect import flow, task

@task
def always_fails_task():
    raise ValueError("I fail successfully")

@task
def always_succeeds_task():
    print("I'm fail safe!")
    return "success"

@flow
def always_succeeds_flow():
    x = always_fails_task.submit().result(raise_on_failure=False)
    y = always_succeeds_task.submit(wait_for=[x])
    return y

if __name__ == "__main__":
    always_succeeds_flow()
```

Running this flow produces the following result &mdash; it succeeds because it returns the future of the task that succeeds:

<div class="terminal">
```bash
18:35:24.965 | INFO    | prefect.engine - Created flow run 'whispering-guan' for flow 'always-succeeds-flow'
18:35:24.965 | INFO    | Flow run 'whispering-guan' - Starting 'ConcurrentTaskRunner'; submitted tasks will be run concurrently...
18:35:25.204 | INFO    | Flow run 'whispering-guan' - Created task run 'always_fails_task-96e4be14-0' for task 'always_fails_task'
18:35:25.205 | INFO    | Flow run 'whispering-guan' - Submitted task run 'always_fails_task-96e4be14-0' for execution.
18:35:25.232 | ERROR   | Task run 'always_fails_task-96e4be14-0' - Encountered exception during execution:
Traceback (most recent call last):
  ...
ValueError: I fail successfully
18:35:25.265 | ERROR   | Task run 'always_fails_task-96e4be14-0' - Finished in state Failed('Task run encountered an exception.')
18:35:25.289 | INFO    | Flow run 'whispering-guan' - Created task run 'always_succeeds_task-9c27db32-0' for task 'always_succeeds_task'
18:35:25.289 | INFO    | Flow run 'whispering-guan' - Submitted task run 'always_succeeds_task-9c27db32-0' for execution.
I'm fail safe!
18:35:25.335 | INFO    | Task run 'always_succeeds_task-9c27db32-0' - Finished in state Completed()
18:35:25.362 | INFO    | Flow run 'whispering-guan' - Finished in state Completed('All states completed.')
```
</div>

### Return multiple states or futures

If a flow returns a mix of futures and states, the final state is determined by resolving all futures to states, then determining if any of the states are not `COMPLETED`.

```python hl_lines="20"
from prefect import task, flow

@task
def always_fails_task():
    raise ValueError("I am bad task")

@task
def always_succeeds_task():
    return "foo"

@flow
def always_succeeds_flow():
    return "bar"

@flow
def always_fails_flow():
    x = always_fails_task()
    y = always_succeeds_task()
    z = always_succeeds_flow()
    return x, y, z
```

Running this flow produces the following result.
It fails because one of the three returned futures failed.
Note that the final state is `Failed`, but the states of each of the returned futures is included in the flow state:

<div class="terminal">
```bash
20:57:51.547 | INFO    | prefect.engine - Created flow run 'impartial-gorilla' for flow 'always-fails-flow'
20:57:51.548 | INFO    | Flow run 'impartial-gorilla' - Using task runner 'ConcurrentTaskRunner'
20:57:51.645 | INFO    | Flow run 'impartial-gorilla' - Created task run 'always_fails_task-58ea43a6-0' for task 'always_fails_task'
20:57:51.686 | INFO    | Flow run 'impartial-gorilla' - Created task run 'always_succeeds_task-c9014725-0' for task 'always_succeeds_task'
20:57:51.727 | ERROR   | Task run 'always_fails_task-58ea43a6-0' - Encountered exception during execution:
Traceback (most recent call last):...
ValueError: I am bad task
20:57:51.787 | INFO    | Task run 'always_succeeds_task-c9014725-0' - Finished in state Completed()
20:57:51.808 | INFO    | Flow run 'impartial-gorilla' - Created subflow run 'unbiased-firefly' for flow 'always-succeeds-flow'
20:57:51.884 | ERROR   | Task run 'always_fails_task-58ea43a6-0' - Finished in state Failed('Task run encountered an exception.')
20:57:52.438 | INFO    | Flow run 'unbiased-firefly' - Finished in state Completed()
20:57:52.811 | ERROR   | Flow run 'impartial-gorilla' - Finished in state Failed('1/3 states failed.')
Failed(message='1/3 states failed.', type=FAILED, result=(Failed(message='Task run encountered an exception.', type=FAILED, result=ValueError('I am bad task'), task_run_id=5fd4c697-7c4c-440d-8ebc-dd9c5bbf2245), Completed(message=None, type=COMPLETED, result='foo', task_run_id=df9b6256-f8ac-457c-ba69-0638ac9b9367), Completed(message=None, type=COMPLETED, result='bar', task_run_id=cfdbf4f1-dccd-4816-8d0f-128750017d0c)), flow_run_id=6d2ec094-001a-4cb0-a24e-d2051db6318d)
```
</div>

!!! note "Returning multiple states"
    When returning multiple states, they must be contained in a `set`, `list`, or `tuple`.
    If other collection types are used, the result of the contained states will not be checked.

### Return a manual state

If a flow returns a manually created state, the final state is determined based on the return value.

```python hl_lines="16-19"
from prefect import task, flow
from prefect.states import Completed, Failed

@task
def always_fails_task():
    raise ValueError("I fail successfully")

@task
def always_succeeds_task():
    print("I'm fail safe!")
    return "success"

@flow
def always_succeeds_flow():
    x = always_fails_task.submit()
    y = always_succeeds_task.submit()
    if y.result() == "success":
        return Completed(message="I am happy with this result")
    else:
        return Failed(message="How did this happen!?")

if __name__ == "__main__":
    always_succeeds_flow()
```

Running this flow produces the following result.

<div class="terminal">
```bash
18:37:42.844 | INFO    | prefect.engine - Created flow run 'lavender-elk' for flow 'always-succeeds-flow'
18:37:42.845 | INFO    | Flow run 'lavender-elk' - Starting 'ConcurrentTaskRunner'; submitted tasks will be run concurrently...
18:37:43.125 | INFO    | Flow run 'lavender-elk' - Created task run 'always_fails_task-96e4be14-0' for task 'always_fails_task'
18:37:43.126 | INFO    | Flow run 'lavender-elk' - Submitted task run 'always_fails_task-96e4be14-0' for execution.
18:37:43.162 | INFO    | Flow run 'lavender-elk' - Created task run 'always_succeeds_task-9c27db32-0' for task 'always_succeeds_task'
18:37:43.163 | INFO    | Flow run 'lavender-elk' - Submitted task run 'always_succeeds_task-9c27db32-0' for execution.
18:37:43.175 | ERROR   | Task run 'always_fails_task-96e4be14-0' - Encountered exception during execution:
Traceback (most recent call last):
  ...
ValueError: I fail successfully
I'm fail safe!
18:37:43.217 | ERROR   | Task run 'always_fails_task-96e4be14-0' - Finished in state Failed('Task run encountered an exception.')
18:37:43.236 | INFO    | Task run 'always_succeeds_task-9c27db32-0' - Finished in state Completed()
18:37:43.264 | INFO    | Flow run 'lavender-elk' - Finished in state Completed('I am happy with this result')
```
</div>

### Return an object

If the flow run returns _any other object_, then it is marked as completed.

```python hl_lines="10"
from prefect import task, flow

@task
def always_fails_task():
    raise ValueError("I fail successfully")

@flow
def always_succeeds_flow():
    always_fails_task().submit()
    return "foo"

if __name__ == "__main__":
    always_succeeds_flow()
```

Running this flow produces the following result.

<div class="terminal">
```bash
21:02:45.715 | INFO    | prefect.engine - Created flow run 'sparkling-pony' for flow 'always-succeeds-flow'
21:02:45.715 | INFO    | Flow run 'sparkling-pony' - Using task runner 'ConcurrentTaskRunner'
21:02:45.816 | INFO    | Flow run 'sparkling-pony' - Created task run 'always_fails_task-58ea43a6-0' for task 'always_fails_task'
21:02:45.853 | ERROR   | Task run 'always_fails_task-58ea43a6-0' - Encountered exception during execution:
Traceback (most recent call last):...
ValueError: I am bad task
21:02:45.879 | ERROR   | Task run 'always_fails_task-58ea43a6-0' - Finished in state Failed('Task run encountered an exception.')
21:02:46.593 | INFO    | Flow run 'sparkling-pony' - Finished in state Completed()
Completed(message=None, type=COMPLETED, result='foo', flow_run_id=7240e6f5-f0a8-4e00-9440-a7b33fb51153)
```
</div>


## State Types

States have names and types. State types are canonical, with specific orchestration rules that apply to transitions into and out of each state type. A state's name, is often, but not always, synonymous with its type. For example, a task run that is running for the first time has a state with the name Running and the type `RUNNING`. However, if the task retries, that same task run will have the name Retrying and the type `RUNNING`. Each time the task run transitions into the `RUNNING` state, the same orchestration rules are applied.

There are terminal state types from which there are no orchestrated transitions to any other state type.

- `COMPLETED`
- `CANCELLED`
- `FAILED`
- `CRASHED`

The full complement of states and state types includes:
  
| Name | Type | Terminal? | Description
| --- | --- | --- | --- |
| Scheduled | SCHEDULED | No | The run will begin at a particular time in the future. |
| Late | SCHEDULED | No | The run's scheduled start time has passed, but it has not transitioned to PENDING (15 seconds by default). |
| <span class="no-wrap">AwaitingRetry</span> | SCHEDULED | No | The run did not complete successfully because of a code issue and had remaining retry attempts. |
| Pending | PENDING | No | The run has been submitted to run, but is waiting on necessary preconditions to be satisfied. |
| Running | RUNNING | No | The run code is currently executing. |
| Retrying | RUNNING | No | The run code is currently executing after previously not complete successfully. |
| Paused | PAUSED | No | The run code has stopped executing until it receives manual approval to proceed. |
| Cancelling | CANCELLING | No | The infrastructure on which the code was running is being cleaned up. |
| Cancelled | CANCELLED | Yes | The run did not complete because a user determined that it should not. |
| Completed | COMPLETED | Yes | The run completed successfully. |
| Failed | FAILED | Yes | The run did not complete because of a code issue and had no remaining retry attempts. |
| Crashed | CRASHED | Yes | The run did not complete because of an infrastructure issue. |

## Returned values

When calling a task or a flow, there are three types of returned values:

- Data: A Python object (such as `int`, `str`, `dict`, `list`, and so on).
- `State`: A Prefect object indicating the state of a flow or task run.
- [`PrefectFuture`](/api-ref/prefect/futures/#prefect.futures.PrefectFuture): A Prefect object that contains both _data_ and _State_.

Returning data  is the default behavior any time you call `your_task()`.

Returning Prefect [`State`](/api-ref/server/schemas/states/) occurs anytime you call your task or flow with the argument `return_state=True`.

Returning [`PrefectFuture`](/api-ref/prefect/futures/#prefect.futures.PrefectFuture) is achieved by calling `your_task.submit()`.

### Return Data

By default, running a task will return data:

```python hl_lines="3-5"
from prefect import flow, task 

@task 
def add_one(x):
    return x + 1

@flow 
def my_flow():
    result = add_one(1) # return int
```

The same rule applies for a subflow:

```python hl_lines="1-3"
@flow 
def subflow():
    return 42 

@flow 
def my_flow():
    result = subflow() # return data
```

### Return Prefect State

To return a `State` instead, add `return_state=True` as a parameter of your task call.

```python hl_lines="3-4"
@flow 
def my_flow():
    state = add_one(1, return_state=True) # return State
```

To get data from a `State`, call `.result()`.

```python hl_lines="4-5"
@flow 
def my_flow():
    state = add_one(1, return_state=True) # return State
    result = state.result() # return int
```

The same rule applies for a subflow:

```python hl_lines="7-8"
@flow 
def subflow():
    return 42 

@flow 
def my_flow():
    state = subflow(return_state=True) # return State
    result = state.result() # return int
```

### Return a PrefectFuture

To get a `PrefectFuture`, add `.submit()` to your task call.

```python hl_lines="3-5"
@flow 
def my_flow():
    future = add_one.submit(1) # return PrefectFuture
```

To get data from a `PrefectFuture`, call `.result()`.

```python hl_lines="4-5"
@flow 
def my_flow():
    future = add_one.submit(1) # return PrefectFuture
    result = future.result() # return data
```

To get a `State` from a `PrefectFuture`, call `.wait()`.

```python hl_lines="4-5"
@flow 
def my_flow():
    future = add_one.submit(1) # return PrefectFuture
    state = future.wait() # return State
```

## Final state determination

The final state of a flow is determined by its return value.  The following rules apply:

- If an exception is raised directly in the flow function, the flow run is marked as `FAILED`.
- If the flow does not return a value (or returns `None`), its state is determined by the states of all of the tasks and subflows within it.
  - If _any_ task run or subflow run failed and none were cancelled, then the final flow run state is marked as `FAILED`.
  - If _any_ task run or subflow run was cancelled, then the final flow run state is marked as `CANCELLED`.
- If a flow returns a manually created state, it is used as the state of the final flow run. This allows for manual determination of final state.
- If the flow run returns _any other object_, then it is marked as successfully completed.

See the [Final state determination](/concepts/flows/#final-state-determination) section of the [Flows](/concepts/flows/) documentation for further details and examples.

## State Change Hooks

State change hooks execute code in response to changes in flow or task run states, enabling you to define actions for specific state transitions in a workflow.

#### A simple example

```python
from prefect import flow

def my_success_hook(flow, flow_run, state):
    print("Flow run succeeded!")

@flow(on_completion=[my_success_hook])
def my_flow():
    return 42

my_flow()
```

### Create and use hooks

#### Available state change hooks

| Type | Flow | Task | Description |
| ----- | --- | --- | --- |
| `on_completion` | ✓ | ✓ | Executes when a flow or task run enters a `Completed` state. |
| `on_failure` | ✓ | ✓ | Executes when a flow or task run enters a `Failed` state. |
| <span class="no-wrap">`on_cancellation`</span> | ✓ | - | Executes when a flow run enters a `Cancelling` state. |
| `on_crashed` | ✓ | - | Executes when a flow run enters a `Crashed` state. |
| `on_running` | ✓ | - | Executes when a flow run enters a `Running` state. |

#### Create flow run state change hooks

```python
def my_flow_hook(flow: Flow, flow_run: FlowRun, state: State):
    """This is the required signature for a flow run state
    change hook. This hook can only be passed into flows.
    """

# pass hook as a list of callables
@flow(on_completion=[my_flow_hook])
```

#### Create task run state change hooks

```python
def my_task_hook(task: Task, task_run: TaskRun, state: State):
    """This is the required signature for a task run state change
    hook. This hook can only be passed into tasks.
    """

# pass hook as a list of callables
@task(on_failure=[my_task_hook])
```

#### Use multiple state change hooks

State change hooks are versatile, allowing you to specify multiple state change hooks for the same state transition, or to use the same state change hook for different transitions:

```python
def my_success_hook(task, task_run, state):
    print("Task run succeeded!")

def my_failure_hook(task, task_run, state):
    print("Task run failed!")

def my_succeed_or_fail_hook(task, task_run, state):
    print("If the task run succeeds or fails, this hook runs.")

@task(
    on_completion=[my_success_hook, my_succeed_or_fail_hook],
    on_failure=[my_failure_hook, my_succeed_or_fail_hook]
)
```

#### Pass `kwargs` to your hooks

The Prefect engine will call your hooks for you upon the state change, passing in the flow, flow run, and state objects.

However, you can define your hook to have additional default arguments:

```python
from prefect import flow

data = {}

def my_hook(flow, flow_run, state, my_arg="custom_value"):
    data.update(my_arg=my_arg, state=state)

@flow(on_completion=[my_hook])
def lazy_flow():
    pass

state = lazy_flow(return_state=True)

assert data == {"my_arg": "custom_value", "state": state}
```

... or define your hook to accept arbitrary keyword arguments:

```python
from functools import partial
from prefect import flow, task

data = {}

def my_hook(task, task_run, state, **kwargs):
    data.update(state=state, **kwargs)

@task
def bad_task():
    raise ValueError("meh")

@flow
def ok_with_failure_flow(x: str = "foo", y: int = 42):
    bad_task_with_a_hook = bad_task.with_options(
        on_failure=[partial(my_hook, **dict(x=x, y=y))]
    )
    # return a tuple of "bar" and the task run state
    # to avoid raising the task's exception
    return "bar", bad_task_with_a_hook(return_state=True)

_, task_run_state = ok_with_failure_flow()

assert data == {"x": "foo", "y": 42, "state": task_run_state}
```

### More examples of state change hooks

- [Send a notification when a flow run fails](/guides/state-change-hooks/#send-a-notification-when-a-flow-run-fails)
- [Delete a Cloud Run job when a flow crashes](/guides/state-change-hooks/#delete-a-cloud-run-job-when-a-flow-crashes)
