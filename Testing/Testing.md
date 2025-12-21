## Testing

Testing is the process of assessing that all parts of the program behave as expected of them. Covering the product with the proper amount of testing, allows you to quickly check later to see if anything in the application is broken after adding new or changing old functionality.

-   ### Unit Tests

    The simplest kind of tests. As a rule, about 70-80% of all tests are exactly [unit-tests](https://en.wikipedia.org/wiki/Unit_testing). "Unit" means that not the whole system is tested, but small and separate parts of it (functions, methods, components, etc.) in isolation from others. All dependent external environment is usually covered by [mocks](https://en.wikipedia.org/wiki/Mock_object).

    -   What are the benefits of unit tests?
        > To give you an example, let's imagine a car. Its "units" are the engine, brakes, dashboard, etc. You can check them individually before assembly and, if necessary, replace or repair them. But you can assemble the car without having tested the units, and it will not go. You will have to disassemble everything and check every detail.
    -   What do I need to start writing unit tests?
        > As a rule, the means of the standard language library are enough to write quality tests. But for more convenient and faster writing of tests, it is better to use third-party tools. For example:
        >
        > -   For , it uses [pytest](https://docs.pytest.org), although the standard [unittest](https://docs.python.org/3/library/unittest.html) is enough to start with.
        > -   For JavaScript/TypeScript, the best choices are [Jest](https://jestjs.io/).
        > -   For Go – [testify](https://github.com/stretchr/testify).
        > -   [And so on...](https://github.com/atinfo/awesome-test-automation#awesome-test-automation)

<details>
    <summary>🔗 <b>References</b></summary>

1. 📺 [**Software Testing Explained in 100 Seconds** – YouTube](https://youtu.be/u6QfIXgjwGQ)
2. 📄 [**How to write your first Unit Test** – medium](https://medium.com/geekculture/how-to-write-your-first-unit-test-in-multiple-programming-languages-6d158d362b3d)
3. 📺 [**Testing JavaScript with Cypress – Full Course** – YouTube](https://youtu.be/u8vMu7viCm8?si=wYAoeR87-dPOIRA4)
4. 📺 [**How To Write Unit Tests For Existing Python Code** – YouTube](https://youtu.be/ULxMQ57engo)
5. 📺 [**Learn How to Test your JavaScript Application** – YouTube](https://youtu.be/ajiAl5UNzBU)
6. 📺 [**Golang Unit Testing and Mock Testing Tutorial** – YouTube](https://youtu.be/XQzTUa9LPU8)

 </details>

<div align="right"><a href="#top">Contents ⬆️</a></div>

-   ### Integration tests

    [Integration testing](https://en.wikipedia.org/wiki/Integration_testing) involves testing individual modules (components) in conjunction with others (that is, in integration). What was covered by a stub during Unit testing is now an actual component or an entire module.

    -   Why it's needed?
        > Integration tests are the next step after units. Having tested each component individually, we cannot yet say that the basic functionality of the program works without errors. Potentially, there may still be many problems that will only surface after the different parts of the program interact with each other.
    -   Strategies for writing integration tests
        > -   **Big Bang**: Most of the modules developed are connected together to form either the whole system or most of it. If everything works, you can save a lot of time this way.
        > -   **incremental approach**: By connecting two or more logically connected modules and then gradually adding more and more modules until the whole system is tested.
        > -   **Bottom-up approach**: each module at lower levels is tested with the modules of the next higher level until all modules have been tested.

<details>
    <summary>🔗 <b>References</b></summary>

1. 📺 [**Unit testing vs. integration testing** – YouTube](https://youtu.be/pf6Zhm-PDfQ)
2. 📺 [**PyTest REST API Integration Testing with Python** – YouTube](https://youtu.be/7dgQRVqF1N0)
3. 📄 [**Integration Testing – Software testing fundamentals**](https://softwaretestingfundamentals.com/integration-testing/)
 </details>

<div align="right"><a href="#top">Contents ⬆️</a></div>

-   ### E2E tests

    <p align="center"><img src="./testing-pyramid_eng.png" alt="Testing pyramid"/></p>

    End-to-end tests imply checking the operation of the entire system as a whole. In this type of testing, the environment is implemented as close to real-life conditions as possible. We can draw the analogy that a robot sits at the computer and presses the buttons in the specified order, as a real user would do.

    -   When to use?
        > E2E is the most complex type of test. They take a long time to write and to execute, because they involve the whole application. So if your application is small (e.g., you are the only one developing it), writing Unit and some integration tests will probably be enough.

<details>
    <summary>🔗 <b>References</b></summary>

1. 📄 [**What is End-to-End Testing and When Should You Use It?** – freeCodeCamp](https://www.freecodecamp.org/news/end-to-end-testing-tutorial/)
2. 📺 [**End to End Testing - Explained** – YouTube](https://youtu.be/68xvfrxlEYo)
3. 📺 [**Testing Node.js Server with Jest and Supertest** – YouTube](https://youtu.be/FKnzS_icp20)
4. 📺 [**End to End - Test Driven Development (TDD) to create a REST API in Go** – YouTube](https://youtu.be/tG9dPO6fe4E)
5. 📺 [**How to test HTTP handlers in Go** – YouTube](https://youtu.be/Ztk9d78HgC0)
6. 📄 [**Awesome Testing** – GitHub](https://github.com/TheJambo/awesome-testing)
 </details>

<div align="right"><a href="#top">Contents ⬆️</a></div>

-   ### Load testing

    When you create a large application that needs to serve a large number of requests, there is a need to test this very ability to withstand heavy loads. There are many utilities available to create [artificial load](https://en.wikipedia.org/wiki/Load_testing).

    -   [JMeter](https://en.wikipedia.org/wiki/Apache_JMeter)
        > User-friendly interface, cross-platform, multi-threading support, extensibility, excellent reporting capabilities, support for many protocols for queries.
    -   [LoadRunner](https://en.wikipedia.org/wiki/LoadRunner)
        > It has an interesting feature of virtual users, who do something with the application under test in parallel. This allows you to understand how the work of some users actively doing something with the service affects the work of others.
    -   [Gatling](<https://en.wikipedia.org/wiki/Gatling_(software)>)
        > A very powerful tool oriented to more experienced users. The Scala programming language is used to describe the scripts.
    -   [Taurus](https://gettaurus.org/)
        > A whole framework for easier work on JMeter, Gatling and so on. JSON or YAML is used to describe tests.

<details>
    <summary>🔗 <b>References</b></summary>

1. 📺 [**Getting started with API Load Testing (Stress, Spike, Load, Soak)** – YouTube](https://youtu.be/r-Jte8Y8zag)
2. 📄 [**How to Load Test: A developer’s guide to performance testing** – medium](https://rhamedy.medium.com/how-to-load-test-a-developers-guide-to-performance-testing-5264faaf4e33)
 </details>

<div align="right"><a href="#top">Contents ⬆️</a></div>

-   ### Regression testing

    [Regression testing](https://en.wikipedia.org/wiki/Regression_testing) is a type of testing aimed at detecting errors in already tested portions of the source code.

    -   Why use it?
        > Statistically, the reappearance of the same bugs in code is quite frequent. And, most interestingly, the patches/fixes issued for them also stop working in time. Therefore, it is considered good practice to create a test for it when fixing a bug and run it regularly for next modifications.

<details>
    <summary>🔗 <b>References</b></summary>

1. 📄 [**What Is Regression Testing? Definition, Tools, Method, And Example**](https://www.softwaretestinghelp.com/regression-testing-tools-and-methods/)
2. 📺 [**Regression testing – What, Why, When, and How to Run It?** – YouTube](https://youtu.be/AWX6WvYktwk)
3. 📺 [**Top-5 Tools for Regression Testing** – YouTube](https://youtu.be/HZvqfuADX8g)
 </details>

<div align="right"><a href="#top">Contents ⬆️</a></div>