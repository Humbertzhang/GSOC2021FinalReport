# Final Report Of "Proposal for Apache SkyWalking: Python agent supports profiling"

Author: Ke Zhang

Mentor: Zhenxu Ke

This is the final report of [Google Summer Of Code 2021 Project 5701806362984448](https://summerofcode.withgoogle.com/dashboard/project/5701806362984448/overview/).

# Final Report Link

Code Location:  
* https://github.com/apache/skywalking-python/tree/97bf7fd8b63aedfbb37b7035c84445c71b32cd07/skywalking/profile
* https://github.com/apache/skywalking-python/tree/97bf7fd8b63aedfbb37b7035c84445c71b32cd07/skywalking/command
* https://github.com/apache/skywalking-python/blob/97bf7fd8b63aedfbb37b7035c84445c71b32cd07/skywalking/utils/atomic_ref.py
* https://github.com/apache/skywalking-python/blob/97bf7fd8b63aedfbb37b7035c84445c71b32cd07/skywalking/utils/array.py
* https://github.com/apache/skywalking-python/blob/97bf7fd8b63aedfbb37b7035c84445c71b32cd07/skywalking/agent/__init__.py#
* https://github.com/apache/skywalking-python/blob/97bf7fd8b63aedfbb37b7035c84445c71b32cd07/skywalking/agent/protocol/grpc.py
* https://github.com/apache/skywalking-python/blob/97bf7fd8b63aedfbb37b7035c84445c71b32cd07/skywalking/client/grpc.py
* https://github.com/apache/skywalking-python/blob/97bf7fd8b63aedfbb37b7035c84445c71b32cd07/skywalking/utils/integer.py

There are also some minor modifications in other files, the detail can be found in below PullRequest links.

**Related Pull Requests(the work I have done can be found here):**

* https://github.com/apache/skywalking-python/pull/127
* https://github.com/apache/skywalking-python/pull/155


# 1. What I have done

* Add `CommandService` in SkyWalking Python Agent for query command from SkyWalking OAPServer.
* Add Command Dispatch class for dispatching command to properly Command Executor.
* Add Profile module for completing the whole function. 
  It includes 
  * `ProfileTaskExecutionService` for check, process and start a `ProfileTask`
  * `ProfileTaskExecutionContext` for maintain and manage all the context related to this target task
  * `ProfileThread` for profiling target thread in another thread.
  * `ThreadProfiler` for doing the actually profile job: it will collect the thead dump of target thread periodically, pack 
    the thread dump into `TracingThreadSnapshot` Object, and send this object to SkyWalking OAPServer. The OAPServer will 
    then analyse the collected data and show the end result in the front end. 

Now, The user of the SkyWalking Python Agent can profile their Python programs like their Java Programs in SkyWalking.

# 2. How to use it
The profile function is enabled by default, you just need to start your application(like below code), open SkyWalking UI, start a new task, 
and request your application, then you can see the result in the dashboard.


The test code is:
```Python
import time
from skywalking import agent, config


def method1():
    time.sleep(0.02)
    return '1'


def method2():
    time.sleep(0.02)
    return method1()


def method3():
    time.sleep(0.02)
    return method2()


if __name__ == '__main__':
    config.service_name = 'provider'
    config.logging_level = 'DEBUG'
    agent.start()

    import socketserver
    from http.server import BaseHTTPRequestHandler

    class SimpleHTTPRequestHandler(BaseHTTPRequestHandler):

        def do_POST(self):
            method3()
            time.sleep(0.5)
            self.send_response(200)
            self.send_header('Content-Type', 'application/json')
            self.end_headers()
            self.wfile.write('{"song": "Despacito", "artist": "Luis Fonsi"}'.encode('ascii'))

    PORT = 9090
    Handler = SimpleHTTPRequestHandler

    with socketserver.TCPServer(("", PORT), Handler) as httpd:
        httpd.serve_forever()
```

And after profile, the result is in line with expectations:

![](https://raw.githubusercontent.com/h10gforks/images/master/finalcapture.png)


# 3. What is the most challenging part?
Well, the most challanging part of this project is how to handle the relationships between those threads. 

For doing this, I Add `AtomicRef`, `AtomicArray` and `AtomicInteger` in `utils` for access and modify variables in a 
thread-safe way. I also add a `Reentrant Lock` in `ProfileTaskExecutionService` for thread safe, And I spend a lot of time 
for debug, but finally i fixed all the concurrency errors.


# 4. What left to be done
The E2E test of Python Agent Profile is needed to be done.

I will add test for it later in another PR.
