# Nodejs - Testing Promises With Sinon and Chai

To help us test our promises we will be using [sinon-as-promised](https://www.npmjs.com/package/sinon-as-promised) and 
[chai-as-promised](https://github.com/domenic/chai-as-promised).

Adding these to your test file is easy. Add the following code to the beginning of your test file. Mine is called `test.js` 
and lives in `test` directory.

    var chai = require('chai');
    var sinonChai = require("sinon-chai");
    var expect = chai.expect;
    var chaiAsPromised = require('chai-as-promised');
    chai.use(chaiAsPromised);
    require('sinon-as-promised');
    var sinon = require('sinon');
    chai.use(sinonChai);


Now image we have the following code, which is written to run on [aws lambda](http://docs.aws.amazon.com/lambda/latest/dg/welcome.html).
`Lambda` is great! It's worth learning and playing around with. It allows you to write serverless api's if you use it along with 
[aws api-gateway](http://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html).
`Lambda` requires we call a __callback__ at the end of everything having been run with either an error or and succes object. 
All `Lambda` functions receive three parameters __event__, __context__ and __callback__.

Consider the following which is part of project that was used to verify webhooks from `Shopify` and run `AWS SQS` methods.

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

We know that we need to check that __callback__ is called with the correct arguments and we know that __manageShopifyWebhook__ takes the
__event__ object and returns a promise.

