.. _remote-spawn:

Remote Spawning of Actors  :sup:`experimental`
==============================================



Remote spawning is an extension of the dynamic spawn using run-time type names
add-custom-actor-type_. The following example assumes a typed actor handle
named ``calculator`` with an actor implementing this messaging
interface named "calculator".


.. code-block:: c++

   void client(actor_system& system, const config& cfg) {
     auto node = system.middleman().connect(cfg.host, cfg.port);
     if (!node) {
       cerr << "*** connect failed: "
            << system.render(node.error()) << endl;
       return;
     }
     auto type = "calculator"; // type of the actor we wish to spawn
     auto args = make_message(); // arguments to construct the actor
     auto tout = std::chrono::seconds(30); // wait no longer than 30s
     auto worker = system.middleman().remote_spawn<calculator>(*node, type,
                                                               args, tout);
     if (!worker) {
       cerr << "*** remote spawn failed: "
            << system.render(worker.error()) << endl;
       return;
     }
     // start using worker in main loop




We first connect to a CAF node with ``middleman().connect(...)``. On
success, ``connect`` returns the node ID we need for
``remote_spawn``. This requires the server to open a port with
``middleman().open(...)`` or ``middleman().publish(...)``.
Alternatively, we can obtain the node ID from an already existing remote actor
handle---returned from ``remote_actor`` for example---via
``hdl->node()``. After connecting to the server, we can use
``middleman().remote_spawn<...>(...)`` to create actors remotely.
