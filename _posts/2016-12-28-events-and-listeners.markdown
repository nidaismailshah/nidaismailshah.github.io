---
layout: post
title:  "Symfony/Drupal: Event Subscribers and Event Listeners."
date:   2016-12-28 10:00:00
categories: drupal
tag: 1
---
Following up on my talk at the <a href="https://events.drupal.org/dublin2016/">DrupalCon Dublin</a> about “<a href="https://events.drupal.org/dublin2016/sessions/state-hooking-drupal">The state of hooking into Drupal</a>", I was presenting on <a href="http://www.slideshare.net/NidaIsmailShah/hooks-and-events-in-drupal-8">Hooks and Events in Drupal 8</a> to my colleagues when someone asked a question about the concept of event subscribers and event listeners. I didn’t have an answer then because I believed both meant the same thing, which they do (somewhat) , however here is an attempt to answer the question in a more elaborate way.


We know that Drupal 8 uses Symfony’s Event Dispatcher component and we also know that to be able to hook into Drupal whenever an event is triggered we have to subscribe to the event. This post is about how we can react to the occurrence of an event  i.e subscribe to an event or listen to it with Event Subscribers and Event Listeners. I'll try to cover both Symfony and Drupal aspects of it.

<blockquote>The <a href="http://symfony.com/doc/current/components/event_dispatcher.html"><strong>EventDispatcher component</strong></a> provides tools that allow your application components to communicate with each other by dispatching events and listening to them.</blockquote>

The basic idea of having events, dispatchers, subscribers and listeners is to make our applications extensible and to be able to run code or perform actions at particular stages through the application life cycle. The stages are pre decided by triggering or dispatching events. The listeners and subscribers can then register themselves to be notified by the dispatcher when the event actually occurs so they can run the code they want or perform actions they want. Read more about events <a href="http://symfony.com/doc/current/components/event_dispatcher.html"><strong>here</strong></a>.

During the life cycle of an application lots of event notifications are triggered. We can listen to these notifications and respond to them by executing any piece of code. One can listen to an event either by creating an event subscriber or by creating an event listener. Event listeners and event subscribers serve the same purpose, just in a slightly different way.

<h2 id="event_listener">Event Listeners.</h2>

An event listener is a class, defined as a service, that has methods that get registered to be executed when certain events occur. All of this happens at the registration time. What method would be executed at what event is set in the service definition. The listener class doesn't know what events it’s listening to. This information is always in the service definition. Each method takes in as an argument the event object that it wants to listen to.

{% highlight ruby %}
# src/AppBundle/EventListener/ExceptionListener.php
namespace AppBundle\EventListener;

use Symfony\Component\HttpKernel\Event\GetResponseForExceptionEvent;

class ExceptionListener
{
    public function onKernelException(GetResponseForExceptionEvent $event)
    {
      # do something.
    }
}

{% endhighlight %}

Each event receives a slightly different type of ```$event``` object. For the ```kernel.exception``` event, it is ```GetResponseForExceptionEvent```. To see what type of object each event listener receives check the documentation about the specific event you're listening to.

Now that the listener class is created, you just need to register it as a service and notify Symfony that it is a <em>listener</em> on the ```kernel.exception``` event by using a special “tag”:
{% highlight ruby %}
{ name: kernel.event_listener, event: kernel.exception }
{% endhighlight %}


{% highlight ruby %}
# app/config/services.yml
services:
    app.exception_listener:
        class: AppBundle\EventListener\ExceptionListener
        tags:
            - { name: kernel.event_listener, event: kernel.exception }
{% endhighlight %}

There is an optional tag attribute called ```method``` which defines which method to execute when the event is triggered. By default the name of the method is <em>on + "camel-cased event name"</em>. If the event is ```kernel.exception``` the method executed by default is ```onKernelException()```.

The other optional tag attribute is called ```priority```, which defaults to 0 and it controls the order in which listeners are executed (the highest the priority, the earlier a listener is executed). This is useful when you need to guarantee that one listener is executed before another. The priorities of the internal Symfony listeners usually range from -255 to 255 but your own listeners can use any positive or negative integer.


