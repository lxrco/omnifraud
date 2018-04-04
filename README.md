# Omnifraud

**An easy to use, consistent fraud prevention processing library for PHP 7.1+**

Inspired by [Omnipay](https://github.com/thephpleague/omnipay), Omnifraud is an fraud prevention library for PHP. It aims at providing a clear and consistent API for interacting with different fraud prevention service.

## TL;DR

Here is a basic example

```php
<?php

/** @var \Omnifraud\Contracts\ServiceInterface $fraudService */
$fraudService = new Omnifraud\Example\ExampleService([
    'api_key' => 'XXX', // Service specific config
]);

// Build request, with data from the current sale
$request = new Omnifraud\Request\Request();
$request->getPurchase()->setId('1');
$request->getPurchase()->setTotal(25100);
$request->getPurchase()->setCurrencyCode('CAD');

$request->getAccount()->setEmail('jane@example.com');
//...

// Send the request to the service
$response = $fraudService->validateRequest($request);

// Does it need to be updated later?
if ($response->isPending()) {
    $this->queueFraudUpdate($response->getRequestUid());
    return;
}

if ($response->isGuaranteed()) {
    // This looks super safe, auto approve it maybe?
    //...
}

if ($response->getScore() < 10.0) {
    // This looks really suspicious, maybe auto refuse?
    //...
}

foreach ($response->getMessages() as $message) {
    if ($message->getType() !== \Omnifraud\Contracts\MessageInterface::TYPE_INFO) {
        $this->saveOrderMessage($message->getType(), $message->getMessage());
    }
}

```
Note: See [MakesTestRequest@makeTestRequest()](https://github.com/lxrco/omnifraud-common/blob/master/src/Testing/MakesTestRequests.php) for a full example of a request, each service might require different fields but they can all handle a full request.

## Installation

Usually all you need to do is installing the service you need, for example:
```bash
composer require omnifraud/signifyd
```

Each package already requires [`omnifraud/common`](https://github.com/lxrco/omnifraud-common),
so you don't need to require it.

You can also install ALL supported services:

```bash
composer require omnifraud/omnifraud
```


## Fraud services

All drivers must implement [ServiceInterface](https://github.com/lxrco/omnifraud-common/blob/master/src/Contracts/ServiceInterface.php).

The following services are officially supported right now:

Service | Composer Package | Maintainer
--- | --- | ---
[Kount](https://github.com/lxrco/omnifraud-kount) | omnifraud/kount | [LXRandCo](https://github.com/lxrco)
[Signifyd](https://github.com/lxrco/omnifraud-signifyd) | omnifraud/signifyd | [LXRandCo](https://github.com/lxrco)


## Service methods

* `trackingCode` - get the frontend code use to track users pre-purchase
* `validateRequest` - submit a fraud review request, typically after an order passed payment validation
* `updateRequest` - update a fraud review to get the updated information 
* `getRequestExternalLink` - get a link to view the fraud review in a browser
* `logRefusedRequest` - log a refused request with the fraud prevention service
* `cancelRequest` - cancel a previous fraud review request

## The fraud review Request

In order to communicate with services in a consistent way, all services accept a [Request](https://github.com/lxrco/omnifraud-common/blob/master/src/Request/Request.php) object for `validateRequest` and `updateRequest`.

The request is mostly a container for the following objects (with the exception of the request ID):
[Account](https://github.com/lxrco/omnifraud-common/blob/master/src/Request/Account.php),
[Address](https://github.com/lxrco/omnifraud-common/blob/master/src/Request/Address.php) *(used for both shipping and billing)*,
[Payment](https://github.com/lxrco/omnifraud-common/blob/master/src/Request/Payment.php),
[Product](https://github.com/lxrco/omnifraud-common/blob/master/src/Request/Product.php),
[Purchase](https://github.com/lxrco/omnifraud-common/blob/master/src/Request/Purchase.php),
[Session](https://github.com/lxrco/omnifraud-common/blob/master/src/Request/Session.php)

Here is a full commented example of building a request:
```php
<?php
use Omnifraud\Request\Data\Account;
use Omnifraud\Request\Data\Address;
use Omnifraud\Request\Data\Payment;
use Omnifraud\Request\Data\Product;
use Omnifraud\Request\Data\Purchase;
use Omnifraud\Request\Data\Session;
use Omnifraud\Request\Request;

$request = new Request();

$purchase = new Purchase();
$purchase->setId('1'); // Unique identifier for this sale
$purchase->setCreatedAt(new \DateTime('2017-09-02 12:12:12')); // Date the order was created at
$purchase->setCurrencyCode('CAD'); // ISO 4217 currency code
$purchase->setTotal(56025); // Total amount of the purchase, NO DECIMAL POINT.
$request->setPurchase($purchase);

$product1 = new Product();
$product1->setSku('SKU1'); // Product unique identifier
$product1->setName('Product number 1'); // Product name
$product1->setUrl('http://www.example.com/product-1'); // Product page
$product1->setImage('http://www.example.com/product-1/cover.jpg'); // Image of the product
$product1->setQuantity(1); // Quantity purchased
$product1->setPrice(6025); // Price of the product, NO DECIMAL POINT.
$product1->setWeight(100); // Weight in grams
$product1->setIsDigital(false); // Is this a digital product
$product1->setCategory('Category1'); // Category name
$product1->setSubCategory('Sub Category 1'); // Sub category name
$purchase->addProduct($product1);

$payment = new Payment();
$payment->setBin(457173); // First six numbers of the card
$payment->setLast4('9000'); // Last four numbers of the card
$payment->setExpiryMonth(9); // Expiration month 1-12
$payment->setExpiryYear(2020); // Expiration year
$payment->setAvs('Y'); // AVS response code, see http://www.emsecommerce.net/avs_cvv2_response_codes.htm
$payment->setCvv('M'); // CVV response code, see http://www.emsecommerce.net/avs_cvv2_response_codes.htm
$request->setPayment($payment);

$account = new Account(); // Customer account
$account->setId('ACCOUNT_ID'); // Account identifier
$account->setUsername('username'); // Username
$account->setEmail('test@example.com'); // Email address
$account->setPhone('1234567890'); // Phone number
$account->setCreatedAt(new \DateTime('2017-01-01 01:01:01')); // Account creation date
$account->setUpdatedAt(new \DateTime('2017-05-12 02:02:02')); // Account last edition date
$account->setLastOrderId('LAST_ORDER_ID'); // Previous sale identifier
$account->setTotalOrderCount(5); // Total number of orders made by this customer in the past
$account->setTotalOrderAmount(128700); // Total amount purchased by this customer, NO DECIMAL POINT.
$request->setAccount($account);

$session = new Session();
$session->setIp('1.2.3.4'); // Browser IP address
$session->setId('SESSION_ID'); // Session ID (same that was passed to the frontend code
$request->setSession($session);

$shippingAddress = new Address();
$shippingAddress->setFullName('John Shipping'); // Shipping name
$shippingAddress->setStreetAddress('1 shipping street');
$shippingAddress->setUnit('25');
$shippingAddress->setCity('Shipping Town');
$shippingAddress->setState('Shipping State');
$shippingAddress->setPostalCode('12345');
$shippingAddress->setCountryCode('US'); // ISO Alpha-2 country code
$shippingAddress->setPhone('1234567891'); // Use as main phone number
$request->setShippingAddress($shippingAddress);

$billingAddress = new Address();
$billingAddress->setFullName('John Billing'); // Name on the card
$billingAddress->setStreetAddress('1 billing street');
$billingAddress->setUnit('1A');
$billingAddress->setCity('Billing Town');
$billingAddress->setState('Billing State');
$billingAddress->setPostalCode('54321');
$billingAddress->setCountryCode('CA'); // ISO Alpha-2 country code
$billingAddress->setPhone('0987654321');
$request->setBillingAddress($billingAddress);
```

## Frontend code

Currently implemented services all require a session id to be passed to the frontend code,
this is done by calling the function defined by the service.

Note: The `trackingCode` needs to be called with a page constent,
either `ServiceInterface::PAGE_ALL` or `ServiceInterface::PAGE_CHECKOUT` as some services require
different code for the checkout page.

```javascript
// Example function to generate session IDs on the frontend and store it in a cookie,
// the ID could also come from the backend.
function getSessionId() {
    var sessionId = Cookies.get('sessionId');
    if(!sessionId) {
        sessionId = '';
        var possible = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
        
        for (var i = 0; i < 32; i++) {
            sessionId += possible.charAt(Math.floor(Math.random() * possible.length));
        }
        Cookies.set('sessionId', sessionId);
    }
    return sessionId;
}

// Must be defined before adding the trackingCodes
var trackingCodes = [];

// Inject one or more service's tracking code
<?= $fraudService->trackingCode(ServiceInterface::PAGE_ALL); ?>

// Call the functions with the code
for(var i in trackingCodes) {
    if(!trackingCodes.hasOwnProperty(i)) continue;
    trackingCodes[i](getSessionId());
}
```
