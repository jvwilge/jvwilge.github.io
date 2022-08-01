---
layout: default
title: Using EasyRandom and Lombok's .toBuilder to improve the sustainability of your Unit tests
exerpt: 
date: 2022-08-01 08:43:55
tags : easy-random java lombok spring
categories : en
---
Using EasyRandom and Lombok's .toBuilder to improve the sustainability of your Unit tests
===

This article will show you an alternative for test fixtures and/or using Spring's `ReflectionTestUtils` by using random pojo's and Lombok's `.toBuilder()` which will greatly improve the long-term understandability of your unit tests.

<!--more-->

I always call setting up large domain objects "decorating the Christmas tree". It is boring, a lot of work and there is no way to see which pieces are important when you're done.

I'll start with a little history of our Christmas trees, so feel free to cut to the [chase](#easyrandom--tobuilder).

Our environment
---

The important ingredients to be aware of are Spring Boot (specifically the `ReflectionTestUtils` class), Lombok (the Builder functionality) and JUnit. 
The example domain object is an `Order` that can contain multiple `Payment` objects.

Setting up domain objects inside the unit test
---

This is the first stage of the evolution of our unit test. For every test you create a new domain object with all the bells and whistles. You can improve this by moving duplicate code to a method, but it is still much work.
An example:

```java
@Test
void testOrder() {
        var payment=Payment.build()
            .creditAccountNumber("c001")
            .debitAccountNumber("d001")
            .amount(88)
            .description("new stapler")
            .build();
        var testOrder=Order.builder()
            .totalAmount(123)
            .payments(List.of(payment1))
            .trackingCode("abcdef")
            .build();
        
        var result = new OrderService().submitOrder(order);
        
        // should overwrite totalAmount with total amount of payments
        assertThat(result.getTotalAmount())
            .isEqualTo(88);
}
```

Biggest disadvantages here are that there is no way to see which fields are important for the test and it is a lot of typing. If you add a field to the `Order` you might have to change all the tests where this class is used.

Test Fixtures
---

The next level of testing would be a test fixture class, here you define the objects in static methods in a Fixture class:

```java
public class OrderFixture {

    public static Order correct(Payment payment) {
        return Order.builder()
                .totalAmount(123)
                .payments(List.of(payment))
                .trackingCode("abcdef")
                .build();
    }
}
```

```java
@Test
void testOrder() {
        var testOrder = OrderFixture.correct(PaymentFixture.correct());
        
        var result = new OrderService().submitOrder(order);
        
        // should overwrite totalAmount with total amount of payments
        assertThat(result.getTotalAmount())
            .isEqualTo(88);
}
```

This will clean up your code and you'll only have to change this class if a field is added. A new downside is that changing a value here might break some tests because you don't know which fields are important for which tests.

In the long term test fixtures also 'rot away'. The values that once were important or valid might not be that anymore and the values of the fields effectively become random values. Objects can even become inconsistent and people usually do a sloppy fix because they didn't touch the breaking tests (this is how developers work, don't fight it, nudge them in the right direction with a solution).

Test Fixtures + `ReflectionTestUtils`
---

`ReflectionTestUtils` is a convenience class to set and get fields via java reflections.
In our case `ReflectionTestUtils` was an improvement to our way of testing, but many people consider this a huge smell.

We introduced `ReflectionTestUtils` to override fields that are important for a test. In the previous example `amount` and `totalAmount` are important. This makes the assumption the fields in the Fixture are not important and can be changed if desired.

```java
@Test
void testOrder() {
        var testPayment = PaymentFixture.correct();
        ReflectionTestUtils(testPayment, "amount", 44);
        
        var testOrder = OrderFixture.correct(testPayment);
        ReflectionTestUtils(testOrder, "totalAmount", 999);
        
        var result = new OrderService().submitOrder(order);
        
        // should overwrite totalAmount with total amount of payments
        assertThat(result.getTotalAmount())
            .isEqualTo(44);
}
```

