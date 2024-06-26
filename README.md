# Content

- [Mork partial test](#test-mock-partial)
- [Test old()](#test-old-helper)
- [Avoid unnecessary work when querying relationships](#unnecessary-work-when-querying-relationships)


## Test mock partial

A partial mock lets you mock specific methods on a class, but allows the class to otherwise work as it normally would. This can be a useful testing technique.

And Laravel, helpful as always, provides a helper on the base TestCase to create partial mocks:

![image](https://github.com/GrytsenkoAndrey/ed-laravel-test-mock-partial/assets/63291871/1b6fee8c-21d0-407e-9740-f0d150118496)

One thing the docs don't mention is that a partial mock created this way will not run the constructor of the class being mocked.

This can be confusing at first, because you might assume the constructor would run as normal. Our intention is only to mock one or two specific methods, after all.

The underlying Mockery library made the decision that you don't want the constructor to be called, even for a partial mock, because it might have some unintended side effects. So this is the default behavior for partial mocks, and this is what the Laravel test helper does.

But if you really need the constructor to run, you can accomplish this by using a different array-based syntax:

![image](https://github.com/GrytsenkoAndrey/ed-laravel-test-mock-partial/assets/63291871/7a52fb63-c533-4410-a7ce-f2c2ee0a0f1b)

Mockery calls this a "partial test double" and refers to the other default kind of partial mock as a "runtime partial double".

There is no Laravel test helper to generate this. If you need your constructor to run on a partial mock, you'll have to manually call Mockery with this syntax instead.

## Test old helper

I was writing a test for a Blade component recently, and there was some value handling in the constructor I wanted to cover. This component was meant for user input, and you could specify a default value, but it would also fall back to old() form input if present.

There are a couple ways you could test this. One approach would be to set up the world where you have data in the session which the old() helper will look for. Then you could call your component, render it, and make assertions against the output.

But this involves a fair amount of setup since that session comes from the active request. And if I want to do a simple Blade component test, I don't even need a request.

Another approach would be to simply assert that the old() helper was called with the expected field name and default value fallback. The advantage here is there's nothing to set up and no request to generate.

I don't think there's one universally right answer here. Some of it boils down to a matter of taste and what else you might be doing in your test or component.

In my case, I chose the second approach. Which leads to the question: How do you do it?

The old() helper is on the Request, so you might start by mocking that Request class. I won't go into all the details here, but this is a bad idea, and will lead you down a difficult path.

Here's how I did it instead:

![image](https://github.com/GrytsenkoAndrey/ed-laravel-clermont/assets/63291871/9426684e-5ec1-4de8-b63a-044ec14faa6e)

The spy doesn't try to mock out the Request class, it just monitors calls into its methods, enabling us to make assertions about them later.

With this test, I'm confident that my Blade component constructor was properly considering old form input, but I don't have to set up a bunch of session data to prove it.

## Unnecessary work when querying relationships

I was working in a controller that was queried by a bunch of receipt printers looking for pending jobs.

The goal was to improve reliability and performance. I saw some code that was looking to see if there was a pending job:

![image](https://github.com/GrytsenkoAndrey/ed-laravel-clermont/assets/63291871/18bdba8b-d3c7-4f37-b736-35f11936154e)

The $pendingJob was only being used as a boolean. The actual model was never used for anything.

At first, I was going to do this:

![image](https://github.com/GrytsenkoAndrey/ed-laravel-clermont/assets/63291871/3c634b4a-e6c8-4c86-948a-4ab7d7af2a8c)

This has a bug though, since the printJobs property is a collection, and there is no exists() method on a collection.

So then my next thought was to do this:

![image](https://github.com/GrytsenkoAndrey/ed-laravel-clermont/assets/63291871/15cc6e22-552b-49ed-8255-5c48843c9b19)

It is true that isNotEmpty works on a collection, and has the same basic meaning as exists, but this is not good code.

Why? If the whole point was to avoid loading and then not using print jobs, I have made absolutely no progress on that goal. I've basically changed nothing about the query.

The SQL query run in both cases looks something like this:

![image](https://github.com/GrytsenkoAndrey/ed-laravel-clermont/assets/63291871/35607dae-b5d0-468b-9dda-0afc8ce53e51)

Notice how we're still querying for all the print jobs, and then Eloquent will hydrate each one of these rows into a model. All of that takes time and memory.

A much better solution is to add a filter to the relationship method:

![image](https://github.com/GrytsenkoAndrey/ed-laravel-clermont/assets/63291871/a3acfb8d-8f2b-42bf-b2e6-2f0c4b0d4e9f)

Notice how I'm now using the relationship method instead of the property, so it runs a much more efficient query.

Now the SQL query looks something like this:

![image](https://github.com/GrytsenkoAndrey/ed-laravel-clermont/assets/63291871/31ef3cfb-18f5-4ac4-9234-a97e90d02391)

So instead of returning all the rows from the database and unnecessarily hydrating models, my query returns a single row with a single boolean value.

This might seem like a small thing, but in my case the code was in an endpoint getting polled every couple seconds by a fleet of printers, so this small improvement actually had a significant impact.






















