The converse.js developer API
=============================

.. note:: The API documented here is available in Converse.js 0.8.4 and higher.
        Earlier versions of Converse.js might have different API methods or none at all.

In the Converse.js API, you traverse towards a logical grouping, from
which you can then call certain standardised accessors and mutators, such as::

    .get
    .set
    .add
    .remove

This is done to increase readability and to allow intuitive method chaining.

For example, to get a contact, you would do the following::

    converse.contacts.get('jid@example.com');

To get multiple contacts, just pass in an array of jids::

    converse.contacts.get(['jid1@example.com', 'jid2@example.com']);

To get all contacts, simply call ``get`` without any jids::

    converse.contacts.get();


**Here follows now a breakdown of all API groupings and methods**:


initialize
----------

.. note:: This method is the one exception of a method which is not logically grouped
    as explained above.

Initializes converse.js. This method must always be called when using
converse.js.

The `initialize` method takes a map (also called a hash or dictionary) of :ref:`configuration-variables`.

Example:

.. code-block:: javascript

    converse.initialize({
            allow_otr: true,
            auto_list_rooms: false,
            auto_subscribe: false,
            bosh_service_url: 'https://bind.example.com',
            hide_muc_server: false,
            i18n: locales['en'],
            keepalive: true,
            play_sounds: true,
            prebind: false,
            show_controlbox_by_default: true,
            debug: false,
            roster_groups: true
        });

send
----

Allows you to send XML stanzas.

For example, to send a message stanza:

.. code-block:: javascript

    var msg = converse.env.$msg({
        from: 'juliet@example.com/balcony',
        to:'romeo@example.net',
        type:'chat'
    });
    converse.send(msg);


The "archive" grouping
----------------------

Converse.js supports the *Message Archive Management*
(`XEP-0313 <https://xmpp.org/extensions/xep-0313.html>`_) protocol,
through which it is able to query an XMPP server for archived messages.

See also the **message_archiving** option in the :ref:`configuration-variables` section, which you'll usually
want to  in conjunction with this API.

query
~~~~~

The ``query`` method is used to query for archived messages.

It accepts the following optional parameters:

* **options** an object containing the query parameters. Valid query parameters
  are ``with``, ``start``, ``end``, ``first``, ``last``, ``after``, ``before``, ``index`` and ``count``.
* **callback** is the callback method that will be called when all the messages
  have been received.
* **errback** is the callback method to be called when an error is returned by
  the XMPP server, for example when it doesn't support message archiving.

Examples
^^^^^^^^

**Requesting all archived messages**

The simplest query that can be made is to simply not pass in any parameters.
Such a query will return all archived messages for the current user.

Generally, you'll however always want to pass in a callback method, to receive
the returned messages.

.. code-block:: javascript

    var errback = function (iq) {
        // The query was not successful, perhaps inform the user?
        // The IQ stanza returned by the XMPP server is passed in, so that you
        // may inspect it and determine what the problem was.
    }
    var callback = function (messages) {
        // Do something with the messages, like showing them in your webpage.
    }
    converse.archive.query(callback, errback))


**Waiting until server support has been determined**

The query method will only work if converse.js has been able to determine that
the server supports MAM queries, otherwise the following error will be raised:

- *This server does not support XEP-0313, Message Archive Management*

The very first time converse.js loads in a browser tab, if you call the query
API too quickly, the above error might appear because service discovery has not
yet been completed.

To work solve this problem, you can first listen for the ``serviceDiscovered`` event,
through which you can be informed once support for MAM has been determined.

For example:

.. code-block:: javascript

    converse.listen.on('serviceDiscovered', function (event, feature) {
        if (feature.get('var') === converse.env.Strophe.NS.MAM) {
            converse.archive.query()
        }
    });

**Requesting all archived messages for a particular contact or room**

To query for messages sent between the current user and another user or room,
the query options need to contain the the JID (Jabber ID) of the user or
room under the  ``with`` key.

.. code-block:: javascript

    // For a particular user
    converse.archive.query({'with': 'john@doe.net'}, callback, errback);)

    // For a particular room
    converse.archive.query({'with': 'discuss@conference.doglovers.net'}, callback, errback);)


**Requesting all archived messages before or after a certain date**

The ``start`` and ``end`` parameters are used to query for messages
within a certain timeframe. The passed in date values may either be ISO8601
formatted date strings, or Javascript Date objects.