{% highlight ruby %}
# app/config/services.yml
services:
    app.exception_listener:
        class: AppBundle\EventListener\ExceptionListener
        tags:
            - { name: kernel.event_listener, event: kernel.exception, method: alterException, priority: 100 }
{% endhighlight %}

An event listener doesn’t necessarily have to be a class on its own. A php callable (even a <a href="http://php.net/manual/en/functions.anonymous.php">php closure</a>) can also be added as a listener to an event.

So another way of registering event listeners is via the ```addListener()``` method. We can add listeners to a dispatcher for any event by calling the ```addListener()``` method and passing in the ```event_name``` of the event we want to listen to as the first argument and the callable, that would get called when the actual event occurs, as the second argument. We can also provide the priority of the execution of our callable as the optional third argument.


{% highlight ruby %}
  $dispatcher = \Drupal::service('event_dispatcher');

  $dispatcher->addListener("event.name", "callable");
  # or
  # $dispatcher->addListener("event.name", "callable", priority);
{% endhighlight %}

For the object orientation part of it, we can also provide the second argument as an array with its first element being the instance of the object our callable belongs to and second element being the name of the method (our callable).


{% highlight ruby %}
  $dispatcher = \Drupal::service('event_dispatcher');

  # Provided OurListenerClass is defined and has implemented the "method".
  $dispatcher->addListener("event.name", array(new OurListenerClass(), "method"));
{% endhighlight %}

<em>Note that Drupal as of Drupal 8.2 doesn’t support the service definition way of adding event listeners for better DX (developer experience) reasons so as to prevent the confusions of having multiple ways of doing the same thing.</em> As of Drupal 8.2, it promotes extending the code and using the event dispatcher component via event subscribers only.

This means that if you were to define a class with a method ```onKernelRequest()``` and define it as a service under a tag ```{ name = kernel.event_listener, event: kernel.request }``` to listen to the ```kernel.request``` event, <strong>it won’t work</strong>. This is because drupal doesn’t support tagging services as event listeners for reasons already mentioned. Drupal only checks for the services tagged with the ```event_subscriber``` tag in the class <a href="http://cgit.drupalcode.org/drupal/tree/core/lib/Drupal/Core/DependencyInjection/Compiler/RegisterEventSubscribersPass.php">Drupal\Core\DependencyInjection\Compiler\RegisterEventSubscribersPass.</a>  See also <a href="https://api.drupal.org/api/drupal/core!core.api.php/group/service_tag/8.2.x">Service tags</a> in Drupal.

However we can still add an event listener to listen to any event by calling the addListener() method on the dispatcher object.


{% highlight ruby %}
  $dispatcher = \Drupal::service('event_dispatcher');

  # Provided OurListenerClass is defined and has implemented the onKernelRequest method.
  $dispatcher->addListener("kernel.request", array(new OurListenerClass, "onKernelRequest"));
{% endhighlight %}

<h2>Event Subscribers</h2>

Event subscriber is a class, defined as a service, that listens to one or more events by defining one or more methods. One method can listen to one or more events and one event can be listened to by one or more methods. All of this is determined at the runtime. What methods listen to what events is known to the event subscriber class unlike event listeners. Event subscribers do this by implementing the ```getSubscribedEvents()``` method which returns an array of all the events that the subscriber would listen to and also the method that would be executed when any of those events occurs.

In a given subscriber, different methods can listen to the same event. The order in which methods are executed is defined by the priority parameter of each method (the higher the priority the earlier the method is called). To learn more about event subscribers, read about <a href="http://symfony.com/doc/current/components/event_dispatcher.html"><strong>The EventDispatcher component</strong></a>.

To create a subscribe we need to create a class, defined as a service, implementing ```Symfony\Component\EventDispatcher\EventSubscriberInterface``` interface and thus also implementing the getSubscribedEvents() method from the EventSubscriberInterface to return an array of all the events we want to listen to with their corresponding callbacks. the callbacks also belong to the same class.

Listing down the things we need to do to create an event subscriber:
<ul>
 <li>Create a subscriber class implementing <em>EventSubscriberInterface</em>.</li>
 <li>Implement <em>getSubscribedEvents()</em> method.</li>
 <li>Write callables.</li>
 <li>Define the subscriber class as a service and tag it with <em>event_subscriber</em>.</li>
 </ul>
