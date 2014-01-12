 q
===
A platform-independent asynchronous IoC library for C++.

Version
----
0.0

License
----
Apache License 2.0

What it is
==========

The library q is not following, but lightly mimicing the [Promises/A+](http://promises-aplus.github.io/promises-spec/) specification, which provides a simplistic and straightforward API for deferring functions, and invoke completions asynchronously. The name is borrowed from the well-known JavaScript implementation with the [same name](http://github.com/kriskowal/q).

q provides, a part from the pure asynchronous IoC methods, a wide range of tools and helpers to make the use as simple and obvious as possible, while still being perfectly type safe. One example if this, is the automatic `std::tuple` expansion of *promise chains*.

The concept of q is that tasks are dispatched on queues, and a queue is attached to an `event_dispatcher`. q comes with its own `blocking_dispatcher`, which is like an event loop, to dispatch asynchronous tasks in order, but it is also easily bridged with existing event loops, such as native Win32, GTK, QT or Node.js. However, q also comes with a thread pool, which also is an `event_dispatcher`.

> One of the most important reasons to use q is that one can run tasks not only asynchronous, but also on different threads, without the need to use mutexes and other locks to isolate data, as one instead will perform certain tasks on certain queues which dispatches the tasks only on certain threads.



Introduction
============

For a C++ programmer, it is likely easiest to learn and to understand the library by some examples.

Asynchronous tasks
------------------

The following example shows how q can be used for networking.
```c++
q::promise< std::tuple< std::string, std::string > > read_message_from_someone()
{
    return connect_to_server( )
    .then( [ ]( connection& c )
    {
        return c.get_next_message( );
    } );
}

...

read_message_from_someone( )
.then( [ ]( std::string&& username, std::string&& msg )
{
    std::cout << "User " << username << " says: " << msg << std::endl;
} )
.fail( [ ]( const ConnectionException& e )
{
    std::cerr << "Connection problem: " << e << std::endl;
} )
.fail( [ ]( std::exception_ptr e )
{
    std::cerr << "Unknown error" << std::endl;
} );
```

Asynchronous termination
------------------------

Terminating an object means to put it in a state where it is not performing anything anymore, and can be freed. However, sometimes there are references to the object, or it is internally not done with its inner tasks. It could e.g. have a thread that is currently working. q comes with a class called `async_termination` which can be subclassed, to provide a smooth way of terminating objects and waiting for them to complete.
```c++
auto object = q::make_shared< my_class >( );
object->perform_background_task( );
object->terminate( )
.then( [ ]( )
{
    std::cout << "object has now completed and can safely be freed" << std::endl;
} );
```

Using threads without locks
---------------------------

Running a function on a newly created thread, "waiting" (asynchronously) for the function to complete, and then using the result in a function scheduled on another thread (or the main thread). Note that we don't need mutexes or semaphores.

```c++
q::run( "thread name", [ ]( )
{
    // Thread function which can perform heavy tasks
    return sort_strings( ); // Returns a vector of strings
} )
->terminate( ) // Will not really terminate, but rather await completion
.then( [ ]( std::vector< std::string >&& strings )
{
    // The result from the thread function is *moved* to this function
    std::cout << strings.size( ) << " strings are now sorted" << std::endl;
} );
```

Awating multiple asynchronously completed tasks
-----------------------------------------------

Lets say you have multiple (two or more) tasks which will complete promises and you want to await the result for all of them. This is done with `q::all( )` which combines the return values of the different promises and unpacks them as function arguments in the following `then( )` task.

```c++
q::promise< std::tuple< double > > a( );
q::promise< std::tuple< std::string, int > > b( );
q::promise< std::tuple< > > c( );
q::promise< std::tuple< std::vector< char > > > d( );

q::all( a( ), b( ), c( ), d( ) )
.fail( [ ]( std::exception_ptr e )
{
    // At least one of the functions failed, or returned a promise that failed.
    std::cerr << q::stream_exception( e ) << std::endl;
} )
.then( [ ]( double d, std::string&& s, int i, std::vector< char >&& v )
{
    // All 4 functions succeeded, as all promises completed successfully.
} );
```

A similar problem is having a variably sized set of *same-type* promises. Such can also be completed with `q::all( )`, although the signature for the following `then( )` task will be slightly different.

```c++
std::vector< q::promise< std::tuple< std::string, int > > > promises;

q::all( promises )
.fail( [ ]( q::combined_promise_exception< std::tuple< std::string > >&& e )
{
    // At least one promise failed.
    // The exception will contain information about which promises failed and which didn't.
} )
.then( [ ]( std::vector< std::tuple< std::string, int > >&& values )
{
    // Data from all promises is collected in one std::vector and *moved* to this function.
    // This means we can move it forward, and the data had never been copied.
    do_stuff( std::move( values ) );
} );
```

Installation
============

q uses CMake to generate build scripts.

Using Makefiles (for multiple platforms)
```sh
git clone git@github.com:grantila/q.git
cd q
./build.sh
```

For Xcode
```sh
git clone git@github.com:grantila/q.git
cd q
BUILDTYPE=Xcode ./build.sh ; open obj/q.xcodeproj
```
