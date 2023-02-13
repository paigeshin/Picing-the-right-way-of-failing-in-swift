# Picing-the-right-way-of-failing-in-swift

One major focus of Swift is compile time safety — enabling us as developers to easily focus on writing code that is more predictable and less prone to runtime errors. However, sometimes things do fail for various reasons — so this week, let’s take a look at how we can handle such failures appropriately, and what tools we have at our disposal for doing so.

A few weeks ago, we took a look at how to handle optionals that are not really optional — in “Handling non-optional optionals in Swift”. In that post I made the case for using preconditionFailure() in combination with guard, instead of force-unwrapping, and introduced the micro framework Require that provides a convenient API for doing so.

Since that post, a lot of people have asked what the difference is between preconditionFailure() and assert(), and how it relates to Swift’s throwing capabilities. So in this post, let’s take a closer look at all those language features, and when to use each one of them.

Let’s start with a list
Here are all (as far as I know) ways that you can handle errors in Swift:

Return nil or an error enum case. The simplest form of error handling is to simply return nil (or an .error case if you’re using a Result enum as your return type) from a function that encountered an error. While this can be really useful in many situations, over-using it for all error handling can quickly lead to APIs that are cumbersome to use, and also risks hiding faulty logic.
Throwing an error (using throw MyError), which requires the caller to handle potential errors using the do, try, catch pattern. Alternatively, errors can be ignored using try? at the call site.
Using assert() and assertionFailure() to verify that a certain condition is true. Per default, this causes a fatal error in debug builds, while being ignored in release builds. It’s therefor not guaranteed that execution will stop if an assert is triggered, so it’s kind of like a severe runtime warning.
Using precondition() and preconditionFailure() instead of asserts. The key difference is that these are always* evaluated, even in release builds. That means that you have a guarantee that execution will never continue if the condition isn’t met.
Calling fatalError() — which you have probably seen in Xcode-generated implementations of init(coder:) when subclassing an NSCoding-conforming system class, such as UIViewController. Calling this directly kills your process.
Calling exit(), which exists your process with a code. This is very useful in command line tools and scripts, when you might want to exit out of the global scope (for example in main.swift).
*Unless you are compiling using the Ounchecked optimization mode.

Recoverable vs non-recoverable
The key thing to consider when picking the right way of failing is to determine whether the error that occurred is recoverable or not.

For example, let’s say that we’re calling our server and we receive an error response. That is something that’s bound to happen, no matter how awesome programmers we are and how solid our server infrastructure is. So treating these type of errors as fatal & non-recoverable is usually a mistake. Instead, what we want is to recover and probably display some form of error screen to our users.

So, how to pick an appropriate way of failing in this case? If we take a look at the list above, we can kind of split it up into recoverable and non-recoverable techniques, like this:

Recoverable

Returning nil or an error enum case
Throwing an error
Non-recoverable

Using assert()
Using precondition()
Calling fatalError()
Calling exit()
In this case, since we’re dealing with an asynchronous task, returning nil or an error enum case is probably the best choice, like this:

class DataLoader {
    enum Result {
        case success(Data)
        case failure(Error?)
    }

    func loadData(from url: URL,
                  completionHandler: @escaping (Result) -> Void) {
        let task = urlSession.dataTask(with: url) { data, _, error in
            guard let data = data else {
                completionHandler(.failure(error))
                return
            }

            completionHandler(.success(data))
        }

        task.resume()
    }
}
For synchronous APIs, throwing is a great option — as it “forces” our API users to handle the error in an appropriate way:

class StringFormatter {
    enum Error: Swift.Error {
        case emptyString
    }

    func format(_ string: String) throws -> String {
        guard !string.isEmpty else {
            throw Error.emptyString
        }

        return string.replacingOccurences(of: "\n", with: " ")
    }
}
However, sometimes an error is not recoverable. For example, let’s say that we need to load a configuration file during app launch. If that configuration file is missing, it’ll put our app in an undefined state — so in this case crashing is better than continuing program execution. For that, using one of the stronger, non-recoverable ways of failing is more appropriate.

In this case, we use preconditionFailure() to stop execution in case the configuration file was missing:

guard let config = FileLoader().loadFile(named: "Config.json") else {
    preconditionFailure("Failed to load config file")
}
Programmer errors vs execution errors
Another distinction that is important to make is whether an error was caused by faulty logic or incorrect configuration, or whether the error should be considered a legitimate part of the application’s flow. Basically whether the programmer caused the error or whether an external factor did.

When protecting against programmer errors, you almost always want to use the non-recoverable techniques. That way, you don’t have to code around extraordinary circumstances all over your app, and a good suite of tests will make sure that those types of errors will get caught as early as possible.

For example, let’s say we’re building a view that requires a view model to be bound to it before it’s used. The view model will be an optional in our code, but we don’t want to have to unwrap it every time we use it. However, we don’t necessarily want to crash the application in production if the view model somehow has gone missing — getting an error about it in debug is good enough. This is a case for using an assert:

class DetailView: UIView {
    struct ViewModel {
        var title: String
        var subtitle: String
        var action: String
    }

    var viewModel: ViewModel?

    override func didMoveToSuperview() {
        super.didMoveToSuperview()

        guard let viewModel = viewModel else {
            assertionFailure("No view model assigned to DetailView")
            return
        }

        titleLabel.text = viewModel.title
        subtitleLabel.text = viewModel.subtitle
        actionButton.setTitle(viewModel.action, for: .normal)
    }
}
Note that we have to return in our guard statement above, since assertionFailure() will silently fail in release builds.

Conclusion
I hope this post helped to clear up the difference between various types of error handling techniques available in Swift. My advice is to not only stick to one technique, but to pick the most appropriate one depending on the situation. In general, I would also suggest to always try to recover from errors if at all possible, as to not disrupt the user experience unless the error should be treated as fatal.

Also, remember that print(error) is not error handling 😉

Feel free to reach out to me on Twitter if you have any questions, suggestions or feedback. I’d also love to hear from you if you have any topic that you’d like me to cover in an upcoming post.

Thanks for reading! 🚀

