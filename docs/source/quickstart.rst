Quickstart
==========

Enabling Trio mode and running your first async tests
-----------------------------------------------------

.. note:: If you used `cookiecutter-trio
   <https://github.com/python-trio/cookiecutter-trio>`__ to set up
   your project, then pytest-trio and Trio mode are already
   configured! You can write ``async def test_whatever(): ...`` and it
   should just work. Feel free to skip to the next section.

Let's make a temporary directory to work in, and write two trivial
tests: one that we expect should pass, and one that we expect should
fail::

   # test_example.py
   import trio

   async def test_sleep():
       start_time = trio.current_time()
       await trio.sleep(1)
       end_time = trio.current_time()
       assert end_time - start_time >= 1

   async def test_should_fail():
       assert False

If we run this under pytest normally, then we get a strange result:

.. code-block:: none

   $ pytest test_example.py

   ======================== test session starts =========================
   platform linux -- Python 3.6.5, pytest-3.6.3, py-1.5.4, pluggy-0.6.0
   rootdir: /tmp, inifile:
   collected 2 items

   test_example.py ..                                             [100%]

   ========================== warnings summary ==========================
   test_example.py::test_sleep
     .../_pytest/python.py:196: RuntimeWarning: coroutine 'test_sleep' was never awaited
       testfunction(**testargs)

   test_example.py::test_should_fail
     .../_pytest/python.py:196: RuntimeWarning: coroutine 'test_should_fail' was never awaited
       testfunction(**testargs)

   -- Docs: http://doc.pytest.org/en/latest/warnings.html
   ================ 2 passed, 2 warnings in 0.02 seconds ================

So ``test_sleep`` passed, which is what we expected... but
``test_should_fail`` also passes, which is strange. And it says that
the whole test run completed in 0.02 seconds, which is weird, because
``test_sleep`` should have taken at least second to run. And then
there are these strange warnings at the bottom... what's going on
here?

The problem is that our tests are async, and pytest doesn't know what
to do with it. So it basically skips running them entirely, and then
reports them as passed. This is not very helpful! If you see warnings
like this, or if your tests seem to pass but your coverage reports
claim that they weren't run at all, then this might be the problem.

Here's the fix:

1. Install pytest-trio: ``pip install pytest-trio``

2. In your project root, create a file called ``pytest.ini`` with
   contents:

   .. code-block:: none

      [pytest]
      trio_mode = true

And we're done! Let's try running pytest again:

.. code-block:: none

   $ pip install pytest-trio

   $ cat <<EOF >pytest.ini
   [pytest]
   trio_mode = true
   EOF

   $ pytest test_example.py
   ======================== test session starts =========================
   platform linux -- Python 3.6.5, pytest-3.6.3, py-1.5.4, pluggy-0.6.0
   rootdir: /tmp, inifile: pytest.ini
   plugins: trio-0.4.2
   collected 2 items

   test_example.py .F                                             [100%]

   ============================== FAILURES ==============================
   __________________________ test_should_fail __________________________

       async def test_should_fail():
   >       assert False
   E       assert False

   test_example.py:7: AssertionError
   ================= 1 failed, 1 passed in 1.05 seconds =================

Notice that now it says ``plugins: trio``, which means that
pytest-trio is installed, and the results make sense: the good test
passed, the bad test failed, no warnings, and it took just over 1
second, like we'd expect.


Trio's magic autojump clock
---------------------------

Tests involving time are often slow and flaky. But we can
fix that. Just add the ``autojump_clock`` fixture to your test, and
it will run in a mode where Trio's clock is virtualized and
deterministic. Essentially, the clock doesn't move, except that whenever all
tasks are blocked waiting, it jumps forward until the next time when
something will happen::

   # Notice the 'autojump_clock' argument: that's all it takes!
   async def test_sleep_efficiently_and_reliably(autojump_clock):
       start_time = trio.current_time()
       await trio.sleep(1)
       end_time = trio.current_time()
       assert start_time - end_time == 1

In the version of this test we saw before that used real time, at the
end we had to use a ``>=`` comparison, in order to account for
scheduler jitter and so forth. If there were a bug that caused
:func:`trio.sleep` to take 10 seconds, our test wouldn't have noticed.
But now we're using virtual time, so the call to ``await
trio.sleep(1)`` takes *exactly* 1 virtual second, and the ``==`` test
will pass every time. Before, we had to wait around for the test to
complete; now, it completes essentially instantaneously. (Try it!)
And, while here our example is super simple, it's integration with
Trio's core scheduling logic allows this to work for arbitrarily
complex programs (as long as they aren't interacting with the outside
world).