.. code-block:: javascript

    var options = {
        'with': 'john@doe.net',
        'start': '2010-06-07T00:00:00Z',
        'end': '2010-07-07T13:23:54Z'
    };
    converse.archive.query(options, callback, errback);


**Limiting the amount of messages returned**

The amount of returned messages may be limited with the ``max`` parameter.
By default, the messages are returned from oldest to newest.

.. code-block:: javascript

    // Return maximum 10 archived messages
    converse.archive.query({'with': 'john@doe.net', 'max':10}, callback, errback);


**Paging forwards through a set of archived messages**

When limiting the amount of messages returned per query, you might want to
repeatedly make a further query to fetch the next batch of messages.

To simplify this usecase for you, the callback method receives not only an array
with the returned archived messages, but also a special RSM (*Result Set
Management*) object which contains the query parameters you passed in, as well
as two utility methods ``next``, and ``previous``.

When you call one of these utility methods on the returned RSM object, and then
pass the result into a new query, you'll receive the next or previous batch of
archived messages. Please note, when calling these methods, pass in an integer
to limit your results.

.. code-block:: javascript

    var callback = function (messages, rsm) {
        // Do something with the messages, like showing them in your webpage.
        // ...
        // You can now use the returned "rsm" object, to fetch the next batch of messages:
        converse.archive.query(rsm.next(10), callback, errback))

    }
    converse.archive.query({'with': 'john@doe.net', 'max':10}, callback, errback);

**Paging backwards through a set of archived messages**

To page backwards through the archive, you need to know the UID of the message
which you'd like to page backwards from and then pass that as value for the
``before`` parameter. If you simply want to page backwards from the most recent
message, pass in the ``before`` parameter with an empty string value ``''``.

