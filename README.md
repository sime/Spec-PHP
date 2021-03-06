# Spec for PHP

Spec for PHP is a tool to implement Behaviour-Driven Development specification
files. It's inspired on [RSpec](http://relishapp.com/rspec) from the Ruby world.

It builds on top of [PHPUnit](http://www.phpunit.de) and [Hamcrest](http://code.google.com/p/hamcrest/)
projects to offer mature and solid functionality and borrow the current
industry support for those projects.

The intent when implementing Spec was to mix normal PHP code and a more
natural language syntax to express expectations. Writing the test code
in pure PHP is great because that is how you are going to use it in your
application and IDE's offer autocompletion for it. However, assertions
are difficult to express and difficult to read with PHP's syntax, thus
it makes sense to write them differently.

To be able to parse Spec's syntax there is an on the fly transformation
that generates valid PHP code. Code generation has its drawbacks, specially
when there is a need to debug a problem, the source file and the executed
one are different so it becomes a nighmare to know what's going on. Spec
takes this into account and will try as hard as possible to generate code
that keeps the original line numbers the same. This is of great help since
you can go to the line number reported in an exception and actually see the
statement that failed. Check the "How does it work?" section for more
details.


> Check [the documentation](http://drslump.github.com/Spec-PHP/) for further
details.


## Features

_Spec for PHP_ is being actively developed and while functional doesn't
have all of its features implemented. As thus it should be considered
_alpha_ quality software for the time being.

### Working features

 - *Describe* and *It* block parsers
 - Natural language expectations parser
 - Coordinated expectations (and, or, but)
 - *Before* and *After* blocks
 - Run expectations against collections/arrays (all, any, none)
 - Parametrized *It* blocks
 - Annotations support (@group, @skip, @todo, @throw, ...)
 - Custom CLI runner tool
 - PHPUnit integration
 - Use annotation to inherit from custom PHPUnit_TestCase class (ie: Zend_Test)
 - Matcher factory to create matchers with a callback function

### Upcoming

 - Review configurable options and extension points

### Later

 - Additional matchers
 - Better descriptions for expectation failures
 - Port features from RSpec 2
 - First class integration of Mock frameworks (PHPUnit, Mockery, ...)


## Requirements

 - PHP 5.3
 - PHPUnit 3 (**Only supports 3.5.15, a version supporting 3.6 is in progress**)
 - Hamcrest matchers library
 - PHP Object_Freezer 1.0.0


## Installation

Install a recent version of PHPUnit and PHP Object_Freezer

    pear channel-discover pear.phpunit.de
    pear install phpunit/PHPUnit-3.5.15
    pear install phpunit/Object_Freezer

Install Hamcrest matchers library

    pear channel-discover hamcrest.googlecode.com/svn/pear
    pear install hamcrest/Hamcrest

Install Spec for PHP

    pear channel-discover pear.pollinimini.net
    pear install drslump/Spec-beta

You can also get the latest version by checking a copy of the repository
in your computer.

To do a test run, simply checkout the repository to get the tests directory
and run the following commands:

    cd tests/
    phpunit AllTests.php

Also try the custom cli runner:

    spec4php tests


## Example

    <?php
    // Implements the StackTest example from PHPUnit manual as a spec file
    describe "Testing array operations with Spec"

        it "should support Push and Pop"

            $stack = array();
            count($stack) should equal 0;

            array_push($stack, 'foo');
            $stack[count($stack)-1] SHOULD == 'foo';
            count($stack) SHOULD BE 1;

            array_pop($stack)
            Should be "foo"

            count($stack)
                should be equal to 0
        end
    end

That's it, really, no `->assert` calls and no classes, methods or other
verbose statements just to fit the code in.

The syntax for blocks is borrowed from [RSpec](http://relishapp.com/rspec),
`describe` groups tests and `it` defines a block of code used to execute a
test for specific functionality.

Note though that this example uses the most _natural language_ syntax
possible with Spec. It could also be writen with closures or using dots
instead of spaces to separate statements, so that it becomes completely
valid PHP syntax. Check the following example:

    <?php
    describe. "Block without closure".
        it("should multiply", function(){
            (2*2) . should.equal(4);
        });
    end;

If you still preffer how a traditional PHPUnit test case looks, you
might still find this library useful, have a look on the section about
PHPUnit compatibility.


## How does it work?

The parser for the custom syntax uses PHP's own tokenizer (`token_get_all`)
to ensure it doesn't choke on weird statements. When it reaches a block
level keyword like `describe` it wraps the contents in a closure function.

The keyword `should` uses a different parsing logic. The statement just
before it is captured as the "value" or "subject" while statemens after it
are captured as the "expectation" or "predicate". It's even able to parse
_complex_ expressions if they are wrapped in parenthesis.

To fool the PHP interpreter so that it receives the transformed code instead
of the original Spec syntax, a custom stream wrapper is used to perform
the transformation. Every file using the `spec://path/to/file.php` notation
will be run thru Spec's parser to transform it if needed.

In order to make some sense of the expectations, the Expect class
applies some simple algorithms like removing common words or managing
sentence combinators like _and_. Take for example the following sentence:

    5 should be an integer and less than 10

would be processed as if it was:

    expect(5)->integer()->and_less(10);


## Compatibility with PHPUnit

One of the design principles when developing Spec was to make it
compatible with PHPUnit, since it's the current standard tool for 
testing in the PHP world.

Spec files are _transformed_ on the fly to be compatible with PHPUnit,
allowing to use its reporting (code coverage, xunit, tap logs) and 
current integrations with IDEs and Continuous integration services.

Feeding a `\DrSlump\Spec\TestSuite` object to PHPUnit it will be able
to execute any spec files it references. For example, by creating the
following class it will run recursively all the spec files found beneath
the directory where the class is defined.

    require_once 'Spec.php';

    class AllSpecs extends \DrSlump\Spec\DirectoryRunnerHelper {
        static function suite() {
            return parent::suite();
        }
    }

If you already have an extended TestCase class with custom assertions
and helper methods, you can make use of it with Spec by using the `class`
annotation. Spec will use the given class as base when generating the
tests. This allows to make use of current PHPUnit extensions like Zend_Test
from Zend Framework for example.

    # class My_TestCase
    describe "My Test" {
      it "should access a method of My_TestCase"
        $this->myMethod();
      end
    }

It's even possible to use the `expect` component in PHPUnit's _native_
test cases.

    class ExpectTest extends PHPUnit_Framework_TestCase {
        function testEqual(){
            expect(1)->to_equal(1);
        }
    }


## Beyond Testing

The `Expect` component is not really tied to PHPUnit or the test harness. It's
primary dependency is the Hamcrest set of matchers, as such, it can also be used
in other scenarios, like for example to validate parameters received in an application
controller.

See the following Zend Framework controller for example:

    class MyController extends Zend_Controller_Action {
        public function myAction() {
            expect($this->getParam('id'))
            ->to_be_numeric->and_more_than(0)
            ->do();

            expect($this->getParam('action'))
            ->to_equal('create')->or('update')->or('delete')
            ->as('Invalid action')
            ->do();

            // ... your logic goes here ...
        }
    }

When an expectation fails an exception is raised so it fits well with most web
frameworks that capture exceptions to render an error page. Of course, the real
power comes by writing custom `matchers`, like an _user exists_ that checks if
an user id is valid for example.


## LICENSE

	The MIT License

	Copyright (c) 2011 Iván -DrSlump- Montes

	Permission is hereby granted, free of charge, to any person obtaining
	a copy of this software and associated documentation files (the
	'Software'), to deal in the Software without restriction, including
	without limitation the rights to use, copy, modify, merge, publish,
	distribute, sublicense, and/or sell copies of the Software, and to
	permit persons to whom the Software is furnished to do so, subject to
	the following conditions:

	The above copyright notice and this permission notice shall be
	included in all copies or substantial portions of the Software.

	THE SOFTWARE IS PROVIDED 'AS IS', WITHOUT WARRANTY OF ANY KIND,
	EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
	MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
	IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
	CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
	TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
	SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

