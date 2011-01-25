HTML mode
=========

.. versionadded:: 1.2

The DOMBuilder API can also be used to generate HTML without having to engage
in extensive ``String`` wrangling. The type of output it generates is controlled
by the :js:attr:`DOMBuilder.mode` attribute.

.. js:attribute:: DOMBuilder.mode

   Determines which kind of objects :js:func:`DOMBuilder.createElement` will
   create.

   The allowable values are:

   +-------------+--------------------------------------------------------------------------+
   | Value       | Output                                                                   |
   +=============+==========================================================================+
   | ``"DOM"``   | DOM elements (default value)                                             |
   +-------------+--------------------------------------------------------------------------+
   | ``"HTML"``  | :js:class:`DOMBuilder.HTMLElement` objects which ``toString()`` to HTML4 |
   +-------------+--------------------------------------------------------------------------+
   | ``"XHTML"`` | :js:class:`DOMBuilder.HTMLElement` objects which ``toString()`` to XHTML |
   +-------------+--------------------------------------------------------------------------+

Yes, that is pretty ugly, but the majority of your usage will depend on the
environment your JavaScript is executing in. If you're in the browser, you're
more likely to want to create DOM elements which are easy to attach event
handlers to, while on the backend you'll probably stick exclusively to one
of the HTML modes.

Of course, there are plenty of scenarios where you would want to generate
HTML in the browser. For example, inserting new content using ``innerHTML``
can be a lot faster than using the DOM methods in scenarios where none of
its limitations or side-effects apply.

To change to HTML mode, set :js:attr:`DOMBuilder.mode` to the appropriate
type of HTML output you want and use DOMBuilder as normal.

.. _mock-dom-objects:

Mock DOM Objects
~~~~~~~~~~~~~~~~

In HTML mode, DOMBuilder will create mock DOM objects which implement a
very small subset of the ``Node`` operations available on their real
counterparts.

With foreknowledge of the available operations (and `requests for additional
operations`_ which would be useful), it's possible to write complex content
creation code which works seamlessly in both DOM and HTML modes.

.. _`requests for additional operations`: http://code.google.com/p/dombuilder/issues/list

Element creation functions will create :js:class:`DOMBuilder.HTMLElement`
objects. Calling the :js:class:`DOMBuilder.HTMLElement.toString()` method
on these objects will produce the appropriate type of HTML based on the
mode at the time they were created.

.. js:class:: DOMBuilder.HTMLElement(tagName[, attributes[, childNodes]])

   A representation of a DOM ``Element``, its attributes and child nodes.

   Arguments are as per :js:func:`DOMBuilder.createElement`.

   .. versionchanged:: 1.3
      Renamed from "Tag" to "HTMLElement"

.. js:function:: DOMBuilder.HTMLElement.appendChild(node)

   Adds to the list of child nodes, for cases where the desired structure
   cannot be built up at creation time.

   .. versionchanged:: 1.3
      Appending a :js:class:`DOMBuilder.HTMLFragment` will append its
      child nodes and clear them from the fragment.

.. js:function:: DOMBuilder.HTMLElement.cloneNode(deep)

   Clones the tag and its attributes - if deep is ``true`` its child nodes
   will also be cloned.

   .. versionadded:: 1.3
      Added to support cloning by an :js:class:`DOMBuilder.HTMLFragment`.

.. js:function:: DOMBuilder.HTMLElement.toString()

   Creates a ``String`` containing the HTML representation of the tag and
   its children. By default, any ``String`` children will be escaped to
   prevent the use of sensitive HTML characters - see the `Escaping`_
   section for details on controlling escaping.

:js:func:`DOMBuilder.fragment` will create :js:class:`DOMBuilder.HTMLFragment`
objects which mimic the behaviour of DOM ``DocumentFragment`` when appended
to another fragment or a :js:class:`DOMBuilder.HTMLElement`.

.. js:class:: DOMBuilder.HTMLFragment([childNodes])

   A representation of a DOM ``DocumentFragment`` and its child nodes.

   :param Array children: initial child nodes

   .. versionadded:: 1.3

.. js:function:: DOMBuilder.HTMLFragment.appendChild(node)

   Adds to the list of child nodes - appending another fragment will
   append its child nodes and clear them from the fragment.

