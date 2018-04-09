<img src="icon.png" align="right">

# Omnifraud · [![Travis Build Status](https://img.shields.io/travis/lxrco/omnifraud-common.svg)](https://travis-ci.org/lxrco/omnifraud-common) [![Packagist Version](https://img.shields.io/packagist/v/omnifraud/common.svg)](https://packagist.org/packages/omnifraud/common) [![Packagist Downloads](https://img.shields.io/packagist/dt/omnifraud/common.svg)](https://packagist.org/packages/omnifraud/common) [![License Version](https://img.shields.io/packagist/l/omnifraud/common.svg)](https://github.com/lxrco/omnifraud-common/blob/master/LICENSE)

> An easy to use, consistent fraud prevention library for **PHP 7.1+**. Inspired by [Omnipay](https://github.com/thephpleague/omnipay).

Omnifraud is an ecommerce fraud prevention library for PHP. The project aims to provide a clear and consistent API for interacting with different fraud prevention, risk assessment, and liability shifting services.

## Table of Contents

- [Motivation](#motivation)
- [Installation](#installation)
- [Basic Example](#basic-example)
- [Fraud Services / Drivers](#fraud-services--drivers)
    - [The `Null` Driver](#the-null-driver)
- [Usage](#usage)
    - [The Session ID](#the-session-id)
    - [Instantiation](#instantiation)
    - [Service Methods](#service-methods)
    - [Front-end Implementation](#front-end-implementation)
    - [Creating Requests](#creating-requests)
        - [Account](#account)
        - [Address](#address)
        - [Payment](#payment)
        - [Product](#product)
        - [Purchase](#purchase)
        - [Session](#session)
        - [Static Creation](#static-creation)
    - [Responses](#responses)
        - [Asynchronous Responses](#asynchronous-responses)
- [Contributing](#contributing)

## Motivation

There are a lot of risk assessment services out there and, although some details differ, the flow is almost always the same.

* Generate a random session ID
* Insert JavaScript tracking code into front end with aforementioned session ID
* On checkout, send a request to risk assessment service with session ID and order information

We created Omnifraud to satiate our own needs. The benefits of using the Omnifraud library are:

* Learn one interface, use throughout projects with different providers.
* Clean separation. Easily swap providers without touching a single line of checkout code.
* Documentation in popular risk assessment services isn't always clear. Put in your API key and go.

## Installation

Usually all you'll need to do is install the service you need. For example:

```bash
composer require omnifraud/signifyd
```

Each package already requires [`omnifraud/common`](https://github.com/lxrco/omnifraud-common),
so you don't need to require it.

To install **ALL** supported services:

```bash
composer require omnifraud/omnifraud
```

## Basic Example

```php
<?php

use Omnifraud\Omnifraud;

/** @var \Omnifraud\Contracts\ServiceInterface $fraudService */
$fraudService = Omnifraud::create('Signifyd', [
    'api_key' => 'XXX',
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
    // Queue for later update
    $this->queueFraudUpdate($response->getRequestUid());
}

if ($response->isGuaranteed()) {
    // The order is guaranteed by our fraud service
    // ...
}

if ($response->getScore() < 10.0) {
    // That's a pretty bad score. Let's bail!
    // ...
}
```
> *Note: See [MakesTestRequest@makeTestRequest()](https://github.com/lxrco/omnifraud-common/blob/master/src/Testing/MakesTestRequests.php) for a full example of a request. Services may differ in which fields are optional and which are required, but every service can handle a completely full request.*

## Fraud Services / Drivers

All drivers implement [ServiceInterface](https://github.com/lxrco/omnifraud-common/blob/master/src/Contracts/ServiceInterface.php).

The following services are officially supported right now:

Service | Composer Package | Alias | Maintainer
--- | --- | --- | ---
[Kount](https://github.com/lxrco/omnifraud-kount) | omnifraud/kount | Kount | [LXRandCo](https://github.com/lxrco)
[Signifyd](https://github.com/lxrco/omnifraud-signifyd) | omnifraud/signifyd | Signifyd | [LXRandCo](https://github.com/lxrco)
[Null](https://github.com/lxrco/omnifraud-common) | omnifraud/common | Null | [LXRandCo](https://github.com/lxrco)

> *Note: Interested in contributing your own implementation? We'd love to include it! Scroll down to [Contributing](#contributing) for information.*

### The `Null` Driver

The `Null` driver does nothing. What else were you expecting?

## Usage

The lifecycle of the fraud request looks like this:

1. User visits www.example.com
2. User is assigned a random session ID or reuses existing session ID (via local storage, cookies, etc.)
3. User attempts checkout with payment
4. Both failed and successful payments are reported to fraud service
5. Once checkout succeeds, a score is given to the fraud request (either synchronously or asynchronously) and is saved alongside the order

In order of implementation, this translates into:

1. Instantiation of a driver
2. Front-end JavaScript implementation
3. Creation of a fraud "request"
4. On checkout, recording *order*, *session*, *account*, and *payment* info in the request
5. Storing the response and fraud score in the database, or queuing for later retrieval if asynchronous

### The Session ID

It's entirely up to you to generate a session ID and re-use the same one throughout the aforementioned fraud request lifecycle. Most services have no opinion on what it looks like, but to be safe accross most vendors you should ensure it:

* is alphanumeric
* is of reasonable length (more than 255 might be overkill)
* has some uniqueness properties (tip: don't do things like `crc32(time())`)

Once you have your random session ID, toss it into a cookie (or custom header, however you want to communicate it) and use it for both the front-end tracker and the back-end fraud requests.

### Instantiation

Instantiation of a driver is incredibly straightforward.

You can create them yourself, passing all configs necessary as the first and only constructor argument.

```php
<?php

use Omnifraud\Kount\KountService;

/** @var \Omnifraud\Contracts\ServiceInterface $fraudService */
$fraudService = new KountService([
    'apiKey' => 'XXX',
    'merchantId' => '123456',
]);
```

Or, use the static `create()` method offered by `Omnifraud\Omnifraud`. The first parameter is the **alias** of the driver (detailed in the [table above](#fraud-services--drivers)), and the second is the same configuration array you would normally supply.

```php
<?php

use Omnifraud\Omnifraud;

/** @var \Omnifraud\Contracts\ServiceInterface $fraudService */
$fraudService = Omnifraud::create('Kount', [
    'apiKey' => 'XXX',
    'merchantId' => '123456',
]);
```

As every driver is an implementation of `Omnifraud\Contracts\ServiceInterface`, you should be careful to ensure that you typehint this contract instead of any concrete implementations. Doing anything else would defeat the purpose of this library.

### Front-end Implementation

All services expose a `trackingCode(string $pageType, string $sessionId)` method which returns a stringified snippet of JavaScript. You can call this method to insert the code necessary to instrument the front-end tracking of your fraud service. The two required parameters are a constant to specify the type of page we're inserting the snippet into, and the clients [session ID](#the-session-id).

Be sure to pass the appropriate constant as some services will differentiate between the two types of pages. It can be one of these two values:

* `ServiceInterface::PAGE_ALL`
* `ServiceInterface::PAGE_CHECKOUT`

Quite simply, the return string is an IIFE including the session ID. To help clarify how this works let's take a peak at the `SignifydService`. Its `trackingCode()` method will return the following JavaScript snippet:

```javascript
(function() {
    var script = document.createElement('script');
    script.setAttribute('src', 'https://cdn-scripts.signifyd.com/api/script-tag.js');
    script.setAttribute('data-order-session-id', {{ $sessionId }});
    script.setAttribute('id', 'sig-api');

    document.body.appendChild(script);
})();
```

By default, anything you pass as `$sessionId` will be quoted and escaped (via `json_encode`). If you wish to pass raw JS (i.e. a variable name, global function, etc.), take advantage of the third, optional `bool $quote = true` parameter of `trackingCode`.

### Services Methods

The interface is quite self-explanatory and you are encouraged to familiarize yourself with it by looking at [ServiceInterface.php](https://github.com/lxrco/omnifraud-common/blob/master/src/Contracts/ServiceInterface.php). All methods are typehinted in both their arguments and returns.

In order to communicate with all services in a consistent way, all implementations accept a [Request Object](https://github.com/lxrco/omnifraud-common/blob/master/src/Request/Request.php) throughout. The methods exposed by all services are:

* `public function validateRequest(Request $request): ResponseInterface;`
* `public function updateRequest(Request $request): ResponseInterface;`
* `public function cancelRequest(Request $request): void;`
* `public function logRefusedPayment(Request $request): void;`
* `public function getRequestExternalLink(string $requestUid): ?string;`
* `public function trackingCode(string $pageType, string $sessionId, bool $quote = true): string;`

### Creating Requests

#### Account

#### Address

#### Payment

#### Product

#### Purchase

#### Session

#### Static Creation

**TODO**: Request::create([]) or new Request([]);

### Responses

#### Asynchronous Responses

## Contributing

---

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
