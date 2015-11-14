---
layout: post
title: Stub & Verify?
comments: true
---

<p style="text-align:justify;">
Several times lately I came across Spock tests, which conceptually looked as follows:
</p>

```groovy
    static final BigDecimal ANY_AMOUNT = new BigDecimal("100.00")
    
    Payment payment = aSamplePayment()
    Client client = aSampleClient()
    
    PaymentCreator paymentCreator = Mock(PaymentCreator)
    PaymentRepository repository = Mock(PaymentRepository)

    @Subject
    Payments payments = new Payments(paymentCreator, repository)

    def "should create outgoing payment for loan"() {
        when:
            Payment actualPayment = payments.saveOutgoingFor(client, ANY_AMOUNT)
        then:
            1 * paymentCreator.createOutgoing(client, ANY_AMOUNT) >> payment
            1 * repository.save(payment)
            actualPayment == payment
    }
```  

<p style="text-align:justify;">
Spock allows you to stub and verify collaborator at once.<a href="https://spockframework.github.io/spock/docs/1.0/interaction_based_testing.html"> See section 'Combining Mocking and Stubbing'</a>. It is followed by sections which describe couple of discouraged solutions to testing, like partial stubs or stubbing static methods. In my opinion above section should also have this sidenote:
</p>

<blockquote class="cite">
      <p>(Think twice before using this feature. It might be better to change the design of the code under specification.)</p>
</blockquote>

***Design smell!***

Consider following implementation under test:

```java
 Payment saveOutgoingFor(Client client, BigDecimal amount) {
        Payment payment = paymentCreator.createOutgoing(client, amount);
        paymentRepository.save(payment);
        return payment;
    }
```

<p style="text-align:justify;">
What it does is creating new payment and saving it in some kind of storage. Actually, the paymentRepository.save() method could return stored payment, so that we could get rid of last line. Such design can be explained like the following "to something with the payment, return it, do something else, and return it". Two actions are being done by two collaborating services. Actually we can cut it to "do something <i>andThen</i> do something else". Rings a bell? It is just function composition and you can read why dependency injection in most cases is just function composition <a href="http://www.nurkiewicz.com/2015/08/dependency-injection-syntax-sugar-over.html">here</a>. What we care about is the result of function composition, which is being done inside one method. We don't care about partial steps and about their implementation. We are interested in the result, to be more precise we check <b>expected state</b>.
</p>

<p style="text-align:justify;">
I bolded state not without a reason. Classical TDD user would tend to use stubs and check state, mockist TDD user likes mocks and <b>behaviours</b>. More about that you can read in <a href="http://martinfowler.com/articles/mocksArentStubs.html">famous Fowler's article</a>. So what is actually wrong in checking partial steps in such scenarios? Well, it leaks implementation details. Now, we force the implementation to actually use PaymentCreator in a fixed way, contrary to stub, which can be desribed as "<b>if</b> you use one of my methods, I'll return this fixture". Both classical and mockists TDD enthusiasts have their rights, but in my opinion those two concepts should not be combined.
</p>

<p style="text-align:justify;">
I express this opinion to one of my friends and he replied: "But what if your first step does something very important that you want to check. Plus it returns value that is used in next lines. Let's say it sends an email". That is a fair question and for simplicity reasons let's assume that mail sender sends information about payment being created to the client and actually really returns something that we care of later (for example a result indicating mail failure/success). The approach showed at the beginning of this article solves that issue, but...
</p>

***Test isolation***

<p style="text-align:justify;">
You want your tests to fail for one reason. That is one of the most ground rules when writing tests. That means that we don't want to check how the payment looks like and if client was informed in the same test. To me, the tests should look like:</p>

```groovy
def "should create outgoing payment for loan"() {
        given:
            paymentCreator.createOutgoing(client, ANY_AMOUNT) >> payment
        when:
            Payment actualPayment = payments.saveOutgoingFor(client, ANY_AMOUNT)
        then:
            1 * repository.save(payment)
            actualPayment == payment
    }

    def "should store the payment"() {
        given:
            paymentCreator.createOutgoing(client, ANY_AMOUNT) >> payment
        when:
            payments.saveOutgoingFor(client, ANY_AMOUNT)
        then:
            1 * repository.save(payment)
    }

    def "should inform the client"() {
        when:
            payments.saveOutgoingFor(client, ANY_AMOUNT)
        then:
            1 * mailSender.informClient()
    }
```
<p style="text-align:justify;">
That way every test fail only for one reason. Not more. It is easier to refractor code, it stops rigidity. Rigidity means that the software is difficult to change. A small change causes a cascade of subsequent changes. In that case, small change breaks unrelated tests.</p>


***My personal rule of thumb***

<p style="text-align:justify;">
I (try to) practise TDD all the time. I faced descirbed problems several times, and I came up with my rule when to use mocks and when go for stubs. I am a big fan of <b>CQRS</b> idea and I truly love it how it fits into above considerations. So when my unit test needs to check whether some action was perfomed (command was sent) I choose mocks and verify behaviours. When i rely on my collaborators to return something (queries) I always go for stubs.
</p>

