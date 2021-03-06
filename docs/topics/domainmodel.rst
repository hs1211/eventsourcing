============
Domain model
============

The library's domain layer has base classes for domain events and entities. These classes show how to
write a domain model that uses the library's event sourcing infrastructure. They can also be used to
develop an event-sourced application as a domain driven design.


Domain events
=============

The purpose of a domain event is to be published when something happens, normally the results from the
work of a command. The library has a base class for domain events called ``DomainEvent``.

Domain events can be freely constructed from the ``DomainEvent`` class. Attributes are
set directly from the constructor keyword arguments.

.. code:: python

    from eventsourcing.domain.model.events import DomainEvent

    domain_event = DomainEvent(a=1)
    assert domain_event.a == 1


The attributes of domain events are read-only. New values cannot be assigned to existing objects.
Domain events are immutable in that sense.

.. code:: python

    # Fail to set attribute of already-existing domain event.
    try:
        domain_event.a = 2
    except AttributeError:
        pass
    else:
        raise Exception("Shouldn't get here")


Domain events can be compared for equality as value objects, instances are equal if they have the
same type and the same attributes.

.. code:: python

    DomainEvent(a=1) == DomainEvent(a=1)

    DomainEvent(a=1) != DomainEvent(a=2)

    DomainEvent(a=1) != DomainEvent(b=1)


Publish-subscribe
-----------------

Domain events can be published, using the library's publish-subscribe mechanism.

The ``publish()`` function is used to publish events. The ``event`` arg is required.

.. code:: python

    from eventsourcing.domain.model.events import publish

    publish(event=domain_event)


The ``subscribe()`` function is used to subscribe a ``handler`` that will receive events.

The optional ``predicate`` arg can be used to provide a function that will decide whether
or not the subscribed handler will actually be called when an event is published.

.. code:: python

    from eventsourcing.domain.model.events import subscribe

    received_events = []

    def receive_event(event):
        received_events.append(event)

    def is_domain_event(event):
        return isinstance(event, DomainEvent)

    subscribe(handler=receive_event, predicate=is_domain_event)

    # Publish the domain event.
    publish(domain_event)

    assert len(received_events) == 1
    assert received_events[0] == domain_event


The ``unsubscribe()`` function can be used to stop the handler receiving further events.

.. code:: python

    from eventsourcing.domain.model.events import unsubscribe

    unsubscribe(handler=receive_event, predicate=is_domain_event)

    # Clean up.
    del received_events[:]  # received_events.clear()


Event library
-------------

The library has a small collection of domain event subclasses, such as ``EventWithOriginatorID``,
``EventWithOriginatorVersion``, ``EventWithTimestamp``, ``EventWithTimeuuid``, ``Created``, ``AttributeChanged``,
``Discarded``.

Some of these classes provide useful defaults for particular attributes, such as a ``timestamp``.
Timestamps can be used to sequence events.

.. code:: python

    from eventsourcing.domain.model.events import EventWithTimestamp
    from eventsourcing.domain.model.events import EventWithTimeuuid
    from uuid import UUID

    # Automatic timestamp.
    assert isinstance(EventWithTimestamp().timestamp, float)

    # Automatic UUIDv1.
    assert isinstance(EventWithTimeuuid().event_id, UUID)


Some classes require particular arguments when constructed. The ``originator_id`` can be used
to identify a sequence to which an event belongs. The ``originator_version`` can be used to
position the event in a sequence.

.. code:: python

    from eventsourcing.domain.model.events import EventWithOriginatorVersion
    from eventsourcing.domain.model.events import EventWithOriginatorID
    from uuid import uuid4

    # Requires originator_id.
    EventWithOriginatorID(originator_id=uuid4())

    # Requires originator_version.
    EventWithOriginatorVersion(originator_version=0)


Some are just useful for their distinct type, for example in subscription predicates.

