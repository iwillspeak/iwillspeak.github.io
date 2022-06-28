---
layout: post
---

It's been a hectic few months. I've just moved from the blistering sunshine of the south of England to the chilly embrace of the north. That means I've not been able to spend that much time writing code outside of work. Now things have calmed down a bit I decided to integrate google test into a C++ application I am developing. This is the story of just how easy that was.

There is a lot of code in the world. A lot of it is C++. Roughly 85% of that is unit testing frameworks. Seriously, the choices are in no way restrictive. Amongst the most highly praised at the moment is the [Google C++ Testing Framework][gtest], or *GTest* for short.

Getting started with the framework is dead simple. Download the source, `./configure`, `make`, and you're done! It's just as easy to integrate it into your project too. *GTest* has auto-discovery of unit tests so all you need to do to create your first test is compile and link in a file with a test definition. Something like this should suffice:

```cpp
#include <gtest/gtest.h>

TEST(MyAwesomeTest,ThatPasses) {
    auto test_is_awesome = true;
    ASSERT_TRUE(test_is_awesome);
} 
```

With that compiled and linked I'll reveal the next neat feature *GTest* has up it's sleeve. If you're writing tests for a command line app you can get guest to run unit tests with a call to `RUN_ALL_TESTS()`. But what if you're not writing a standalone tool but a dynamic library? Well *GTest* has a minimalist test harness available as a library alongside the main `libgtest` too. Just link `-lgtest_main` on the command line along with your dynamic library and you should have yourself a working unit test  runner.

One of the nicest features of *GTest* for me is the ability to `EXPECT_` as well as `ASSERT_`, allowing you to continue to check more things in a test, even if one of your assumptions has failed. This gives you a bit more insight when a test fails as to the underlying cause of the failure.

I've barely touched the surface of what *GTest* can do here, quite deliberately I should add. There are plenty of resources out there to help you get started. *GTest* has tonnes of neat features, like test fixtures; parameterised tests; and what it calls 'death tests', which test program termination. Next time you find yourself looking for a C++ unit test framework *GTest* definitely deserves some considering.

[gtest]: http://code.google.com/p/googletest/