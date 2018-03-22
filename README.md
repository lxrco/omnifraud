# Omnifraud

**And easy to use, constistent fraud prevention processing library for PHP 7.1+**

Inpired by [Omnipay](https://github.com/thephpleague/omnipay), Omnifraud is an fraud prevention livrary for PHP. It aims at providing a clear and consisten API for interacting with different fraud prevention service.


## TL;DR

Here is a basic example

```php
<?php

/** @var \Omnifraud\Contracts\ServiceInterface $fraudService */
$fraudService = new Omnifraud\Example\ExampleService([
    'api_key' => 'XXX', // Service specific config
]);

$request = new Omnifraud\Request\Request();
$request->getPurchase()->setId('1');
$request->getPurchase()->setTotal(25100);
$request->getPurchase()->setCurrencyCode('CAD');

$request->getAccount()->setEmail('jane@example.com');
//...

$response = $fraudService->validateRequest($request);

// Does it need to be updated later?
if ($response->isAsync()) {
    $this->queueFraudUpdate($response->getRequestUid());
    return;
}

if ($response->isGuaranteed()) {
    // This looks super safe, auto approve it maybe?
    //...
}

if ($response->getPercentScore() < 10.0) {
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


## Fraud services

All drivers must implement [ServiceInterface](https://github.com/lxrco/omnifraud-common/blob/master/src/Contracts/ServiceInterface.php).

The following services are supported:

Service | Composer Package | Maintainer
--- | --- | ---
[Kount](https://github.com/lxrco/omnifraud-kount) | omnifraud/kount | [LXRandCo](https://github.com/lxrco)
[Signifyd](https://github.com/lxrco/omnifraud-kount) | omnifraud/signifyd | [LXRandCo](https://github.com/lxrco)
