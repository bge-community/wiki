# Networking using `asyncio`

> Level: Advanced

## Event-loops vs Threads

The Python `asyncio` module proposes an event-loop implementation and utilities for I/O operations.

Event-loops were designed to address a simple issue: In some cases, your program is mostly waiting for I/O operations to complete.

This means that if you read from a file, or make an http request, or anything that requires some time, your thread will block until your get a response.

Event-loops solve that issue by making you write your program in such a way that you can execute something else while requests are pending.

But this is not magic, in order for an event-loop to work and "parallelize" your code, it needs to "take control of your thread".

## About Threads

A program only ever has one thread of execution by default, you can spawn more threads but there are some issues:

- It is relatively heavy to create a new thread from Python.

  Spawning threads require some processing from the runtime and the OS.

- You cannot expect to spawn 1 thread per task you want to complete.

  If you are parallelizing 5 tasks, why not. But if you have 100+ tasks, threads are not the way to go anymore. You can use pools, where you define a maximum limit of threads that can run at the same time, then "submit" a function to the pool and it will try to find available threads to run the submitted function. Problem is that if all the threads are busy, new work won't happen.

- Code actually runs in parallel.

  While this might be interesting to "speed up" your processing, it is also very easy to hit problems if you access the same resources from different threads. It is especially an issue in the BGE, since there are specific times at which the BGE is "ready" to execute logic code, but if your run functions in separate threads, you might call APIs outside of these "windows" and the BGE might crash or raise an exception in your code running in the thread.

Threads are not inherently bad, but they must be used with a lot of care, especially in the BGE.

## Back to Event-Loops

So how can an event-loop help you?

First of all, it can run on just one thread and it can run multiple operations _concurrently_, without blocking everything because someone is waiting for an answer.

Note that multiple things don't happen at the same time, but the event-loop will enter a phase where it looks for new events that you registered yourself onto, and dispatches said event to each of your listeners, one after the other. You don't know when the event will be trigger, meaning that you also don't know when your "listening code" will be executed, hence why we call this programming style "asynchronous" despite **not being parrallel**.

## Illustrations

But let's take a look at how **synchronous** code may look like (script for a Python Controller in "module" mode):

```py
# Echo client program
import socket

HOST = 'daring.cwi.nl'    # The remote host
PORT = 50007              # The same port as used by the server

def pycontroller_on_event(controller):
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as client_socket:
        client_socket.connect((HOST, PORT))
        client_socket.sendall(b'Hello, world')
        # the following line may block your game:
        data = client_socket.recv(1024)

    print('Received', repr(data))
```

This code looks fairly simple, it basically creates a socket, sends a message (`.sendall`), and finally waits for a response (`.recv`).

The issue here is that the whole program is blocked on the receive statement. Because of the way it is written, you don't want your program to continue if you did not receive anything, that wouldn't make any sense. So it blocks.

That's the price of writing code for only one thread, if it blocks, you'll have to wait.

In the BGE this would translate into your game freezing, because it will wait for your script to finish, which itself waits for a response.

To solve this issue, you have three solutions:

1. Make the socket non-blocking.

    This will make the `.recv` call return and raise an error if there was nothing to read.

    You need to write your code to branch off and wait for the next execution to try and read again, until something can be read.

    It works, but throwing errors on each logic frame and re-executing the same code over and over until it works is a bit annoying to reason about.

    ```py
    # Echo client program (non-blocking)
    import socket

    HOST = 'daring.cwi.nl'    # The remote host
    PORT = 50007              # The same port as used by the server

    def pycontroller_on_event(controller):
        owner = controller.owner

        client_socket = owner.get('socket', None)

        # Test if someone already expects a response somehow
        if client_socket is None:
             client_socket = owner['socket'] = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
             client_socket.settimeout(0)
             client_socket.connect((HOST, PORT))
             client_socket.sendall(b'Hello, world')

     def pycontroller_always(controller):
         owner = controller.owner

         client_socket = owner.get('socket', None)

         # Test if someone somehow expects a response
         if client_socket is not None:
             try:
                 data = client_socket.recv(1024)
             except socket.timeout:
                 pass

             if data is not None: # finally got something to read
                 print('Received', repr(data))
                 client_socket.close()
                 del owner['socket']
    ```

    In this version, we are constantly "polling" on a socket (if present) in order to see if we get a response. In order to do this, we need to store our "useful state" inside the game object that executes the code, and the way  it is written here, we can only have one pending request at the time. Allowing for multiple request is left to the reader.

    The main conclusion I have from this is that while it works, it requires to store the state "outside" of your functions, which can make it harder to design bigger systems. The polling is also a bit ugly, even though this is a naive implementation, supporting more advanced use cases is yet another funny task.

