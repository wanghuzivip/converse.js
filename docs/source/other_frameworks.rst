Integrating converse.js into other frameworks
=============================================

Angular.js
----------

Angular.js has the concept of a `service <https://docs.angularjs.org/guide/services#!>`_,
which is a special kind of `provider <https://docs.angularjs.org/guide/providers>`_.

An angular.js service is a constructor or object which provides an API defined by the
writer of the service. The goal of a service is to organize and share code, so
that it can be used across an application.

So, if we wanted to properly integrate converse.js into an angular.js
application, then putting it into a service is a good approach.

This lets us avoid having a global ``converse`` API object (accessible via
``windows.converse``), and instead we can get hold of the converse API via
angular.js dependency injection, when we specify it as a dependency for our
angular components.

Below is an example code that wraps converse.js as an angular.js service.

.. code-block:: javascript

    angular.module('converse', []).service('converse', function() {
        var deferred = new $.Deferred(),
            promise = deferred.promise();

        var service = {
            'load': _.constant(promise),
            'initialize': function initConverse(options) {
                this.load().done(_.partial(this.api.initialize, options));
            }
        };

        define("converse", [
            "converse-api",
            // START: Removable components
            // --------------------
            // Any of the following components may be removed if they're not needed.
            "locales",               // Translations for converse.js. This line can be removed
                                     // to remove *all* translations, or you can modify the
                                     // file src/locales.js to include only those
                                     // translations that you care about.

            "converse-chatview",     // Renders standalone chat boxes for single user chat
            "converse-controlbox",   // The control box
            "converse-bookmarks",    // XEP-0048 Bookmarks
            "converse-mam",          // XEP-0313 Message Archive Management
            "converse-muc",          // XEP-0045 Multi-user chat
            "converse-vcard",        // XEP-0054 VCard-temp
            "converse-otr",          // Off-the-record encryption for one-on-one messages
            "converse-register",     // XEP-0077 In-band registration
            "converse-ping",         // XEP-0199 XMPP Ping
            "converse-notification", // HTML5 Notifications
            "converse-minimize",     // Allows chat boxes to be minimized
            "converse-dragresize",   // Allows chat boxes to be resized by dragging them
            "converse-headline",     // Support for headline messages
            // END: Removable components

        ], function(converse_api) {
            service.api = converse_api;
            return deferred.resolve();
        });
        require(["converse"]);
        return service;
    });

The above code is a modified version of the file `src/converse.js <https://github.com/jcbrand/converse.js/blob/master/src/converse.js>`_
which defines the converse AMD module and specifies which plugins will go into
this build.

You should replace the contents of that file with the above, if you want such a
service registered. Then, you should run `make build`, to create new build
files in the `dist` directory, containing your new angular.js service.

The above code registers an angular.js module and service, both named ``converse``.

This module should then be added as a dependency for your own angular.js
modules, for example:

.. code-block:: javascript

    angular.module('my-module', ['converse']);

Then you can have the converse service dependency injected into
your components, for example:

.. code-block:: javascript

    angular.module('my-module').provider('my-provider', function(converse) {
        // Your custom code can come here..

        // Then when you're ready, you can initialize converse.js
        converse.load().done(function () {
            converse.initialize({
                'allow_logout': false,
                'auto_login': 'true',
                'auto_reconnect': true,
                'bosh_service_url': bosh_url,
                'jid': bare_jid,
                'keepalive': true,
                'credentials_url': credentials_url,
            });

        // More custom code could come here...
        });
    });

You might have noticed the ``load()`` method being called on the ``converse``
service. This is a special method added to the service (see the implementation
example above) that makes sure that converse.js is loaded and available. It
returns a jQuery promise which resolves once converse.js is available.

This is necessary because with otherwise you might run into race-conditions
when your angular application loads more quickly then converse.js.

Lastly, the API of converse is available via the ``.api`` attribute on the service.
So you can call it like this for example:

.. code-block:: javascript

    converse.api.user.status.set('online');
