---
layout: post
title: Event sourcing with CQRS sample revisited
comments: true

---

<p style="text-align:justify;">
<a href=http://pillopl.github.io/event-sourcing-with-cqrs/>Previously</a> I posted a short note about my sample event sourced application that follows Command Query Responsibility Segregation principle. Today I would like to go a little bit further and describe its particular components and their responsibilities. 
</p>

***Overall structure***
<p style="text-align:justify;">
There are 4 sub packages:
<ul>
  <li><b>domain</b> - which is the core of the business. In so called onion architecture it stays in the middle, it does not depend on any other package. It does not even depend on spring, since no container is needed there. Hence, there are only unit tests provided. It exposes one interface to get ShopItem, this interface is called <a href="https://github.com/pilloPl/event-source-cqrs-sample/blob/master/src/main/java/io/pillopl/eventsource/domain/shopitem/ShopItemRepository.java">ShopItemRepository</a>. Its presence let the application to inverse the dependencies - every implementation of ShopItemRepository points towards domain</li>
  <li><b>boundary</b> - entry point to the application. It contains only one service, called <a href="https://github.com/pilloPl/event-source-cqrs-sample/blob/master/src/main/java/io/pillopl/eventsource/boundary/ShopItems.java">ShopItems. It receives <a href="https://github.com/pilloPl/event-source-cqrs-sample/tree/master/src/main/java/io/pillopl/eventsource/domain/shopitem/commands">commands exposed by domain</a> and invokes them on proper ShopItem instance.</a>. This service could be represented as a REST controller (although it does not handle REST, but a few more annotations could change that)</li>
  <li><b>eventstore</b> - it exposes one public class - <a href="https://github.com/pilloPl/event-source-cqrs-sample/blob/master/src/main/java/io/pillopl/eventsource/eventstore/EventSourcedShopItemRepository.java">EventSourcedShopItemRepository</a>, which implements ShopItemRepository exposed by domain. Hence, it can be treated as plug-in shipped with application. Currently, it uses relational database to store events, but probably document database would be more suitable choice, since events <a href="https://github.com/pilloPl/event-source-cqrs-sample/blob/master/src/main/java/io/pillopl/eventsource/eventstore/EventSerializer.java">are serialized to JSON by EventSerializer</a>. Every single stored event is also published to any other interested party with the help of <a href="http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/context/ApplicationEventPublisher.html">ApplicationEventPublisher</a></li>
  <li><b>readmodel</b> - listens to events published by mentioned ApplicationEventPublisher and mutates denormalized shop item model represented by <a href="https://github.com/pilloPl/event-source-cqrs-sample/blob/master/src/main/java/io/pillopl/eventsource/readmodel/ShopItemDto.java">ShopItemDto.</a></li>
</ul>

</p>


***Package: domain***
<p style="text-align:justify;">
Application simulates a shop with various items. Since shopping domain is much more complicated, it probably would not be modeled like this in a real software. Anyway, a shop item can be in 4 states: INITIALIZED, BOUGHT, PAID, PAYMENT_MISSING. Changing a state means emitting an event. Let's take a look at state diagram.

<img src="/images/states.png">


</p>

<p style="text-align:justify;">
The most important mind shift is how the shop item aggregate is modeled. <a href="https://github.com/pilloPl/event-source-cqrs-sample/blob/master/src/main/java/io/pillopl/eventsource/domain/shopitem/ShopItem.java">It is not a traditional entity, but a simple POJO</a>. Hence, no external dependencies are needed. Note that ShopItem is immutable, which favors functional programming and gives a lot of benefits. Every business method returns new instance of ShopItem. 
</p>

<p style="text-align:justify;">
Secondly, it is worth seeing how easy it is to test such an aggregate. We only care about what is being emitted as event, we don't care about internal state representations. Thus, test for paying for shop item looks as follows:
</p>

```java
 def 'should emit item paid event when paying for bought item'() {
        when:
            ShopItem tx = bought(uuid).pay(now())
        then:
            tx.getUncommittedChanges().size() == 1
            tx.getUncommittedChanges().head().type() == ItemPaid.TYPE
    }
```

<p style="text-align:justify;">
<b></i>getUncommitedChanges()</b></i> returns events created during business method invocation. We also should take care of illegal state transitions, for example marking payment as missing when when someone already has paid should be impossible:
</p>

```java
def 'cannot mark payment missing when item already paid'() {
        when:
            paid(uuid).markTimeout(now())
        then:
            thrown(IllegalStateException)
    }
```

<p style="text-align:justify;">
Last, but not least - we want those operations to be idempotent. Idempotency in this context means not emitting the same event twice.
</p>

```java
   def 'marking payment timeout should be idempotent'() {
        when:
            ShopItem tx = withTimeout(uuid).markTimeout(now())
        then:
            tx.getUncommittedChanges().isEmpty()
    }
```

***Package: eventstore***

<p style="text-align:justify;">
Mentioned above</i>getUncommitedChanges()</i>method is crucial point for implementing storage with events. Every time we try to save a new change, we call "save" in EventSourceShopItemRepository, which looks as follows:
</p>

```java
 public ShopItem save(ShopItem aggregate) {
        final List<DomainEvent> pendingEvents = aggregate.getUncommittedChanges();
        eventStore.saveEvents(
                aggregate.getUuid(),
                pendingEvents
                        .stream()
                        .map(eventSerializer::serialize)
                        .collect(toList()));
        return aggregate.markChangesAsCommitted();
    }
```

