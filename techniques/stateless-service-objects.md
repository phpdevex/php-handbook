---
description: Avoiding accidental global state
---

# Stateless service objects

Here's a pattern I've seen a few times, that I've found can cause some problems. Let's jump right in with an example.

Imagine we're calling an API to send a letter, through `api.postie.example.com`. This fictional service has an HTTP API, and we decide to use Symfony's API client to do it.

### First attempt at a service class

```php
use Symfony\Component\HttpClient\HttpClientInterface;

final class Postie
{
    private string $apiKey;
    private Customer $customer;

    public function __construct(
        private readonly HttpClientInterface $client
    ) {}
    
    public function setApiKey(string $key): void
    {
        $this->apiKey = $key;
    }
    
    public function setCustomer(Customer $customer): void
    {
        $this->customer = $customer;
    }
    
    public function sendLetter(Document $letter): void
    {
        $this->client->request(
            'POST',
            "https://api.mail.example.com/letters/{$this->customer->getPostieId()}",
            [
                'body' => $letter->toPdfString(), 
            ]           
        );
    }
}
```

We can then call it like this:

```php
final class PostieQueueWorker
{
    public function __construct(
        private readonly Postie $postie
    ) {}
    
    public function __invoke(PostieJob $job): void
    {
        $this->postie->setApiKey(getenv('POSTIE_API_KEY'));
        $this->postie->setCustomer($job->customer);
        $this->postie->sendLetter($job->letter);
    }
}
```

### A potential problem

I've left out some details, like how the worker and job run. I've also not gone into any detail about how we construct these objects. Often that's configured via a service container -- but that's the point, this code shouldn't really care about how that object is constructed.

Since we don't know how the object is constructed, we also don't know much about the lifecycle of the object. Do other services hold references to this object? Have they set a different customer or API key? If we forget to call `setCustomer`, will we accidentally sent a letter to the wrong customer, just because they were earlier in the queue?

### Stateless services to the rescue

Sure, you can avoid all those potential pitfalls by making sure we get our service container to make new objects each time, by never reusing a class, and by always setting all the properties correctly, but we all make mistakes.

By making our service stateless, we can avoid these problems and be more confident we won't introduce these sorts of bugs.

`apiKey` and `customer` are both stateful properties here, but we can handle them in different ways. The API key isn't going to change between sending letters, so we can move that to the constructor. The customer is different for every letter, but we can move that into `sendLetter`.

```php
use Symfony\Component\HttpClient\HttpClientInterface;

final class Postie
{
    public function __construct(
        private readonly HttpClientInterface $client,
        private readonly string $apiKey
    ) {}
        
    public function sendLetter(Customer $customer, Document $letter): void
    {
        $this->client->request(
            'POST',
            "https://api.mail.example.com/letters/{$customer->getPostieId()}",
            [
                'body' => $letter->toPdfString(), 
            ]           
        );
    }
}
```
