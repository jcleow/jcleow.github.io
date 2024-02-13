## Understanding pytest-xdist

* TOC
{:toc}
### A. Preface
Over at `$work` we have been using alot of [Pytest](https://docs.pytest.org/en/8.0.x/) and its associated libraries to help us with testing alot of of our application logic.

I have not had to fully deep dive into pytest and its associated plugins for the testing needed at $work. But recently, we also started exploring the usage of [Pytest-xdist](https://pytest-xdist.readthedocs.io/en/stable/) at `$work`, which allows tests to be run in parallel. With that, came a slew of flaky tests that required debugging.

In this post - I aim to consolidated my learnings about Pytest and Pytest-xdist throughout my time debugging their workings. Specifically, I wish to note down the parts that caused me confusion, things that I learnt and hopefully these notes can serve to aid your understanding with this library.

---

### B. An Introduction to Pytest-xdist
>The pytest-xdist plugin extends pytest with new test execution modes, with the most common usage being distributing tests across multiple CPUs to speed up test execution:

As the docs suggest, the library helps to distribute and run your suite of tests concurrently.

The library offers a few cool modes, but for the sake of brevity, I will focus more on the core usage modes meant for distribution which are related to:
1. `load` - which sends pending tests to any worker available in no guaranteed order.
2. `loadscope` - The default behaviour is that tests are grouped by module and test functions are grouped by class. <br><br> What this does is that all of the tests in a group run in the same process/worker. This is usually helpful when we have expensive module-level or class level fixtures that can be shared across all of these tests. <br><br>
3. `loadgroup` - Tests are grouped by the `xdist_group` mark. Groups are distributed to available workers as whole units. This guarantees that all tests with same `xdist_group` name run in the same worker.

---

#### C. A Unit of (Py)Test - Node
In pytest, a unit of test, be it a single function or a collection of tests, is known as a `node`, and their references or `node ids` their scopes are usually delimited by `::` identifiers.

Most notably, it identifies the particular subset of tests that is to be run.

E.g we can do something like:
```
pytest test_module.py::TestClass::test_method
```
to execute a particular `test_method` nested under `TestClass` in the `test_module.py` file.

Essentially the tests identified or proverbially `collected` in Pytest are also identified using such a specification format.

From pytest:
>Node IDs are of the form module.py::class::method or module.py::function. Node IDs control which tests are collected, so module.py::class will select all test methods on the class. Nodes are also created for each parameter of a parametrized fixture or test, so selecting a parametrized test must include the parameter value, e.g. module.py::function[param].

[See Source](https://docs.pytest.org/en/latest/example/markers.html#node-id)

In pytest, the [collector](https://docs.pytest.org/en/7.1.x/reference/reference.html#pytest.Collector) collects all the node ids and iteratively builds a tree out of all of these tests to perform testing later on.

---

#### D. Process of distributing tests in Pytest-xdist
In `pytest-xdist`, the process generally goes like this:
1. We specify the number of workers(nodes) we want to perform the parallel testing.

2. `pytest-xdist` spins up a new `distributed test session` or `DSession`. This `DSession` spins up a `Node Manager` which then spins up the required number of workers or nodes. The Node Manager is also synonymous to the `Controller`.

3. The `Controller` spins up the required number of workers by connecting to different (sub)processes running, each running a Python interpreter via execnet using gateways.
    * This can be done locally or to remote server.
    * Each of these processes are also known as a worker or node.
<br/>


![Pytest Workers](/docs/assets/2024-02-12-pytest-workers.png)

4. Each worker performs a full test collection of all the tests available and sends these tests or otherwise their `node ids` back to the `Controller`.
<br/>

5. The `Controller` constructs a list of `node_ids` based off the `node_ids`received and check if all workers collected the same tests.
    </br>
    * This is an important step as the implementation relies on a centralized in memory list of `node_ids` (i.e test ids).
    Based off this list, the Controller will then again [assign the tests](https://github.com/pytest-dev/pytest-xdist/blob/8d57046bd465b559f45f66c2406e5acedb1c1c21/src/xdist/scheduler/loadscope.py#L260-L264), based on scope or grouping to the different workers using the tests' indexed positions in this list.
    <br/>
    * Because the work assignment relies on this centralized list, if any of the workers collects any tests differently be it in [order or number](https://pytest-xdist.readthedocs.io/en/stable/known-limitations.html#order-and-amount-of-test-must-be-consistent), the test will not be able to execute and result in an error like `Failed due to gw0`.
6. Each worker is essentially a pytest runner, which reimplements `pytest_runtestloop` and `pytest_runtest_protocol`, but adjusted in `xdist` to manage the controller-workernode relationship ([see docs](https://github.com/pytest-dev/pytest-xdist/blob/95b309e980796a261045d770f69c016ca741473d/docs/how-it-works.rst)).
What is interesting here is learning about `pytest-xdist` extending pytest's existing hooks with its own custom implementation. We exploreo more of this in the next section.

But first here is a diagram that illustrates the process of from test collection to test execution and reporting.

![Pytest Workers](/docs/assets/2024-02-12-pytest-worker-flow.png)

A more complete process of how it works can also be found [here](https://github.com/pytest-dev/pytest-xdist/blob/95b309e980796a261045d770f69c016ca741473d/docs/how-it-works.rst)

---

#### E.1. Pytest-xdist: Scoping, grouping and running of tests in the same worker.
* As mentioned in section C, we can distribute the tests by their `scopes` either through the original implementation `loadscope` as well a derived implementation`loadgroup`.
* The tests distributed across the various workers are also guaranteed to be [run only once](https://github.com/pytest-dev/pytest-xdist/blob/8d57046bd465b559f45f66c2406e5acedb1c1c21/src/xdist/scheduler/loadscope.py#L12-L13).
* In `loadscope`, how it works is that for each test(or node id as discussed in section C) flows through the [_split_scope](https://github.com/pytest-dev/pytest-xdist/blob/8d57046bd465b559f45f66c2406e5acedb1c1c21/src/xdist/scheduler/loadscope.py#L268-L290) function.

For convenience this is the documentation:
```
Determine the scope (grouping) of a nodeid.
    There are usually 3 cases for a nodeid::
    example/loadsuite/test/test_beta.py::test_beta0
    example/loadsuite/test/test_delta.py::Delta1::test_delta0
    example/loadsuite/epsilon/__init__.py::epsilon.epsilon

    #. Function in a test module.
    #. Method of a class in a test module.
    #. Doctest in a function in a package.

    This function will group tests with the scope determined by splitting
    the first ``::`` from the right. That is, classes will be grouped in a
    single work unit, and functions from a test module will be grouped by
    their module. In the above example, scopes will be::

    example/loadsuite/test/test_beta.py
    example/loadsuite/test/test_delta.py::Delta1
    example/loadsuite/epsilon/__init__.py
```


To rephrase, what it does is that `_split_scope` determines the scope via the first `::` separator of the node ids. I.e the scope is created from a file/module -> class to function level.
* By doing this, `xdist` achieves grouping of tests together by their scopes. A scope of tests will be guaranteed to run in a single worker/process.
* What this also means is that we can overwrite this function to determine custom way of ensuring certain tests will always run in the same worker.

For example, we can take a look at the the `loadgroup` implementation. Specifically it redefines the [_split_scope](https://github.com/pytest-dev/pytest-xdist/blob/8d57046bd465b559f45f66c2406e5acedb1c1c21/src/xdist/scheduler/loadgroup.py#L19-L54) implementation to identify based on the `@` operator within the node ids.

Such a node id might look like this `tests/test.py::TestGroup1::test_get_now_only@group1` given the example below.


```python
# investigation/tests/test.py

import pytest

@pytest.mark.xdist_group(name="group1")
class TestGroup1:
    def test_val_1(self):
        val = 1
        assert val == 1


@pytest.mark.xdist_group(name="group2")
class TestGroup2:
    def test_val_2(self):
        val = 2
        assert val == 2
```

Test Results
```
(venv3117xdist) ➜  investigation git:(master) ✗ pytest tests/test.py -n 2 --dist loadgroup -vvv
=========================================================================================================================================== test session starts ============================================================================================================================================
platform darwin -- Python 3.11.0rc2, pytest-8.1.0.dev176+g7690a0ddf.d20240210, pluggy-1.4.0 -- /Users/jitcorn/.pyenv/versions/3.11.0rc2/bin/python3.11
cachedir: .pytest_cache
Using --randomly-seed=1707796509
rootdir: /Users/jitcorn/pytest-xdist
configfile: tox.ini
plugins: timeout-2.2.0, randomly-3.1.0, asyncio-0.20.3, tornado-0.8.1, cov-4.0.0, xdist-3.5.0, forked-1.3.0, anyio-4.2.0, mock-3.5.1
asyncio: mode=Mode.STRICT
2 workers [2 items]
scheduling tests via LoadGroupScheduling

tests/test.py::TestGroup2::test_val_2@group2
tests/test.py::TestGroup1::test_val_1@group1
[gw1] [ 50%] PASSED tests/test.py::TestGroup2::test_val_2@group2
[gw0] [100%] PASSED tests/test.py::TestGroup1::test_val_1@group1
```

From this contrived test example, we demonstrate the usage of `loadgroup` and can make a few observations.
1. We first run the tests in the verbose mode `-vvv` and we can observe that there are 2 workers `gw0` and `gw1` working on the tests.
This is because we specified 2 workers in `-n 2` (we can also use `-n auto` to determine based on the number of CPUs your server has).
`gw0` and `gw1` are also knokwn as `worker_id` in `xdist`.
2. We can see that the the tests are also grouped and distributed evevnly to each of the workers.
3. The group names `group1` and `group2` are appended as `@<group_num>` at the end of each of the node ids.
* Also note while the existing example on the documentation shows the grouping by individual test functions, you can also group tests by their classes, instead of individual test functions.
This means you don't have to put `@pytest.mark.xdist_group` over every single function if they are already grouped by classes, which is very convenient!

#### E.2 Customising the scoping capabilities of pytest-xdist

As seen from the previous section, the scoping mechanism can be customised by specifying a custom algortihm in the `_split_scope` function.

Suppose you wish to devise your own custom `split_scope` methodology specific to your repo, you can modify `conftest.py` in this manner to modify the scoping functionality.

```python
import os
import logging
from xdist.scheduler.loadscope import LoadScopeScheduling

SINGLE_PROCESS_TESTFILES_TO_SCOPE_MAPPING = {
    "investigation/tests/test.py": "investigation-tests-test-scope",
}

class TestsScheduler(LoadScopeScheduling):
    def fetch_highest_test_scope(self, node_id):
        """
        Currently used for scoping out tests that cannot be run concurrently, i.e in a single worker only
        """
        first_delimiter = node_id.find("::")
        return node_id[:first_delimiter]

    def _split_scope(self, node_id):
        """
        Override pytest-xdist's _split_scope method to have scoped/grouped tests run by the same worker.
        Each node_id entails a single scope/worker.
        See: https://github.com/pytest-dev/pytest-xdist/blob/ef344e9b55a0365c1aabb738ce3db97324ed553e/src/xdist/scheduler/loadscope.py#L268-L290
        """
        highest_test_scope = self.fetch_highest_test_scope(node_id)
        if highest_test_scope in SINGLE_PROCESS_TESTFILES_TO_SCOPE_MAPPING:
            return SINGLE_PROCESS_TESTFILES_TO_SCOPE_MAPPING.get(highest_test_scope)

        return node_id

def pytest_xdist_make_scheduler(config, log):
    return TestsScheduler(config,log)
```

* This example shows that in a list of all of the tests collected, we ensure that all the tests inside test.py are guaranteed to run within the same worker.
* This could be useful in cases where you might experience race conditions amongst different tests that are read or writing to the same table or index in a database for example.
* The way how the `xdist` groups tests by scope is by the heavy usage of ordered dicts, specially [here](https://github.com/pytest-dev/pytest-xdist/blob/8d57046bd465b559f45f66c2406e5acedb1c1c21/src/xdist/scheduler/loadscope.py#L356). The exact debugging shall be left to the reader as an exercise :p.


**Important!**
Be it grouping by `@` or defining your own `_split_scope` function, at the point of writing of pytest version 3.5.0, while a grouped/scoped tests are guaranteed to run in the same worker, there is no mechanism that prevents 2 groups of scoped tests from running in the same worker as it comes in via a round-robin fashion*.

I.e if we have 2 workers with 3 grouped tests:
```
[gw0] - test_group_1
[gw1] - test_group_2
[gw0] - test_group_3
```

test_group_1 and test_group_3 are going to run in the same `gw0` worker. This may pose a problem if there is no properly tear down performed after tear_group_1. As you can imagine, if you wish to separate test_group_1 and test_group_3 from separate workers (maybe they involve writing to the same DB), then pytest-xdist does not manage this for you at the moment. A better way to circumvent this problem is to ensure that the tests are properly setup and teardown in both test_group_1 and test_group_3.

---

#### F. Pytest: What are hooks?
*  As this [author](https://pytest-with-eric.com/hooks/pytest-hooks/#:~:text=Pytest%20hooks%20are%20intervention%20points,or%20when%20exceptions%20are%20raised.) puts it, Pytest hooks are gateway points that allow users to inject logic at specific stages of the test execution process to modify or extend the behaviour of tests based on test events.
* In the case of our pytest-xdist workers, it mostly reimplements or makes use of these 2 hooks:
    * [pytest_runtestloop](https://docs.pytest.org/en/7.1.x/reference/reference.html#pytest.hookspec.pytest_runtestloop) is the default hook implementation that performs the `pytest_runtest_protocol`. What it means that it was built to be extendable via pytest hooks.
    * [pytest_runtest_protocol](https://pytest-with-eric.com/hooks/pytest-hooks/#:~:text=Pytest%20hooks%20are%20intervention%20points,or%20when%20exceptions%20are%20raised.) is basically the series of steps done for each test in a `runtestloop`.
    * Each stage usually performs 4 actions, `call`, `create-report`, `log-report` and `handle-exception`, respectively:
        * `call` - `pytest_runtest_setup` - essentiallly collecting values such as the fixtures required for the test item
        * `create-report` - `pytest_runtest_makereport` - creates a TestReport object meant to report the outcomes of a test
        * `log-report` - `pytest_runtest_logreport` - processes the created TestReport
        * `handle-exception` - `pytest_exception_interact`- interactive handling when an exception is thrown

    * Hence in a `runtest_protocol`, these 4 actions are done in each phase:
        1. Setup phase - This involves processes such as:
        2. Call phase - which performs the actual test function call
        3. Teardown phase - tearing down all of the tests

    This is indeed a similar idea to the 3 As of testing, together with how we normally have setups and teardowns in our day to day test writing!
<br/>

---

#### G. Pytest-xdist: Why are the workers collecting the tests instead of the controller?
* For more details please refer to the [How it works](https://github.com/pytest-dev/pytest-xdist/blob/95b309e980796a261045d770f69c016ca741473d/docs/how-it-works.rst) section, which also explains the entire process in greater details.

---

#### H. Debugging the pytest-xdist and pytest libraries
* A quick way of testing out any library installed on a repo you working on is by:
1. Installing the library locally
2. Installing the library in your target repo by using the `editable` mode.

Step 1:
Specifically in the context of pytest-xdist we need to first clone it and then install all of the dependencies locally and install its dependencies.
I am currently using python 3.11:
```
git clone git@github.com:pytest-dev/pytest-xdist.git
tox -e py311
```

This should set up all the necessary dependencies in your local pytest-xdist setup.

Step 2:
In our repo that is using pytest-xdist, we can symlink it to this local copy of the package by install it in `--editable` mode. This developers to implement and test changes iteratively before releasing a package.

```cli
pip install -e <local_directory_of_package>
```

With this set up, the `pytest-xdist` library will be symlinked from your repo to the library locally. You can then place debuggers like `import pdb; pdb.set_trace()` inside to debug the flow of the programme!

---

#### I. Takeaways

Here are my takeaways after spending alot of time trying to understand `pytest-xdist` internals:

1. There was some confusion when deconstructing the notion of a node: in `pytest` it refers to a unit of test while in `pytest-xdist`, it refers to a particular worker.
2. A in memory ordered list of `node_ids` is the underlying structure used to reference and schedule work units to each worker node. The creation of this list is based on scoping and heavy usage of `OrderedDict()` in the actual implementation
3. The `@pytest.mark.xdist_group(name="<name>")` decoractor can also be applied on classes for more convenient grouping
4. While scoping/grouping of tests ensures they run in the same worker, it does not prevent 2 groups of tests running in the same worker. Proper test set ups and tear downs are recommended if race conditions are a concern.
5. `Pytest-xdist` makes use of existing pytest-hooks to help extend pytest functionalities to make test running in a distributed setting possible.
6. A similar arrange, act, assert, complete with setup and teardown structure is also present in the inner workings of pytest!


N.B
\* Whether the scoped tests are indeed assigned on a round robin fashion or based on FIFO manner or some other mechanism needs more investigation.



<!-- ```mermaid
graph TD;
    c[Controller]
    w1[WorkerNode1]
    w2[WorkerNode2]
    w3[WorkerNode3]
    g[Gateway]

    style c fill:#c0c0c0,stroke:#333,stroke-width:1px
    style w1 fill:#30D5C8,stroke:#333,stroke-width:1px
    style w2 fill:#30D5C8,stroke:#333,stroke-width:1px
    style w3 fill:#30D5C8,stroke:#333,stroke-width:1px

    style g fill:#FFD580,stroke:#333,stroke-width:1px

    c-.connects via.->g<-.collects all tests.->w1
    c-.connects via.->g<-.collects all tests.->w2
    c-.connects via.->g<-.collects all tests.->w3
``` -->

<!--```mermaid
sequenceDiagram
    participant Controller
    participant Gateway
    participant Node1

    Controller->>Gateway: establishes connection
    Gateway->>Node1: creates a new Python interpreter process
    Node1->>Node1: collects tests
    Node1->>Controller: return collected tests
    Controller->>Controller: checks all the tests collected from all workers
    Controller->>Node1: assigns subset of all tests to node to process
    Node1->>Node1: runs tests on assigned node_ids i.e runtest_protocol
    Node1->>Controller: returns tests results
``` -->

^ Just found out that GH pages doesnt support mermaid diagrams at moment, have to live with some low fidelity screenshots for now.