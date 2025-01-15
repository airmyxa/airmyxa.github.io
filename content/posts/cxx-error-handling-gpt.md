Title: Thoughts on Error Handling

Date: 2024-12-15T11:03:02+04:00

Draft: true


Have you ever thought properly about error handling in your code? Some languages have their own idioms for error handling, like Go with err != nil or Rust with the Result type. But others offer the freedom to implement whatever you want. Here are some of my thoughts on how to handle errors when you have a choice of approach.

As a language for examples, I'll use C++ because it allows the use of all error handling approaches I want to discuss here. As a task to illustrate the problem, consider a function Foo that accepts some data, communicates with a remote server over a network, and returns the data back to the calling side. The code might look something like this:

c++
Ret Foo(D dependencies, R request) {
    ...
    return response;
}

This looks pretty simple and well-readable, but do you see any problems with this? This code assumes the client always responds with some good response with the awaited data, which is expected on the calling side. But is this always the case? Of course not. Let's see what can go wrong:

When dealing with network communication, we should be aware of network errors like timeouts when reaching the server.
When working with network communication protocols, we must process all possible responses that the server might return.

Let's explore ways to handle all the errors we could encounter during this seemingly simple task.

Conditions
First, let's set up some points for good error handling:

We do not ignore errors.
We need to capture the context of the error somehow.
We need to retrieve the value if no error occurs.

With these conditions in mind, let's discuss the approaches.

Optional Return Value
The method I most often see in production code is returning an optional value when an error occurs. On the calling side, we check if we have a value and process it differently depending on its presence. To satisfy the condition of getting the error context, we log with an appropriate level from info to error. Here's a sample:

c++
std::optional<Ret> Foo(D dependencies, R request) {
    ...
    LOG_ERROR() << "Could not get data from remote service myservice: " << error;
    return opt_response;
}

Do you already see the problem?

It's a good idea to think about error handling from the calling point of view. Here, the calling side might look like this:

c++
auto data = Foo(deps, request);
if (!data.has_value()) {
    LOG_INFO() << "Could not get data but it's because we can gently degrade in our functionality";
}

What do we get here:

Two logs with different levels for the same event - should we consider this as an error or info? Two identical logs don't make error research easier.
The calling side doesn't know that the function already logged all the info, so it logs again. But what if this data was crucial for our functionality? Then we would need to take actions on the calling side depending on the error type: timeout or non-2xx HTTP response code. Here, we can't do this due to a lack of error context.

Throw on Error
Another common approach is to throw an exception on error and return on success:

c++
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
            .message = "Could not get data from remote server: " + error.description,
            .context = MakeContext(error),
        };
    }
    return response;
}

This feels like a standard way to handle errors in languages where exceptions are supported. But let's look at how the calling side of the code looks:

cpp
auto data = Foo(deps, req);

Do you see anything indicating that Foo can throw an exception? I don't. Moreover, even if you know Foo throws an exception, how do you understand what type of exception it throws? You go look through the call stack until you find one? But if you're involved in network communication using a protocol like HTTP, there can be multiple reasons for an error, leading to multiple exception types, which can't be in a linear hierarchy.

The algorithm to handle errors for this function API:

First, understand (feel with all your experience) that the function might throw an exception.
Hopefully, you find documentation comments listing all possible exception types, but if it's not a well-maintained library, you're unlikely to find any documentation, so you must explore the call stack to understand all exception types and their hierarchy.
After this journey through the code, you can handle exceptions, but you must do it carefully in the correct order to not lose information.

When handling exceptions, remember some key points about call stack behavior:

If an exception occurs in a constructor, the destructor won't be invoked.
What happens if you get an exception while processing an exception in a destructor? Yes, the best thing possible - terminate.
And more...

Also, remember that using exceptions adds another level of indirection. For example, if you open a resource like a database connection and then an exception occurs, the connection might be left in a dangling state unless you provide a mechanism like C++ RAII to manage it.

