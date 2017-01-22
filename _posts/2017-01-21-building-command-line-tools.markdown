---
layout: post
title:  "Building command line tools with swift."
date:   2017-01-21 10:28:00
categories: Xcode
---

Building basic command line tools with swift is pretty straight forward.
But it gets complicated when you want to use frameworks, run your tests and release the tool.

In this post I'll show you how to setup a simple hello world command line tool that uses CocoaPods to setup frameworks and runs tests and how to use that tool from anywhere.

Start by creating a new Xcode project, in the first step go to *macOS* and choose the *Cocoa Application* template.
It is important not to choose the obvious *Command Line Tool* option, because then you can not easily use frameworks.

![Create a new Cocoa Appliaction](/assets/images/swift-command-line/01.png)

In the next step I'll name my tool *hello* and uncheck the box for Unit Tests.
We will add the tests later.

Next go to your Xcode project and create a new framework called *helloFramework*.
This time make sure to check the *include unit tests* button to create an additional unit test target.

Your project structure should now look like this:
![Targets](/assets/images/swift-command-line/02.png)

For this demo I'm using just two frameworks.
*Commander* is a framework that allows you to craft beautiful command line interfaces, *RainbowSwift* can help you create coloured output on the terminal.

Create a podfile with the dependencies and run pod install.
{% highlight ruby %}
target 'hello' do
  use_frameworks!
  pod 'Commander', '~> 0.6'
end

target 'helloFramework' do
  use_frameworks!
  pod 'RainbowSwift', '~> 2.0'
  target 'helloFrameworkTests' do
    inherit! :search_paths
  end
end
{% endhighlight %}

If you now run the tests in Xcode you see a window pop up.
That's because we selected the *Cocoa Application* template when creating the project.
To prevent that just delete the *MainMenu.xib* and the *Assets.xcassets* files in your project navigator.
When you run the tests again everything seems to work but you do not get the tests succeeded notification.
To investigate the issue run the tests on the command line:

{% highlight bash %}
xcodebuild -workspace hello.xcworkspace -scheme 'hello' build test | xcpretty -c
{% endhighlight %}

You'll get this error:

```
Testing failed:
	Test target helloFrameworkTests encountered an error (Early unexpected exit, operation never finished bootstrapping - no restart will be attempted)
```

To fix it, go to the *Info.plist* and delete the *Main nib file base name* entry.
Now the tests should be working again.

Time to add some code to you application.
In this setup you can only test the helloFramework, not the main target.
I recommend putting everything you really want to test, like your business logic and complex algorithms in the *helloFramework* target.
The main *hello* target is only used for setting up commands with *Commander* that make calls to your *helloFramework*.

Let's first go to the *helloFramework*, create a file named *Hello.swift* and add this code:

{% highlight swift %}
import Foundation
import RainbowSwift

public final class Hello {
    public init() {}
    
    public func sayHello(to name: String) -> String {
        return "Hello \(name)!".green
    }
}
{% endhighlight %}

Then go to the "helloFrameworkTests" and replace the code in the *helloFrameworkTests.swift* file by this:

{% highlight swift %}
import XCTest
@testable import helloFramework

class helloFrameworkTests: XCTestCase {
    func testExample() {
        let result = Hello().sayHello(to: "World")
        XCTAssert(result.contains("Hello World!"))
    }
}
{% endhighlight %}

Try to run your tests in the console and you'll get an error.
But there is an easy fix: go to the *helloFrameworkTests* target and on the general tab set the *Host Application* to *None*.

![Host Application](/assets/images/swift-command-line/05.png)

Now your tests should be running again.

Rename your *AppDelegate.swift* to *main.swift*, delete everything inside and add something like this:

{% highlight swift %}
import Commander
import helloFramework

let main = command { (name: String) in
    print(Hello().sayHello(to: name))
}

main.run()
{% endhighlight %}

This code sets up a *command* that calls into the *helloFramework*.
You can now edit your *hello* Scheme, go to *Run* and add a launch argument.

![Add a launch argument](/assets/images/swift-command-line/03.png)

When you now run the app you'll see something like this:

```
[32mHello Justus![0m
```

The characters at the beginning and end of the output will change the outputs colour in a real terminal window, but do not work in the Xcode terminal.

But how do we build the tool for the terminal?
In the *Products* folder in your Project Navigator you'll see an item called *hello.app*.
Right click on it and choose *Show in Finder*.
This will open a finder window at the location of the built app.
Right click on *hello.app* in the finder, choose *Show Package Contents* and inside go to the *Contents* folder.
In the *MacOS* folder is your command line tool.
But if you copy it somewhere else and try to run it from your terminal you'll get an error again.

```
dyld: Library not loaded: @rpath/Commander.framework/Versions/A/Commander
  Referenced from: /Users/justuskandzi/Desktop/./hello
  Reason: image not found
fish: './hello' terminated by signal SIGABRT (Abort)
```

To make it run you'll also need the *Frameworks* folder.
Copy it to the same location as the *hello* executable.
Now everything should be working as expected.

![working hello tool](/assets/images/swift-command-line/04.png)

You can take a look at the [full project](https://github.com/jkandzi/hello) on github.