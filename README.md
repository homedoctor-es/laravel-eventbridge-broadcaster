# Laravel EventBridge Broadcaster

[![Latest Version on Packagist](https://img.shields.io/packagist/v/pod-point/laravel-aws-pubsub.svg?style=flat-square)](https://packagist.org/packages/pod-point/laravel-aws-pubsub)
![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/pod-point/laravel-aws-pubsub/run-tests.yml?branch=main&label=0.4.x)
[![Software License](https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat-square)](LICENSE.md)
[![Total Downloads](https://img.shields.io/packagist/dt/pod-point/laravel-aws-pubsub.svg?style=flat-square)](https://packagist.org/packages/pod-point/laravel-aws-pubsub)

**The Pub**

Similar to [Pusher](https://laravel.com/docs/broadcasting#pusher-channels), this package provides [Laravel Broadcasting](https://laravel.com/docs/broadcasting) driver for [AWS EventBridge](https://aws.amazon.com/eventbridge/) in order to publish server-side events.

We understand [Broadcasting](https://laravel.com/docs/broadcasting) is usually used to "broadcast" your server-side Laravel [Events](https://laravel.com/docs/events) over a WebSocket connection to your client-side JavaScript application. However, we believe this approach of leveraging broadcasting makes sense for a Pub/Sub architecture where an application would like to broadcast a server-side event to the outside world about something that just happened.

In this context, "channels" can be assimilated to "topics" when using the SNS driver and "event buses" when using the EventBridge driver.

**The Sub**

This part is not implemented here, but we use this package [Laravel EventBridge sqs consumer](https://github.com/homedoctor-es/laravel-eventbridge-sqs-consumer) to consume those events in other of your service.

For this purpose, we're using an architecture like this:
1. "n" publishers who publish events in EventBridge
2. Each event is publishes in one SNS topic.
3. Each one of these topics can be consumed by one or more SQS queues.
4. Each one of these queues correspond to one app.

## Installation

You can install the package on a Laravel 8+ application via composer:

```bash
composer require homedoctor-es/laravel-eventbridge-broadcaste
```

Then, add HomedoctorEs\EventBridgeBroadcaster\EventBridgeBroadcasterServiceProvider::class to load the driver in config/app.php file automatically.

## Publishing / Broadcasting

### Configuration

You will need to add the following connection and configure your AWS credentials in the `config/broadcasting.php` configuration file:

```php
'connections' => [

    'eventbridge' => [ //The connection name will be used as default eventbus
        'driver' => 'eventbridge',
        'region' => env('AWS_DEFAULT_REGION'),
        'key' => env('AWS_ACCESS_KEY_ID'),
        'endpoint' => env('AWS_URL'),
        'secret' => env('AWS_SECRET_ACCESS_KEY'),
        'source' => env('AWS_EVENTBRIDGE_SOURCE'),
    ],
    // ...
],
```

Make sure to define your [environment variables](https://laravel.com/docs/configuration#environment-configuration) accordingly:

```dotenv
# both drivers require:
AWS_DEFAULT_REGION=you-region
AWS_ACCESS_KEY_ID=your-aws-key
AWS_SECRET_ACCESS_KEY=your-aws-secret

# EventBridge driver only:
AWS_EVENTBRIDGE_SOURCE=com.your-app-name
```



Next, you will need to make sure you're using the `sns` broadcast driver as your default driver when broadcasting in your `.env` file:

```php
BROADCAST_DRIVER=eventbridge
```

**Remember** that you can define the connection at the Event level if you ever need to be able to use [two drivers concurrently](https://github.com/laravel/framework/pull/38086).

### Usage

Simply follow the default way of broadcasting Laravel events, explained in the [official documentation](https://laravel.com/docs/broadcasting#defining-broadcast-events).

In a similar way, you will have to make sure you're implementing the `Illuminate\Contracts\Broadcasting\ShouldBroadcast` interface and define which channel you'd like to broadcast on.

```php
use App\Models\Order;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Broadcasting\InteractsWithBroadcasting;
use Illuminate\Queue\SerializesModels;

class OrderShipped implements ShouldBroadcast
{
    use SerializesModels;

    /**
     * The order that was shipped.
     *
     * @var \App\Models\Order
     */
    public $order;

    /**
     * Create a new event instance.
     *
     * @param  \App\Models\Order  $order
     * @return void
     */
    public function __construct(Order $order)
    {
        $this->order = $order;
        $this->broadcastVia('eventbridge');
    }

    /**
     * Get the topics that model events should broadcast on.
     *
     * @return array
     */
    public function broadcastOn()
    {
        return ['orders']; // This is the Event bus name
    }
}
```

#### Broadcast Data

By default, the package will publish the default Laravel payload which is already used when broadcasting an Event. Once published, its JSON representation could look like this:

```json
{
    "order": {
        "id": 1,
        "name": "Some Goods",
        "total": 123456,
        "created_at": "2021-06-29T13:21:36.000000Z",
        "updated_at": "2021-06-29T13:21:36.000000Z"
    },
    "connection": null,
    "queue": null
}

```

Using the `broadcastWith` method, you will be able to define exactly what kind of payload gets published.

```php
/**
 * Get and format the data to broadcast.
 *
 * @return array
 */
public function broadcastWith()
{
    return [
        'action' => 'parcel_handled',
        'data' => [
            'order-id' => $this->order->id,
            'order-total' => $this->order->total,
        ],
    ];
}
```

Now, when the event is being triggered, it will behave like a standard Laravel event, which means other listeners can listen to it, as usual, but it will also broadcast to the Topic defined by the `broadcastOn` method using the payload defined by the `broadcastWith` method.

#### Broadcast Name / Subject

In a Pub/Sub context, it can be handy to specify a `Subject` on each notification which broadcast to SNS. This can be an easy way to configure a Listeners for each specific kind of subject you can receive and process later on within queues.

By default, the package will use the standard [Laravel broadcast name](https://laravel.com/docs/broadcasting#broadcast-name) in order to define the `Subject` of the notification sent. Feel free to customize it as you wish.

```php
/**
 * The event's broadcast name/subject.
 *
 * @return string
 */
public function broadcastAs()
{
    return "orders.{$this->action}";
}
```

#### Model Broadcasting

If you're familiar with [Model Broadcasting](https://laravel.com/docs/broadcasting#model-broadcasting), you already know that Eloquent models dispatch several events during their lifecycle and broadcast them accordingly.

In the context of model broadcasting, only the following model events can be broadcasted:

- `created`
- `updated`
- `deleted`
- `trashed` _if soft delete is enabled_
- `restored` _if soft delete is enabled_

In order to broadcast the model events, you need to use the `Illuminate\Database\Eloquent\BroadcastsEvents` trait on your Model and follow the official [documentation]((https://laravel.com/docs/broadcasting#model-broadcasting)).

You can use `broadcastOn()`, `broadcastWith()` and `broadcastAs()` methods on your model in order to customize the Topic names, the payload and the Subject respectively.

> **Note:** Model Broadcasting is **only available from Laravel 8.x**.

## Credits

- [laravel-aws-pubsub](https://github.com/Pod-Point/laravel-aws-pubsub) for some inspiration

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.