.. code:: python

    from eventsourcing.domain.model.events import Created, Discarded

    def is_created(event):
        return isinstance(event, Created)

    def is_discarded(event):
        return isinstance(event, Discarded)

    assert is_created(Created()) is True
    assert is_created(Discarded()) is False
    assert is_created(DomainEvent()) is False

    assert is_discarded(Created()) is False
    assert is_discarded(Discarded()) is True
    assert is_discarded(DomainEvent()) is False

    assert is_domain_event(Created()) is True
    assert is_domain_event(Discarded()) is True
    assert is_domain_event(DomainEvent()) is True


Custom events
-------------

Custom domain events can be coded by subclassing the library's domain event classes.

Domain events are normally named using the past participle of a common verb, for example
a regular past participle such as "started", "paused", "stopped", or an irregular past
participle such as "chosen", "done", "found", "paid", "quit", "seen".

.. code:: python

    class SomethingHappened(DomainEvent):
        """
        Published whenever something happens.
        """


It is possible to code domain events as inner or nested classes.

.. code:: python

    class Job(object):

        class Seen(EventWithTimestamp):
            """
            Published when the job is seen.
            """

        class Done(EventWithTimestamp):
            """
            Published when the job is done.
            """


Inner or nested classes can be used, and are used in the library, to define the domain events of a domain entity
on the entity class itself.

.. code:: python

    seen = Job.Seen(job_id='#1')
    done = Job.Done(job_id='#1')

    assert done.timestamp > seen.timestamp


Domain entities
===============

A domain entity is an object that is not defined by its attributes, but rather by a thread of continuity and its
identity. The attributes of a domain entity can change, directly by assignment, or indirectly by calling a method of
the object.

The library provides a domain entity class ``VersionedEntity``, which has an ``id`` attribute, and a ``version``
attribute.

.. code:: python

    from eventsourcing.domain.model.entity import VersionedEntity

    entity_id = uuid4()

    entity = VersionedEntity(id=entity_id, version=0)

    assert entity.id == entity_id
    assert entity.version == 0


Entity library
--------------

There is a ``TimestampedEntity`` that has ``id`` and ``created_on`` attributes. It also has a ``last_modified``
attribute which is normally updated as events are applied.

.. code:: python

    from eventsourcing.domain.model.entity import TimestampedEntity

    entity_id = uuid4()

    entity = TimestampedEntity(id=entity_id, timestamp=123456789)

    assert entity.id == entity_id
    assert entity.created_on == 123456789
    assert entity.last_modified == 123456789


There is also a ``TimestampedVersionedEntity`` that has ``id``, ``version``, ``created_on``, and ``last_modified``
attributes.

.. code:: python

    from eventsourcing.domain.model.entity import TimestampedVersionedEntity

    entity_id = uuid4()

    entity = TimestampedVersionedEntity(id=entity_id, version=0, timestamp=123456789)

    assert entity.id == entity_id
    assert entity.version == 0
    assert entity.created_on == 123456789
    assert entity.last_modified == 123456789


A timestamped, versioned entity is both a timestamped entity and a versioned entity.

.. code:: python

    assert isinstance(entity, TimestampedEntity)
    assert isinstance(entity, VersionedEntity)


Entity events
-------------

The library's domain entities have domain events as inner classes: ``Event``, ``Created``, ``AttributeChanged``, and
``Discarded``. These inner event classes are all subclasses of ``DomainEvent`` and can be freely constructed, with
suitable arguments.

.. code:: python

    created = VersionedEntity.Created(
        originator_version=0,
        originator_id=entity_id,
    )

    attribute_a_changed = VersionedEntity.AttributeChanged(
        name='a',
        value=1,
        originator_version=1,
        originator_id=entity_id
    )

    attribute_b_changed = VersionedEntity.AttributeChanged(
        name='b',
        value=2,
        originator_version=2,
        originator_id=entity_id
    )

    entity_discarded = VersionedEntity.Discarded(
        originator_version=3,
        originator_id=entity_id
    )


The class ``VersionedEntity`` has a method ``_increment_version()`` which can be used, for example by a mutator
function, to increment the version number each time an event is applied.

