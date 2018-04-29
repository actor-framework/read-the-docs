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

.. _registry:

Registry
========

The actor registry in CAF keeps track of the number of running actors and allows to map actors to their ID or a custom atom (see § \ `:ref:`atom` <#atom>`__) representing a name. The registry does *not* contain all actors. Actors have to be stored in the registry explicitly. Users can access the registry through an actor system by calling ``system.registry()``. The registry stores actors using ``strong_actor_ptr`` (see § `:ref:`actor-pointer` <#actor-pointer>`__).

Users can use the registry to make actors system-wide available by name. The middleman (see § \ `:ref:`middleman` <#middleman>`__) uses the registry to keep track of all actors known to remote nodes in order to serialize and deserialize them. Actors are removed automatically when they terminate.

It is worth mentioning that the registry is not synchronized between connected actor system. Each actor system has its own, local registry in a distributed setting.

.. raw:: latex

   \small

+--------------------------------------------+-------------------------------------------------+
| **Types**                                  |                                                 |
+============================================+=================================================+
| ``name_map``                               | ``unordered_map<atom_value, strong_actor_ptr>`` |
+--------------------------------------------+-------------------------------------------------+
|                                            |                                                 |
+--------------------------------------------+-------------------------------------------------+
| **Observers**                              |                                                 |
+--------------------------------------------+-------------------------------------------------+
| ``strong_actor_ptr get(actor_id)``         | Returns the actor associated to given ID.       |
+--------------------------------------------+-------------------------------------------------+
| ``strong_actor_ptr get(atom_value)``       | Returns the actor associated to given name.     |
+--------------------------------------------+-------------------------------------------------+
| ``name_map named_actors()``                | Returns all name mappings.                      |
+--------------------------------------------+-------------------------------------------------+
| ``size_t running()``                       | Returns the number of currently running actors. |
+--------------------------------------------+-------------------------------------------------+
|                                            |                                                 |
+--------------------------------------------+-------------------------------------------------+
| **Modifiers**                              |                                                 |
+--------------------------------------------+-------------------------------------------------+
| ``void put(actor_id, strong_actor_ptr)``   | Maps an actor to its ID.                        |
+--------------------------------------------+-------------------------------------------------+
| ``void erase(actor_id)``                   | Removes an ID mapping from the registry.        |
+--------------------------------------------+-------------------------------------------------+
| ``void put(atom_value, strong_actor_ptr)`` | Maps an actor to a name.                        |
+--------------------------------------------+-------------------------------------------------+
| ``void erase(atom_value)``                 | Removes a name mapping from the registry.       |
+--------------------------------------------+-------------------------------------------------+
