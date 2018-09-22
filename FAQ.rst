.. _faq:

Frequently Asked Questions
==========================

This Section is a compilation of the most common questions via GitHub, chat, and mailing list.

.. _can-i-encrypt-caf-communication:

Can I Encrypt CAF Communication?
--------------------------------

Yes, by using the OpenSSL module .

.. _can-i-create-messages-dynamically:

Can I Create Messages Dynamically?
----------------------------------

Yes.

Usually, messages are created implicitly when sending messages but can also be created explicitly using ``make_message``. In both cases, types and number of elements are known at compile time. To allow for fully dynamic message generation, CAF also offers ``message_builder``:

::

   message_builder mb;
   // prefix message with some atom
   mb.append(strings_atom::value);
   // fill message with some strings
   std::vector<std::string> strings{/*...*/};
   for (auto& str : strings)
     mb.append(str);
   // create the message
   message msg = mb.to_message();

.. _what-debugging-tools-exist:

What Debugging Tools Exist?
---------------------------

The ``scripts/`` and ``tools/`` directories contain some useful tools to aid in development and debugging.

``scripts/atom.py`` converts integer atom values back into strings.

``scripts/demystify.py`` replaces cryptic ``typed_mpi<...>`` templates with ``replies_to<...>::with<...>`` and ``atom_constant<...>`` with a human-readable representation of the actual atom.

``scripts/caf-prof`` is an R script that generates plots from CAF profiler output.

``caf-vec`` is a (highly) experimental tool that annotates CAF logs with vector timestamps. It gives you happens-before relations and a nice visualization via `ShiViz <https://bestchai.bitbucket.io/shiviz/>`__.