.. code:: python

    entity._increment_version()

    assert entity.version == 1


Mutator functions
-----------------

For an application to be event sourced, the state of the application must be mutated by applying domain events.

The entity mutator function ``mutate_entity()`` can be used to apply a domain event to an entity.

.. code:: python

    from eventsourcing.domain.model.entity import mutate_entity

    entity = mutate_entity(entity, attribute_a_changed)

    assert entity.a == 1


When a versioned entity is updated in this way, the version number is normally incremented.

.. code:: python

    assert entity.version == 2


Apply and publish
-----------------

Events are normally published after they are applied. The method ``_apply_and_publish()``
can be used to both apply and then publish, so the event mutates the entity
and is then received by subscribers.

.. code:: python

    # Apply and publish a domain event.
    entity._apply_and_publish(attribute_b_changed)

    # Check the event was applied.
    assert entity.b == 2
    assert entity.version == 3


For example, the method ``change_attribute()`` constructs an ``AttributeChanged`` event and then calls
``_apply_and_publish()``. In the code below, the event is received and checked.

.. code:: python

    entity = VersionedEntity(id=entity_id, version=0)

    assert len(received_events) == 0
    subscribe(handler=receive_event, predicate=is_domain_event)

    # Apply and publish an AttributeChanged event.
    entity.change_attribute(name='full_name', value='Mr Boots')

    # Check the event was applied.
    assert entity.full_name == 'Mr Boots'

    # Check the event was published.
    assert received_events[0].__class__ == VersionedEntity.AttributeChanged
    assert received_events[0].name == 'full_name'
    assert received_events[0].value == 'Mr Boots'

    # Clean up.
    unsubscribe(handler=receive_event, predicate=is_domain_event)
    del received_events[:]  # received_events.clear()


Discarding entities
-------------------

The entity method ``discard()`` can be used to discard the entity, by applying and publishing
a ``Discarded`` event, after which the entity is unavailable for further changes.

.. code:: python

    from eventsourcing.exceptions import EntityIsDiscarded

    entity.discard()

    # Fail to change an attribute after entity was discarded.
    try:
        entity.change_attribute('full_name', 'Mr Boots')
    except EntityIsDiscarded:
        pass
    else:
        raise Exception("Shouldn't get here")


The mutator function will return ``None`` after mutating an entity with a ``Discarded`` event.

.. code:: python

    entity = VersionedEntity(id=entity_id, version=3)

    entity = mutate_entity(entity, entity_discarded)

    assert entity is None


That means a sequence of events that ends with a ``Discarded`` event will result in the same
state as an empty sequence of events, when the sequence is replayed by an event player for example.


Custom entities
---------------

The library entity classes can be subclassed.

.. code:: python

    from eventsourcing.domain.model.decorators import attribute


    class User(VersionedEntity):
        def __init__(self, full_name, *args, **kwargs):
            super(User, self).__init__(*args, **kwargs)
            self.full_name = full_name


An entity factory method can construct, apply, and publish the first event of an entity's lifetime. After the event
is published, the new entity will be returned by the factory method.

.. code:: python

    def create_user(full_name):
        created_event = User.Created(full_name=full_name, originator_id='1')
        assert created_event.originator_id
        user_entity = mutate_entity(event=created_event, initial=User)
        publish(created_event)
        return user_entity

    user = create_user(full_name='Mrs Boots')

    assert user.full_name == 'Mrs Boots'


Subclasses can extend the entity base classes, by adding event-based properties and methods.


Custom attributes
-----------------

The library's ``@attribute`` decorator provides a property getter and setter, which will apply and publish an
``AttributeChanged`` event when the property is assigned. Simple mutable attributes can be coded as
decorated functions without a body, such as the ``full_name`` function of ``User`` below.

.. code:: python

    from eventsourcing.domain.model.decorators import attribute


    class User(VersionedEntity):

        def __init__(self, full_name, *args, **kwargs):
            super(User, self).__init__(*args, **kwargs)
            self._full_name = full_name

        @attribute
        def full_name(self):
            pass