2. Read in a different thread.

    This way, your main thread will finish, but the new one will wait until something is received.

    The issue here is that you don't know when it will finish, and it might be at a time the BGE is not ready.

    ```py
    # Echo client program (threading)
    import threading
    import socket

    HOST = 'daring.cwi.nl'    # The remote host
    PORT = 50007              # The same port as used by the server

    def recv_thread(sock):
        '''
        This will be run in its own thread
        '''
        data = sock.recv(1024)
        print('Received', repr(data))
        sock.close()

    def pycontroller_on_event(controller):
        client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM):
        client_socket.connect((HOST, PORT))
        client_socket.sendall(b'Hello, world')
        thread = threading.Thread(target=recv_thread, args=(client_socket,))
        thread.start()
    ```

    In this scenario, the code looks a bit cleaner, but this is only because this is a simple example.

    Once you start doing more complex things, especially upon reception, you will start enjoying using threads.

    If you try to use a gameobject reference inside the thread, using the BGE API will also only bring problems.

3. Use `asyncio`.

    ```py
    # python >= 3.6
    import asyncio

    async def tcp_echo_client(message):
        reader, writer = await asyncio.open_connection(
            '127.0.0.1', 8888)

        print(f'Send: {message!r}')
        writer.write(message.encode())

        data = await reader.read(100)
        print(f'Received: {data.decode()!r}')

        print('Close the connection')
        writer.close()
        await writer.wait_closed()

    def pycontroller_on_event(controller):
        asyncio.create_task(tcp_echo_client('Hello World!'))

    def pycontroller_always(controller):
        logic_frame_time = 1 / bge.logic.getLogicTicRate()
        asyncio.run(asyncio.sleep(logic_frame_time / 3)) # arbitrary number
    ```

    Or for older Python runtimes:

    ```py
    # python < 3.6
    import asyncio

    @asyncio.coroutine
    def tcp_echo_client(message, loop):
        reader, writer = yield from asyncio.open_connection(
            '127.0.0.1', 8888, loop=loop)

        print('Send: %r' % message)
        writer.write(message.encode())

        data = yield from reader.read(100)
        print('Received: %r' % data.decode())

        print('Close the socket')
        writer.close()

    def pycontroller_on_event(controller):
        loop = asyncio.get_event_loop()
        loop.ensure_future(tcp_echo_client('Hello World!', loop))

    def pycontroller_always(controller):
        logic_frame_time = 1 / bge.logic.getLogicTicRate()
        loop = asyncio.get_event_loop()
        loop.run_until_complete(asyncio.sleep(logic_frame_time / 3)) # arbitrary number
    ```

    In this case, you start using coroutines that have to be executed by an event-loop. The first thing we can notice is how close the "business" code looks like compared to our original synchronous version:

    ```py
    async def tcp_echo_client(message):
        reader, writer = await asyncio.open_connection(
            '127.0.0.1', 8888)

        print(f'Send: {message!r}')
        writer.write(message.encode())

        # the following line will pause this function until we get a response,
        # but your game will keep running!
        data = await reader.read(100)
        print(f'Received: {data.decode()!r}')

        writer.close(); await writer.wait_closed()
        print('Connection closed.')
    ```

    Versus:

    ```py
    def pycontroller_on_event(controller):
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as client_socket:
            client_socket.connect(('127.0.0.1', 8888))

            print(f'Send: {message!r}')
            client_socket.sendall(b'Hello, world')

            # the following line may block your game:
            data = client_socket.recv(1024)
            print(f'Received: {data.decode()!r}')

        print('Connection closed.')

    ```

    The major difference is the new `async/await` syntax, but also the slightly different API (reader/writers vs pure sockets).

    Ultimately, asynchronous code is closer to what you would write synchronously, except that it will not block your game if it has to wait for an event. It will also always execute code when you actually run the event loop: In our example, we run it for 1/3rd of a frame during a logic tick:

    ```py
    def pycontroller_always(controller):
        logic_frame_time = 1 / bge.logic.getLogicTicRate()
        loop = asyncio.get_event_loop()
        loop.run_until_complete(asyncio.sleep(logic_frame_time / 3)) # arbitrary number
    ```

    Compared to a thread, that would execute Python code at random times, which could lead to unexpected ACCESS_VIOLATION_EXCEPTION errors, crashing the game or Blender.