<p style="text-align:justify;">
So basically we get all changes emitted by an aggregate, serialize them to JSON, store in database and flush pending events in aggregate. Let's look how event store handles saving an event.
</p>

```java
interface EventStore extends JpaRepository<EventStream, Long> {

    Optional<EventStream> findByAggregateUUID(UUID uuid);

    default void saveEvents(UUID aggregateId, List<EventDescriptor> events) {
        final EventStream eventStream = findByAggregateUUID(aggregateId)
                .orElseGet(() -> new EventStream(aggregateId));
        eventStream.addEvents(events);
        save(eventStream);
    }

    default List<EventDescriptor> getEventsForAggregate(UUID aggregateId) {
        return findByAggregateUUID(aggregateId)
                        .map(EventStream::getEvents)
                        .orElse(emptyList());

    }
}
```

<p style="text-align:justify;">
It just looks for <a href="https://github.com/pilloPl/event-source-cqrs-sample/blob/master/src/main/java/io/pillopl/eventsource/eventstore/EventStream.java">EventStream</a> connected with this aggregate UUID and adds new serialized events in a form of <a href="https://github.com/pilloPl/event-source-cqrs-sample/blob/master/src/main/java/io/pillopl/eventsource/eventstore/EventDescriptor.java">EventDescriptor</a> with event serialized to JSON. If there is no stream yet (means we try to store the brand new item, new stream is created). Everything is implemented with help of spring data jpa repository. Concurrent changes done in EventStream (adding new EventDescriptors to it) can be done with optimistic locking.
</p>

<p style="text-align:justify;">
Having EventStore implemented helped us to create <a href="https://github.com/pilloPl/event-source-cqrs-sample/blob/master/src/main/java/io/pillopl/eventsource/eventstore/EventSourcedShopItemRepository.java">EventSourcedShopItemRepository</a>. It delegates to event store and looks for events connected with given aggregate. Next, it applies them sequentially, creating an aggregate instance. <b>State can be reconstructed to represent aggregate from any given time. This is visible in the following test case.</b>
</p>

```java
	 def 'should reconstruct item at given moment'() {
		given:
		    ShopItem stored = initialized()
		            .buy(uuid, TOMORROW, PAYMENT_DEADLINE_IN_HOURS)
		            .pay(DAY_AFTER_TOMORROW)
		when:
		    shopItemRepository.save(stored)
		and:
		    ShopItem bought = shopItemRepository.getByUUIDat(uuid, TOMORROW)
		    ShopItem paid = shopItemRepository.getByUUIDat(uuid, DAY_AFTER_TOMORROW)

		then:
		    bought.state == BOUGHT
		    paid.state == PAID
	    }
```


***Package: readmodel***

<p style="text-align:justify;">
<a href="https://github.com/pilloPl/event-source-cqrs-sample/blob/master/src/main/java/io/pillopl/eventsource/readmodel/ReadModelOnDomainEventUpdater.java">Take a look how read model is updated.</a> Basically it uses jdbc and direct sql calls to update denormalized database schema. No fany ORM needed here. Read model update is triggered with help of <a href="http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/transaction/event/TransactionalEventListener.html">TransactionalEventListener</a>.   
</p>

```java
	 //...

	 @TransactionalEventListener
	    public void handle(DomainEvent event) {
		if (event instanceof ItemBought) {
		    final ItemBought itemBought = (ItemBought) event;
		    jdbcReadModelUpdater.updateOrCreateItemAsBlocked(event.uuid(), event.when(), itemBought.getPaymentTimeoutDate());
		} else if (event instanceof ItemPaid) {
		    jdbcReadModelUpdater.updateItemAsPaid(event.uuid(), event.when());
		} else if (event instanceof ItemPaymentTimeout) {
		    jdbcReadModelUpdater.updateItemAsPaymentMissing(event.uuid(), event.when());
		} else {
		    throw new IllegalArgumentException("Cannot handle event " + event.getClass());
		}
	    }
```

<p style="text-align:justify;">
Note that we update read model in the same transaction, hence usage of TransactionalEventListener. Thus, the read model is consistent with write model, because they share the same data source. Normally we want our write model to be decoupled from read model, preferably in different data source. Read model would be eventually consistent and would read all the events stored by write model in event store.

There is one more interesting thing in readmodel package. Since it knows about all the data and has it in denormalized form, it is able to perform actions. Here, <a href="https://github.com/pilloPl/event-source-cqrs-sample/blob/master/src/main/java/io/pillopl/eventsource/readmodel/PaymentTimeoutChecker.java">PaymentTimeoutChecker</a> periodically checks whether payment is missing. If that happens, it <b>throws a command to write model</b>. That causes aggregate state change.
</p>

***Conclusion***

<p style="text-align:justify;">
With usage of CQRS we have simple and performant read model, that we can query without complex joins. Our read model is decoupled from write model and does not pollute domain model. That means that it is just a projection of domain events and has no impact of how domain is implemented. 

Having all the events stored in event store gives us huge debugging possibilities. We can reconstruct aggregate state from any given moment. Storing only events results in having simple shema for write model (only 2 tables here), throwing away all the work that db admins need to do when storing complex domain in relational database. 
</p>
