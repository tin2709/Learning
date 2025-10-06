
# PHP: The Right Way

**Last Updated:** 2025-08-17 07:16:32 +0000

"PHP: The Right Way" is an easy-to-read, quick reference for PHP popular coding standards, authoritative tutorials, and best practices. It aims to guide both new and seasoned PHP developers through crucial topics, offering suggestions for multiple options rather than prescribing single tools. This is a living document, continually updated with helpful information and examples.

## Table of Contents

1.  [Welcome](#welcome)
2.  [Translations](#translations)
3.  [How to Contribute](#how-to-contribute)
4.  [Getting Started](#getting-started)
    *   [Use the Current Stable Version (8.4)](#use-the-current-stable-version-84)
    *   [Built-in Web Server](#built-in-web-server)
    *   [macOS Setup](#macos-setup)
    *   [Windows Setup](#windows-setup)
    *   [Linux Setup](#linux-setup)
    *   [Common Directory Structure](#common-directory-structure)
5.  [Code Style Guide](#code-style-guide)
6.  [Language Highlights](#language-highlights)
    *   [Programming Paradigms](#programming-paradigms)
    *   [Namespaces](#namespaces)
    *   [Standard PHP Library (SPL)](#standard-php-library-spl)
    *   [Command Line Interface (CLI)](#command-line-interface-cli)
    *   [Xdebug](#xdebug)
7.  [Dependency Management](#dependency-management)
    *   [Composer and Packagist](#composer-and-packagist)
    *   [PEAR](#pear)
8.  [Coding Practices](#coding-practices)
    *   [The Basics](#the-basics)
    *   [Date and Time](#date-and-time)
    *   [Design Patterns](#design-patterns)
    *   [Working with UTF-8](#working-with-utf-8)
    *   [Internationalization and Localization (i18n & l10n)](#internationalization-and-localization-i18n--l10n)
    *   [Dependency Injection](#dependency-injection)
9.  [Databases](#databases)
    *   [MySQL Extension](#mysql-extension)
    *   [PDO Extension](#pdo-extension)
    *   [Interacting with Databases](#interacting-with-databases)
    *   [Abstraction Layers](#abstraction-layers)
10. [Templating](#templating)
    *   [Benefits](#benefits)
    *   [Plain PHP Templates](#plain-php-templates)
    *   [Compiled Templates](#compiled-templates)
11. [Errors and Exceptions](#errors-and-exceptions)
    *   [Errors](#errors)
    *   [Exceptions](#exceptions)
12. [Security](#security)
    *   [Web Application Security](#web-application-security)
    *   [Password Hashing](#password-hashing)
    *   [Data Filtering](#data-filtering)
    *   [Configuration Files](#configuration-files)
    *   [Register Globals](#register-globals)
    *   [Error Reporting](#error-reporting)
13. [Testing](#testing)
    *   [Test Driven Development (TDD)](#test-driven-development-tdd)
    *   [Behavior Driven Development (BDD)](#behavior-driven-development-bdd)
    *   [Complementary Testing Tools](#complementary-testing-tools)
14. [Servers and Deployment](#servers-and-deployment)
    *   [Platform as a Service (PaaS)](#platform-as-a-service-paas)
    *   [Virtual or Dedicated Servers](#virtual-or-dedicated-servers)
    *   [Shared Servers](#shared-servers)
    *   [Building Your Application](#building-your-application)
    *   [Server Provisioning](#server-provisioning)
    *   [Continuous Integration](#continuous-integration)
15. [Virtualization](#virtualization)
    *   [Vagrant](#vagrant)
    *   [Docker](#docker)
16. [Caching](#caching)
    *   [Opcode Cache](#opcode-cache)
    *   [Object Caching](#object-caching)
17. [Documenting your Code](#documenting-your-code)
    *   [PHPDoc](#phpdoc)
18. [Resources](#resources)
    *   [From the Source](#from-the-source)
    *   [People to Follow](#people-to-follow)
    *   [PHP PaaS Providers](#php-paas-providers)
    *   [Frameworks](#frameworks)
    *   [Components](#components)
    *   [Other Useful Resources](#other-useful-resources)
    *   [Video Tutorials](#video-tutorials)
    *   [Books](#books)
    *   [Community](#community)
19. [Credits](#credits)

---

## Welcome

PHP: The Right Way provides an up-to-date, easy-to-read reference for popular PHP coding standards and best practices. It addresses common misconceptions and outdated information, guiding developers toward secure and efficient coding. It is a living document, continuously updated.

## Translations

The document is available in multiple languages, including English, Spanish, French, Japanese, Simplified Chinese, and more. A book version (PDF, EPUB, MOBI) is also available on Leanpub.

## How to Contribute

Contributions are welcome! You can help make this resource even better by contributing on [GitHub](https://github.com/phptherightway/phptherightway.com).

## Getting Started

### Use the Current Stable Version (8.4)

Always start with the current stable release, currently PHP 8.4. PHP 8.x offers significant performance improvements and new features over older versions. Ensure you upgrade to the latest stable version, as older versions like PHP 7.4 are End of Life.

### Built-in Web Server

PHP 5.4+ includes a built-in web server, perfect for local development. Run `php -S localhost:8000` from your project's web root.

### macOS Setup

Install PHP via:
*   **Homebrew**: `brew install php` (supports multiple versions).
*   **Macports**: `sudo port install php83`.
*   **phpbrew**: Manages multiple PHP versions.
*   **Liip's binary installer**: `php-osx.liip.ch`.
*   **Compile from Source**: For full control.
*   **All-in-One Installers**: MAMP, XAMPP (convenient but less flexible).

### Windows Setup

Download binaries from [windows.php.net/download](https://windows.php.net/download). Set `PATH` to your PHP folder. For all-in-one solutions, use XAMPP, EasyPHP, OpenServer, or WAMP. For production on Windows, IIS7 with `phpmanager` is recommended. Consider a Virtual Machine if deploying to Linux.

### Linux Setup

Most distributions have PHP in repositories, often older.
*   **Ubuntu/Debian**: Use Ondřej Surý's PPA/bikeshed (`sudo add-apt-repository ppa:ondrej/php`, `sudo apt update`).
*   **RPM-based (CentOS, Fedora)**: Use Remi's RPM repository.
*   Alternatively, use containers or compile from source.

### Common Directory Structure

A recommended structure (Standard PHP Package Skeleton) for web projects:
*   `public/`: DocumentRoot points here (for public scripts).
*   `tests/`: Unit tests.
*   `vendor/`: Third-party libraries (Composer).
*   Configuration files and private data should be outside `public/`.

## Code Style Guide

Adhering to a common code style is crucial for collaboration and library integration. The Framework Interop Group (PHP-FIG) proposes [PSR-1](https://www.php-fig.org/psr/psr-1/), [PSR-12](https://www.php-fig.org/psr/psr-12/), [PSR-4](https://www.php-fig.org/psr/psr-4/) and [PER Coding Style](https://www.php-fig.org/per/coding-style/).
*   **Tools**: [PHP_CodeSniffer](https://github.com/squizlabs/PHP_CodeSniffer) (check), [PHP Coding Standards Fixer](https://github.com/FriendsOfPHP/PHP-CS-Fixer) (fix), [PHP Code Beautifier and Fixer](https://github.com/squizlabs/PHP_CodeSniffer/wiki/Fixing-Errors-Automatically) (fix).
*   **Commands**:
    ```bash
    phpcs -sw --standard=PSR1 file.php
    phpcbf -w --standard=PSR1 file.php
    php-cs-fixer fix -v --rules=@PSR1 file.php
    ```
*   English is preferred for symbol names and code infrastructure. Refer to [Clean Code PHP](https://github.com/jupeter/clean-code-php) for supplementary advice.

## Language Highlights

### Programming Paradigms

PHP supports:
*   **Object-oriented Programming**: Classes, abstract classes, interfaces, inheritance, exceptions, [traits](https://www.php.net/manual/en/language.oop5.traits.php).
*   **Functional Programming**: First-class functions, [anonymous functions (closures)](https://www.php.net/manual/en/functions.anonymous.php) (PHP 5.3+), higher-order functions, recursion.
*   **Meta Programming**: [Reflection API](https://www.php.net/manual/en/book.reflection.php), [Magic Methods](https://www.php.net/manual/en/language.oop5.magic.php) (`__get`, `__set`, `__call`, `__callStatic`).

### Namespaces

Namespaces prevent naming collisions between different libraries. [PSR-4](https://www.php-fig.org/psr/psr-4/) provides a standard for file, class, and namespace conventions, enabling plug-and-play code. (PSR-0 is deprecated but still used in older projects.)

### Standard PHP Library (SPL)

A collection of built-in classes and interfaces for commonly needed data structures (stack, queue, heap) and iterators.
*   [Read about the SPL](https://www.php.net/manual/en/book.spl.php)

### Command Line Interface (CLI)

PHP is useful for CLI scripting, automating tasks like testing and deployment.
*   `php -i`: Prints PHP configuration.
*   `php -a`: Interactive shell.
*   Example (`hello.php`): Access arguments via `$argc` (count) and `$argv` (values).
    ```php
    <?php
    if ($argc !== 2) {
        echo "Usage: php hello.php <name>" . PHP_EOL;
        exit(1);
    }
    $name = $argv[1];
    echo "Hello, $name" . PHP_EOL;
    ```
*   [Learn about running PHP from the command line](https://www.php.net/manual/en/features.commandline.php)

### Xdebug

PHP's debugger for tracing code execution, monitoring stack, and profiling.
*   **Remote Debugging**: Essential for local development/VMs. Configure `xdebug.remote_host` and `xdebug.remote_port` (e.g., 9000).
*   Trigger with `http://your-website.example.com/index.php?XDEBUG_SESSION_START=1`.
*   IDEs offer graphical debugging support.
*   [Learn more about Xdebug](https://xdebug.org/)

## Dependency Management

PHP projects rely on external libraries. Two major package managers exist: Composer (current standard) and PEAR (legacy).

### Composer and Packagist

**Recommended dependency manager for PHP.**
*   **Installation**: Follow official instructions, install globally (`mv composer.phar /usr/local/bin/composer`). Windows users can use `ComposerSetup.exe`.
*   **Define & Install**: List dependencies in `composer.json` (e.g., `composer require twig/twig:^2.0` or `composer init`). Then `composer install`.
*   **Autoloading**: Include `require 'vendor/autoload.php';` in your application.
*   **Updating**: `composer update` (for dev), `composer install` (for deploy with `composer.lock`).
*   **Security**: Use [Local PHP Security Checker](https://github.com/sensiolabs/security-checker) to audit `composer.lock`.
*   **Global Dependencies**: `composer global require phpunit/phpunit` and add `~/.composer/vendor/bin` to `PATH`.
*   [Learn about Composer](https://getcomposer.org/)

### PEAR

A veteran package manager, but less flexible than Composer as packages need to be specifically structured for PEAR and are installed globally.
*   **Installation**: Download `.phar` installer or use distribution package manager (`apt php-pear`).
*   **Install Package**: `pear install foo` or specify a channel.
*   **Composer Integration**: You can handle PEAR dependencies via Composer by defining a `package` repository in `composer.json`.
*   [Learn about PEAR](https://pear.php.net/)

## Coding Practices

### The Basics

A reminder to adhere to fundamental coding practices for efficiency and maintainability.

### Date and Time

Use the `DateTime` class for robust date and time operations, including time zones.
*   **Creation**: `DateTime::createFromFormat()`, `new DateTime()`.
*   **Formatting**: `$datetime->format('Y-m-d')`.
*   **Calculations**: Use `DateInterval` with `add()`, `sub()`, `diff()`.
*   **Iteration**: `DatePeriod` for recurring events.
*   **Enhancement**: [Carbon](https://carbon.nesbot.com/docs/) (inherits `DateTime`, adds localization, testing features).
*   [Read about DateTime](https://www.php.net/manual/en/class.datetime.php)

### Design Patterns

Using common design patterns improves code management and understanding. Frameworks often dictate higher-level patterns (MVC).
*   [refactoring.guru/design-patterns/php](https://refactoring.guru/design-patterns/php)
*   [designpatternsphp.readthedocs.io](https://designpatternsphp.readthedocs.io/)

### Working with UTF-8

PHP's low-level Unicode support requires care.
*   **PHP Level**: Use `mb_*` functions (`mb_strpos`, `mb_strlen`, `mb_substr`) from the Multibyte String Extension. Call `mb_internal_encoding('UTF-8');` and `mb_http_output('UTF-8');`. Always specify `UTF-8` in functions like `htmlentities()`. Consider `symfony/polyfill-mbstring`.
*   **Database Level**: Use `utf8mb4` character set and collation for your database and tables. Specify `charset=utf8mb4` in PDO connection string.
*   **Browser Level**: `header('Content-Type: text/html; charset=UTF-8')` and `<meta charset="UTF-8">`.
*   [Further reading](https://www.php.net/manual/en/book.mbstring.php)

### Internationalization and Localization (i18n & l10n)

*   **i18n**: Structuring code for language/region adaptation (done once).
*   **l10n**: Translating content based on i18n (done for each language).
*   **Pluralization**: Rules for number-sensitive strings (complex for many languages).
*   **Recommended Tool**: [Gettext](https://www.php.net/manual/en/book.gettext.php) (Unix tool, complete, powerful). Use [Poedit](https://poedit.net/) for GUI editing of `.po` files.
*   **File Structure**: `<project root>/locales/<locale_code>/LC_MESSAGES/<domain>.mo/.po`.
*   **Usage**: `gettext('string')` (or `_()`), `ngettext('singular', 'plural', $count)`.

### Dependency Injection

A software design pattern that provides a component with its dependencies, removing hard-coded dependencies and increasing flexibility, testability, and scalability.
*   **Basic Concept**: Pass dependencies (e.g., `MySqlAdapter`) into a class's constructor rather than instantiating them within the class itself.
*   **Complex Problem (IoC & SOLID)**: Solves "Inversion of Control" and adheres to SOLID principles (Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion).
*   **Containers**: Convenience utilities to manage dependencies, but can be misused as Service Locators, creating new hard dependencies. Modern frameworks often include their own DI containers.
*   [Further Reading](https://www.martinfowler.com/articles/injection.html)

## Databases

### MySQL Extension

**Deprecated and removed in PHP 7.0.** Do not use `mysql_*` functions. Upgrade to `mysqli` or `PDO`.

### PDO Extension

**Recommended database connection abstraction library (PHP 5.1.0+).**
*   **Common Interface**: Provides a consistent API for various databases (MySQL, SQLite, etc.).
*   **Security**: Crucially, supports **prepared statements and bound parameters** to prevent SQL injection attacks.
    ```php
    $pdo = new PDO('sqlite:/path/db/users.db');
    $stmt = $pdo->prepare('SELECT name FROM users WHERE id = :id');
    $id = filter_input(INPUT_GET, 'id', FILTER_SANITIZE_NUMBER_INT); // Filter first!
    $stmt->bindParam(':id', $id, PDO::PARAM_INT);
    $stmt->execute();
    ```
*   Connections implicitly close when the PDO object is destroyed.
*   [Learn about PDO](https://www.php.net/manual/en/book.pdo.php)

### Interacting with Databases

Separate database interaction logic from presentation logic. Encapsulate queries in classes (e.g., a "Model") and use separate files for presentation ("View"). This improves testability and readability.

### Abstraction Layers

Many frameworks and libraries provide higher-level abstraction layers (e.g., ORMs) on top of PDO, offering database-agnostic features and further simplifying interactions.
*   Examples: Atlas, Aura SQL, Doctrine2 DBAL, Medoo, Propel, laminas-db.

## Templating

Templates separate controller/domain logic from presentation logic (HTML, XML). Often called "views" in MVC.

### Benefits

*   **Separation of Concerns**: Cleaner, more readable code.
*   **Code Organization & Reuse**: Templates in 'views' folders, use partials (header, footer).
*   **Security**: Automatic escaping, sandboxing (depending on library).

### Plain PHP Templates

Use native PHP code within templates.
*   **Pros**: No new syntax, fast (no compilation).
*   **Frameworks**: Most modern PHP frameworks use plain PHP templates by default.
*   **Libraries**: [Plates](https://platesphp.com/), [Aura.View](https://aura.php.net/packages/Aura.View).

### Compiled Templates

Offer a new, dedicated templating syntax.
*   **Pros**: Easier to write, cleaner to read, safer to use (auto-escaping), inheritance, simplified control structures, can be cross-language (Mustache).
*   **Libraries**: [Twig](https://twig.symfony.com/), [Brainy](https://github.com/brainy/brainy), [Smarty](https://www.smarty.net/) (note: auto-escaping not always default).

## Errors and Exceptions

### Errors

PHP is "exception-light," often continuing execution with notices or warnings.
*   **Severity**: `E_ERROR` (fatal), `E_NOTICE` (advisory), `E_WARNING` (non-fatal), `E_STRICT` (compile-time suggestions).
*   **Reporting**: Configure `error_reporting()`, `display_errors`, `log_errors` differently for development vs. production.
*   **Inline Suppression (`@`)**: Silences errors but has performance implications and hides critical issues. Use the null coalescing operator (`??`) in PHP 7+ as an alternative.
*   **ErrorException**: Convert PHP errors into exceptions for better handling. Frameworks like Symfony and Laravel use this.
*   [Error Control Operators](https://www.php.net/manual/en/language.operators.errorcontrol.php)

### Exceptions

Standard in most languages, allow dynamic handling of errors.
*   **Benefit**: Forces developers to acknowledge and handle issues, making applications more robust.
*   **SPL Exceptions**: PHP provides specialized exception types (e.g., `BadMethodCallException`) extending the generic `Exception` class. Use these or custom exceptions for specific error conditions.
*   [Read about Exceptions](https://www.php.net/manual/en/language.exceptions.php)
*   [Read about SPL Exceptions](https://www.php.net/manual/en/book.spl-exceptions.php)

## Security

**The best resource**: [The 2018 Guide to Building Secure PHP Software by Paragon Initiative](https://paragonie.com/blog/2018/01/2018-guide-building-secure-php-software).

### Web Application Security

Key topics:
*   Code-data separation (prevents SQL Injection, XSS, RFI).
*   Application logic (auth/auth controls, input validation).
*   Operating environment (PHP versions, third-party libraries, OS).
*   Cryptography weaknesses.
*   Refer to [OWASP Security Guide](https://owasp.org/www-project-top-ten/).

### Password Hashing

*   **Hashing vs. Encryption**: Hashing is irreversible and one-way (good for passwords); encryption is reversible.
*   **Salting**: Individually salt passwords (random string) before hashing to prevent dictionary attacks and rainbow tables.
*   **Algorithms**: Use specialized password hashing algorithms like Argon2 (PHP 7.2+), Scrypt, or Bcrypt (PHP 5.5+).
*   **`password_hash()`**: PHP 5.5+ provides `password_hash()` (uses Bcrypt by default) and `password_verify()` which handle salting and algorithm management automatically.
    ```php
    $passwordHash = password_hash('secret-password', PASSWORD_DEFAULT);
    if (password_verify('bad-password', $passwordHash)) { /* Correct */ } else { /* Wrong */ }
    ```
*   [Learn about password_hash()](https://www.php.net/manual/en/function.password-hash.php)

### Data Filtering

**Never trust foreign input.** Always sanitize and validate.
*   **Sources**: `$_GET`, `$_POST`, `$_SERVER`, HTTP request body, uploaded files, session data, cookies, third-party services.
*   **Sanitization**: Removes/escapes illegal/unsafe characters.
    *   XSS prevention: `strip_tags()`, `htmlentities()`, `htmlspecialchars()`.
    *   Command line: `escapeshellarg()`.
    *   File paths: remove `"/"`, `"../"`, null bytes.
    *   `unserialize()` is dangerous with untrusted data; use JSON instead.
*   **Validation**: Ensures input matches expectations (e.g., email format, numeric age).
*   **Functions**: `filter_var()`, `filter_input()`.
*   [Learn about data filtering](https://www.php.net/manual/en/book.filter.php)

### Configuration Files

*   Store configuration outside the document root.
*   If inside, name with `.php` extension to prevent plain text disclosure.
*   Protect with encryption or file system permissions.
*   Do not commit sensitive info (passwords, API tokens) to source control.

### Register Globals

**Removed as of PHP 5.4.0.** For legacy applications (< 5.4.0), ensure `register_globals` is `Off` to prevent security issues caused by variables from `$_POST`, `$_GET`, etc., being globally available and potentially overriding declared variables.

### Error Reporting

Configure `php.ini` differently for environments:
*   **Development**: `display_errors = On`, `display_startup_errors = On`, `error_reporting = -1`, `log_errors = On` (show all errors).
*   **Production**: `display_errors = Off`, `display_startup_errors = Off`, `error_reporting = E_ALL`, `log_errors = On` (log errors, hide from users).
*   [PHP manual: error_reporting](https://www.php.net/manual/en/function.error-reporting.php)

## Testing

Automated testing is a best practice, ensuring application stability and correctness during development and changes.

### Test Driven Development (TDD)

A development process where failing tests are written *before* code, then code is written to pass the tests, and finally refactored.
*   **Unit Testing**: Tests individual functions, classes, and methods.
    *   Tools: [PHPUnit](https://phpunit.de/), [atoum](https://atoum.org/), [Kahlan](https://kahlan.github.io/), [Peridot](https://peridot-php.github.io/), [Pest](https://pestphp.com/), [SimpleTest](http://simpletest.org/).
*   **Integration Testing**: Combines and tests modules as a group. Uses similar tools to unit testing.
*   **Functional Testing (Acceptance Testing)**: Uses tools to simulate actual users interacting with the application.
    *   Tools: [Codeception](https://codeception.com/), [Cypress](https://www.cypress.io/), [Mink](http://mink.behat.org/), [Selenium](https://www.selenium.dev/), [Storyplayer](https://github.com/Storyplayer/Storyplayer).

### Behavior Driven Development (BDD)

*   **StoryBDD**: Writes human-readable stories describing application behavior.
    *   Tool: [Behat](https://behat.org/) (inspired by Ruby's Cucumber, uses Gherkin DSL).
*   **SpecBDD**: Writes specifications describing how code should behave.
    *   Tool: [PHPSpec](https://www.phpspec.net/) (inspired by Ruby's RSpec).

### Complementary Testing Tools

*   [Selenium](https://www.selenium.dev/): Browser automation.
*   [Mockery](https://mockery.github.io/): Mock object framework.
*   [Prophecy](https://prophecy.phpspec.net/): PHP object mocking framework.
*   [php-mock](https://github.com/php-mock/php-mock): Mocks PHP native functions.
*   [Infection](https://infection.github.io/): Mutation Testing for test effectiveness.
*   [PHPUnit Polyfills](https://github.com/dg/phpunit-polyfills): PHPUnit cross-version compatible tests.

## Servers and Deployment

### Platform as a Service (PaaS)

Provides system and network architecture for PHP apps with minimal configuration. Popular for deploying and scaling. See [PHP PaaS Providers](#php-paas-providers) in Resources.

### Virtual or Dedicated Servers

Offer complete control.
*   **nginx and PHP-FPM**: Lightweight, high-performance web server paired with PHP-FPM for efficient request handling.
*   **Apache and PHP**: Widely configurable, but uses more resources than nginx by default. Common configurations: `prefork MPM` with `mod_php` (simple) or `worker/event MPM` with `mod_fastcgi`/`mod_fcgid`/`mod_proxy_fcgi` (more performant).

### Shared Servers

Cheap, but beware of neighboring tenants affecting performance or security. Ensure latest PHP versions are offered.

### Building Your Application

Automate build and deployment tasks to avoid manual errors: dependency management, asset compilation/minification, running tests, documentation, packaging, deployment.
*   **Deployment Tools**:
    *   [Phing](https://phing.info/): XML-based build system (like Apache Ant).
    *   [Capistrano](https://capistranorb.com/): Ruby-based for remote command execution (can deploy PHP).
    *   [Ansistrano](https://ansistrano.com/): Ansible roles for deployment (Ansible port of Capistrano).
    *   [Deployer](https://deployer.org/): PHP-written deployment tool with recipes for frameworks.
    *   [Magallanes](https://github.com/andres-montanez/Magallanes): PHP-written with YAML config, atomic deployment.

### Server Provisioning

Automate server configuration and management (especially for many servers). Integrates with cloud providers.
*   [Ansible](https://www.ansible.com/): YAML-based, simple, scalable infrastructure management.
*   [Puppet](https://puppet.com/): Own language for server management (master/client or master-less).
*   [Chef](https://www.chef.io/): Ruby-based system integration framework.

### Continuous Integration

Team members integrate work frequently.
*   [Travis CI](https://www.travis-ci.com/): Hosted CI service, GitHub integration, supports PHP.
*   [GitHub Actions](https://github.com/features/actions): CI/CD workflows integrated with GitHub.
*   Others: Jenkins, PHPCI, PHP Censor, Teamcity.

## Virtualization

Running dev and prod environments on different setups leads to bugs. Virtualization ensures consistency.

### Vagrant

Builds virtual boxes (VirtualBox, VMware) using a single config file.
*   **Provisioning**: Use tools like Puppet or Chef to automate box setup.
*   **Shared Folders**: Edit code on host, run in VM.
*   Easy to destroy and recreate "fresh" installations.
*   [Learn more about Vagrant](https://www.vagrantup.com/)

### Docker

Lightweight alternative using "containers" (isolated environments) built from "images."
*   **Efficiency**: Quicker installs/downloads, less RAM, faster start/stop times.
*   **Usage**: Create containers from command line or `docker-compose.yml`.
    ```bash
    docker run -d --name my-php-webserver -p 8080:80 -v /path/to/your/php/files:/var/www/html/ php:apache
    ```
*   [PHPDocker.io](https://phpdocker.io/) can auto-generate stack files.
*   [Docker Website](https://www.docker.com/)

## Caching

Speed up applications by reducing remote connections, file loads, etc.

### Opcode Cache

Prevents redundant compilation of PHP files into opcodes.
*   **Zend OPcache**: Built into PHP since 5.5 (check `opcache.enable` in `phpinfo()`).
*   **WinCache**: Extension for MS Windows Server.
*   Significant speed improvement.
*   [PHP Preloading](https://www.php.net/manual/en/opcache.preloading.php) (PHP 7.4+).

### Object Caching

Stores individual objects/data in memory for fast access (e.g., expensive database calls).
*   **APCu**: Excellent for single-server object caching, simple API. Tied to server/PHP processes.
*   **Memcached**: Separate service, accessed across network, scales better for multiple servers.
*   **Redis**: In-memory data structure store, often used for caching.
*   **WinCache Functions**: Provides API for data caching.
*   [APCu Documentation](https://www.php.net/manual/en/book.apcu.php), [Memcached](https://memcached.org/), [Redis](https://redis.io/)

## Documenting your Code

### PHPDoc

An informal standard for commenting PHP code.
*   **Tags**: `@author`, `@link`, `@param`, `@return`, `@throws`, etc.
*   Provides type hints and descriptions for classes, methods, and properties.
*   `@return void` explicitly states no return value.
*   [PHPDoc manual](https://docs.phpdoc.org/latest/guides/references/phpdoc/tags/index.html)

## Resources

### From the Source

*   [PHP Website](https://www.php.net/)
*   [PHP Documentation](https://www.php.net/manual/en/)

### People to Follow

*   [ogprogrammer.com/2017/06/28/how-to-get-connected-with-the-php-community/](https://www.ogprogrammer.com/2017/06/28/how-to-get-connected-with-the-php-community/)
*   [x.com/CalEvans/lists/phpeople](https://x.com/CalEvans/lists/phpeople)

### PHP PaaS Providers

Amezmo, AWS Elastic Beanstalk, Bref Cloud, Cloudways, DigitalOcean App Platform, Divio, Engine Yard Cloud, fortrabbit, Google App Engine, Heroku, IBM Cloud, Lumen, Microsoft Azure, Pivotal Web Services, Platform.sh, Red Hat OpenShift, Virtuozzo.

### Frameworks

*   **Micro-frameworks**: Route HTTP requests (e.g., for HTTP services).
*   **Full-Stack Frameworks**: Comprehensive (ORMs, authentication, etc.).
*   **Component-based frameworks**: Collections of specialized libraries.

### Components

*   [Packagist](https://packagist.org/) (official Composer repository)
*   [PEAR](https://pear.php.net/)
*   Component-based frameworks/vendors: Aura, CakePHP, FuelPHP, Hoa Project, Symfony Components, The League of Extraordinary Packages, Laravel's Illuminate Components.

### Other Useful Resources

*   **Cheatsheets**: PHP Cheatsheets, Modern PHP Cheatsheet, OWASP Security Cheatsheets.
*   **More best practices**: PHP Best Practices, Why You Should Be Using Supported PHP Versions.
*   **Newsletters**: PHP Weekly, JavaScript Weekly, Frontend Focus, Mobile Web Weekly.
*   **PHP universe**: PHP Developer blog.

### Video Tutorials

*   **YouTube Channels**: Learn PHP The Right Way Series, PHP Academy, The New Boston, Sherif Ramadan, Level Up Tuts.
*   **Paid Videos**: PHP Training on Pluralsight, LinkedIn.com, Tutsplus, Laracasts, SymfonyCasts.

### Books

*   **Free Books**: PHP Pandas, PHP The Right Way (this website as a book), Using Libsodium in PHP Projects.
*   **Paid Books**: PHP & MySQL, Build APIs You Won’t Hate, Modern PHP, Building Secure PHP Apps, Modernizing Legacy Applications In PHP, Securing PHP: Core Concepts, Scaling PHP, Signaling PHP, Minimum Viable Tests, Domain-Driven Design in PHP.

### Community

*   **User Groups (PUGs)**: [PHP.ug](https://www.php.ug/), Meetup.com. Special mention: [NomadPHP](https://nomadphp.com/) (online), [PHPWomen](http://phpwomen.org/) (diversity/support).
*   **Conferences**: Find a [PHP Conference](https://www.php.net/conferences/).
*   **Online**: IRC (#phpc on irc.libera.chat), @phpc on X, Mastodon, Discord, StackOverflow.
*   **Elephpants**: PHP project mascot, plush toys at conferences.

## Credits

**Created and maintained by:**
Josh Lockhart
Phil Sturgeon

**Project Contributors**

PHP: The Right Way by Josh Lockhart is licensed under a [Creative Commons Attribution-NonCommercial-ShareAlike 3.0 Unported License](http://creativecommons.org/licenses/by-nc-sa/3.0/).
Based on a work at [www.phptherightway.com](http://www.phptherightway.com/).
