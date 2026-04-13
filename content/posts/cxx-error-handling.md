---
date: '2025-01-15T9:03:02+04:00'
draft: false
title: 'Thoughts on error handling'
---

---

Have you ever properly thought about error handling in your code?
We encounter errors all the time: when network errors occur, when processing user input, and generally whenever side effects are involved.

Some languages have their own idioms for error handling, like Go's `err != nil` or Rust's `Result` type. 
Others provide an ability to do whatever rubbish you want. So here are some of my thoughts on handling errors when you have the freedom to choose your strategy.

For examples, I’ll use C++, as it supports all the error-handling approaches I’ll discuss here.
Let’s consider a function Foo that accepts some data, communicates with a remote server, and returns a response to the caller. The code might look like this:
```c++
Ret Foo(D dependencies, R request) {
	const auto response = dependencies.client.FetchData(request);
	return response;
}
```

At first glance, this seems simple enough, but do you notice any issues? This code assumes the server will always respond correctly with the expected data.
But is that always the case? Of course not! Let’s explore what could go wrong:

* Network errors, such as timeouts, might occur when trying to reach the server.
* Protocol-specific issues might arise, requiring us to handle a variety of potential server responses.

Now, let’s examine the various approaches to error handling.

---

### Conditions

Before diving into specific methods, let’s establish some principles of good error handling:

* Do not ignore errors.
* Provide sufficient context about the error.
* Ensure access to the expected value when no error occurs.

With these conditions in mind, let’s proceed.

---

### Optional return value

A common approach is to return an optional value when an error occurs. The caller checks for the presence of a value and handles the situation accordingly.
To provide error context, we log a message with an appropriate severity level. Here’s an example:

```c++
std::optional<Ret> Foo(D dependencies, R request) {
	...
	LOG_ERROR() << "Could not get data from remote service myservice: " << error;
	return opt_response;
}
```

I see this approach to error handling every day millions of times in production code. Do you already see the problem?

It's a good idea to think about error handling from the calling side's point of view. Here is how the calling side may look:

```c++
auto data = Foo(deps, request);
if (!data.has_value()) {
	LOG_INFO() << "Could not get data but it's because we can gently degrade in our functionality";
}
```

So what do we get here:

