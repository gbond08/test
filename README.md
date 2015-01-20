# test
#A brief guide to becoming productive with FilterCoffee

This guide is aimed to help people who plan on using the new functional testing rails (FilterCoffee) in the near future. We will start by covering the motivation behind building FilterCoffee, its architecture, and what features it provides to simplify the testing process. We will then get hands-on with some real examples that should help you understand how the test are written for different use-cases. Finally, we illustrate what the introduction of a new service (REST/ASF) typically entails. By the end of this guide you should be equipped to add tests for existing and new features/products. If you have any questions feel free to email me at sahjain@paypal.com

## Table of Contents
* **[Motivation](#motivation)**
* **[Architecture](#architecture)**
 * **[An overview of atomic units: ApiUnit, ValidationUnit, ExecutionUnit](#an-overview-of-atomic-units-apiunit-validationunit-executionunit)**
 * **[Arriving at the design](#arriving-at-the-design)**
 * **[An overview of the flow of the engine under the hood](#an-overview-of-the-flow-of-the-engine-under-the-hood)**
 * **[Architectural diagram](#architectural-diagram)**
* **[Examples](#examples)**
 * **[A simple example with light validation](#light-validation)**
 * **[A simple example with heavier validation](#heavier-validation)**
 * **[An example where predefined templates aren't enough](#new-validation)**
* **[Adding to FilterCoffee](#adding-to-filtercoffee)**

##Motivation
FilterCoffee started as an attempt to simplify the functional testing process. With several lessons learned from the existing functional tests, we sought to address the following: 
* Wide array of redundancies across test cases and packages
* Discoverability of potentially re-usable code
* Inconsistencies in testing quality/validations
* Verbosity of test cases and general util/base classes
* Unnatural design for writing data-driven tests
* Slow performance in execution time
* Reliability in the face of parallel execution
* Maintainability

The existing functional tests' biggest strength was its flexibility, but that turned out to be a double edged sword which in turn led to many of the problems outlined above. It became apparent that the right way to address the above problems was to introduce a set of patterns; these patterns together is what we've dubbed a "framework/rails" of the name FilterCoffee.

##Architecture:
In order to properly understand the architecture, it's important to first survey the core building blocks of FilterCoffee and understand what role they play. After that the general flow, architectural patterns and assumptions will begin to make more sense.

###An overview of atomic units: ApiUnit, ValidationUnit, ExecutionUnit
Let's start with something simple. If you're trying to talk to an Api, what are the minimum possible things you need? For starters, you would need to know what api exactly you're trying to call. In the context of a RESTful api call, this means knowing the HTTP verb (ex: GET/POST etc) along with the endpointURL (ex: /v1/payments/checkout-sessions on a stage with the right port). In the context of an ASF call, this means knowing the exact api "method" (ex: planpaymentv2 via the MoneyPlanning client interface). Without getting too pedantic, we can just say that the first requirement for anyone to call an api is know what the "apiMethod/endpoint" is. Next, depending on the api requirements, there could be an associated request payload that you have to supply. Also, it's important to declare who *you* are as a client in charge of triggering the api (usually a merchant/seller). Since we're in the business of enabling e-commerce, it's not unusual to have a "buyer" purchasing something from the said merchant/seller.

So in the context of a transaction, all we really need to successfully call an api is: 
   * Api method (always)
   * Request payload (sometimes)
   * Seller (always)
   * Buyer (sometimes)

These fields together constitute an [ApiUnit](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/SymphonyBluefinTests/src/test/java/com/paypal/test/filtercoffee/source/ApiUnit.java) in FilterCoffee. It's just a simple POJO abstraction that captures all the necessary details required to "call" an api.

While this is all great news to "capture" necessary details neatly to "call" an api, what about being able to "test" the response? At its most basic level, "testing" can be thought of asserting whether an actual value is the same as the expected value. But actual value of what exactly? It usually helps to have the smallest level of granularity, and since any response can be visualized as a composition of its subfields, the simplest way to capture an "expectation" is to outline the "field" you're interested in, along with its "expectedValue". This is exactly what a [ValidationUnit](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/SymphonyBluefinTests/src/test/java/com/paypal/test/filtercoffee/source/ValidationUnit.java) is: a simple POJO capturing the expectedValue of your desired field in a response.

Finally, an [ExecutionUnit](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/SymphonyBluefinTests/src/test/java/com/paypal/test/filtercoffee/source/ExecutionUnit.java) is nothing more than a POJO that encapsulates an ApiUnit and a list of ValidationUnits.

###Arriving at the design
Instead of immediately explaining the current design, we'll walk through a basic functional test iteratively and see how it could motivate the design for filterCoffee. Please note that this is simply a discussion of design requirements at a conceptual level. If you feel like you already understand filterCoffee at a high level, free to skip this section.

```java
public void endToEndTestPseudoCode() {
  // orchestration assuming buyer/seller have been created
  SetEC();
  createCheckoutSession();
  approveCheckoutSession();
  GetEC();
  DoEC();
}
```

The above is a highly stripped down version of what a simple end to end orchestration may look like, but there are several key missing pieces. For starters:
* Methods like SetEC() and DoEC() need to operate on the seller's information. createCheckoutSession() and approveCheckoutSession() need both the buyer and seller. How can these methods not consume this critical information?

This is a legitimate concern, so let's go ahead and add the relevant inputs:

```java
public void endToEndTestPseudoCode() {
  // orchestration assuming buyer/seller have been created
  SetEC(seller);
  createCheckoutSession(buyer, seller);
  approveCheckoutSession(buyer, seller);
  GetEC(seller);
  DoEC(seller);
}
```

While this is a mild improvement, it raises more important questions:
* SetEC() and DoEC() on their own don't make sense. After all, there could be several flavors of SetEC/DoEC. How can we dictate which one we want to use?

There's many ways to tackle this. You could manually take up the responsibility of creating the request on your own and supply the request as a parameter to the SetEC() method. Alternatively, you could get the SetEC() method to delegate to a request constructor of some sort underneath provided that you can supply it some metadata about the nature of the request. This looks like a good opportunity to use payloadTemplates. If you recall the previous section where we covered the concept of payloadTemplates, you'll remember that they are essentially concise aliases that help in guiding the way a request is constructed (usually a request body/payload) for a particular api call. The same applies here such that we supply the appopriate payloadTemplate to the SetEC() method, and it takes care of everything else underneath. In FilterCoffee's case, if you're wondering where to find these predefined templates for SetEC/DoEC, just think about the underlying service that is relevant: ExpressCheckout. You'll find a class called [ExpressCheckoutPayloadTemplates](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/SymphonyBluefinTests/src/test/java/com/paypal/test/filtercoffee/payloadtemplate/ExpressCheckoutPayloadTemplates.java) in the [payloadtemplate package](https://github.paypal.com/Payments-R/paymentapiplatformserv/tree/develop/SymphonyBluefinTests/src/test/java/com/paypal/test/filtercoffee/payloadtemplate) that captures all the EC-related payloadTemplates for SetEC, GetEC, and DoEC. If you goto this class you'll find a [SetECTemplate](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/2193b4a2d27dfbde983fdacb0122aa960ab5eb4a/SymphonyBluefinTests/src/test/java/com/paypal/test/filtercoffee/payloadtemplate/ExpressCheckoutPayloadTemplates.java#L22) enum that lists different flavors like "SIMPLE_SALE_USD". Another example: If you're interested in calling POST/Cart (i.e. createCart), you'll find yourself faced with the same question - "Which kind of cart am I supplying, exactly?" Now you know that you can goto the right package and look for [CartPayloadTemplates](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/SymphonyBluefinTests/src/test/java/com/paypal/test/filtercoffee/payloadtemplate/CartPayloadTemplates.java) where you'll find an enum called [CreateCartTemplate](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/2193b4a2d27dfbde983fdacb0122aa960ab5eb4a/SymphonyBluefinTests/src/test/java/com/paypal/test/filtercoffee/payloadtemplate/CartPayloadTemplates.java#L22). Looking to create a payment in a specific fashion? You'll find a class called PaymentPayloadTemplates where you'll see an enum called CreatePaymentTemplate. That's the pattern to follow and hopefully this gives you a sense of how to locate relevant PayloadTemplates while authoring tests.

So to recap, you've defined what api you want to call (SetEC), what entities you want involved (buyer/seller), and you've also defined what payloadTemplate to use (ex: SetECTemplate.SIMPLE_SALE_USD). But how exactly is this template consumed? This is where we arrive at a notion of RequestMappers. These mappers are responsible for using the payloadTemplates to construct the appropriate request. So once again, if you wanted to see how this template is consumed conceptually (without being too quick to hit Ctrl+Shift+G on your IDE), you can find a relevant class called [ExpressCheckoutServiceRequestMapper](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/SymphonyBluefinTests/src/test/java/com/paypal/test/filtercoffee/service/mapper/ExpressCheckoutServiceRequestMapper.java) located in the [mapper package](https://github.paypal.com/Payments-R/paymentapiplatformserv/tree/develop/SymphonyBluefinTests/src/test/java/com/paypal/test/filtercoffee/service/mapper). In this class, you'll find separate public methods like mapSetECRequest(), mapGetECRequest() and mapDoECRequest(). In the mapSetECRequest(), you'll notice that we handle the different SetECTemplates in a switch statement, which then route to an ecApiFactory method (ex: ecApiFactory.buildSimpleSetExpressCheckoutRequest()). In this case, there's a simple static mapping between an alias and the request construction.

Equipped with this information, we can revise our previous test like the following:

```java
public void endToEndTestPseudoCode() {
  // orchestration assuming buyer/seller have been created
  SetEC(SetECTemplate.SIMPLE_SALE_USD, seller);
  createCheckoutSession(buyer, seller);
  approveCheckoutSession(buyer, seller);
  GetEC(seller);
  DoEC(DoECTemplate.SIMPLE_SALE_WITH_INSURANCE_AND_SHIPPING_DISC, seller);
}
```

This alleviates some anxiety, but wait! We've arrived at yet another important question:
* createCheckouSession() couldn't possibly be successful without providing the cart_id in the request body/payload. The cart_id comes from the SetEC response, so who's taking care of that? approveCheckoutSession() also can't be successful without providing the cart_id in the uri itself, so who's taking care of that?

If you've understood the previous section you'll know that this responsiblity lies squarely in the domain of the relevant RequestMapper (in this case, the CheckoutSessionServiceRequestMapper). If there was some _simple way_ for us to "supply" the result of the SetEC api response to the mapCreateCheckoutSessionRequest() method, then we should be able to comfortably build the request body with the cart_id (i.e. ec-Token). But how is the requestMapper supposed to know the preceding api's response? What is this nebulous _simple way_ that we speak of? If you think about it, this problem can be solved by the introduction of some kind of "transaction api response context" that __records__ every api's response such that the response could be used for the construction of a subsequent request. Let's say that want to tackle this in the following way:

```java
public void endToEndTestPseudoCode() {
  // orchestration assuming buyer/seller have been created
  
  // create map that is responsible for "holding" results
  Map<SomeApi, ThatApisResult> apiResponseMap = new HashMap<SomeApi, ThatApisResult>();
  
  // call api and store result
  SetECResponse = SetEC(SetECTemplate.SIMPLE_SALE_USD, seller, apiResponseMap); // empty map at this point
  apiResponseMap.put(SetEC, SetECResult);
  
  // call api and store result
  CreateXOSessionResponse = createCheckoutSession(buyer, seller, apiResponseMap);
  apiResponseMap.put(CreateCheckoutSession, CreateXOSessionResponse);
  
  // call api and store result
  ApproveXOSessionResponse = approveCheckoutSession(buyer, seller, apiResponseMap);
  apiResponseMap.put(ApproveCheckoutSession, ApproveXOSessionResponse);
  
  // call api and store result
  GetECResponse = GetEC(seller, apiResponseMap);
  apiResponseMap.put(GetEC, GetECResponse);
  
  // call api and store result
  DoECResponse = DoEC(DoECTemplate.SIMPLE_SALE_WITH_INSURANCE_AND_SHIPPING_DISC, seller, apiResponseMap);
  apiResponseMap.put(DoEC, DoECResponse); //We could technically do without this line
}
```

Using this approach it becomes clear that the createCheckoutSession() method isn't doomed. It has a simple way of reaching into the apiResponseMap and finding the precending SetEC response, which it can extract the ec-Token from and "apply" that as the cart_id in the request payload. __< PedanticAside >__ Of course, you could argue that instead of supplying the whole map, if our method in the pseudocode just supplied the ec-Token alone we should be fine. While this applies in the above example, a closer survey of other examples will show that sometimes the supply of a token alone isn't enough. There are many examples where a contingency scenario necessitates the inclusion of an additional query parameter in the uri depending on the flow, so instead of creating mutliple methods with different signatures having different parameters, this is a simple way to consolidate it behind one singular interface. DoEC would require the GetEC response (payerID) and the SetEC response (ec-Token), so instead of creating a method signature taking in both results, we can just make the sure the method takes in the apiResponseMap and handles the extraction of relevant results. This way every single api starts having a standard interface and avoids polymorphic pollution. __< /PedanticAside >__

So it looks like we've resolved the createCheckoutSession dilemma now that it knows __where__ to find the SetEC result (in the apiResponseMap). The same would apply for the approveCheckoutSession where we can extract the ec-Token to apply in the URI itself (but not the request body which is an empty json). If you think about GetEC(), all it needs is the ec-Token and seller credentials in order to get a payerID from EC. Since it has the apiResponse handy, this should be taken care of as well.

All this was a fun pseudo-code exercise that outlines the simplest way we can successfully orchestrate a number of apis. But what does this look like using FilterCoffee? Well, not so different actually:

```java
public void endToEndTestRealCode() {
  // orchestration assuming buyer/seller have been created
  
  filterCoffee.orchestrate(
      ExpressCheckout.Set, SetECTemplate.SIMPLE_SALE_USD,
      CheckoutSession.CREATE,
      CheckoutSession.APPROVE,
      ExpressCheckout.GET,
      ExpressCheckout.DO, DoECTemplate.SIMPLE_SALE_WITH_INSURANCE_AND_SHIPPING_DISC,
      buyer, seller);
  )
}
```

Here's what happens the moment you invoke "orchestrate": 
###__Step1: All the input data passed passes through a utility called ExecutionUnitListBuilder__
As you may have guessed, that utility is responsible for building executionUnits. In our example, this entire orchestration becomes composed of an executionUnit per api call (so, 5 executionUnits). The 1st exectionUnit will contain the ExpressCheckout.SET (i.e. SetExpressCheckout) apiMethod, the SetECTemplate.SIMPLE_SALE_USD payloadTemplate, buyer and seller. The 2nd one will have CheckoutSession.CREATE (i.e. createCheckoutSession) apiMethod, no payloadTemplate, buyer and seller etc. At this point, it should become intuitive that every single orchestration can be visualized as a sequence list of execution units. In the case of the filterCoffee engine, this is nothing more than a simple for-loop that handles the list of execution units.

###__Step2: The relevant serviceImpl is invoked __
When you supply something like CheckoutSession.CREATE, it's obvious that there needs to be an entity that can handle the (a) delegation of request body mapping (b) construction of headers and uri (c) invocation of the api (d) delegation of the response assertion/validation. This is where the notion of serviceImpls comes in. If you survey the CheckoutSession endpoint enum, you'll notice that it supplied the CheckoutSessionServiceImpl which in turn handles all the checkout-session based endpoints. This serviceImpl manages the uri path construction on its own, delegates to the CheckoutSessionServiceRequestMapper to generate the request body, invokes the api endpoint through the ServiceBridge, passes the result to the CheckoutSessionServiceResponseValidator in order to validation the response. This step will be discussed in greater detail later in this documentation.

###__Step3: The response is stored __
Once the api call has been triggered and validated, the result is stored in an apiResponseMap that is incrementally built. Remember from earlier how we managed the interdependence of api responses and requests? This is taken care of automatically in the orchestrate method, and the map is passed down to each serviceImpl that may need it for selective extraction.

What we haven't covered so far:
###This is a large 

##Examples

##Adding to FilterCoffee

A walkthrough of a simple example with light validation:
< Go step by step explaining what the test does under the hood >

Talk about what the significance of payloadTemplates and validationTemplates is

A walkthrough of an example with heavier validation:
< Pick up an example of a Prox/DoEC test case that inhales the cart and performs a patch >

A walkthrough of an example with dynamically generated templates:
< Pick up the buyersetup example and talk about reducing the template explosion >

A walkthrough of implementing a new mapping within the existing templates:
< Pick some unimplemented mapping and talk about what all that could influence >

A walkthrough of implementing a new RESTful service end to end in filterCoffee

A walkthrough of implementing a new ASF service end to end in filterCoffee

A note on generous dataProvider use and disciplined formatting

FAQ