Async fixtures
--------------

We can write async fixtures::

   @pytest.fixture
   async def db_connection():
       return await some_async_db_library.connect(...)

   async def test_example(db_connection):
       await db_connection.execute("SELECT * FROM ...")

If you need to run teardown code, you can use ``yield``, just like a
regular pytest fixture::

   # DB connection that wraps each test in a transaction and rolls it
   # back afterwards
   @pytest.fixture
   async def rollback_db_connection():
       # Setup code
       connection = await some_async_db_library.connect(...)
       await connection.execute("START TRANSACTION")

       # The value of this fixture
       yield connection

       # Teardown code, executed after the test is done
       await connection.execute("ROLLBACK")


.. _server-fixture-example:

Running a background server from a fixture
------------------------------------------

Here's some code to implement an echo server. It's supposed to take in
arbitrary data, and then send it back out again::

   async def echo_server_handler(stream):
       while True:
           data = await stream.receive_some(1000)
           if not data:
               break
           await stream.send_all(data)

   # Usage: await trio.serve_tcp(echo_server_handler, ...)

Since this is such complicated and sophisticated code, we want to
write lots of tests to make sure it's working correctly. To test it,
we want to start a background task running the echo server itself, and
then we'll connect to it, send it test data, and see how it responds.
Here's a first attempt::

   # Let's cross our fingers and hope no-one else is using this port...
   PORT = 14923

   # Don't copy this -- we can do better
   async def test_attempt_1():
       async with trio.open_nursery() as nursery:
           # Start server running in the background
           nursery.start_soon(
               partial(trio.serve_tcp, echo_server_handler, port=PORT)
           )

           # Connect to the server.
           client = await trio.open_tcp_stream("127.0.0.1", PORT)
           # Send some test data, and check that it gets echoed back
           async with client:
               for test_byte in [b"a", b"c", b"c"]:
                   await client.send_all(test_byte)
                   assert await client.receive_some(1) == test_byte

This will mostly work, but it has a few problems. The first one is
that there's a race condition: We spawn a background task to call
``serve_tcp``, and then immediately try to connect to that server.
Sometimes this will work fine. But it takes a little while for the
server to start up and be ready to accept connections – so other
times, randomly, our connection attempt will happen too quickly, and
error out. After all – ``nursery.start_soon`` only promises that the
task will be started *soon*, not that it's actually happened.

To solve this race condition, we should switch to using ``await
nursery.start(...)``. You can `read its docs for full details
<https://trio.readthedocs.io/en/latest/reference-core.html#trio.The%20nursery%20interface.start>`__,
but basically the idea is that both ``nursery.start_soon(...)`` and
``await nursery.start(...)`` create background tasks, but only
``start`` waits for the new task to finish getting itself set up. This
requires some cooperation from the background task: it has to notify
``nursery.start`` when it's ready. Fortunately, :func:`trio.serve_tcp`
already knows how to cooperated with ``nursery.start``, so we can
write::

   # Let's cross our fingers and hope no-one else is using this port...
   PORT = 14923

   # Don't copy this -- we can do better
   async def test_attempt_2():
       async with trio.open_nursery() as nursery:
           # Start server running in the background
           # AND wait for it to finish starting up before continuing
           await nursery.start(
               partial(trio.serve_tcp, echo_server_handler, port=PORT)
           )

           # Connect to the server
           client = await trio.open_tcp_stream("127.0.0.1", PORT)
           async with client:
               for test_byte in [b"a", b"c", b"c"]:
                   await client.send_all(test_byte)
                   assert await client.receive_some(1) == test_byte

That solves our race condition. Next issue: hardcoding the port number
like this is a bad idea, because port numbers are a machine-wide
resource, so if we're unlucky some other program might already be
using it. What we really want to do is to tell :func:`~trio.serve_tcp`
to pick a random port that no-one else is using. It turns out that
this is easy: if you request port 0, then the operating system will
pick an unused one for you automatically. Problem solved!

But wait... if the operating system is picking the port for us, how do
we know figure out which one it picked, so we can connect to it later?

