# Ray.Aop

## Aspect Oriented Framework

[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/ray-di/Ray.Aop/badges/quality-score.png?b=2.x)](https://scrutinizer-ci.com/g/Ray-Di/Ray.Aop/?branch=2.x)
[![Code Coverage](https://scrutinizer-ci.com/g/ray-di/Ray.Aop/badges/coverage.png?b=2.x)](https://scrutinizer-ci.com/g/Ray-Di/Ray.Aop/?branch=2.x)
[![Build Status](https://travis-ci.org/ray-di/Ray.Aop.svg?branch=2.x)](https://travis-ci.org/ray-di/Ray.Aop)

[\[Japanese\]](https://github.com/ray-di/Ray.Aop/blob/2.x/README.ja.md)

**Ray.Aop** package provides method interception. This feature enables you to write code that is executed each time a matching method is invoked. It's suited for cross cutting concerns ("aspects"), such as transactions, security and logging. Because interceptors divide a problem into aspects rather than objects, their use is called Aspect Oriented Programming (AOP).

A [Matcher](http://bearsunday.github.io/builds/Ray.Aop/api/class-Ray.Aop.Matchable.html) is a simple interface that either accepts or rejects a value. For Ray.AOP, you need two matchers: one that defines which classes participate, and another for the methods of those classes. To make this easy, there's factory class to satisfy the common scenarios.

[MethodInterceptors](http://bearsunday.github.io/builds/Ray.Aop/api/class-Ray.Aop.MethodInterceptor.html) are executed whenever a matching method is invoked. They have the opportunity to inspect the call: the method, its arguments, and the receiving instance. They can perform their cross-cutting logic and then delegate to the underlying method. Finally, they may inspect the return value or exception and return. Since interceptors may be applied to many methods and will receive many calls, their implementation should be efficient and unintrusive.

Example: Forbidding method calls on weekends
--------------------------------------------

To illustrate how method interceptors work with Ray.Aop, we'll forbid calls to our pizza billing system on weekends. The delivery guys only work Monday thru Friday so we'll prevent pizza from being ordered when it can't be delivered! This example is structurally similar to use of AOP for authorization.

To mark select methods as weekdays-only, we define an annotation. (Ray.Aop uses Doctrine Annotations)

```php
<?php
/**
 * NotOnWeekends
 *
 * @Annotation
 * @Target("METHOD")
 */
final class NotOnWeekends
{
}
```

...and apply it to the methods that need to be intercepted:

```php
<?php
class RealBillingService
{
    /**
     * @NotOnWeekends
     */
    chargeOrder(PizzaOrder $order, CreditCard $creditCard)
    {
```

Next, we define the interceptor by implementing the org.aopalliance.intercept.MethodInterceptor interface. When we need to call through to the underlying method, we do so by calling `$invocation->proceed()`:

```php
<?php
class WeekendBlocker implements MethodInterceptor
{
    public function invoke(MethodInvocation $invocation)
    {
        $today = getdate();
        if ($today['weekday'][0] === 'S') {
            throw new \RuntimeException(
                $invocation->getMethod()->getName() . " not allowed on weekends!"
            );
        }
        return $invocation->proceed();
    }
}
```

Finally, we configure everything. In this case we match any class, but only the methods with our `@NotOnWeekends` annotation:

```php
<?php

use Ray\Aop\Sample\Annotation\NotOnWeekends;
use Ray\Aop\Sample\Annotation\RealBillingService;

$pointcut = new Pointcut(
    (new Matcher)->any(),
    (new Matcher)->annotatedWith(NotOnWeekends::class),
    [new WeekendBlocker]
);
$bind = new Bind(RealBillingService::class, [$pointcut]);
$billing = (new Compiler($tmpDir))->newInstance(RealBillingService::class, [], $bind);

try {
    echo $billing->chargeOrder();
} catch (\RuntimeException $e) {
    echo $e->getMessage() . "\n";
    exit(1);
}
```

Putting it all together, (and waiting until Saturday), we see the method is intercepted and our order is rejected:

```php
<?php
RuntimeException: chargeOrder not allowed on weekends! in /apps/pizza/WeekendBlocker.php on line 14

Call Stack:
    0.0022     228296   1. {main}() /apps/pizza/main.php:0
    0.0054     317424   2. Ray\Aop\Weaver->chargeOrder() /apps/pizza/main.php:14
    0.0054     317608   3. Ray\Aop\Weaver->__call() /libs/Ray.Aop/src/Weaver.php:14
    0.0055     318384   4. Ray\Aop\ReflectiveMethodInvocation->proceed() /libs/Ray.Aop/src/Weaver.php:68
    0.0056     318784   5. Ray\Aop\Sample\WeekendBlocker->invoke() /libs/Ray.Aop/src/ReflectiveMethodInvocation.php:65
```

Explicit method name match
---------------------------

```php
<?php
    $bind = (new Bind)->bindInterceptors('chargeOrder', [new WeekendBlocker]);
    $compiler = new Compiler($tmpDir);
    $billing = $compiler->newInstance('RealBillingService', [], $bind);
    try {
        echo $billing->chargeOrder();
    } catch (\RuntimeException $e) {
        echo $e->getMessage() . "\n";
        exit(1);
    }
```

Own matcher
-----------

You can have your own matcher.
To create `contains` matcher, You need to provide a class which have two method. One is `matchesClass` for class match.
The other one is `matchesMethod` method match. Both return the boolean result of matched.

```php
use Ray\Aop\AbstractMatcher;
use Ray\Aop\Matcher;

class IsContainsMatcher extends AbstractMatcher
{
    /**
     * {@inheritdoc}
     */
    public function matchesClass(\ReflectionClass $class, array $arguments)
    {
        list($contains) = $arguments;

        return (strpos($class->name, $contains) !== false);
    }

    /**
     * {@inheritdoc}
     */
    public function matchesMethod(\ReflectionMethod $method, array $arguments)
    {
        list($contains) = $arguments;

        return (strpos($method->name, $contains) !== false);
    }
}
```

```php
$pointcut = new Pointcut(
		(new Matcher)->any(),
		new IsContainsMatcher('charge'),
		[new WeekendBlocker]
);
$bind = new Bind(RealBillingService::class, [$pointcut]);
$billing = (new Compiler($tmpDir))->newInstance(RealBillingService::class, [], $bind);
```

Priority
--------

The order of interceptor invocation are determined by following rules.

 * Basically, it will be invoked in bind order.
 * `PriorityPointcut` has most priority.
 * Annotation method match is followed by `PriorityPointcut`. Invoked in annotation order as follows.

```php
/**
 * @Auth    // 1st
 * @Cache   // 2nd
 * @Log     // 3rd
 */
```

Limitations
-----------

Behind the scenes, method interception is implemented by generating code at runtime. Ray.Aop dynamically creates a subclass that applies interceptors by overriding methods.

This approach imposes limits on what classes and methods can be intercepted:

 * Classes must be *non-final*
 * Methods must be *public*
 * Methods must be *non-final*

AOP Alliance
------------

The method interceptor API implemented by Ray.Aop is a part of a public specification called [AOP Alliance](http://aopalliance.sourceforge.net/doc/org/aopalliance/intercept/MethodInterceptor.html).

Requirements
------------

* PHP 5.4+
* hhvm

Installation
------------

The recommended way to install Ray.Aop is through [Composer](https://github.com/composer/composer).

```bash
# Add Ray.Aop as a dependency
$ composer require ray/aop ~2.0
```

Testing Ray.Aop
---------------

Here's how to install Ray.Aop from source and run the unit tests and demos.

```bash
$ composer create-project ray/aop:~2.0 Ray.Aop
$ cd Ray.Aop
$ phpunit
$ php docs/demo/run.php
```

See also the DI framework [Ray.Di](https://github.com/koriym/Ray.Di) which integrates DI and AOP.

* This documentation for the most part is taken from [Guice/AOP](https://github.com/google/guice/wiki/AOP).
