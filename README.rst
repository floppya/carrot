##############################################
 carrot - AMQP Messaging Framework for Python
##############################################

:Version: 0.10.4

**NOTE** This release contains backward-incompatible changes.
Please read the `Changelog`_ for more information.

.. _`Changelog`: http://ask.github.com/carrot/changelog.html


Introduction
------------

`carrot` is an `AMQP`_ messaging queue framework. AMQP is the Advanced Message
Queuing Protocol, an open standard protocol for message orientation, queuing,
routing, reliability and security.

The aim of `carrot` is to make messaging in Python as easy as possible by
providing a high-level interface for producing and consuming messages. At the
same time it is a goal to re-use what is already available as much as possible.

`carrot` has pluggable messaging back-ends, so it is possible to support
several messaging systems. Currently, there is support for `AMQP`_
(`py-amqplib`_, `pika`_), `STOMP`_ (`python-stomp`_). There's also an
in-memory backend for testing purposes, using the `Python queue module`_.

Several AMQP message broker implementations exists, including `RabbitMQ`_,
`ZeroMQ`_ and `Apache ActiveMQ`_. You'll need to have one of these installed,
personally we've been using `RabbitMQ`_.

Before you start playing with ``carrot``, you should probably read up on
AMQP, and you could start with the excellent article about using RabbitMQ
under Python, `Rabbits and warrens`_. For more detailed information, you can
refer to the `Wikipedia article about AMQP`_.

.. _`RabbitMQ`: http://www.rabbitmq.com/
.. _`ZeroMQ`: http://www.zeromq.org/
.. _`AMQP`: http://amqp.org
.. _`STOMP`: http://stomp.codehaus.org
.. _`python-stomp`: http://bitbucket.org/asksol/python-stomp
.. _`Python Queue module`: http://docs.python.org/library/queue.html
.. _`Apache ActiveMQ`: http://activemq.apache.org/
.. _`Django`: http://www.djangoproject.com/
.. _`Rabbits and warrens`: http://blogs.digitar.com/jjww/2009/01/rabbits-and-warrens/
.. _`py-amqplib`: http://barryp.org/software/py-amqplib/
.. _`pika`: http://github.com/tonyg/pika
.. _`Wikipedia article about AMQP`: http://en.wikipedia.org/wiki/AMQP

Documentation
-------------

Carrot is using Sphinx, and the latest documentation is available at GitHub:

    http://github.com/ask/carrot/

Installation
============

You can install ``carrot`` either via the Python Package Index (PyPI)
or from source.

To install using ``pip``,::

    $ pip install carrot


To install using ``easy_install``,::

    $ easy_install carrot


If you have downloaded a source tarball you can install it
by doing the following,::

    $ python setup.py build
    # python setup.py install # as root


Terminology
===========

There are some concepts you should be familiar with before starting:

    * Publishers

        Publishers sends messages to an exchange.

    * Exchanges

        Messages are sent to exchanges. Exchanges are named and can be
        configured to use one of several routing algorithms. The exchange
        routes the messages to consumers by matching the routing key in the
        message with the routing key the consumer provides when binding to
        the exchange.

    * Consumers

        Consumers declares a queue, binds it to a exchange and receives
        messages from it.

    * Queues

        Queues receive messages sent to exchanges. The queues are declared
        by consumers.

    * Routing keys

        Every message has a routing key.  The interpretation of the routing
        key depends on the exchange type. There are four default exchange
        types defined by the AMQP standard, and vendors can define custom
        types (so see your vendors manual for details).

        These are the default exchange types defined by AMQP/0.8:

            * Direct exchange

                Matches if the routing key property of the message and
                the ``routing_key`` attribute of the consumer are identical.

            * Fan-out exchange

                Always matches, even if the binding does not have a routing
                key.

            * Topic exchange

                Matches the routing key property of the message by a primitive
                pattern matching scheme. The message routing key then consists
                of words separated by dots (``"."``, like domain names), and
                two special characters are available; star (``"*"``) and hash
                (``"#"``). The star matches any word, and the hash matches
                zero or more words. For example ``"*.stock.#"`` matches the
                routing keys ``"usd.stock"`` and ``"eur.stock.db"`` but not
                ``"stock.nasdaq"``.


