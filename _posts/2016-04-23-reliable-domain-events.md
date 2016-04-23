---
layout: post
title: Reliable Domain Events
comments: true
category: norss
---

<p style="text-align:justify;">
We already know that domain events are a great tool to integrate different parts of your system (so called bounded contexts) and decouple them. Instead of directly calling outside component, we can notify our infrastructure, which in turn informs interested parties about what has just occurred. This another form of <b>Inversion of Control</b> helps us to achieve our goal. Consider following piece of code:  
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

    public class UserRegistered implements DomainEvent {

        private final UserRegistered userId;
        private final LocalDateTime at;

        //getters, constructor

    }

    class DomainEventsPublisher {

        //...

        private final Register handlersRegistry;

        public static void publish(DomainEvent event) {
            handlersRegistry
                .getHandlers(event.getClass())
                .forEach(handler -> handler.handle(event));
        }

    }

    class SendCommunicationOnUserRegistered implements DomainEventHandler<UserRegistered> {

    @Override
    public void handle(UserRegistered event) {
        Communication c = sendRegistrationEmail(event.getUserId());
        storeCommunication(c);
    }

```  

<p style="text-align:justify;">
There is a big problem with this simple <b>Observer</b> pattern in above example. Namely, everything happens in the same database transaction, which has started at the time of calling <i>register</i> method. This has several implications. First of all, we did not decouple user registration from e-mail sending and we just have mentioned this what domain events are for. If our mail server is down, registration fails. Why would our user care about an email? He correctly filled registration form and does not even know there is an email coming. 

From the <b>use case</b> point of view, sending an email should not imply successful invocation. Secondly, even though it looks like the code deals only with users (it registers and marks as suspicious when needed), it modifies another aggregate - communications (communication would be probably modeled as a Generic Subdomain, but to simplify things consider it as another bounded context). The rule of thumb says that we should not modify several aggregates in one transaction, because these concepts should not be so directly related. Sending an email (or any other action done as a consequence of registering user) may take a lot of time, do I/O calls, etc. 

But things get worse. Consider that everything was fine wit our mail server, we want to save user to database and something fails and transaction rollbacks. Now we have a big problem, because confused user got error page as a response, but seconds later successful email with welcoming greetings.
</p>

<p style="text-align:justify;">
This clearly shows that those concepts should be unrelated. One may come with an idea to fire all handlers asynchronously, but that solves only one issue: we could register users when we cannot send emails. We need something better. Fortunately, we can fire our emails just after transaction commit. Thus, we are sure everything was fine during registration and without doubts we can send the welcoming message. Spring gives us possibility to do that in a few lines of code with <a href="http://docs.spring.io/spring/docs/3.0.6.RELEASE_to_3.1.0.BUILD-SNAPSHOT/3.0.6.RELEASE/org/springframework/transaction/support/TransactionSynchronizationManager.html" TransactionSynchronizationManager</a>:
</p>

```java
    class DomainEventsPublisher {

        //...

        public static void publish(DomainEvent event) {
            handlersRegistry
                .getHandlers(event.getClass())
                .forEach(handler -> handleAsynchronously(event, handlers));
        }

        private static void handleAsynchronously(DomainEvent event, DomainEventHandler handler) {
            if (TransactionSynchronizationManager.isActualTransactionActive()) {
                processAfterCommit(event, handler);
            } else {
                processNow(event, handler);
            }
        }

        private static void processNow(final DomainEvent event, DomainEventHandler handler) {
            handler.handle(event);
        }

        private static void processAfterCommit(final DomainEvent event, DomainEventHandler handler) {
            TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
                @Override
                public void afterCommit() {
                    handler.handle(event);
                }
            });
        }
   }

```  

</p>

<p style="text-align:justify;">
Thanks to this code, we are fine with cause and effect relationship. Taking advantage of java memory model nomenclature: we know that successful registration <b>happens before</b> sending an email. One may argue that there is a slight window of time when we are not consistent. As stated at the beginning, we don't need to be consistent when working with several bounded context. The important is that they will be <b>eventually consistent</b>. Anyway, we still did not solve the problem with unresponsive mail server. When it fails, the information is lost. We can implement something which will retry this operation with a reasonable back-off, but when our application crashes, we still don't have any mean to recover to former state. We need to deal with that issue.
</p>

*** Event Store ***

<p style="text-align:justify;">
It gets clearer that we need to store somewhere the <b>intent</b> of sending the email. The intent was of course <i>UserRegistered</i> event. By exposing this intent to for example JMS infrastructure, we can implement the retry mechanism. We can go further and store any event coming from a given bounded context. Repository of all domain events is in fact called Event Store. Below example contain code that serializes events to JSON and publishes them to JMS. We could publish them anywhere else, for example to a file on a local disk or to an Akka actor. 
</p>

```java
class ExternalEventStore {
 