* Two identical logs with different levels — when we see them, should we consider it an error or info? [Two identical logs don't make error investigation easier](https://github.com/uber-go/guide/blob/master/style.md#handle-errors-once).
* The calling side doesn't know that the function already logged all the info, so it logs one more time. But what if this data was crucial for our functionality? Then we would need to take some actions on
the calling side depending on the error type: timeout or non-2xx HTTP response code. We can't do this here because of the lack of error context.

---

### Throw on error

Another common method is to throw an exception on error and return the expected value on success:

```c++
struct FooError : std::exception {
	int code;
	std::string message;
	Context context;
};

Ret Foo(D dependencies, R request) {
	...
	if (error) {
		throw FooError {
			.code = error.code,
			.message = "error: " + error.description,
			.context = MakeContext(error),
		};
	}

	return response;
}
```

This feels like a standard way to process errors in languages where exceptions are supported.
But let's see how the calling side of code looks like:

```cpp
auto data = Foo(deps, req);
```

Do you see anything that shows that Foo can throw an exception? I don't. More than that, even if you know that Foo throws an exception, how do you figure out
what type of exception it throws? Should you go look through the call stack till you find one? But if we take part in network communication using some protocol like HTTP, we can
have multiple reasons for error, so we can have multiple exception types, and they might not be in a linear hierarchy or could be thrown not only in one place.

So the algorithm to understand how to handle errors for this kind of function API:

1. First of all you should understand (feel with all of your experience) that the function throws an exception.
2. Hopefully you find doc comments with all exception types it throws, and hopefully the doc comments contain the correct structure. But if it's not some well-maintained lib,
you are not likely to find any docs, so go look through the call stack and find all the exception types it uses and understand the hierarchy.
3. After all the journey to the depths of the code, you can finally handle exceptions. But you need to do it carefully with the correct order of handling to not lose any information.

When you get all your work done and handling exceptions seems OK, you'd better suddenly recall some cool facts about call stack behavior when an exception occurs:

* If an exception occurs in a constructor, the destructor will not be invoked.
* What happens when you get an exception while processing an exception in a destructor? Yes, the best thing possible — terminate.
* And more...

And also you should remember that using exceptions always adds indirection. For example, you open some resource like a DB connection. After that, an exception
occurs. The DB connection will be left in a dangling state if you don't provide some mechanism like C++ RAII to rollback and close the connection in a destructor.

---

### Return struct

If we need to return more than one type from a function, we usually use structs. Like in our situation: data and error. Why not use our beloved method? Let's see.

Code using struct would look like this:

```cpp
struct FooResult {
	std::optional<R> data;
	std::optional<E> error;
};

FooResult Foo(D dependencies, R request) {
...
	if (error) {
		return FooResult {
			.data = std::nullopt,
			.error = MakeErrorContext(...),
		}
	}
	return FooResult {
		.data = response,
		.error = std::nullopt,
	};
}
```

Error can be any structured data with rich context to process it from outside of Foo.

And calling side may look like this:

```cpp
auto [data, error] = Foo(deps, req);
if (error.has_value()) {
	// handle error
} else if (data.has_value()) {
	// handle data
} else {
	// oh shit I don't know what to do with it...
}
```

See the problem? I do. Have you ever encountered std::optional<bool>? Same story.
Because you have two optional types, you really have four possible data sets:
1. value, value
2. nil, value
3. value, nil
4. nil, nil

We can have an agreement that if we have an error value, we don't look at data. It looks fine for most real cases. But it removes only one
redundant case. We still have `nil, nil` which can't be processed properly. More than that, we have access to one type that really should not be present at all. I mean, why can we get access to
the error type if no error takes place? Or the opposite.

Most likely you find it familiar if you've ever programmed in Go. I guess the closest C++ way to express Go-style error handling is using tuples:
```cpp
std::tuple<R, E> Foo(D dependencies, R request);
```

---

### Sum types

Sum types is a concept you'd better read about somewhere else, like [wikipedia](https://en.wikipedia.org/wiki/Algebraic_data_type). 
But here we will talk about the standard way of expressing data in case we need to have one of multiple possible types.
In C++ the standard way is to use `std::variant`. It allows us to store only one value, we always know what value type it can store,
and it uses just enough space to store the biggest value type. In other languages you may see more ways to implement this behavior, like Rust enums or Haskell sum types.

```cpp
std::variant<R, E> Foo(D dependencies, R request) {
	...
	if (error) {
		return MakeError(...);
	}
	return data;
}
```

So here we can see that we construct and return only data that we need to have. No error construction when no error occurs and so on.
Let's see what happens on the calling side of code:
```cpp
auto result = Foo(deps, req);
std::visit(Overloaded{
	[](R& data) {/*process data*/},
	[](E& error) {/*process error*/},
}, result);
```

Or we can use other construction to check what value is present in variant:
```cpp

auto result = Foo(deps, req);

if (auto* data = std::get_if<R>(&result)) {
	// process data
} else if (auto* error = std::get_if<E>(&result)) {
	// process error
} else {/*welp...*/}
```

Something similar we get using index() function that returns index of holding type or npos if variant is in invalid state.

So what do we get using this kind of error handling? First of all — absolutely ugly syntax, yes. But it's C++, so we're used to it. Let's move on to advantages:

* Using `std::visit` gives us the ability to process all variant types safely. If we forget one, it fails to compile. We don't have any redundant state of data.
* We store only data we need.
* We don't have access to data we don't have.
* We don't add any indirection to our control flow.

But (yes, there's always a but...)

* The syntax is complex
* All the functions on the call stack above Foo that don't process the error will need to somehow check if an error occurred and pass it further. With `std::variant` it doesn't look like a
good decision.

So here comes...

---

### Result interface

Let's now wrap our variant into a nice interface:

```cpp
template <typename Ok, typename Err>
class Result {
public:
	Result(Ok);
	Result(Err);

	bool IsOk() const;

	Ok Value() const;

	Err Error() const;

private:
	std::variant<Ok, Err> result_;
};

Result<Res, E> Foo(D dependencies, R request) {
	if (error) {
		return Result(MakeError(...));
	}
	return Result(MakeData(...));
}
```

Here is the calling side:

```cpp
auto result = Foo(deps, req);
if (!result.IsOk()) {
	auto error = result.Error();
	// handle error
} else {
	auto value = result.Value();
	// handle data
}
```

So now we have all the advantages of variant but don't have the problems related to C++ verbosity and complex syntax. Cool, isn't it?
Now we see from the function call side a type that restricts direct access to a possibly nonexistent value, forces each function to explicitly process the error if it occurs,
and makes our control flow linear and safe.

Of course this Result interface example is a bit castrated, and folks familiar with Rust would rather castrate me for this, but I'm not pretending to write production-quality code
in this article, so I don't care.

For C++ gangsters: Result is [in the standard library now](https://en.cppreference.com/w/cpp/utility/expected)!

---

### What I didn't mention

Of course there are some more ways of error handling available with different levels of comfort in different programming languages. Let's list them:

#### Return error, value is inout parameter

```cpp
std::optional<E> Foo(D dependencies, R request, Ret& inout);

Ret value;
auto error = Foo(deps, req, value);
if (error.has_value()) {...}
```

Or the other way around.

#### Error codes

Oh no... Not this, please. I don't want to get into this, let's skip it.

#### Functional style

```cpp
void Foo(D dependencies, R request, std::function<void(OkResponse)> on_success, std::function<void(ErrResponse)> on_error) {
	if (error) {
		return on_error(error);
	}
	return on_success(response);
}
```

You think this is not really used much and you don't care? Ha! I worked with a codebase where this was the prescribed way to handle errors!
What a great time it was writing 9 level recursive closures in C++...

---

### What to choose?

There is no correct or incorrect answer here. But I'll try to share some points on what I think:

* Provide enough context to the calling side to make correct decisions how to handle error
* Make interface explicit from error handling point of view
* Don't handle an error twice.
* Use what is already used in your project — it's the simplest and most likely correct answer. Changing the error handling approach can lead to many error handling approaches at once,
and that would be much worse.

---

## Remarks about the code

* I didn't try to write correct or production-quality code. You won't find r/l/g/x-value C++ references or any language-specific constructs. This article is not about that.
* It's not important what is under the hood of reaching the remote service and how our client returns us a value or an error. We just don't care. Here we look at what happens next when we already know
the result of execution and try to provide an API for it.