Examples
========

Creating a connection
---------------------

    You can set up a connection by creating an instance of
    ``carrot.messaging.BrokerConnection``, with the appropriate options for
    your broker:

    >>> from carrot.connection import BrokerConnection
    >>> conn = BrokerConnection(hostname="localhost", port=5672,
    ...                           userid="test", password="test",
    ...                           virtual_host="test")


    If you're using Django you can use the
    ``carrot.connection.DjangoBrokerConnection`` class instead, which loads
    the connection settings from your ``settings.py``::

       BROKER_HOST = "localhost"
       BROKER_PORT = 5672
       BROKER_USER = "test"
       BROKER_PASSWORD = "secret"
       BROKER_VHOST = "/test"

    Then create a connection by doing:

        >>> from carrot.connection import DjangoBrokerConnection
        >>> conn = DjangoBrokerConnection()



Receiving messages using a Consumer
-----------------------------------

First we open up a Python shell and start a message consumer.

This consumer declares a queue named ``"feed"``, receiving messages with
the routing key ``"importer"`` from the ``"feed"`` exchange.

The example then uses the consumers ``wait()`` method to go into consume
mode, where it continuously polls the queue for new messages, and when a
message is received it passes the message to all registered callbacks.

    >>> from carrot.messaging import Consumer
    >>> consumer = Consumer(connection=conn, queue="feed",
    ...                     exchange="feed", routing_key="importer")
    >>> def import_feed_callback(message_data, message)
    ...     feed_url = message_data["import_feed"]
    ...     print("Got feed import message for: %s" % feed_url)
    ...     # something importing this feed url
    ...     # import_feed(feed_url)
    ...     message.ack()
    >>> consumer.register_callback(import_feed_callback)
    >>> consumer.wait() # Go into the consumer loop.

Sending messages using a Publisher
----------------------------------

Then we open up another Python shell to send some messages to the consumer
defined in the last section.

    >>> from carrot.messaging import Publisher
    >>> publisher = Publisher(connection=conn,
    ...                       exchange="feed", routing_key="importer")
    >>> publisher.send({"import_feed": "http://cnn.com/rss/edition.rss"})
    >>> publisher.close()


Look in the first Python shell again (where ``consumer.wait()`` is running),
where the following text has been printed to the screen::

   Got feed import message for: http://cnn.com/rss/edition.rss  


Serialization of Data
-----------------------

By default every message is encoded using `JSON`_, so sending
Python data structures like dictionaries and lists works.
`YAML`_ and Python's built-in ``pickle`` module is also supported,
and if needed you can register any custom serialization scheme you
want to use.

.. _`JSON`: http://www.json.org/
.. _`YAML`: http://yaml.org/

Each option has its advantages and disadvantages.

``json`` -- JSON is supported in many programming languages, is now
    a standard part of Python (since 2.6), and is fairly fast to 
    decode using the modern Python libraries such as ``cjson or 
    ``simplejson``.
    
    The primary disadvantage to ``JSON`` is that it limits you to 
    the following data types: strings, unicode, floats, boolean, 
    dictionaries, and lists.  Decimals and dates are notably missing.
    
    Also, binary data will be transferred using base64 encoding, which
    will cause the transferred data to be around 34% larger than an 
    encoding which supports native binary types. 
    
    However, if your data fits inside the above constraints and 
    you need cross-language support, the default setting of ``JSON``
    is probably your best choice. 
    
``pickle`` -- If you have no desire to support any language other than
    Python, then using the ``pickle`` encoding will gain you 
    the support of all built-in Python data types (except class instances), 
    smaller messages when sending binary files, and a slight speedup
    over ``JSON`` processing.

``yaml`` -- YAML has many of the same characteristics as ``json``, 
    except that it natively supports more data types (including dates, 
    recursive references, etc.)
    
    However, the Python libraries for YAML are a good bit slower
    than the libraries for JSON. 
    
    If you need a more expressive set of data types and need to maintain
    cross-language compatibility, then ``YAML`` may be a better fit
    than the above. 

