.. raw:: latex

   \definecolor{lightgrey}{rgb}{0.9,0.9,0.9}

.. raw:: latex

   \definecolor{lightblue}{rgb}{0,0,1}

.. raw:: latex

   \definecolor{grey}{rgb}{0.5,0.5,0.5}

.. raw:: latex

   \definecolor{blue}{rgb}{0,0,1}

.. raw:: latex

   \definecolor{violet}{rgb}{0.5,0,0.5}

.. raw:: latex

   \definecolor{darkred}{rgb}{0.5,0,0}

.. raw:: latex

   \definecolor{darkblue}{rgb}{0,0,0.5}

.. raw:: latex

   \definecolor{darkgreen}{rgb}{0,0.5,0}

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
    expected<typename infer_handle_from_fun<F>::type>
    spawn_client(F fun, const std::string& host, uint16_t port, Ts&&... xs);

    template <spawn_options Os = no_spawn_options,
              class F = std::function<void(broker*)>, class... Ts>
    expected<typename infer_handle_from_fun<F>::type>
    spawn_server(F fun, uint16_t port, Ts&&... xs);

The function ``spawn_broker`` simply spawns a broker. The convenience function ``spawn_client`` tries to connect to given host and port and returns a broker managing this connection on success. Finally, ``spawn_server`` opens a local port and spawns a broker managing it on success.

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

.. raw:: latex

   \clearpage

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

Closes a connection or port.

.. _broker-related-message-types:

Broker-related Message Types
----------------------------

Brokers receive system messages directly from the middleman for connection and acceptor events.

**Note:** brokers are *required* to handle these messages immediately regardless of their current state. Not handling the system messages from the broker results in loss of data, because system messages are *not* delivered through the mailbox and thus cannot be skipped.

::

    struct new_connection_msg {
      accept_handle source;
      connection_handle handle;
    };

Indicates that ``source`` accepted a new (TCP) connection identified by ``handle``.

::

    struct new_data_msg {
      connection_handle handle;
      std::vector<char> buf;
    };

Contains raw bytes received from ``handle``. The amount of data received per event is controlled with ``configure_read`` (see `1.2 <#broker-class>`__). It is worth mentioning that the buffer is re-used whenever possible.

::

    struct connection_closed_msg {
      connection_handle handle;
    };

    struct acceptor_closed_msg {
      accept_handle handle;
    };

A ``connection_closed_msg`` or ``acceptor_closed_msg`` informs the broker that one of it handles is no longer valid.

::

    struct connection_passivated_msg {
      connection_handle handle;
    };

    struct acceptor_passivated_msg {
      accept_handle handle;
    };

A ``connection_passivated_msg`` or ``acceptor_passivated_msg`` informs the broker that one of it handles entered passive mode and no longer accepts new data or connections (see § `1.4 <#trigger>`__).

.. _trigger:

Manually Triggering Events :sup:`experimental` 
-----------------------------------------------

Brokers receive new events as ``new_connection_msg`` and ``new_data_msg`` as soon and as often as they occur, per default. This means a fast peer can overwhelm a broker by sending it data faster than the broker can process it. In particular if the broker outsources work items to other actors, because work items can accumulate in the mailboxes of the workers.

Calling ``self->trigger(x, y)``, where ``x`` is a connection or acceptor handle and ``y`` is a positive integer, allows brokers to halt activities after ``y`` additional events. Once a connection or acceptor stops accepting new data or connections, the broker receives a ``connection_passivated_msg`` or ``acceptor_passivated_msg``.

Brokers can stop activities unconditionally with ``self->halt(x)`` and resume activities unconditionally with ``self->trigger(x)``.
