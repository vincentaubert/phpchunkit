# PHPChunkit

[![Build Status](https://secure.travis-ci.org/jwage/phpchunkit.png?branch=master)](http://travis-ci.org/jwage/phpchunkit)
[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/jwage/phpchunkit/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/jwage/phpchunkit/?branch=master)
[![Code Coverage](https://scrutinizer-ci.com/g/jwage/phpchunkit/badges/coverage.png?b=master)](https://scrutinizer-ci.com/g/jwage/phpchunkit/?branch=master)

PHPChunkit is a library that sits on top of PHPUnit and adds additional
functionality to make it easier to work with large unit and functional
test suites. The primary feature is test chunking and database sandboxing
which gives you the ability to run your tests in parallel chunks on the
same server or across multiple servers.

In order to run functional tests in parallel on the same server, you need to
have a concept of database sandboxing. You are responsible for implementing
the sandbox preparation, database creation, and sandbox cleanup. PHPChunkit
provides a framework for you to hook in to so you can prepare your application
environment sandbox.

## Parallel Execution Example

Imagine you have 100 tests and each test takes 1 second. When the tests are
ran serially, it will take 100 seconds to complete. But if you split the 100
tests in to 10 even chunks and run the chunks in parallel, it will in theory
take only 10 seconds to complete.

Now imagine you have a two node Jenkins cluster. You can spread the run of each
chunk across the 2 servers with 5 parallel jobs running on each server:

### Jenkins Server #1 with 5 job workers

    phpchunkit --num-chunks=10 --chunk=1 --sandbox --create-dbs
    phpchunkit --num-chunks=10 --chunk=2 --sandbox --create-dbs
    phpchunkit --num-chunks=10 --chunk=3 --sandbox --create-dbs
    phpchunkit --num-chunks=10 --chunk=4 --sandbox --create-dbs
    phpchunkit --num-chunks=10 --chunk=5 --sandbox --create-dbs

### Jenkins Server #2 with 5 job workers

    phpchunkit --num-chunks=10 --chunk=6 --sandbox --create-dbs
    phpchunkit --num-chunks=10 --chunk=7 --sandbox --create-dbs
    phpchunkit --num-chunks=10 --chunk=8 --sandbox --create-dbs
    phpchunkit --num-chunks=10 --chunk=9 --sandbox --create-dbs
    phpchunkit --num-chunks=10 --chunk=10 --sandbox --create-dbs

## Screenshot

![PHPChunkit Screenshot](https://raw.githubusercontent.com/jwage/PHPChunkit/master/docs/phpchunkit.png?1)

## Installation

Install in your project with composer:

    composer require jwage/phpchunkit
    ./vendor/bin/phpchunkit

Install globally with composer:

    composer global require jwage/phpchunkit
    ln -s /home/youruser/.composer/vendor/bin/phpchunkit /usr/local/bin/phpchunkit
    cd /path/to/your/project
    phpchunkit

Install Phar:

    wget https://github.com/jwage/phpchunkit/raw/master/phpchunkit.phar
    chmod +x phpchunkit.phar
    sudo mv phpchunkit.phar /usr/local/bin/phpchunkit
    cd /path/to/your/project
    phpchunkit

## Setup

As mentioned above in the introduction, you are responsible for implementing
the sandbox preparation, database creation and sandbox cleanup processes
by adding [EventDispatcher](http://symfony.com/doc/current/components/event_dispatcher.html)
listeners. You can listen for the following events:

- `sandbox.prepare` - Use the `Events::SANDBOX_PREPARE` constant.
- `databases.create` - Use the `Events::DATABASES_CREATE` constant.
- `sandbox.cleanup` - Use the `Events::SANDBOX_CLEANUP` constant.

Take a look at the listeners implemented in this projects test suite for an example:

- [SandboxPrepare.php](https://github.com/jwage/phpchunkit/blob/master/tests/Listener/SandboxPrepare.php)
- [DatabasesCreate.php](https://github.com/jwage/phpchunkit/blob/master/tests/Listener/DatabasesCreate.php)
- [SandboxCleanup.php](https://github.com/jwage/phpchunkit/blob/master/tests/Listener/SandboxCleanup.php)

### Configuration

Here is an example `phpchunkit.xml` file. Place this in the root of your project:

```xml
<?xml version="1.0" encoding="UTF-8"?>

<phpchunkit
    bootstrap="./tests/phpchunkit_bootstrap.php"
    root-dir="./"
    tests-dir="./tests"
    phpunit-path="./vendor/bin/phpunit"
    memory-limit="512M"
    num-chunks="2"
>
    <watch-directories>
        <watch-directory>./src</watch-directory>
        <watch-directory>./tests</watch-directory>
    </watch-directories>

    <database-names>
        <database-name>testdb1</database-name>
        <database-name>testdb2</database-name>
    </database-names>

    <events>
        <listener event="sandbox.prepare">
            <class>PHPChunkit\Test\Listener\SandboxPrepare</class>
        </listener>

        <listener event="sandbox.cleanup">
            <class>PHPChunkit\Test\Listener\SandboxCleanup</class>
        </listener>

        <listener event="databases.create">
            <class>PHPChunkit\Test\Listener\DatabasesCreate</class>
        </listener>
    </events>
</phpchunkit>

```

The `tests/phpchunkit_bootstrap.php` file is loaded after the XML is loaded
and gives you the ability to do more advanced things with the [Configuration](https://github.com/jwage/phpchunkit/blob/master/src/Configuration.php).

Here is an example:

```php
<?php

use PHPChunkit\Events;

// Manipulate $configuration which is an instance of PHPChunkit\Configuration

$rootDir = $configuration->getRootDir();

$configuration = $configuration
    ->setWatchDirectories([
        sprintf('%s/src', $rootDir),
        sprintf('%s/tests', $rootDir)
    ])
    ->setTestsDirectory(sprintf('%s/tests', $rootDir))
    ->setPhpunitPath(sprintf('%s/vendor/bin/phpunit', $rootDir))
    ->setDatabaseNames(['testdb1', 'testdb2'])
    ->setMemoryLimit('256M')
    ->setNumChunks(2)
;

$eventDispatcher = $configuration->getEventDispatcher();

$eventDispatcher->addListener(Events::SANDBOX_PREPARE, function() {
    // prepare the sandbox
});

$eventDispatcher->addListener(Events::SANDBOX_CLEANUP, function() {
    // cleanup the sandbox
});

$eventDispatcher->addListener(Events::DATABASES_CREATE, function() {
    // create databases
});
```

## Available Commands

Run all tests:

    phpchunkit

Run just unit tests:

    phpchunkit --exclude-group=functional

Run 4 chunks of tests across 2 parallel processes:

    phpchunkit --exclude-group=functional --num-chunks=4 --parallel=2

Run all functional tests:

    phpchunkit --group=functional

Run a specific chunk of functional tests:

    phpchunkit --num-chunks=5 --chunk=1

Run test paths that match a filter:

    phpchunkit --filter=BuildSandbox

Run a specific file:

    phpchunkit --file=tests/Command/BuildSandboxTest.php

Run tests that contain the given content:

    phpchunkit --contains="SOME_CONSTANT_NAME"

Run tests that do not contain the given content:

    phpchunkit --group=functional --not-contains="SOME_CONSTANT_NAME"

Run tests for changed files:

> Note: This relies on git to know which files have changed.

    phpchunkit --changed

Watch your code for changes and run tests:

    phpchunkit watch

Create databases:

    phpchunkit create-dbs

Generate a test skeleton from a class:

    phpchunkit generate "MyProject\ClassName"

Save the generated test to a file:

    phpchunkit generate "MyProject\ClassName" --file=tests/MyProject/Test/ClassNameTest.php

Pass through options to PHPUnit when running tests:

    phpchunkit --phpunit-opt="--coverage-html /path/to/save/coverage"

List all the available options:

    phpchunkit --help

Help information for setting up PHPChunkit:

    phpchunkit setup

## Demo Project

Take a look at [jwage/phpchunkit-demo](https://github.com/jwage/phpchunkit-demo) to see how it can be integrated in to an existing PHPUnit project.

## IRC

Please join irc.freenode.net/phpchunkit to ask questions.