To instruct carrot to use an alternate serialization method, 
use one of the following options.

    1.  Set the serialization option on a per-Publisher basis: 
        
            >>> from carrot.messaging import Publisher
            >>> publisher = Publisher(connection=conn,
            ...                       exchange="feed", routing_key="importer",
            ...                       serializer="yaml")

    2.  Set the serialization option on a per-call basis

            >>> from carrot.messaging import Publisher
            >>> publisher = Publisher(connection=conn,
            ...                       exchange="feed", routing_key="importer")
            >>> publisher.send({"import_feed": "http://cnn.com/rss/edition.rss"}, 
            ...                serializer="pickle")
            >>> publisher.close()

Note that ``Consumer``s do not need the serialization method specified in 
their code.  They can auto-detect the serialization method since we supply 
the ``Content-type`` header as part of the AMQP message.


Sending raw data without Serialization
---------------------------------------

In some cases, you don't need your message data to be serialized. If you
pass in a plain string or unicode object as your message, then carrot will
not waste cycles serializing/deserializing the data.

You can optionally specify a ``content_type`` and ``content_encoding``
for the raw data:

    >>> from carrot.messaging import Publisher
    >>> publisher = Publisher(connection=conn,
    ...                       exchange="feed",
                              routing_key="import_pictures")
    >>> publisher.send(open('~/my_picture.jpg','rb').read(), 
                       content_type="image/jpeg", 
                       content_encoding="binary")
    >>> publisher.close()
    
The ``message`` object returned by the ``Consumer`` class will have a 
``content_type`` and ``content_encoding`` attribute. 


Receiving messages without a callback
--------------------------------------

You can also poll the queue manually, by using the ``fetch`` method.
This method returns a ``Message`` object, from where you can get the
message body, de-serialize the body to get the data, acknowledge, reject or
re-queue the message.

    >>> consumer = Consumer(connection=conn, queue="feed",
    ...                     exchange="feed", routing_key="importer")
    >>> message = consumer.fetch()
    >>> if message:
    ...    message_data = message.payload
    ...    message.ack()
    ... else:
    ...     # No messages waiting on the queue.
    >>> consumer.close()

Sub-classing the messaging classes
----------------------------------

The ``Consumer``, and ``Publisher`` classes can also be sub classed. Thus you
can define the above publisher and consumer like so:

    >>> from carrot.messaging import Publisher, Consumer

    >>> class FeedPublisher(Publisher):
    ...     exchange = "feed"
    ...     routing_key = "importer"
    ...
    ...     def import_feed(self, feed_url):
    ...         return self.send({"action": "import_feed",
    ...                           "feed_url": feed_url})

    >>> class FeedConsumer(Consumer):
    ...     queue = "feed"
    ...     exchange = "feed"
    ...     routing_key = "importer"
    ...
    ...     def receive(self, message_data, message):
    ...         action = message_data["action"]
    ...         if action == "import_feed":
    ...             # something importing this feed
    ...             # import_feed(message_data["feed_url"])
                    message.ack()
    ...         else:
    ...             raise Exception("Unknown action: %s" % action)

    >>> publisher = FeedPublisher(connection=conn)
    >>> publisher.import_feed("http://cnn.com/rss/edition.rss")
    >>> publisher.close()

    >>> consumer = FeedConsumer(connection=conn)
    >>> consumer.wait() # Go into the consumer loop.

Getting Help
============

Mailing list
------------

Join the `carrot-users`_ mailing list.

.. _`carrot-users`: http://groups.google.com/group/carrot-users/

Bug tracker
===========

If you have any suggestions, bug reports or annoyances please report them
to our issue tracker at http://github.com/ask/carrot/issues/

Contributing
============

Development of ``carrot`` happens at Github: http://github.com/ask/carrot

You are highly encouraged to participate in the development. If you don't
like Github (for some reason) you're welcome to send regular patches.

License
=======

This software is licensed under the ``New BSD License``. See the ``LICENSE``
file in the top distribution directory for the full license text.
