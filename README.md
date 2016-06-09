# Nodejs - Testing Promises With Sinon and Chai

## Chai-as-promised and Sinon-as-promised - Amazing helpers

To help us test our promises we will be using [sinon-as-promised](https://www.npmjs.com/package/sinon-as-promised) and 
[chai-as-promised](https://github.com/domenic/chai-as-promised). These are two awesome tools that make testing promises
so much more simple!


Adding these to your test file is easy. Add the following code to the beginning of your test file. Mine is called `test.js` 
and lives in `test` directory.

```javascript
var chai = require('chai');
var sinonChai = require("sinon-chai");
var expect = chai.expect;
var chaiAsPromised = require('chai-as-promised');
chai.use(chaiAsPromised);
require('sinon-as-promised');
var sinon = require('sinon');
chai.use(sinonChai);
```



## Our code

### AWS LAMBDA - Quick reference

Note the following code, which is written to run on [aws lambda](http://docs.aws.amazon.com/lambda/latest/dg/welcome.html).
`Lambda` is great! It's worth learning and playing around with. It allows you to write serverless api's if you use it along with 
[aws api-gateway](http://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html).
`Lambda` requires we call a __callback__ at the end of everything having been run with either an error or and succes object. 
All `Lambda` functions receive three parameters __event__, __context__ and __callback__.

### Example code

Consider the following which is part of project that was used to verify webhooks from `Shopify` and run `AWS SQS` methods.

__lambda.js__

```javascript
'use strict';

var main = require('./lib/main');

//Verifies shopify webhooks
//@params {object} event
//@params {string} event.srcHmac
//@params {string} event.rawBody
//@params {string} event.shopName - <shop-name.myshopify.com>
//@params {string} event.productId
exports.shopifyVerifyWebHook = function (event, context, callback) {
  console.log('---- EVENT ----');
  console.log(event);
  main.manageShopifyWebhook(event)
    .then(function(result) {
      callback(null, result);
    })
    .catch(function(err) {
      callback(err, null);
    });
};
```

We know that we need to check that __callback__ is called with the correct arguments and we know that __manageShopifyWebhook__ takes the
__event__ object and returns a promise.

Because our functions are no longer returning/calling callbacks, but promises, which are objects, we cannot stub and call them as 
one would normally do:

```javascript
//Imagine a 'method' that takes a string as its only parameter and returns a string
// You would stub it and have it return the value like so:
var exampleStub = sinon.stub(module, 'method');
exampleStub.withArgs('Hello').returns('Hello World');

//OR

//Imagine a 'method' that has a callback with standard (err, res) as parameters - function method(param1, callback)
// You would stub it and have it call its callback with a successful result like so:
var exampleStub = sinon.stub(module, 'method');
exampleStub.callsArgWith(1, null, 'HelloWorld');
```
    
__For promises__ we would write it a little differently. From the code above we know that `manageShopifyWebhook` returns a
promise.
    
### Testing our example code from above

__test.js__

```javascript
'use strict';

var chai = require('chai');
var sinonChai = require("sinon-chai");
var expect = chai.expect;
var chaiAsPromised = require('chai-as-promised');
chai.use(chaiAsPromised);
require('sinon-as-promised');
var sinon = require('sinon');
chai.use(sinonChai);
var proxyquire = require('proxyquire').noCallThru();

describe('LAMBDA', function() {
  var testedModule,
    mainShopStub,
    callbackSpy,
    mainModule,
    fakeEvent;

  before(function() {
    callbackSpy = sinon.spy();
    fakeEvent = {
      srcHmac: '12345',
      rawBody: 'helloworld',
      shopName: 'mario-test.myshopify.com',
      productId: '6789'
    };
    testedModule = require('../lambda');
    mainModule = require('../lib/main');
    mainShopStub = sinon.stub(mainModule, 'manageShopifyWebhook'); //we stub our method
  });

  after(function() {
    mainShopStub.restore(); // once we have finished the tests, we need to restore the original function
  });

  it('calling shopifyVerifyWebHook returns an error', function() {
    var fakeError = new Error('Error running lambda');
    // define our stub behaviour and write our assertions/expectations
    mainShopStub.rejects(fakeError)().catch(function (error) {
      // this is only called eventually once the stub has run its course
      expect(callbackSpy).has.been.called.and.calledWith(error, null);
    });

    // call our tested module and appy its callback with the arguments that it will receive
    testedModule.shopifyVerifyWebHook(fakeEvent, {}, function() {
      callbackSpy.apply(null, arguments);
    });
  });

  it('calling shopifyVerifyWebHook return a data object', function() {
    var fakeObj = {message: 'success'};
    mainShopStub.resolves(fakeObj)().then(function (result) {
      expect(callbackSpy).has.been.called.and.calledWith(null, result);
    });

    testedModule.shopifyVerifyWebHook(fakeEvent, {}, function() {
      expected.resolves(fakeObj);
      callbackSpy.apply(null, arguments);
    });
  });
});
```
### More examples

#### The example above is great if we have a single promise to test and not a chain. In the following snipped of code, we have a module that calls a series of promises in a chain. How would you control that flow so that you can test that each promise triggers the final catch method if there is an error or run successfully end to end?

#### Proxyquire

For our tests I often use [proxyquire](https://github.com/thlorenz/proxyquire), which is great when you want to override
dependencies when testing. What this does is proxy node's `require` method. It's also pretty straight forward to use and avoids
using node's cache when it requires modules, keeping you tests clean.
Instead of requiring all of your dependencies and the module you want to test individually and then spying/stubbing/mocking them,
you would do the following:

```javascript
var example2methodStub = sinon.stub();
var testedModule = proxyquire('../lib/modules/example, { //the path to your target module is relative to the test file. No file extension
    './utils/example2': { // the path the dependency is relative to the target module
        'methodName': example2methodStub // here we replace 'methodName' with a stub
        }
    });

// AND now we could do whatever we need to do with our stub!!!
```
    
__main.js__

```javascript
'use strict';

var mySQS = require('./modules/sqs/sqs-manager');
var sWebHook = require('./modules/webhooks/shopify/webhooks');

var main = {};

//@params {object} params
//@params {string} params.srcHmac
//@params {string} params.rawBody
//@params {string} params.shopName - <shop-name.myshopify.com>
//@params {string} params.productId

main.manageShopifyWebhook = function (params) {
  return new Promise(function(resolve, reject) {
    sWebHook.verify(params.srcHmac, params.rawBody, params.shopName.split('.myshopify.com')[0], params.productId)
      .then(function(data) {
        var body = {
          "params": {
            "productId": data.productId,
            "shopName": data.shopName
          },
          "job": "call-update-item"
        };
        return mySQS.create_Queue(body);
      })
      .then(mySQS.send_Message)
      .then(resolve)
      .catch(function(err) {
        reject(err);
      });
  });
};

module.exports = main;
```

__These tests continue in the same file as above's tests - test.js__

```javascript
describe('MAIN', function() {
  require('sinon-as-promised');
  var testedModule,
    sWebHookStub,
    sqsQueueStub,
    sqsSendMsgStub,
    callbackSpy,
    fakeDataObj;

// we only want to set these values once, so we use the `before` function
  before(function() {
    sWebHookStub = sinon.stub();
    sqsQueueStub = sinon.stub();
    sqsSendMsgStub = sinon.stub();
    callbackSpy = sinon.spy();
    fakeDataObj = {
      srcHmac: '12345',
      rawBody: 'helloworld',
      shopName: 'mario-test.myshopify.com',
      productId: '6789'
    };
    // require our tested module and stub its dependencies
    testedModule = proxyquire('../lib/main', {
      './modules/webhooks/shopify/webhooks': {
        'verify': sWebHookStub
      },
      './modules/sqs/sqs-manager': {
        'create_Queue': sqsQueueStub,
        'send_Message': sqsSendMsgStub
      }
    });
  });

  it('calling shopifyVeriWebhook returns an error when trying to VERIFY WEBHOOK', function() {
    var fakeError = new Error('Error verifying webhook');
    // we define the behaviour we want our stub to have and we determine what we expect once that has happened
    sWebHookStub.rejects(fakeError)().catch(function(error) {
    // this will eventually be run
      expect(shopifyWebhook).to.eventually.equal(error);
    });
    // we need to call our tested module so that our stub can be called too
    var shopifyWebhook = testedModule.manageShopifyWebhook(fakeDataObj);
  });

  it('calling shopifyVeriWebhook returns an error when trying to CREATE SQS QUEUE', function() {
    var fakeBody = {
      "params": {
        "productId": '1234',
        "shopName": 'name'
      },
      "job": "call-update-item"
    };
    var fakeError = new Error('Error creating sqs queue');
    sWebHookStub.resolves(fakeBody)().then(function(result) {
      sqsQueueStub.rejects(fakeError)().catch(function(error) {
        expect(shopifyWebhook).to.eventually.equal(error);
      });
    });
    var shopifyWebhook = testedModule.manageShopifyWebhook(fakeDataObj);
  });

  it('calling shopifyVeriWebhook returns an error when trying to SEND SQS MESSAGE', function() {
    var fakeData = {
      queueUrl: '5678',
      payLoad: '{"message": "Hello World"'
    };
    var fakeBody = {
      "params": {
        "productId": '1234',
        "shopName": 'name'
      },
      "job": "call-update-item"
    };
    var fakeError = new Error('Error sending sqs message');
    sWebHookStub.resolves(fakeBody)().then(function(result) {
      sqsQueueStub.resolves(fakeData)().then(function(result) {
        sqsSendMsgStub.rejects(fakeError)().catch(function(error) {
          expect(shopifyWebhook).to.eventually.equal(error);
        });
      });
    });
    var shopifyWebhook = testedModule.manageShopifyWebhook(fakeDataObj);
  });

  it('calling shopifyVeriWebhook is SUCCESSFUL', function() {
    var fakeData = {
      queueUrl: '5678',
      payLoad: '{"message": "Hello World"'
    };
    var fakeBody = {
      "params": {
        "productId": '1234',
        "shopName": 'name'
      },
      "job": "call-update-item"
    };
    var fakeResponse = {
      'message': 'success'
    };
    sWebHookStub.resolves(fakeBody)().then(function(result) {
      sqsQueueStub.resolves(fakeData)().then(function(result) {
        sqsSendMsgStub.resolves(fakeResponse)().then(function(result) {
          expect(shopifyWebhook).to.eventually.equal(result);
        });
      });
    });
    var shopifyWebhook = testedModule.manageShopifyWebhook(fakeDataObj);
  });
});
```
