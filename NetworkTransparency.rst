.. _middleman:

Middleman
=========

The middleman is the main component of the I/O module and enables distribution. It transparently manages proxy actor instances representing remote actors, maintains connections to other nodes, and takes care of serialization of messages. Applications install a middleman by loading ``caf::io::middleman`` as module (see :ref:`system-config`). Users can include ``"caf/io/all.hpp"`` to get access to all public classes of the I/O module.

.. _class-middleman:

Class ``middleman``
-------------------

+--------------------------------------------------------------+------------------------+
| **Remoting**                                                 |                        |
+==============================================================+========================+
| ``expected<uint16> publish(T, uint16, const char*, bool)``   | See :ref:`remoting`.   |
+--------------------------------------------------------------+------------------------+
| ``expected<void> unpublish(T x, uint16)``                    | See :ref:`remoting`.   |
+--------------------------------------------------------------+------------------------+
| ``expected<actor> remote_actor(string, uint16)``             | See :ref:`remoting`.   |
+--------------------------------------------------------------+------------------------+
| ``expected<T> typed_remote_actor(string, uint16)``           | See :ref:`remoting`.   |
+--------------------------------------------------------------+------------------------+
| ``expected<T> spawn_broker(F fun, ...)``                     | See :ref:`broker`.     |
+--------------------------------------------------------------+------------------------+
| ``expected<T> spawn_client(F, string, uint16, ...)``         | See :ref:`broker`.     |
+--------------------------------------------------------------+------------------------+
| ``expected<T> spawn_server(F, uint16, ...)``                 | See :ref:`broker`.     |
+--------------------------------------------------------------+------------------------+

.. _remoting:

Publishing and Connecting
-------------------------

The member function ``publish`` binds an actor to a given port, thereby allowing other nodes to access it over the network.

::

    template <class T>
    expected<uint16_t> middleman::publish(T x, uint16_t port,
                                          const char* in = nullptr,
                                          bool reuse_addr = false);

The first argument is a handle of type ``actor`` or ``typed_actor<...>``. The second argument denotes the TCP port. The OS will pick a random high-level port when passing 0. The third parameter configures the listening address. Passing null will accept all incoming connections (``INADDR_ANY``). Finally, the flag ``reuse_addr`` controls the behavior when binding an IP address to a port, with the same semantics as the BSD socket flag ``SO_REUSEADDR``. For example, with ``reuse_addr = false``, binding two sockets to 0.0.0.0:42 and 10.0.0.1:42 will fail with ``EADDRINUSE`` since 0.0.0.0 includes 10.0.0.1. With ``reuse_addr = true`` binding would succeed because 10.0.0.1 and 0.0.0.0 are not literally equal addresses.

The member function returns the bound port on success. Otherwise, an ``error`` (see :ref:`error`) is returned.

::

    template <class T>
    expected<uint16_t> middleman::unpublish(T x, uint16_t port = 0);

The member function ``unpublish`` allows actors to close a port manually. This is performed automatically if the published actor terminates. Passing 0 as second argument closes all ports an actor is published to, otherwise only one specific port is closed.

The function returns an ``error`` (see :ref:`error`) if the actor was not bound to given port.

::

    expected<actor> middleman::remote_actor(std::string host, uint16_t port);

    template <class ActorHandle>
    expected<ActorHandle> middleman::typed_remote_actor(std::string host, uint16_t port);

After a server has published an actor with ``publish``, clients can connect to the published actor by calling ``remote_actor`` or ``typed_remote_actor``:

::

    // node A
    auto ping = spawn(ping);
    system.middleman().publish(ping, 4242);

    // node B
    auto ping = system.middleman().remote_actor("node A", 4242);
    if (! ping) {
      cerr << "unable to connect to node A: "
           << system.render(ping.error()) << std::endl;
    } else {
      self->send(*ping, ping_atom::value);
    }

There is no difference between server and client after the connection phase. Remote actors use the same handle types as local actors and are thus fully transparent.