.. js:function:: DOMBuilder.HTMLFragment.cloneNode(deep)

   Clones the fragment - if deep is ``true``, its child nodes will also
   be cloned.

Temporarily Switching Mode
~~~~~~~~~~~~~~~~~~~~~~~~~~

If you're going to be working with mixed output types, forgetting to reset
:js:attr:`DOMBuilder.mode` would be catastrophic, so DOMBuilder provides
:js:func:`DOMBuilder.withNode` to manage it for you.

.. js:function:: DOMBuilder.withNode(mode, func)

   Calls a function, with :js:attr:`DOMBuilder.mode` set to the given value
   for the duration of the function call, and returns its output.

The following `FireBug`_ console session shows :js:func:`DOMBuilder.withNode` in action::

    >>> function createParagraph() { return P("Bed and", BR(), "BReakfast"); }
    >>> createParagraph().toString() // DOM mode by default
    "[object HTMLParagraphElement]"
    >>> DOMBuilder.withMode("HTML", createParagraph).toString();
    "<p>Bed and<br>BReakfast</p>"
    >>> DOMBuilder.withMode("XHTML", createParagraph).toString();
    "<p>Bed and<br />BReakfast</p>"
    >>> DOMBuilder.withMode("HTML", function() { return createParagraph() + " " + DOMBuilder.withMode("XHTML", createParagraph); })
    "<p>Bed and<br>BReakfast</p> <p>Bed and<br />BReakfast</p>"

.. _Firebug: http://www.getfirebug.com

Escaping
~~~~~~~~

HTML mode was initially introduced with backend use in mind - specifically,
for generating forms and working with user input. As such, autoescaping was
implemented to protect the developer from malicious user input. The same can
still apply on the frontend, so :js:func:`DOMBuilder.HTMLElement.toString`
automatically escapes the following characters in any ``String`` contents it
finds, replacing them with their equivalent HTML entities::

   < > & ' "

If you have a ``String`` which is known to be safe for inclusion without
escaping, pass it through :js:func:`DOMBuilder.markSafe` before adding it
to a :js:class:`DOMBuilder.HTMLElement`.

.. js:function:: DOMBuilder.markSafe(value)

   :param String value: A known-safe string.
   :returns: A ``SafeString`` object.

There is also a corresponding method to determine if a ``String`` is
already marked as safe.

.. js:function:: DOMBuilder.isSafe(value)

   :returns: ``true`` if the given ``String`` is marked as safe, ``false``
       otherwise.

Assuming we're in HTML mode, this example shows how autoescaping deals with
malicious input::

   >>> var input = "<span style=\"font-size: 99999px;\" onhover=\"location.href='whereveriwant'\">Free money!</span>";
   >>> P("Steve the dog says: ", input).toString()
   "<p>Steve the dog says: &lt;span style=&quot;font-size: 99999px;&quot; onhover=&quot;location.href=&#39;whereveriwant&#39;&quot;&gt;Free money!&lt;/span&gt;</p>"

But say you have a ``String`` containing HTML which you trust and do want to
render, like a status message you've just created, or an ``XMLHTTPRequest``
response::

   >>> var response = "You have <strong>won the internet!</strong>";
   >>> P("According to our experts: ", response).toString()
   "<p>According to our experts: You have &lt;strong&gt;won the internet!&lt;/strong&gt;</p>"
   >>> P("According to our experts: ", DOMBuilder.markSafe(response)).toString()
   "<p>According to our experts: You have <strong>won the internet!</strong></p>"

.. warning::

   ``String`` operations performed on a ``String`` which was marked safe will
   produce a ``String`` which is no longer marked as safe.

To avoid accidentally removing safe status from a ``String``, try not to mark it
safe until it's ready for use::

   >>> var response = "<span style=\"font-family: Comic Sans MS\">Your money is safe with us!</span>";
   >>> function tasteFilter(s) { return s.replace(/Comic Sans MS/gi, "Verdana"); }
   >>> var safeResponse = DOMBuilder.markSafe(response);
   >>> P("Valued customer: ", safeResponse).toString()
   "<p>Valued customer: <span style="font-family: Comic Sans MS">Your money is safe with us!</span></p>"
   >>> P("Valued customer: ", tasteFilter(safeResponse)).toString()
   "<p>Valued customer: &lt;span style=&quot;font-family: Verdana&quot;&gt;Your money is safe with us!&lt;/span&gt;</p>"