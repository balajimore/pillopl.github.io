---
layout: post
title: Stub & Verify?
comments: true
---

<p style="text-align:justify;">
Lately I several times came across Spock tests, which conceptually looked as follows:

```java
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
</p>
