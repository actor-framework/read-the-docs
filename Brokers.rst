.. _broker:

Network I/O with Brokers
========================

When communicating to other services in the network, sometimes low-level socket I/O is inevitable. For this reason, CAF provides *brokers*. A broker is an event-based actor running in the middleman that multiplexes socket I/O. It can maintain any number of acceptors and connections. Since the broker runs in the middleman, implementations should be careful to consume as little time as possible in message handlers. Brokers should outsource any considerable amount of work by spawning new actors or maintaining worker actors.

.. _spawning-brokers:

Spawning Brokers
----------------

Brokers are implemented as functions and are spawned by calling on of the three following member functions of the middleman.

::

    template <spawn_options Os = no_spawn_options,
              class F = std::function<void(broker*)>, class... Ts>
    typename infer_handle_from_fun<F>::type
    spawn_broker(F fun, Ts&&... xs);

    template <spawn_options Os = no_spawn_options,
              class F = std::function<void(broker*)>, class... Ts>
    typename infer_handle_from_fun<F>::type
    spawn_client(F fun, const std::string& host, uint16_t port, Ts&&... xs);

    template <spawn_options Os = no_spawn_options,
              class F = std::function<void(broker*)>, class... Ts>
    typename infer_handle_from_fun<F>::type
    spawn_server(F fun, uint16_t port, Ts&&... xs);

The function ``spawn_broker`` simply spawns a broker. The convenience function ``spawn_client`` spawns a broker and immediately connects it to given host and port. Finally, ``spawn_server`` immediately adds an acceptor for the given port to the new broker.

.. _broker-class:

Class ``broker``
----------------

::

    void configure_read(connection_handle hdl, receive_policy::config config)

Modifies the receive policy for the connection identified by ``hdl``. This will cause the middleman to enqueue the next ``new_data_msg`` according to the given ``config`` created by ``receive_policy::exactly(x)``, ``receive_policy::at_most(x)``, or ``receive_policy::at_least(x)`` (with ``x`` denoting the number of bytes).

::

    void write(connection_handle hdl, size_t num_bytes, const void* buf)

Writes data to the output buffer.

::

    void flush(connection_handle hdl)

Sends the data from the output buffer.

::

    template <class F, class... Ts>
    actor fork(F fun, connection_handle hdl, Ts&&... xs)

Spawns a new broker that takes ownership of given connection.

::

    size_t num_connections()

Returns the number of open connections.

::

    void close(connection_handle hdl)
    void close(accept_handle hdl)

Closes a connection or acceptor.

.. _broker-related-message-types:

Broker-related Message Types
----------------------------

Brokers receive system messages directly from the middleman whenever an event on one of it handles occurs.

::

    struct new_connection_msg {
      accept_handle source;
      connection_handle handle;
    };

Whenever a new incoming connection (identified by the ``handle`` field) has been accepted for one of the broker’s accept handles, it will receive a ``new_connection_msg``.

::

    struct new_data_msg {
      connection_handle handle;
      std::vector<char> buf;
    };

New incoming data is transmitted to the broker using messages of type ``new_data_msg``. The raw bytes can be accessed as buffer object of type ``std::vector<char>``. The amount of data, i.e., how often this message is received, can be controlled using ``configure_read`` (see :ref:`broker-class])` It is worth mentioning that the buffer is re-used whenever possible. This means, as long as the broker does not create any new references to the message by copying it, the middleman will always use only a single buffer per connection.

::

    struct connection_closed_msg {
      connection_handle handle;
    };

::

    struct acceptor_closed_msg {
      accept_handle handle;
    };

A ``connection_closed_msg`` or `` acceptor_closed_msg`` informs the broker that one of it handles is no longer valid.
