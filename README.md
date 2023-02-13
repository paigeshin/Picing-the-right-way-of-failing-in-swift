Let’s start with a list

# Here are all (as far as I know) ways that you can handle errors in Swift:

### Return nil or an error enum case. 

The simplest form of error handling is to simply return nil (or an .error case if you’re using a Result enum as your return type) from a function that encountered an error. While this can be really useful in many situations, over-using it for all error handling can quickly lead to APIs that are cumbersome to use, and also risks hiding faulty logic.

###  Throwing an error (using throw MyError)

This requires the caller to handle potential errors using the do, try, catch pattern. Alternatively, errors can be ignored using try? at the call site.

###  Using assert() and assertionFailure() 

To verify that a certain condition is true. Per default, ***this causes a fatal error in debug builds, while being ignored in release builds.*** It’s therefor not guaranteed that execution will stop if an assert is triggered, so it’s kind of like a severe runtime warning.

### Using precondition() and preconditionFailure() instead of asserts. 

The key difference is that these are always* evaluated, even in release builds. That means that you have a guarantee that execution will never continue if the condition isn’t met.

###  Calling fatalError() 

You have probably seen in Xcode-generated implementations of init(coder:) when subclassing an NSCoding-conforming system class, such as UIViewController. Calling this directly kills your process.

### Calling exit()

It exists your process with a code. This is very useful in command line tools and scripts, when you might want to exit out of the global scope (for example in main.swift).

*Unless you are compiling using the Ounchecked optimization mode.

# Recoverable

- Returning nil or an error enum case
- Throwing an error

# Non-recoverable

- Using assert()
- Using precondition()
- Calling fatalError()
- Calling exit()

## Recoverable 

### Return nil or an error enum case.

```swift
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
```

### Throwing an error (using throw MyError)

```swift
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
```

### Using precondition() and preconditionFailure() instead of asserts.

```swift
guard let config = FileLoader().loadFile(named: "Config.json") else {
    preconditionFailure("Failed to load config file")
}
```

### Using assert() and assertionFailure() 

```swift
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
```
