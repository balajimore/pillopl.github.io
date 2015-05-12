---
layout: post
title: Why injecting by constructor should be preffered?
comments: true
---
<p style="text-align:justify;">
Seeing the famous <a href="http://google-collections.googlecode.com/svn/trunk/javadoc/com/google/common/annotations/VisibleForTesting.html">@VisibleForTesting</a> guava's annotation for the first time made me thinking. Without looking at the implementation I was pretty sure it is magical piece of code which silently makes our properties visibile only in tests without breaking carefully (?) created encapsulation. Later on, I decided to look at the implementation and javadocs. Here goes the latter one:
</p>
<blockquote class="cite">
      <p>An annotation that indicates that the visibility of a type or member has been relaxed to make the code testable.</p>
</blockquote>
<p style="text-align:justify;">
So it becomes useful for developers who write unit tests - which is good, because everyone should do so. But then it hit me. At the same time it's handy for **lazy** ones - which is bad. Let's hear that again - <i>visibility of a type or member has been relaxed to make the code testable</i>. Changing the production code for the sake of tests' simplicity is just fine and in fact, this is among others what tests are intended for. But breaking encapsulation, which is one of the foundations of OOP programming seems like a step too far. In fact, enthusiast of Test Driven Development should be worried - should not the tests themselves lead to proper design?
		   
Indeed. They should. And in most cases they probably will, but what if we are to test legacy code or modify already existed tests? Consider this excerpt:</p>

```java
/* CarService.java */

class CarService {
	@Autowired private RepairService repairService;
	@Autowired private WashService washService;
}

```  
<p style="text-align:justify;">
At first glance it looks just fine. Spring's field injection feature will work it out by using reflection. And how would the unit test of that class look like? We have to wire up the dependencies somehow, so in the early days we would have ended up in relaxining the visibility and (in the best scenario) annoting fields/setters with above mentioned @VisibleForTesting annotation. Then, the idea of <a href="http://docs.mockito.googlecode.com/hg/1.9.5/org/mockito/InjectMocks.html">Mockito's @InjectMocks</a> was born. The mechanism is pretty much the same as in Spring - Mockito uses reflection to setup dependencies. In fact, Spring provides it's own mechanism to setup the test suite, it is called <a href="http://docs.spring.io/spring-framework/docs/2.5.x/api/org/springframework/test/util/ReflectionTestUtils.html">ReflectionTestUtils</a>. It is even simplier with <a href="https://code.google.com/p/spock/">Spock</a>, because groovy's magic allows us to modify hidden fields. Anyways, traditional mockito-like tests most probably would look as follows:</p>

```java
/* CarServiceTest.java */

class CarServiceTestg {
	@Mock private RepairService repairService
	@Mock private WashService washService
	@InjectMocks private CarService carService
	
	@Test
	public should...
}

```  
<p style="text-align:justify;">
Test cases are omitted for brevity. One may ask what is wrong with that code sample. The are at least two things to consider. Let's examin Mockito's javadoc.</p>

<blockquote class="cite">
      <p>Mockito will try to inject mocks only either by constructor injection, setter injection, or property injection in order and as described below. If any of the following strategy fail, then Mockito <strong>won't report failure</strong>; i.e. you will have to provide dependencies yourself.</p>
</blockquote>
<p style="text-align:justify;">
Mockito, unlike Spring's default behaviour, will fail silently if it cannot inject dependencies. That means that if someone adds new dependency, say CarRegister, to our CarService then test will probably fail with undescriptive NullPointerException. Neither can we use final fields, hence forget about immutability. Practicing TDD would never lead us to add mock of CarRegister in our test suite, because this dependency **is not visible** for CarService's clients.</p>

***Do not inject by fields!***
<p style="text-align:justify;">
The root of the all evil is not Mockito, which is a brilliant testing tool. The problem lies in code's design. To be more specific, the field-based injection causes all the commotion. Let's take a look at refractored version of Car:</p>

```java
/* CarService.java */

class CarService {

	private final RepairService repairService;
	private final WashService washService;
	
	CarService(WashService washService, RepairService repairServie) {
		this.repairService = Objects.requireNotNull(repairService);
		this.washService = Objects.requireNotNull(washService);
	}
}

```  
<p style="text-align:justify;">
One may ask, what value bring this refractoring to us? The answer is: a lot. Firstly, all of the sudden CarService became immutable. Moreover, the dependencies are clear and visible to all CarService clients. All of this was achived by using different strategy of dependency injection. In my opinion, the gain is pretty big. Last, but not least - now the test is straightforward and we do not need to use Mockito's hidden magic tricks
	
But constructor-based injection comes in handy not only in tests. The constuctor's signature tells about the only possible way to create a valid instance of some class. In that way, we will not end up with **half-created object**, which could not wire up not present fields (in given example, Spring will raise an exception, but consider a more general picture). That means that we are forced to initilize objects as follows:</p>

```java

CarService carService = new CarService(washService, repairService);

```  

or even like that:

```java

CarService carService = new CarService(null, null);

```  
<p style="text-align:justify;">
So even if we ignore dependencies we stricly have to prove being aware of their presence. Forget about injecting mock by Mockito's tricks.
	
One may say, that adding new dependency by field-based injection does not force us to modify already complex constructor. While constructor-based injection indeed requires more code to write, the problem is elsewhere. Adding new property is super-simple, but enlarging constructor should make us thinking about the design. Presence of tone of dependencies in our constructor might be clean sign that our class violates <a href="http://code.tutsplus.com/tutorials/solid-part-1-the-single-responsibility-principle--net-36074">Single Responsibility Principle</a> and should be sliced. Yet another advantage of injecting by constructor.</p>

****