Return Struct
If we need to return more than one type from a function, we typically use structs. Here, we could return both data and error:

cpp
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
        };
    }
    return FooResult {
        .data = response,
        .error = std::nullopt,
    };
}

The error can be any structured data providing rich context for processing outside Foo.

And on the calling side:

cpp
auto [data, error] = Foo(deps, req);
if (error.has_value()) {
    // handle error
} else if (data.has_value()) {
    // handle data
} else {
    // oh shit, I don't know what to do with it...
}

Do you see the problem? I do. Have you ever encountered std::optional<bool>? It's similar here. With two optional types, you actually have four possible states:

value, value
nil, value
value, nil
nil, nil

We can agree that if there's an error value, we don't look at data, which seems fine for most real cases. But this agreement removes only one redundant case. We still have nil, nil, which can't be processed properly. Plus, why should we have access to one type when another should not be present?

Most likely, this situation feels familiar if you've programmed in Go. The closest C++ way to express Go's error handling might be using tuples:

cpp
std::tuple<R, E> Foo(D dependencies, R request);

Sum Types
Sum types are a concept better explained elsewhere, like Wikipedia. Here, we'll focus on the standard way to express data when you need one of multiple possible types. In C++, this is std::variant, allowing only one value to be stored, ensuring we know what type is stored, and using just enough space for the largest type. Other languages might use Rust enums or Haskell sum types.

cpp
std::variant<R, E> Foo(D dependencies, R request) {
    if (error) {
        return MakeError(...);
    }
    return data;
}

On the calling side:

cpp
auto result = Foo(deps, req);
std::visit(Overloaded{
    [](R& data) {/*process data*/},
    [](E& error) {/*process error*/},
}, result);

Or you might check the variant this way:

cpp
auto result = Foo(deps, req);

if (auto* data = std::get_if<R>(&result)) {
    // process data
} else if (auto* error = std::get_if<E>(&result)) {
    // process error
} else {/*fck...*/}

Using std::variant gives:

Safe processing of all types with std::visit - if you forget one, it won't compile.
We only store necessary data.
No access to non-existent data.
No added indirection to control flow.

But:

The syntax is complex.
Functions in the call stack that don't process errors need to check and pass them along, which isn't ideal with std::variant.

Result
Now, let's wrap std::variant into a more user-friendly interface:

cpp
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

Here's the calling side:

cpp
auto result = Foo(deps, req);
if (!result.IsOk()) {
    // handle error
} else {
    // handle data
}

This gives us variant's advantages without C++'s verbosity and complex syntax. 

For C++ enthusiasts: Result is now in the standard library!

What I Didn't Mention
There are additional ways of error handling:

Return Error, Value as Inout Parameter
cpp
std::optional<E> Foo(D dependencies, R request, Ret& inout);

Ret value;
auto error = Foo(deps, req, value);
if (error.has_value()) {...}

Error Codes
I'd rather not delve into this; let's skip it.

Functional Style
cpp
void Foo(D dependencies, R request, std::function<void(OkResponse)> on_success, std::function<void(ErrResponse)> on_error) {
    if (error) {
        return on_error(error);
    }
    return on_success(response);
}

You might think this isn't used much, but I've worked with code bases where this was the guideline for error handling. Good times writing 9-level recursive closures in C++...

What to Choose?
No one-size-fits-all answer here, but here are some points:

Provide enough context to the calling side to make informed decisions on error handling.
Make the interface explicit regarding error handling.
Don't handle errors twice. If you return an error, consider it will be handled elsewhere.
Use what's already established in your project - it's simple and likely correct. Changing error handling can lead to a mix of approaches, which could be worse.

Remarks about the Code
I didn't aim for correct or production-ready code. You won't find r/l/g/x-value references or language-specific constructs. This article isn't about that.
The specifics of how we reach a remote service or how our client returns values or errors aren't important here. We're focusing on what happens next, once we know the execution result, and how to design an API for it.
