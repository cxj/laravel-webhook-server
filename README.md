# Send webhooks from Laravel apps

[![Latest Version on Packagist](https://img.shields.io/packagist/v/spatie/laravel-webhook-server.svg?style=flat-square)](https://packagist.org/packages/spatie/laravel-webhook-server)
[![Build Status](https://img.shields.io/travis/spatie/laravel-webhook-server/master.svg?style=flat-square)](https://travis-ci.org/spatie/laravel-webhook-server)
[![Quality Score](https://img.shields.io/scrutinizer/g/spatie/laravel-webhook-server.svg?style=flat-square)](https://scrutinizer-ci.com/g/spatie/laravel-webhook-server)
[![Total Downloads](https://img.shields.io/packagist/dt/spatie/laravel-webhook-server.svg?style=flat-square)](https://packagist.org/packages/spatie/laravel-webhook-server)

A webhook is a way for an app to provide information to another app about a certain event. The way the two apps communicate is with a simple HTTP request. 

This package allows you to easily configure and send webhooks in a Laravel app. It has support for [signing calls](https://github.com/spatie/laravel-webhook-server#how-signing-requests-works), [retrying calls and backoff strategies](https://github.com/spatie/laravel-webhook-server#retrying-failed-webhooks).

If you need to receive and process webhooks take a look at our [laravel-webhook-client](https://github.com/spatie/laravel-webhook-client) package.

## Installation

You can install the package via composer:

```bash
composer require spatie/laravel-webhook-server
```

You can publish the config file with:
```bash
php artisan vendor:publish --provider="Spatie\WebhookServer\WebhookServerServiceProvider"
```

This is the contents of the file that will be published at `config/webhook-server.php`:

```php
return [

    /*
     *  The default queue that should be used to send webhook requests.
     */
    'queue' => 'default',

    /*
     * The default http verb to use.
     */
    'http_verb' => 'post',

    /*
     * This class is responsible for calculating the signature that will be added to
     * the headers of the webhook request. A webhook client can use the signature
     * to verify the request hasn't been tampered with.
     */
    'signer' => \Spatie\WebhookServer\Signer\DefaultSigner::class,

    /*
     * This is the name of the header where the signature will be added.
     */
    'signature_header_name' => 'Signature',

    /*
     * These are the headers that will be added to all webhook requests.
     */
    'headers' => [],

    /*
     * If a call to a webhook takes longer that this amount of seconds
     * the attempt will be considered failed.
     */
    'timeout_in_seconds' => 3,

    /*
     * The amount of times the webhook should be called before we give up.
     */
    'tries' => 3,

    /*
     * This class determines how many seconds there should be between attempts.
     */
    'backoff_strategy' => \Spatie\WebhookServer\BackoffStrategy\ExponentialBackoffStrategy::class,

    /*
     * By default we will verify that the ssl certificate of the destination
     * of the webhook is valid.
     */
    'verify_ssl' => true,
];
```

By default the package uses queues to retry failed webhook requests. Be sure to setup a real queue other than `sync` in non-local environments.

## Usage

This is the simplest way to call a webhook:

```php
Webhook::create()
   ->url('https://other-app.com/webhooks')
   ->payload(['key' => 'value'])
   ->useSecret('sign-using-this-secret')
   ->call();
```

This will send a post request to `https://other-app.com/webhooks`. The body of the request will be json encoded version of the array passed to `payload`. The request will have a header called `Signature` that will contain a signature the receiving app can use [to verify](https://github.com/spatie/laravel-webhook-server#how-signing-requests-works) the payload hasn't been tampered with. 

If the receiving app doesn't respond with a response code starting with `2`, the package will retry calling the webhook after 10 seconds. If that second attempt fails, the package will attempt to call the webhook a final time after 100 seconds. Should that attempt fail the `FinalWebhookCallFailedEvent` will be raised.

### How signing requests works

When setting up it's common to generate, store and share a secret between your app and the app that wants to receive webhooks. Generating the secret could be done with `Illuminate\Support\Str::random()`, but it's really entirely up to you. The package will use the secret to sign a webhook call.

By default the package will add a header called `Signature` that will contain a signature the receiving app can use the payload hasn't been tampered with. This is how that signature is calculated.

```php
// payload is the array passed to the `payload` method of the webhook
// secret is the string given to the `signUsingSecret` method on the webhook.

$payloadJson = json_encode($payload); 

$signature = hash_hmac('sha256', $payloadJson, $secret);
```

### Customizing signing requests

If you want to customize the signing process you can create your own custom signer. A signer is any class that implements `Spatie\WebhookServer\Signer`.

This is what that interface looks like.

```php
namespace Spatie\WebhookServer\Signer;

interface Signer
{
    public function signatureHeaderName(): string;

    public function calculateSignature(array $payload, string $secret): string;
}
```

After creating your signer you can specify it's class name in the `signer` key of the `webhook-server` config file. Your signer will than be used by default in all webhook calls.

You can also specify a signer for a specific webhook call

```php
Webhook::create()
    ->signUsing(YourCustomSigner::class)
    ...
    ->call();
```

If you just want to customize the name of the header you don't need to use a custom signer, but you can just change the value in the `signature_header_name` in the `webhook-server` config file.

### Retrying failed webhooks

When the app to which we're sending the webhook fails to send a response with a `2xx` status code the package will consider the call as failed. The call will also be considered failed if the remote app doesn't respond within 3 seconds.

You can configure that default timeout in the `timeout_in_seconds` key of the `webhook-server` config file. Alternatively you can override the timeout for a specific webhook like this:

```php
Webhook::create()
    ->timeoutInSeconds(5)
    ...
    ->call();
```

When a webhook call fails we'll retry the call two more times. You can set the default amount of times we just retry the webhook call in the `tries` key of the config file. Alternatively, you can specify the amount of tries for a specific webhook like this:

```php
Webhook::create()
    ->maximumTries(5)
    ...
    ->call();
```

To not hammer the remote app we'll wait some time between each attempts. By default we wait 10 seconds between the first and second attempt, 100 seconds between the third and the fourth, 1000 between the fourth and the fifth and so on. The maximum amount of seconds that we'll wait is 100 000, which is about 27 hours. This behaviour is implemented in the default `ExponentialBackoffStrategy`.

You can define your own backoff strategy by creating a class that implements `Spatie\WebhookServer\BackoffStrategy\BackoffStrategy`. This is what that interface looks like:

```php
namespace Spatie\WebhookServer\BackoffStrategy;

interface BackoffStrategy
{
    public function waitInSecondsAfterAttempt(int $attempt): int;
}
```

You can make your custom strategy the default strategy by specifying it's fully qualified classname in the `backoff_strategy` of the `webhook-server` config file. Alternatively you can specify a strategy for a specific webhook like this.

```php
Webhook::create()
    ->useBackoffStrategy(YourBackoffStrategy::class)
    ...
    ->call();
```

Under the hood the retrying of the webhook calls is implemented using [delayed dispatching](https://laravel.com/docs/master/queues#delayed-dispatching). Amazon SQS only has support for a small maximum delay. If you're using Amazon SQS for your queues, make sure you do not configure the package in away so there is more than 15 minutes between each attempt.


### Customizing the http verb

By default all webhooks will use the `post` method. You can customize that by specify the http verb you want in the `http_verb` key of the `webhook-server` config file.

You can also override the default for a specific call by use the `useHttpVerb` method.

```php
Webhook::create()
    ->useHttpVerb('get')
    ...
    ->call();
```

### Adding extra headers

You can extra headers by adding the to the `headers` key in the `webhook-server` config file. If you want to add extra headers for a specific webhook you can use the `withHeaders` call.

```php
Webhook::create()
    ->withHeaders([
        'Another Header' => 'Value of Another Header'
    ])
    ...
    ->call();
```

### Verifying the SSL certificate of the receiving app

When using an url that starts with `https://` the package will verify if the SSL certificate of the receiving party is valid. If it is not, we will consider the webhook call failed. We don't recommend this but you can turn off this verification by setting the `verify_ssl` key in the `webhook-server` config file to `false`.

You can also disable the verification per webhook call with the `doNotVerifySsl` method.

```php
Webhook::create()
    ->doNotVerifySsl()
    ...
    ->call();
```

### Adding meta information

You can add extra meta information to the webhook. This meta information will not be transmitted, it will only be used to pass to [the events this package fires](#events).

This is how you can add meta information:

```php
Webhook::create()
    ->meta($arrayWithMetaInformation)
    ...
    ->call();
```

### Adding tags

If you're using [Laravel Horizon](https://laravel.com/docs/5.8/horizon) for your queues you'll be happy to know that we support [tags](https://laravel.com/docs/5.8/horizon#tags). 

To add tags to the underlying job that'll perform the webhook call, simply specify them in the `tags` key of the `webhook-server` config file or use the `withTags` method:

```php
Webhook::create()
    ->withTags($tags)
    ...
    ->call();
```

### Events

The package fires these events:
- `WebhookCallSucceededEvent`: the remote app responded with a `2xx` response code.
- `WebhookCallFailedEvent`: the remote app responded with a non `2xx` response code or it did not respond at all
- `FinalWebhookCalledFailedEvent`: the final attempt to call the webhook failed.

All these events have these properties:

- `httpVerb`: the verb used to perform the request
- `webhookUrl`: the url to where the request was sent
- `payload`: the used payload
- `headers`: the headers that were sent. This array includes the signature header
- `meta`: the array of values passed to the webhook with [the `meta` call](#adding-meta-information)
- `tags`: the array of [tags](#adding-tags) used
- `attempt`: the attempt number
- `response`: the response returned by the remote app. Can be an instance of `\GuzzleHttp\Psr7\Response` or `null`.

## Testing

``` bash
composer test
```

## Changelog

Please see [CHANGELOG](CHANGELOG.md) for more information on what has changed recently.

## Contributing

Please see [CONTRIBUTING](CONTRIBUTING.md) for details.

### Security

If you discover any security related issues, please email freek@spatie.be instead of using the issue tracker.

## Postcardware

You're free to use this package, but if it makes it to your production environment we highly appreciate you sending us a postcard from your hometown, mentioning which of our package(s) you are using.

Our address is: Spatie, Samberstraat 69D, 2060 Antwerp, Belgium.

We publish all received postcards [on our company website](https://spatie.be/en/opensource/postcards).

## Credits

- [Freek Van der Herten](https://github.com/freekmurze)
- [All Contributors](../../contributors)

## Support us

Spatie is a webdesign agency based in Antwerp, Belgium. You'll find an overview of all our open source projects [on our website](https://spatie.be/opensource).

Does your business depend on our contributions? Reach out and support us on [Patreon](https://www.patreon.com/spatie). 
All pledges will be dedicated to allocating workforce on maintenance and new awesome stuff.

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.
