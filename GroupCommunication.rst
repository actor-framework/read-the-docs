.. _groups:

Group Communication
===================

CAF supports publish/subscribe-based group communication. Actors can join and leave groups and send messages to groups.

::

    actor_system system;
    std::string group_module = ...;
    std::string group_id = ...;
    auto grp = system.groups().get("local", "foo");
    scoped_actor self{system};
    self->join(grp);
    self->send(grp, "test");
    self->leave(grp);

.. _anonymous-group:

Anonymous Groups
----------------

Groups created on-the-fly with ``system.groups().anonymous()`` can be used to coordinate a set of workers. Each call to this function returns a new, unique group instance.

.. _local-group:

Local Groups
------------

The ``"local"`` group module creates groups for in-process communication. For example, a group for GUI related events could be identified by ``system.groups().get("local", "GUI events")``. The group ID ``"GUI events"`` uniquely identifies a singleton group instance of the module ``"local"``.

.. _remote-group:

Remote Groups
-------------

Remote groups are available only if a middleman (see :ref:`middleman`) is loaded.

Calling ``system.middleman().publish_local_groups(port, addr)`` makes a group available to other nodes in the network. The first argument denotes the port, while the second (optional) parameter can be used to whitelist IP addresses.

After publishing the group at one node, other nodes can get a handle for that group by using the “remote” module: ``system.middleman().get("remote", "<group>@<host>:<port>")``. This implementation uses N-times unicast underneath and the group is only available as long as the hosting server is alive.