## Conclusion

I heavily recommend reading the official's Python documentation about async programming: https://docs.python.org/3/library/asyncio-task.html#coroutine.

What I tried to explain in this document is how to run `asyncio` in the BGE, and not so much how to use asynchronous programming in Python.

Something that can also greatly help would be learning JavaScript (mostly NodeJS scripts): The main difference with Python is the syntax (of course, it is a different language), but also the fact that in JavaScript there is always an event-loop, that runs after parsing your main file. After a certain point, if you did it well, your whole application is running solely based on "callback functions", which means that it is the runtime (event-loop) that drives everything, only executing your code when required. So there is the good old callback-based API, as well as `Promise` objects, that would map to Python's `Future` objects. `Promise` objects can be used with the `async/await` syntax, and/or with the callback-based API, it is really simple yet powerful.

Understanding how async programming works in the JavaScript world can help a great deal when doing it in Python, except that it will be more verbose using the latter.

## Playground

If you want to understand a bit the magic behind coroutines in Python, you can try the following thing:

```py

class AwaitableObject:
    '''
    We need an object of this kind in order to break a coroutine's execution.
    In this example we will do something silly, but asyncio uses the same
    mechanism except with specific semantics for the event-loop to work.
    '''

    def __init__(self, step):
        self.step = step

    def __await__(self):
        print('awaiting inside the object...')
        sent = yield f'break {self.step} !'
        print(f'received: {sent}')
        return yield f'break {self.step} ! (wow)'

async def asyncFunction(count):
    print('started!')
    for i in range(count):
        print(f'iteration: {i}')
        # we await an "awaitable" instance here:
        result = await AwaitableObject(i)
        print(f'await done: {result}')

coroutine = asyncFunction(5)

# ... nothing happens??

# Of course! We need to execute the coroutine, until something awaited yields.
# A little quirk: We MUST send `None` on the first execution:
out = coroutine.send(None)

# output: started!
# output: iteration: 0
# output: awaiting inside the object...

print(out)

# output: break 0 !

# Ok, so since we are manually triggering the coroutine, we are the ones to
# catch things sent through the `yield` statements of the awaitables...
# The awaiting function has no idea what we are receiving now, but we can
# keep sending things to this coroutine:

out = coroutine.send('aaa')

# output: received: aaa

print(out)

# output: break 0 ! (wow)

out = coroutine.send('bbb')

# output: await done: bbb
# output: iteration 1
# output: awaiting inside the object...

print(out)

# output: break 1 !

...
```

To be fair, this is rather confusing the first time. But if you play enough with this, you'll get a better picture of what actually happens in your code.

Coroutines basically are functions that must be executed in "chunks" or "parts", they don't do everything in one go. And this is precisely what is needed to write concurrent code: When an async function wants to wait for something, it can await some special awaitable object, that will tell whoever steps the coroutine to put it on the side until some condition is met. Meanwhile, the executor can look for other paused coroutines to execute.

Good thing `asyncio` does all that for us, right? :)