Now we're stuck with `ReflectionTestUtils` but it is clear which fields are **probably** important (probably because the data in the Fixture might also be important, but we don't know). At least we can show our intentions.
Another downside here is that if we rename a field a bunch of tests will probably break because of the hardcoded fields names in the `ReflectionTestUtils` constructor.


Test Fixtures + `.toBuilder`
---

To prevent tests breaking after renaming fields we added `toBuilder = true` to all our `@Builder` and `@SuperBuilder` annotations. This is still a bit smelly since we adapted production code for our unit tests, but it will make our test much more maintainable.

The `.toBuilder` method creates a new object from an existing one (by copying all the fields, leaving the old object intact). We now can ditch `ReflectionTestUtils` and make refactoring easier. So you basically override fields on the new object.

```java
@Test
void testOrder() {
        var testPayment = PaymentFixture.correct().toBuilder()
            .amount(44)
            .build();
        
        var testOrder = OrderFixture.correct(testPayment).toBuilder()
            .totalAmount(999)
            .build();
        
        var result = new OrderService().submitOrder(order);
        
        // should overwrite totalAmount with total amount of payments
        assertThat(result.getTotalAmount())
            .isEqualTo(44);
}
```


# EasyRandom + `.toBuilder`

Our most recent improvement was the introduction of (EasyRandom)[https://github.com/j-easy/easy-random] to create random domain objects. This article isn't about how to use EasyRandom and there are lots of articles to get started already. So I'll skip that part.

Our whole EasyRandom setup is hidden behind a static object called `EASY_RANDOM` (for starters you can use `public static EasyRandom EASY_RANDOM = new EasyRandom();`, this should cover many cases).

```java
@Test
void testOrder() {
        var testPayment = EASY_RANDOM.nextObject(Payment.class).toBuilder()
            .amount(44)
            .build();
        
        var testOrder = EASY_RANDOM.nextObject(Order.class).toBuilder()
            .payment(testPayment) // omit this and you will get a random Payment
            .totalAmount(999)
            .build();
        
        var result = new OrderService().submitOrder(order);
        
        // should overwrite totalAmount with total amount of payments
        assertThat(result.getTotalAmount())
            .isEqualTo(44);
}
```

At first sight this might look a lot like the previous step, but now each time you run the test the assumed unimportant values will change. This might give you flaky tests, but they should be easy to fix since you're probably randomizing a field that is being used in the unit tests and should be there anyway. Be careful with types with few values (like booleans or simple enums), they might succeed many times and then suddenly fail (and of course this will happen in you CI/CD environment).

An `Order` converted to json might look like this (before overriding the fields):

```json
{
  "totalAmount": -1188957731,
  "payments": [
    {
      "creditAccountNumber": "eOMtThyhVNLWUZNRcBaQKxI",
      "debitAccountNumber": "RYtGKbgicZaHCBRQDSx",
      "amount": 1295249578,
      "description": "yedUsFwdkelQbxeTeQOvaScfqIOOmaa"
    }
  ],
  "trackingCode": "PBzMiJFouxILNv"
}
```

Note that `totalAmount` has a negative value, this might trigger some unexpected things (like a validation failing or some branch in your code going hay wire). You can change the behaviour of EasyRandom by setting `easyRandomParameters` on the `EasyRandom` object. Example:

```java
new EasyRandomParamters()
.randomize(FieldPredicates.named("amount").and(ofType(BigDecimal.class)), () -> {
    // there are currencies with no decimals, int makes sure this always works
    return new BigDecimal(getEasyRandom.get().nextInt()).abs();
})
```

In the end there are some downsides like flaky tests, but you will also discover that some values don't act as you'd expect which will make your code more robust. The readability in my opinion far outweighs the downsides, especially with very old test fixtures.



Conclusion
---

I hope this article was helpful.

If you have any comments, improvements or spotted a mistake please reach out on [twitter.com/jvwilge](https://twitter.com/jvwilge), thank you for reading!


> First published on August 1, 2022 at [jvwilge.github.io](http://jvwilge.github.io)