In the code below, after the entity has been created, assigning to the ``full_name`` attribute causes the entity to be
updated, and an ``AttributeChanged`` event to be published. Both the ``Created`` and ``AttributeChanged`` events are
received by a subscriber.

.. code:: python

    assert len(received_events) == 0
    subscribe(handler=receive_event, predicate=is_domain_event)

    # Publish a Created event.
    user = create_user('Mrs Boots')
    assert user.full_name == 'Mrs Boots'

    # Publish an AttributeChanged event.
    user.full_name = 'Mr Boots'
    assert user.full_name == 'Mr Boots'

    assert len(received_events) == 2
    assert received_events[0].__class__ == VersionedEntity.Created
    assert received_events[0].full_name == 'Mrs Boots'

    assert received_events[1].__class__ == VersionedEntity.AttributeChanged
    assert received_events[1].value == 'Mr Boots'
    assert received_events[1].name == '_full_name'

    # Clean up.
    unsubscribe(handler=receive_event, predicate=is_domain_event)
    del received_events[:]  # received_events.clear()


Custom commands
---------------

The entity base classes can also be extended by adding "command" methods that publish events. In general, the arguments
of a command will be used to perform some work. Then, the result of the work will be used to construct a domain event
that represents what happened. And then, the domain event will be applied and published.

Methods like this, for example the ``set_password()`` method of the ``User`` entity below, normally have no return
value. The method creates an encoded string from a raw password, and then uses the ``change_attribute()`` method to
apply and publish an ``AttributeChanged`` event for the ``_password`` attribute with the encoded password.

.. code:: python

    from eventsourcing.domain.model.decorators import attribute


    class User(VersionedEntity):

        def __init__(self, *args, **kwargs):
            super(User, self).__init__(*args, **kwargs)
            self._password = None

        def set_password(self, raw_password):
            # Do some work using the arguments of a command.
            password = self._encode_password(raw_password)

            # Construct, apply, and publish an event.
            self.change_attribute('_password', password)

        def check_password(self, raw_password):
            password = self._encode_password(raw_password)
            return self._password == password

        def _encode_password(self, password):
            return ''.join(reversed(password))


    user = User(id='1')

    user.set_password('password')
    assert user.check_password('password')


A custom entity can also have custom methods that publish custom events. In the example below, a method
``make_it_so()`` publishes a domain event called ``SomethingHappened``.


Custom mutator
--------------

To be applied to an entity, custom event classes must be supported by a custom mutator function. In the code below,
the ``mutate_world()`` mutator function extends the library's ``mutate_entity`` function to support the event
``SomethingHappened``. The ``_mutate()`` function of ``DomainEntity`` has been overridden so that ``mutate_world()``
will be called when events are applied.

.. code:: python

    from eventsourcing.domain.model.decorators import mutator

    class World(VersionedEntity):

        def __init__(self, *args, **kwargs):
            super(World, self).__init__(*args, **kwargs)
            self.history = []

        class SomethingHappened(VersionedEntity.Event):
            """Published when something happens in the world."""

        def make_it_so(self, something):
            # Do some work using the arguments of a command.
            what_happened = something

            # Construct an event with the results of the work.
            event = World.SomethingHappened(
                what=what_happened,
                originator_id=self.id,
                originator_version=self.version
            )

            # Apply and publish the event.
            self._apply_and_publish(event)

        @classmethod
        def _mutate(cls, initial, event):
            return mutate_world(event=event, initial=initial)


    @mutator
    def mutate_world(initial, event):
        return mutate_entity(initial, event)

    @mutate_world.register(World.SomethingHappened)
    def _(self, event):
        self.history.append(event)
        self._increment_version()
        return self


    world = World(id='1')
    world.make_it_so('dinosaurs')
    world.make_it_so('trucks')
    world.make_it_so('internet')

    assert world.history[0].what == 'dinosaurs'
    assert world.history[1].what == 'trucks'
    assert world.history[2].what == 'internet'