Talking about Drupal 8 there are multiple examples one can cite where event subscribers have been used. Following is an example from Drupal Core from <a href="http://cgit.drupalcode.org/drupal/tree/core/lib/Drupal/Core/Config/ConfigFactory.php">Drupal\Core\Config\ConfigFactory</a>

{% highlight ruby %}
namespace Drupal\Core\Config;
 ...

class ConfigFactory implements ConfigFactoryInterface, EventSubscriberInterface {
  ...

  /**
   * Updates stale static cache entries when configuration is saved.
   *
   * @param ConfigCrudEvent $event
   *   The configuration event.
   */
  public function onConfigSave(ConfigCrudEvent $event) {
    # Ensure that the static cache contains up to date configuration objects by
    # replacing the data on any entries for the configuration object apart
    # from the one that references the actual config object being saved.
    $saved_config = $event->getConfig();
    foreach ($this->getConfigCacheKeys($saved_config->getName()) as $cache_key) {
      $cached_config = $this->cache[$cache_key];
      if ($cached_config !== $saved_config) {
        # We can not just update the data since other things about the object
        # might have changed. For example, whether or not it is new.
        $this->cache[$cache_key]->initWithData($saved_config->getRawData());
      }
    }
  }

  /**
   * Removes stale static cache entries when configuration is deleted.
   *
   * @param \Drupal\Core\Config\ConfigCrudEvent $event
   *   The configuration event.
   */
  public function onConfigDelete(ConfigCrudEvent $event) {
    # Ensure that the static cache does not contain deleted configuration.
    foreach ($this->getConfigCacheKeys($event->getConfig()->getName()) as $cache_key) {
      unset($this->cache[$cache_key]);
    }
  }

  /**
   * {@inheritdoc}
   */
  static function getSubscribedEvents() {
    $events[ConfigEvents::SAVE][] = array('onConfigSave', 255);
    $events[ConfigEvents::DELETE][] = array('onConfigDelete', 255);
    return $events;
  }

  ...
}
{% endhighlight %}

Now that the subscriber class is created, you just need to register it as a service and notify Symfony/Drupal that it is an “event subscriber"  by using a special “tag”: { name: event_subscriber }


{% highlight ruby %}
# core.services.yml
config.factory:
    class: Drupal\Core\Config\ConfigFactory
    tags:
      - { name: event_subscriber }
      - { name: service_collector, tag: 'config.factory.override', call: addOverride }
    arguments: ['@config.storage', '@event_dispatcher', '@config.typed']
{% endhighlight %}

<em>Note that how we subscribe to events in Drupal may change based on the issue here: <a href="https://www.drupal.org/node/2023613">https://www.drupal.org/node/2023613</a></em>

<h3>Event Listeners versus Event Subscribers.</h3>

Although Drupal developers don’t have much of a choice here and should be using event subscribers wherever needed but event listeners and event subscribers can be used in any application independently. The choice is personal but subscribers are easier to reuse because the knowledge of the events is kept in the class rather than in the service definition. This is the reason why Symfony uses subscribers internally and probably why drupal also supports only event subscribers only.


<h3>Summary</h3>
<ul>
<li>Event listeners and Subscribers serve the same purpose and can be used in an application indistinctly.</li>
<li>Event listeners can be added via service definition and also with <em>addListener()</em> method.</li>
<li>Drupal doesn't support service definition way of adding event listeners. Listeners can be added only using the <em>addListener()</em> method.</li>
<li>Event subscribers are added via service definition and by implementing the <em>getSubscribedEvents()</em> method.</li>
<li>Event subscribers are easier to use and reuse.</li>
<li>Event listener is registered specifying the events on which it listens. The subscriber has a method telling the dispatcher what events it is listening to.</li>
</ul>

<span class="disclaimer-tag">Note: Some of the definitions and examples have been taken from <a href="http://symfony.com/doc/current/index.html">Symfony documentation</a>.</span>
<span class="fix-tag">If you find any typo or any error, please feel free to fix and raise a pull request <a href="https://github.com/nidaismailshah/nidaismailshah.github.io">here</a>.</span>