.. code-block:: javascript

    converse.archive.query({'before': '', 'max':5}, function (message, rsm) {
        // Do something with the messages, like showing them in your webpage.
        // ...
        // You can now use the returned "rsm" object, to fetch the previous batch of messages:
        rsm.previous(5); // Call previous method, to update the object's parameters,
                         // passing in a limit value of 5.
        // Now we query again, to get the previous batch.
        converse.archive.query(rsm, callback, errback);
    }

The "connection" grouping
-------------------------

This grouping collects API functions related to the XMPP connection.

connected
~~~~~~~~~

A boolean attribute (i.e. not a callable) which is set to `true` or `false` depending
on whether there is an established connection.

disconnect
~~~~~~~~~~

Terminates the connection.


The "user" grouping
-------------------

This grouping collects API functions related to the current logged in user.

jid
~~~

Return's the current user's full JID (Jabber ID).

.. code-block:: javascript

    converse.user.jid()
    // Returns for example jc@opkode.com/conversejs-351236

login
~~~~~

Logs the user in. This method can accept a map with the credentials, like this:

.. code-block:: javascript

    converse.user.login({
        'jid': 'dummy@example.com',
        'password': 'secret'
    });

or it can be called without any parameters, in which case converse.js will try
to log the user in by calling the `prebind_url` or `credentials_url` depending
on whether prebinding is used or not.

logout
~~~~~~

Log the user out of the current XMPP session.

.. code-block:: javascript

    converse.user.logout();


The "status" sub-grouping
~~~~~~~~~~~~~~~~~~~~~~~~~

Set and get the user's chat status, also called their *availability*.

get
^^^

Return the current user's availability status:

.. code-block:: javascript

    converse.user.status.get(); // Returns for example "dnd"

set
^^^

The user's status can be set to one of the following values:

* **away**
* **dnd**
* **offline**
* **online**
* **unavailable**
* **xa**

For example:

.. code-block:: javascript

    converse.user.status.set('dnd');

Because the user's availability is often set together with a custom status
message, this method also allows you to pass in a status message as a
second parameter:

.. code-block:: javascript

    converse.user.status.set('dnd', 'In a meeting');

The "message" sub-grouping
^^^^^^^^^^^^^^^^^^^^^^^^^^

The ``user.status.message`` sub-grouping exposes methods for setting and
retrieving the user's custom status message.

.. code-block:: javascript

    converse.user.status.message.set('In a meeting');

    converse.user.status.message.get(); // Returns "In a meeting"


The "contacts" grouping
-----------------------

get
~~~

This method is used to retrieve roster contacts.

To get a single roster contact, call the method with the contact's JID (Jabber ID):

.. code-block:: javascript

    converse.contacts.get('buddy@example.com')

To get multiple contacts, pass in an array of JIDs:

.. code-block:: javascript

    converse.contacts.get(['buddy1@example.com', 'buddy2@example.com'])

To return all contacts, simply call ``get`` without any parameters:

.. code-block:: javascript

    converse.contacts.get()


The returned roster contact objects have these attributes:

+----------------+-----------------------------------------------------------------------------------------------------------------+
| Attribute      |                                                                                                                 |
+================+=================================================================================================================+
| ask            | If ask === 'subscribe', then we have asked this person to be our chat buddy.                                    |
+----------------+-----------------------------------------------------------------------------------------------------------------+
| fullname       | The person's full name.                                                                                         |
+----------------+-----------------------------------------------------------------------------------------------------------------+
| jid            | The person's Jabber/XMPP username.                                                                              |
+----------------+-----------------------------------------------------------------------------------------------------------------+
| requesting     | If true, then this person is asking to be our chat buddy.                                                       |
+----------------+-----------------------------------------------------------------------------------------------------------------+
| subscription   | The subscription state between the current user and this chat buddy. Can be `none`, `to`, `from` or `both`.     |
+----------------+-----------------------------------------------------------------------------------------------------------------+
| id             | A unique id, same as the jid.                                                                                   |
+----------------+-----------------------------------------------------------------------------------------------------------------+
| chat_status    | The person's chat status. Can be `online`, `offline`, `busy`, `xa` (extended away) or `away`.                   |
+----------------+-----------------------------------------------------------------------------------------------------------------+
| user_id        | The user id part of the JID (the part before the `@`).                                                          |
+----------------+-----------------------------------------------------------------------------------------------------------------+
| resources      | The known resources for this chat buddy. Each resource denotes a separate and connected chat client.            |
+----------------+-----------------------------------------------------------------------------------------------------------------+
| groups         | The roster groups in which this chat buddy was placed.                                                          |
+----------------+-----------------------------------------------------------------------------------------------------------------+
| status         | Their human readable custom status message.                                                                     |
+----------------+-----------------------------------------------------------------------------------------------------------------+
| image_type     | The image's file type.                                                                                          |
+----------------+-----------------------------------------------------------------------------------------------------------------+
| image          | The Base64 encoded image data.                                                                                  |
+----------------+-----------------------------------------------------------------------------------------------------------------+
| url            | The buddy's website URL, as specified in their VCard data.                                                      |
+----------------+-----------------------------------------------------------------------------------------------------------------+
| vcard_updated  | When last the buddy's VCard was updated.                                                                        |
+----------------+-----------------------------------------------------------------------------------------------------------------+

add
~~~

Add a contact.

Provide the JID of the contact you want to add:

.. code-block:: javascript

    converse.contacts.add('buddy@example.com')

You may also provide the fullname. If not present, we use the jid as fullname:

.. code-block:: javascript

    converse.contacts.add('buddy@example.com', 'Buddy')

The "chats" grouping
--------------------

Note, for MUC chat rooms, you need to use the "rooms" grouping instead.

get
~~~

Returns an object representing a chat box.

To return a single chat box, provide the JID of the contact you're chatting
with in that chat box:

.. code-block:: javascript

    converse.chats.get('buddy@example.com')

To return an array of chat boxes, provide an array of JIDs:

.. code-block:: javascript

    converse.chats.get(['buddy1@example.com', 'buddy2@example.com'])

To return all open chat boxes, call the method without any JIDs::

    converse.chats.get()

open
~~~~

Opens a chat box and returns an object representing a chat box.

To open a single chat box, provide the JID of the contact:

.. code-block:: javascript

    converse.chats.open('buddy@example.com')

To return an array of chat boxes, provide an array of JIDs:

.. code-block:: javascript

    converse.chats.open(['buddy1@example.com', 'buddy2@example.com'])


*The returned chat box object contains the following methods:*

+-------------+------------------------------------------+
| Method      | Description                              |
+=============+==========================================+
| endOTR      | End an OTR (Off-the-record) session.     |
+-------------+------------------------------------------+
| get         | Get an attribute (i.e. accessor).        |
+-------------+------------------------------------------+
| initiateOTR | Start an OTR (off-the-record) session.   |
+-------------+------------------------------------------+
| maximize    | Minimize the chat box.                   |
+-------------+------------------------------------------+
| minimize    | Maximize the chat box.                   |
+-------------+------------------------------------------+
| set         | Set an attribute (i.e. mutator).         |
+-------------+------------------------------------------+
| close       | Close the chat box.                      |
+-------------+------------------------------------------+
| open        | Opens the chat box.                      |
+-------------+------------------------------------------+

*The get and set methods can be used to retrieve and change the following attributes:*

+-------------+-----------------------------------------------------+
| Attribute   | Description                                         |
+=============+=====================================================+
| height      | The height of the chat box.                         |
+-------------+-----------------------------------------------------+
| url         | The URL of the chat box heading.                    |
+-------------+-----------------------------------------------------+

The "rooms" grouping
--------------------

get
~~~

Returns an object representing a multi user chat box (room).
It takes 3 parameters:

* the room JID (if not specified, all rooms will be returned).
* a map (object) containing any extra room attributes For example, if you want
  to specify the nickname, use ``{'nick': 'bloodninja'}``. Previously (before
  version 1.0.7, the second parameter only accepted the nickname (as a string
  value). This is currently still accepted, but then you can't pass in any
  other room attributes. If the nickname is not specified then the node part of
  the user's JID will be used.
* a boolean, indicating whether the room should be created if not found (default: `false`)

.. code-block:: javascript

    var nick = 'dread-pirate-roberts';
    var create_if_not_found = true;
    converse.rooms.open('group@muc.example.com', {'nick': nick}, create_if_not_found)

open
~~~~

Opens a multi user chat box and returns an object representing it.
Similar to chats.get API

It takes 2 parameters:

* the room JID (if not specified, all rooms will be returned).
* a map (object) containing any extra room attributes. For example, if you want
  to specify the nickname, use ``{'nick': 'bloodninja'}``.

To open a single multi user chat box, provide the JID of the room:

.. code-block:: javascript

    converse.rooms.open('group@muc.example.com')

To return an array of rooms, provide an array of room JIDs:

.. code-block:: javascript

    converse.rooms.open(['group1@muc.example.com', 'group2@muc.example.com'])

To setup a custom nickname when joining the room, provide the optional nick argument:

.. code-block:: javascript

    converse.rooms.open('group@muc.example.com', {'nick': 'mycustomnick'})

close
~~~~~

Lets you close open chat rooms. You can call this method without any arguments
to close all open chat rooms, or you can specify a single JID or an array of
JIDs.

The "settings" grouping
-----------------------

This grouping allows you to get or set the configuration settings of converse.js.

get(key)
~~~~~~~~

Returns the value of a configuration settings. For example:

.. code-block:: javascript

    converse.settings.get("play_sounds"); // default value returned would be false;

set(key, value) or set(object)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Set one or many configuration settings. For example:

.. code-block:: javascript

    converse.settings.set("play_sounds", true);

or :

.. code-block:: javascript

    converse.settings.set({
        "play_sounds", true,
        "hide_offline_users" true
    });

Note, this is not an alternative to calling ``converse.initialize``, which still needs
to be called. Generally, you'd use this method after converse.js is already
running and you want to change the configuration on-the-fly.

The "tokens" grouping
---------------------

get
~~~

Returns a token, either the RID or SID token depending on what's asked for.

Example:

.. code-block:: javascript

    converse.tokens.get('rid')


.. _`listen-grouping`:

The "listen" grouping
---------------------

Converse.js emits events to which you can subscribe from your own Javascript.

Concerning events, the following methods are available under the "listen"
grouping:

* **on(eventName, callback, [context])**:

    Calling the ``on`` method allows you to subscribe to an event.
    Every time the event fires, the callback method specified by ``callback`` will be
    called.

    Parameters:

    * ``eventName`` is the event name as a string.
    * ``callback`` is the callback method to be called when the event is emitted.
    * ``context`` (optional), the value of the `this` parameter for the callback.

    For example:

.. code-block:: javascript

        converse.listen.on('message', function (event, messageXML) { ... });

* **once(eventName, callback, [context])**:

    Calling the ``once`` method allows you to listen to an event
    exactly once.

    Parameters:

    * ``eventName`` is the event name as a string.
    * ``callback`` is the callback method to be called when the event is emitted.
    * ``context`` (optional), the value of the `this` parameter for the callback.

    For example:

.. code-block:: javascript

        converse.listen.once('message', function (event, messageXML) { ... });

* **not(eventName, callback)**

    To stop listening to an event, you can use the ``not`` method.

    Parameters:

    * ``eventName`` is the event name as a string.
    * ``callback`` refers to the function that is to be no longer executed.

    For example:

.. code-block:: javascript

        converse.listen.not('message', function (event, messageXML) { ... });

