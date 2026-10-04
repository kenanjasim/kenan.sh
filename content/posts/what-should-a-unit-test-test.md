---
title: 'What should a unit test actually test?'
date: "2026-10-04"
author: "Kenan Jasim"
tags: ["golang", "testing", "software engineering"]
readTime: true
toc: true
summary: "Why I stopped testing implementation details, and what testing behaviour through the public API looks like in Go."
description: "Why I stopped testing implementation details, and what testing behaviour through the public API looks like in Go."
---
For a long time I treated the three rules of TDD as law. Don't write production code until you have a failing test. Don't write more of a test than is needed to fail. Don't write more code than is needed to pass. My old notes from 2021 even set a target of 90 to 100% unit test coverage, and I was proud of hitting it.

The problem is that on real projects, the tests slowly became the thing slowing us down. A small refactor would break dozens of tests even though nothing the user could see had changed. It took me a while to work out that TDD itself wasn't the problem. I had misunderstood what I was supposed to be testing.

## How I used to write tests

The way I was taught, every class or function got its own test file. Every public method got at least one test. Anything the code depended on was mocked, so each test exercised exactly one piece of code in isolation.

It felt rigorous. The coverage numbers looked great, and every function had a test sitting next to it that described what it did. I don't think this was a stupid way to work. It is what most books and courses teach, and for small codebases it mostly holds up.

## Where it fell apart

The cracks only showed once a codebase had been around for a year or two.

We regularly had more test code than production code. That isn't a problem in itself, but a lot of it was mock setup that described *how* a function did its job rather than *what* it did. When we changed the how, the mocks had to be rewritten, even though the result was the same.

Refactoring started to feel expensive. And when refactoring is expensive, people stop doing it, which is the opposite of what TDD is meant to encourage. The tests were supposed to give us the confidence to change the design. Instead they locked it in place.

Eventually we had red tests nobody could explain. Was a test failing because the feature was broken, or because someone had renamed a private helper? After enough of those, people stopped trusting the suite and started re-running it until it passed.

The lesson I took from all of this is simple. A test that breaks when the behaviour *hasn't* changed is testing the wrong thing.

## The misunderstanding

Ian Cooper's talk [TDD, Where Did It All Go Wrong](https://www.youtube.com/watch?v=EZ05e7EMOLM) changed how I think about this. His main point is that the "unit" in unit test was never meant to be a class or a function. The unit of isolation is the *test*. Tests should be isolated from each other, so they can run in any order and in parallel. They don't need to isolate every function from every other function.

Once you see it that way, the right thing to test is behaviour through the public API. The internals of a package are implementation details, and you should be free to rewrite them without touching a single test.

This is also what makes the refactor step of red, green, refactor work. You get to green however you can, then clean up the design while the tests stay exactly as they are. If the refactor makes you edit the tests, you aren't really refactoring. You're changing two things at once and hoping they still agree.

## What this looks like in Go

Go makes this easier than most languages, because a test file can declare itself as an external package. If you write `package cache_test` instead of `package cache`, your tests can only see what is exported. The compiler stops you reaching into the internals.

Take a small generic cache with `Set` and `Get`, where items expire after a TTL. The test I would have written a few years ago looks like this:

```go
package cache

func TestSet(t *testing.T) {
	c := New[string, int](time.Minute, time.Now)
	c.Set("a", 1)

	// Reaching into the internal map
	if len(c.items) != 1 {
		t.Fatalf("expected 1 item, got %d", len(c.items))
	}
	if c.items["a"].val != 1 {
		t.Fatalf("expected value 1, got %d", c.items["a"].val)
	}
}
```

It passes, and it's useless. Rename `items`, swap the map for something else, or change how entries are stored, and it breaks. Worse, it doesn't test what `Set` is actually for. Nobody calls `Set` to put something in a map. They call it so that a later `Get` returns the value. The only meaningful way to test `Set` is through `Get`.

The version I would write now only uses the public API:

```go
package cache_test

func TestGetExpiresAfterTTL(t *testing.T) {
	now := time.Now()
	c := cache.New[string, int](time.Minute, func() time.Time { return now })

	c.Set("a", 1)

	if got, ok := c.Get("a"); !ok || got != 1 {
		t.Fatalf("Get(a) = %d, %v; want 1, true", got, ok)
	}

	// Move the clock past the TTL
	now = now.Add(time.Minute + time.Second)

	if _, ok := c.Get("a"); ok {
		t.Fatal("expected item to have expired")
	}
}
```

There is still something being faked here, the clock, but that is the point. Time is a real boundary of the system, like the network or the disk. Those are the places where a fake earns its keep. Faking things *inside* your own package is where tests start to rot.

This test also catches a real bug. An older cache I wrote only removed expired items on a background ticker, so `Get` happily returned stale values until the next tick. The first test would never have noticed. A behaviour test like the second one would have failed straight away.

## So is TDD dead?

No. When David Heinemeier Hansson [declared TDD dead](https://dhh.dk/2014/tdd-is-dead-long-live-testing.html) back in 2014, he was reacting to the same pain I have described above. But the answer isn't to stop writing tests first. It's to stop coupling them to the implementation.

I still often write the test first, because it forces me to think about how the code will be used before I think about how it works. What has changed is that coverage is no longer the goal. If you test every behaviour that matters through the public API, high coverage tends to follow. If you chase the number directly, you end up with tests like the first example.

## Takeaways

1. Test behaviour, not implementation.
2. The test is the unit of isolation, not the class or function.
3. If a refactor breaks a test without changing behaviour, the test was wrong.
4. Only fake things at the edges of your system, such as time, network and disk.
5. In Go, use external `_test` packages so the compiler keeps you honest.

Good tests aren't the ones that give you the highest coverage. They're the ones the team still trusts six months from now, when someone needs to change the code underneath them.
