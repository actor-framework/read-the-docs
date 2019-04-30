.. _overview:

Overview
========

Compiling CAF requires CMake and a C++11-compatible compiler. To get and compile the sources on UNIX-like systems, type the following in a terminal:

::

   git clone https://github.com/actor-framework/actor-framework
   cd actor-framework
   ./configure
   make
   make install [as root, optional]

We recommended to run the unit tests as well:

::

   make test

If the output indicates an error, please submit a bug report that includes (a) your compiler version, (b) your OS, and (c) the content of the file ``build/Testing/Temporary/LastTest.log``.

.. _features:

Features
--------

-  Lightweight, fast and efficient actor implementations

-  Network transparent messaging

-  Error handling based on Erlang’s failure model

-  Pattern matching for messages as internal DSL to ease development

-  Thread-mapped actors for soft migration of existing applications

-  Publish/subscribe group communication

.. _minimal-compiler-versions:

Minimal Compiler Versions
-------------------------

-  GCC 4.8

-  Clang 3.4

-  Visual Studio 2015, Update 3

.. _supported-operating-systems:

Supported Operating Systems
---------------------------

-  Linux

-  Mac OS X

-  Windows (static library only)

.. _hello-world-example:

Hello World Example
-------------------

.. code-block:: C++

   #include <string>
   #include <iostream>
   
   #include "caf/all.hpp"
   
   using std::endl;
   using std::string;
   
   using namespace caf;
   
   behavior mirror(event_based_actor* self) {
     // return the (initial) actor behavior
     return {
       // a handler for messages containing a single string
       // that replies with a string
       [=](const string& what) -> string {
         // prints "Hello World!" via aout (thread-safe cout wrapper)
         aout(self) << what << endl;
         // reply "!dlroW olleH"
         return string(what.rbegin(), what.rend());
       }
     };
   }
   
   void hello_world(event_based_actor* self, const actor& buddy) {
     // send "Hello World!" to our buddy ...
     self->request(buddy, std::chrono::seconds(10), "Hello World!").then(
       // ... wait up to 10s for a response ...
       [=](const string& what) {
         // ... and print it
         aout(self) << what << endl;
       }
     );
   }
   
   void caf_main(actor_system& system) {
     // create a new actor that calls 'mirror()'
     auto mirror_actor = system.spawn(mirror);
     // create another actor that calls 'hello_world(mirror_actor)';
     system.spawn(hello_world, mirror_actor);
     // system will wait until both actors are destroyed before leaving main
   }
   
   // creates a main function for us that calls our caf_main
   CAF_MAIN()