    //...

    private final ProducerTemplate producerTemplate;
    private final ObjectMapper objectMapper = new ObjectMapper();
    
    void publish(DomainEvent event) throws JsonProcessingException {
        final String body = objectMapper.writeValueAsString(event);
        producerTemplate.sendBody("queue.url", InOnly, body);
    }
}

```

<p style="text-align:justify;">
We could call that component from our <i>handleNow</i> method from previous example and make some other component consume those messages and send emails. But that would be just delegating the problem from mail server to another part of infrastructure - our queue. Putting a message to a queue might fail and the message would be lost. Moreover, our application can fail somewhere in between commit and invoking this asynchronous transaction listener. We also cannot move this code back to the transaction, because that would make us stuck at the beginning of our problem and queue failures would result in our users not being able to register. 

We have to develop a consistent solution in which an <b>occurrence of an event reflects that it really happened in our system</b>. Also <b>when it really happened, it should be followed by an event</b>. In other words, those two statements should be in bi-conditional logical connective.

The problem is that our JMS component is not backed by the same data source as our domain model. One solution is to use global transaction and two-phase commits. The problem is that it might decrease performance significantly. Plus, not every part of our infrastructure must support that mechanism. More clever idea is to share the same data source for our messaging infrastructure and domain model (if those two parts support that, of course). The side effect here is that we need to share the same schema, which might not be the nicest solution. In my opinion the best solution is to translate our events to our domain model storage and save them in the same transaction. Later on, another pool of threads can process them and publish somewhere else. Now our event store would look as follows:
</p>

```java

public class InternalEventStore {

    //...

    private final ObjectMapper eventSerializer;
    private final SessionFactory sessionFactory;

    void store(DomainEvent event) throws JsonProcessingException  {
        session().save(serialized(event));
    }

    private PersistentEvent serialized(DomainEvent event) throws JsonProcessingException   {
        return new PersistentEvent(eventSerializer.writeValueAsString(event));
    }

    List<PersistentEvent> listPending(int limit) {
       //...
    }
}

public class PersistentEvent {

    public enum Status {
        PENDING, SENT
    }

    private Long id;

    private String uuid = UUID.randomUUID().toString();

    private LocalDateTime occuredAt = LocalDateTime.now();

    private LocalDateTime sentAt;

    private String body;

    private Status status = PENDING;

    public void sent(LocalDateTime at) {
        this.sentAt = at;
        this.status = SENT;
    }

    //...

}

```
<p style="text-align:justify;">
and it would be call in <i>DomainEventProcessor</i> in the same transaction (so we came back to synchronous observator pattern):
</p>

```java

 class DomainEventsPublisher {

    //...

    private final Register handlersRegistry;
    private final EventStore eventStore;

    public static void publish(DomainEvent event) {
        handlersRegistry
            .getHandlers(event.getClass())
            .forEach(handler -> handler.handle(event));
        eventStore.store(event);
    }
}

```

<p style="text-align:justify;">
Note that we left the register with handlers working under the same transaction. It is because we may need to listen to this event somewhere around the same bounded context. That way we won't modify another aggregates, so leaving this like that is fine.

But we still want our events to appear in our JMS infrastructure. We can run a periodic job which scans list of our events and sends them to queue or topic:
</p>

```java
class PublishPendingEventsScheduler {
    
    private final ExternalEventStore eventStore;
    private final ExternalEventStore publisher;

    @Scheduled(initialDelay = 3000, fixedDelayString = "${events.publisher.freq:3000}")
    public void sendEvents() {
        eventStore.listPending(100).forEach(this::publish);
    }
    
    private void publish(PersistentEvent event) {
        publisher.publish(event); // publisher needs to have publish(PersistentEvent event) method
        event.sent(now());
    }
}

```

<p style="text-align:justify;">
We mark every event as sent, so that it won't be picked up in further processing. If our message infrastructure fails, we try again soon. We have implemented a  We might argue that this code suffers from the same problem as the whole example. Our database might be done at the time we want to save this event as sent. That is fair concern, because we already have put it to JMS. That means it can arrive at consumer side several times. It is important that consumer handles those events in idempotent way by for example storing PersistentEvent's uuid and doing de-duplication. Resending at producer side and idempotency at consumer side gives us <b>at most once delivery</b>
</p>

<p>In this post I tried to describe un reliable domain events mechanism by implementing simple event store. Actually, events stores have much more benefits: we can examine every historical result of commands invoked in our system, run some forecasting algorithms, do event sourcing (reconstructing an aggregate by looking at its events) in different bounded contexts</p>




