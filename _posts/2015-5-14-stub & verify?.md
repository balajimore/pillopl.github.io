---
layout: post
title: Stub & Verify?
comments: true
---

<p style="text-align:justify;">
Lately I several times came across Spock tests, which conceptually looked as follows:</p>

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
            actualPayment == payment
    }
```
<p style="text-align:justify;">
Spock allows you to stub and verify collaborator at once.<a href="https://spockframework.github.io/spock/docs/1.0/interaction_based_testing.html"> See section 'Combining Mocking and Stubbing'</a>. It is followed by sections which describe couple of discouraged solutions to testing, like partial stubs or stubbing static methods. In my opinion above section should also have this sidenote:

<blockquote class="cite">
      <p>(Think twice before using this feature. It might be better to change the design of the code under specification.)</p>
</blockquote>

</p>