Reflexive mutator
-----------------

The ``WithReflexiveMutator`` class tries to call a function called ``mutate()`` on the
event class itself. This means each event class can define how an entity is mutated by it.

A custom base entity class, for example ``Entity`` in the code below, may help to adopt
this style across all entity classes in an application.

.. code:: python

    from eventsourcing.domain.model.entity import WithReflexiveMutator


    class Entity(WithReflexiveMutator, VersionedEntity):
        """
        Custom base class for domain entities in this example.
        """

    class World(Entity):
        """
        Example domain entity, with mutator function on domain event.
        """
        def __init__(self, *args, **kwargs):
            super(World, self).__init__(*args, **kwargs)
            self.history = []

        def make_it_so(self, something):
            what_happened = something
            event = World.SomethingHappened(
                what=what_happened,
                originator_id=self.id,
                originator_version=self.version
            )
            self._apply_and_publish(event)

        class SomethingHappened(VersionedEntity.Event):
            # Define mutator function for entity on the event class.
            def mutate(self, entity):
                entity.history.append(self)
                entity._increment_version()


    world = World(id='1')
    world.make_it_so('dinosaurs')
    world.make_it_so('trucks')
    world.make_it_so('internet')

    assert world.history[0].what == 'dinosaurs'
    assert world.history[1].what == 'trucks'
    assert world.history[2].what == 'internet'


Aggregate root
==============

The library has a domain entity class called ``AggregateRoot`` that can be useful in a domain driven design, where a
command can cause many events to be published. The ``AggregateRoot`` class has a ``save()`` method, which publishes
a list of pending events, and overrides the ``_publish()`` method of the base class to append events to a pending list.

The ``AggregateRoot`` class inherits from both ``TimestampedVersionedEntity`` and
``WithReflexiveMutator``, and can be subclassed to define custom aggregate root entities.

.. code:: python

    from eventsourcing.domain.model.aggregate import AggregateRoot


    class World(AggregateRoot):
        """
        Example domain entity, with mutator function on domain event.
        """
        def __init__(self, *args, **kwargs):
            super(World, self).__init__(*args, **kwargs)
            self.history = []

        def make_things_so(self, *somethings):
            for something in somethings:
                self._trigger(World.SomethingHappened, what=something)

        class SomethingHappened(VersionedEntity.Event):
            def mutate(self, entity):
                entity.history.append(self)
                entity._increment_version()


    # World factory.
    def create_new_world():
        created = World.Created(originator_id=1)
        world = World._mutate(event=created)
        world._publish(created)
        return world


An ``AggregateRoot`` entity will postpone the publishing of all events, pending the next call to its
``save()`` method.

.. code:: python

    assert len(received_events) == 0
    subscribe(handler=receive_event)

    # Create new entity.
    world = create_new_world()
    assert isinstance(world, World)

    # Command that publishes many events.
    world.make_things_so('dinosaurs', 'trucks', 'internet')

    assert world.history[0].what == 'dinosaurs'
    assert world.history[1].what == 'trucks'
    assert world.history[2].what == 'internet'


When the ``save()`` method is called, all such pending events are published as a
single list of events to the publish-subscribe mechanism.

.. code:: python

    # Events are pending actual publishing until the save() method is called.
    assert len(received_events) == 0
    world.save()

    # Pending events were published as a single list of events.
    assert len(received_events) == 1
    assert len(received_events[0]) == 4


Publishing all events from a single command in a single list allows all the events to be written to a database as a
single atomic operation.

That avoids the risk that some events will be stored successfully but other events from the
same command will fall into conflict and be lost, because another thread has operated on the same aggregate at the
same time, causing an inconsistent state that would also be difficult to repair.

It also avoids the risk of other threads picking up only some events caused by a command, presenting the aggregate
in an inconsistent or unusual and perhaps unworkable state.

.. code:: python

    unsubscribe(handler=receive_event)
    del received_events[:]  # received_events.clear()
