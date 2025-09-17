# Temporal Cheat Sheet -- Getting Started Commands, Patterns, and More

## Build Workflow
Here is some basic workflow starter code: 

```python
from temporalio import workflow

from datetime import timedelta
from temporalio import workflow

# Import activity, passing it through the sandbox without reloading the module
with workflow.unsafe.imports_passed_through():
    from translate import TranslateActivities


@workflow.defn
class GreetSomeone:
    @workflow.rrun
    async def run(self, name: str) -> str:
        greeting = await workflow.execute_activity_method(
            TranslateActivities.greet_in_spanish,
            name,
            start_to_close_timeout=timedelta(seconds=5),
        )

        return greeting
```
## Launch Workflow:

**SDK Starter Code:** 
```python
greeting_handle = workflow.start_activity_method(
    TranslateActivities.greet_in_spanish,
    name,
    start_to_close_timeout=timedelta(seconds=5)
)

greeting = await greeting_handle
```


## Show Workflow:

**Standard Workflow:**
$ temporal workflow show --workflow-id {my-workflow}


**Detailed Worfklow:** 
$ temporal workflow show --workflow-id {my-workflow} --detailed


## Activity Starter Code

**Function Style**

```python
from temporalio import activity

@activity.defn
async def greet_in_french(name: str) -> str:
    return f"Bonjour {name}!"
```

**Method Style**

```python
class TranslateActivities:
    def __init__(self, session: aiohttp.ClientSession):
        self.session = session

    @activity.defn
    async def greet_in_spanish(self, name: str, stem: str) -> str:
        base = f"http://localhost:9999/{stem}"
        url = f"{base}?name={urllib.parse.quote(name)}"

        async with self.session.get(url) as response:
            translation = await response.text()

        return translation
```




# Non-Standard Topics
## Sync vs Async Activites, Workers
In general, we write temporal workflows with python as asyc. However, there are some cases where we need to write an activity as a sync function.
Below is an example of a sync activity: 

```python
@activity.defn
def greet_in_spanish(name: str) -> str:
    greeting = call_service("get-spanish-greeting", name)
    return greeting

# Utility method for making calls to the microservices
def call_service(stem: str, name: str) -> str:
    base = f"http://localhost:9999/{stem}"
    url = f"{base}?name={urllib.parse.quote(name)}"

    response = requests.get(url)
    return response.text
```

The worker code also needs modification for sychronous activities. When executing Synchronous Activities, you must pass an activity_executor to the Worker. Currently the Temporal Python SDK supports Thread Pools or Multiprocessing as your Activity Executor. Which one to use is dependent on your use case and deployment. This example will show how to use a ThreadPoolExecutor as your Activity Executor.

```python
import concurrent.futures
import asyncio

from temporalio.client import Client
from temporalio.worker import Worker

from translate import TranslateActivities
from greeting import GreetSomeone


async def main():
    client = await Client.connect("localhost:7233", namespace="default")

    activities = TranslateActivities()

    with concurrent.futures.ThreadPoolExecutor(max_workers=100) as activity_executor:
        worker = Worker(
            client,
            task_queue="greeting-tasks",
            workflows=[GreetSomeone],
            activities=[activities.greet_in_spanish, activities.farewell_in_spanish],
            activity_executor=activity_executor,
        )
        print("Starting the worker....")
        await worker.run()


if __name__ == "__main__":
    asyncio.run(main())
```

Asynchronous Activities have many advantages, such as potential speed up of execution. However, as discussed above, making unsafe calls within the async event loop can cause sporadic and difficult to diagnose bugs. For this reason, we recommend using asynchronous Activities only when you are certain that your Activities are async safe and don't make blocking calls.

If you experience bugs that you think may be a result of an unsafe call being made in an asynchronous Activity, convert it to a synchronous Activity and see if the
issue resolves.

Sometimes, you will need to execute both sync and async activities from the same worker. Here's an exaple of what that looks like: 

```python
import concurrent.futures
import aiohttp
import asyncio

from temporalio.client import Client
from temporalio.worker import Worker

from translate import TranslateActivities
from greeting import GreetSomeone


async def main():
    client = await Client.connect("localhost:7233", namespace="default")

    async with aiohttp.ClientSession() as session:
        activities = TranslateActivities(session)
        with concurrent.futures.ThreadPoolExecutor(max_workers=100) as activity_executor:
            worker = Worker(
                client,
                task_queue="greeting-tasks",
                workflows=[GreetSomeone],
                activities=[activities.greet_in_spanish, activities.thank_you_in_spanish],
                activity_executor=activity_executor,
            )
            print("Starting the worker....")
            await worker.run()


if __name__ == "__main__":
    asyncio.run(main())
```

## Sandbox challenges and passing through imported packages
In the Temporal Python SDK, Workflow files are reloaded in a sandbox for every run. In order to keep from reloading an import on every run, you can mark it as pass through so it reuses the module from outside the sandbox. Standard library modules and temporalio modules are passed through by default. All other modules that are used in a deterministic way, such as activity function references or third-party modules, should be passed through this way.

Because of this situation, we recommend keeping your workflow files separate from the rest of your code including input/output types, Activities, etc.

This section of code should be included in your Workflow below your standard library and temporalio imports:

```python
# Import activity, passing it through the sandbox without reloading the module
with workflow.unsafe.imports_passed_through():
    from translate import TranslateActivities
```