Well, there's no way to predict the port ahead of time. But after
:func:`~trio.serve_tcp` has opened a port, it can check and see what
it got. So we need some way to pass this data back out of
:func:`~trio.serve_tcp`. Fortunately, ``nursery.start`` handles this
too: it lets the task pass out a piece of data after it's started. And
it just so happens that what :func:`~trio.serve_tcp` passes out is a
list of :class:`~trio.SocketListener` objects. And there's a handy
function called :func:`trio.testing.open_stream_to_socket_listener`
that can take a :class:`~trio.SocketListener` and make a connection to
it.

Putting it all together::

   from trio.testing import open_stream_to_socket_listener

   # Don't copy this -- we can do better
   async def test_attempt_3():
       async with trio.open_nursery() as nursery:
           # Start server running in the background
           # AND wait for it to finish starting up before continuing
           # AND find out where it's actually listening
           listeners = await nursery.start(
               partial(trio.serve_tcp, echo_server_handler, port=0)
           )

           # Connect to the server.
           # There might be multiple listeners (example: IPv4 and
           # IPv6), but we don't care which one we connect to, so we
           # just use the first.
           client = await open_stream_to_socket_listener(listeners[0])
           async with client:
               for test_byte in [b"a", b"c", b"c"]:
                   await client.send_all(test_byte)
                   assert await client.receive_some(1) == test_byte

Okay, this is getting closer... but if we try to run it, we'll find
that it just hangs instead of completing. What's going on?

The problem is that we need to shut down the server – otherwise it
will keep waiting for new connections forever, and the test will never
finish. We can fix this by cancelling everything in the nursery once
our test is finished::

   # Don't copy this -- it finally works, but we can still do better!
   async def test_attempt_4():
       async with trio.open_nursery() as nursery:
           # Start server running in the background
           # AND wait for it to finish starting up before continuing
           # AND find out where it's actually listening
           listeners = await nursery.start(
               partial(trio.serve_tcp, echo_server_handler, port=0)
           )

           # Connect to the server.
           # There might be multiple listeners (example: IPv4 and
           # IPv6), but we don't care which one we connect to, so we
           # just use the first.
           client = await open_stream_to_socket_listener(listeners[0])
           async with client:
               for test_byte in [b"a", b"c", b"c"]:
                   await client.send_all(test_byte)
                   assert await client.receive_some(1) == test_byte

           # Shut down the server now that we're done testing it
           nursery.cancel_scope.cancel()

Okay, finally this test works correctly. But that's a lot of
boilerplate. Remember, we need to write lots of tests for this server,
and we don't want to have to copy-paste all that stuff into every
test. Let's factor it out into a fixture.

Probably our first attempt will look something like::

   # DON'T DO THIS, IT DOESN'T WORK
   @pytest.fixture
   async def echo_server_connection():
       async with trio.open_nursery() as nursery:
           await nursery.start(...)
           yield ...
           nursery.cancel_scope.cancel()

Unfortunately, this doesn't work. **You cannot make a fixture that
opens a nursery or cancel scope, and then yields from inside the
nursery or cancel scope block.** Sorry 🙁. We're `working on it
<https://github.com/python-trio/pytest-trio/issues/55>`__.

Instead, pytest-trio provides a built-in fixture called
``test_nursery``. This is a nursery that's created before each test,
and then automatically cancelled after the test finishes. Which is
exactly what we need – in fact it's even simpler than our first try,
because now we don't need to worry about cancelling the nursery
ourselves.

So here's our complete, final version::

   # Final version -- copy this!
   from functools import partial
   import pytest
   import trio
   from trio.testing import open_stream_to_socket_listener

   # The code being tested:
   async def echo_server_handler(stream):
       while True:
           data = await stream.receive_some(1000)
           if not data:
               break
           await stream.send_all(data)

   # The fixture:
   @pytest.fixture
   async def echo_server_connection(test_nursery):
       listeners = await test_nursery.start(
           partial(trio.serve_tcp, echo_server_handler, port=0)
       )
       client = await open_stream_to_socket_listener(listeners[0])
       async with client:
           yield client

   # A test using the fixture:
   async def test_final(echo_server_connection):
       for test_byte in [b"a", b"c", b"c"]:
           await echo_server_connection.send_all(test_byte)
           assert await echo_server_connection.receive_some(1) == test_byte

No race conditions, no hangs, simple, clean, and reusable.
