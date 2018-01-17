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

.. _using-aout-a-concurrency-safe-wrapper-for-cout:

Using ``aout`` – A Concurrency-safe Wrapper for ``cout``
========================================================

When using ``cout`` from multiple actors, output often appears interleaved. Moreover, using ``cout`` from multiple actors – and thus from multiple threads – in parallel should be avoided regardless, since the standard does not guarantee a thread-safe implementation.

By replacing ``std::cout`` with ``caf::aout``, actors can achieve a concurrency-safe text output. The header ``caf/all.hpp`` also defines overloads for ``std::endl`` and ``std::flush`` for ``aout``, but does not support the full range of ostream operations (yet). Each write operation to ``aout`` sends a message to a ‘hidden’ actor. This actor only prints lines, unless output is forced using ``flush``. The example below illustrates printing of lines of text from multiple actors (in random order).

::

    #include <random>
    #include <chrono>
    #include <cstdlib>
    #include <iostream>

    #include "caf/all.hpp"
    #include "caf/io/all.hpp"

    using namespace caf;
    using std::endl;

    void caf_main(actor_system& system) {
      for (int i = 1; i <= 50; ++i) {
        system.spawn([i](blocking_actor* self) {
          aout(self) << "Hi there! This is actor nr. "
                     << i << "!" << endl;
          std::random_device rd;
          std::default_random_engine re(rd());
          std::chrono::milliseconds tout{re() % 10};
          self->delayed_send(self, tout, 42);
          self->receive(
            [i, self](int) {
              aout(self) << "Actor nr. "
                         << i << " says goodbye!" << endl;
            }
          );
        });
      }
    }

    CAF_MAIN()
