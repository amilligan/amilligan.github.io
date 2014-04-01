Imagine you need to create a software product that models going to a bank teller and withdrawing money.  You have a Customer instance and a Teller instance.  When you send the `withdraw` message to the customer, you expect it to interact with the teller.  When you're testing the interaction you don't want a real teller to withdraw real money, so you might test with a mock object:

```objc
describe(@"-withdraw", ^{
  beforeEach(^{ [customer withdraw:money fromAccount:account]; });
  it(@"should begin a transaction with the teller", ^{
    teller should have_received("transactRequest:");
  });
```

Perhaps the teller expects you to specify the details of the request (e.g. withdrawal vs. deposit, which account to use, and the amount) with a TransactionRequest object.  You could test this like so:

```objc
  it(@"should correctly specify the details of the transaction", ^{
    transactionRequest.type should equal(TransactionRequestTypeWithdrawal);
    transactionRequest.account should equal(account);
    transactionRequest.amount should equal(money.amount);
    transactionRequest.currency should equal(money.currency);
  });
```

Now the teller has to muck about behind the teller window for a bit.  Perhaps the customer should express impatience in some way:

```objc
  it(@"should cause the customer to wait impatiently", ^{
    customer.isTappingFoot should be_truthy;
  });
});
```

Eventually the teller should return to the window and either hand cash to the customer, or notify the customer that the account has insufficient funds.  Since the interaction is asynchronous, the teller could respond to the customer via delegate methods.  Now our teller object is a mock, so we don't expect it to really do anything, but we do want to test the behavior of the customer when the transaction completes:

```objc
describe(@"when the withdrawal succeeds", ^{
  before(^{ [customer transactionSucceeded]; });
  it(@"should cease waiting impatiently", ^{
    customer.isTappingFoot should_not be_truthy;
  });

  it(@"should now be able to buy that pony", ^{
    customer.canBuyPony should be_truthy;
  });
});

describe(@"when the withdrawal fails", ^{
  before(^{ [customer transactionFailed:reason]; });
  it(@"should cease waiting impatiently", ^{
    customer.isTappingFoot should_not be_truthy;
  });

  it(@"should weep inconsolably", ^{
    customer should_not be_consolable;
  });
});
```

This is a fairly straightforward example of unit testing a class with a mock object.  This is also *exactly* how you might test a client making a network request to a server:

1. Create request with URL, parameters, and HTTP method
2. Invoke request via NSURLSession
3. Wait, displaying some form of activity indicator
4. Handle success or failure response

Tragically, and unsurprisingly, Apple's new(ish) `NSURLSession` makes it practically impossible to write unit tests for network activity that run quickly and deterministically.  Like with the interaction with the Teller, we don't want to make actual network requests[^1], but instead interact with a stubbed version of the network infrastructure.  How, then, to stub the network infrastructure?

We could try reopening `NSURLSessionDataTask` in the test target.  Code that is invoking a network request first instantiates a task object, then calls `resume` to start the request; we can reimplement that method to store the provided `NSURLRequest` for later interrogation.  Unfortunately, instantiated tasks are not actually instances of `NSURLSessionDataTask`, but `__NSCFURLSessionDataTask`; it's a class cluster.  Since we don't have access to the "private" implementation class, we can't reimplement the actual `resume` method.  Strike one.

Perhaps we can reimplement the initializer for `NSURLSessionDataTask` to return a test stub?  That doesn't work either, because the initializers for the task objects are not publicly accessible.  Strike two.

We can't directly reimplement, or call, the initializer for `NSURLSessionDataTask`, but you create a task object by calling `dataTaskWithURL:` or `dataTaskWithURL:completionHandler:` on an `NSURLSession` instance.  Perhaps we can reimplement these factory methods to return test stubs?  Unfortunately no, because `NSURLSession` is a class cluster as well, and we can't access the `__NSCFURLSession` class.  We've already run afoul of this problem, so let's say strike two and a half.

We can't seem to crack [`NSURLSession`](https://developer.apple.com/library/ios/documentation/Foundation/Reference/NSURLSession_class/Introduction/Introduction.html), so how about [`NSURLProtocol`](https://developer.apple.com/library/mac/documentation/cocoa/reference/foundation/classes/NSURLProtocol_Class/Reference/Reference.html)?  The Cocoa networking infrastructure allows you to register protocol classes for custom handling of network requests; we can provide custom handling for network requests invoked during tests.  Unfortunately, prior to the introduction of `NSURLSession`, which `NSURLProtocol` class handled a network request was entirely independent of the code making the request, but with `NSURLSession` you must register the `NSURLProtocol` class with the specific `NSURLSession` instance.  This means that the code making the requests must be aware of the protocol class; we can't register a test protocol class in our test target only.  Foul tip.

So we create an `NSURLProtocol` subclass that always returns `NO` in the `+canInitWithRequest` method.  In the test target we reimplement this method to return `YES`, and reimplement `-startLoading` to simply record the associated request.  Unfortunately, by the time the networking code has reached this point it's running in another thread, and tests will not pass deterministically (if at all).  We can use a `dispatch_semaphore` to ensure the code runs in the proper order, but this requires sprinkling semaphore waits throughout the unit tests; forget a semaphore, and you have non-deterministic (or simply broken) tests.  Strike three.

<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<


1: Sure, lots of people have written about connecting to a proxy server and testing that way.  That's nice for manual testing, but unit tests should run without external dependencies of any kind; they should emphasize speed and determinism above all else (except, perhaps, correctness).

