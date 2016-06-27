---
layout: post
title: Event sourcing with Command Query Responsibility Segregation
comments: true
category: norss
---

<p style="text-align:justify;">
<a href="http://pillopl.github.io/reliable-domain-events/">In my last post</a> I wrote about domain events publishing. Events were published at the end of aggregate public methods as a natural consequence of internal state modification. Let's look again at this code sample:
</p>

```java
    class User {

        //...

        @Transactional
        public void register(LocalDateTime at, RegisteredBy by) {
            markRegistered();
            markSuspiciousIfAtNight(at);
            DomainEventsPublisher.publish(new UserRegistered(this.id(), at));
        }

    }

```  

<p style="text-align:justify;">
The problem that we might face is different source of truth between created <i>UserRegistered</i> and internal changes done by <i>markRegistered</i> and <i>markSuspiciousIfAtNight(at)</i> method invocations. The path to mistake is fairly simple, we could throw this domain event with <i>Instant.now()</i> instead of <i>at</i> parameter. Thus, misleading listeners of domain events, because internal state was modified differently. 
</p>
<p style="text-align:justify;">
If we were storing our aggregate state with the help of <a href="http://martinfowler.com/eaaDev/EventSourcing.html">event sourcing</a>, so by serializing domain events, this mistake cannot be done. I've created <a href="https://github.com/pilloPl/event-source-cqrs-sample">a sample application that might show how to deal with such an approach</a>. Those serialized events can be published to external listeners. The same events from which we rebuild our domain object</a>

<p style="text-align:justify;">
Take a look <a href="https://github.com/pilloPl/event-source-cqrs-sample/blob/master/src/test/groovy/io/pillopl/eventsource/domain/ShopItemSpec.groovy">how ease it is to test such an aggregate</a>. By invocking public method, we expect that proper domain events were raised. Simple as that. It is also tested in <a href="https://github.com/pilloPl/event-source-cqrs-sample/blob/master/src/test/groovy/io/pillopl/eventsource/integration/shopitem/ShopItemsIntegrationSpec.groovy">integration test</a>, together with storage. Please note, that with this approach it is simple to create <a href="https://github.com/pilloPl/event-source-cqrs-sample/blob/master/src/main/java/io/pillopl/eventsource/domain/shopitem/ShopItem.java">Shop Item as truly immutable aggregate</a> and favour functional programming.
</p>

***Event Store***
<p style="text-align:justify;">
Event store was implemented very simply - as database table <a href="https://github.com/pilloPl/event-source-cqrs-sample/blob/master/src/main/java/io/pillopl/eventsource/store/EventStream.java">EventStream</a> that represents all events comming from one aggregate instance, which holds collection of all events (<a href="https://github.com/pilloPl/event-source-cqrs-sample/blob/master/src/main/java/io/pillopl/eventsource/store/EventDescriptor.java">EventDescriptors</a>).
</p>

***CQRS***
<p style="text-align:justify;">
Read model in CQRS applcations is build when changes to write model are performed. That is why event sourcing can naturally lead to CQRS usage - is there any better way for listening for write model changes than subscribing to events raised from write model? The same events, from which write model is built? Take a look <a href="https://github.com/pilloPl/event-source-cqrs-sample/blob/master/src/main/java/io/pillopl/eventsource/readmodel/ReadModelOnDomainEventUpdater.java">how simply it is to update read model</a> in event sourced application. Read model there is implemented by directly using JDBC queries with <a href="https://github.com/pilloPl/event-source-cqrs-sample/blob/master/src/main/java/io/pillopl/eventsource/readmodel/ShopItemDto.java">denormalized table<a/>, beacuse why not?
</p>
