WebdriverIO [![Build Status](https://travis-ci.org/webdriverio/webdriverio.png?branch=master)](https://travis-ci.org/webdriverio/webdriverio) [![Coverage Status](https://coveralls.io/repos/webdriverio/webdriverio/badge.png?branch=master&)](https://coveralls.io/r/webdriverio/webdriverio?branch=master)
===========

[![Selenium Test Status](https://saucelabs.com/browser-matrix/webdriverio.svg)](https://saucelabs.com/u/webdriverio)

This library is a webdriver module for Node.js. It makes it possible to write
super easy selenium tests in your favorite BDD or TDD test framework. It was
originated by [Camilo Tapia's](https://github.com/camme) inital selenium project
called WebdriverJS.

Have a look at the many [examples](examples/).

For news or announcements follow [@webdriverio](https://twitter.com/WebdriverIO) on Twitter.

## How to install it

```shell
npm install webdriverio
```

## Usage

Make sure you have a running Selenium standalone/grid/hub. Or use [selenium-standalone](https://github.com/vvo/selenium-standalone)
package to run one easily.

Once you initialized your WebdriverIO instance you can chain all available [protocol and action commands](http://webdriver.io/api.html)
to execute asynchronous requests sequentially. WebdriverIO supports callback and promise based chaining. You can
either pass a callback as last parameter to handle with the command results:

```js
var webdriverio = require('../index');
var options = {
    desiredCapabilities: {
        browserName: 'chrome'
    }
};

webdriverio
    .remote(options)
    .init()
    .url('http://www.google.com')
    .getTitle(function(err, title) {
        console.log('Title was: ' + title);
    })
    .end();
```

or you can handle it like a A+ promise:

```js
webdriverio
    .remote(options)
    .init()
    .url('http://www.google.com')
    .getTitle()
        .then(function(title) {
            console.log('Title was: ' + title);
        })
        .reject(function(error) {
            console.log('uups something went wrong', error);
        })
    .end();
```

Using promised based assertion libraries like [chai-as-promised](https://github.com/domenic/chai-as-promised/)
makes functional testing with WebdriverIO super easy. No nested callbacks anymore! No confusion whether to use
callbacks or promises!

```js
var chai = require("chai");
var chaiAsPromised = require("chai-as-promised");
chai.use(chaiAsPromised);
chai.should();
chaiAsPromised.transferPromiseness = client.transferPromiseness;

webdriverio
    .remote(options)
    .init()
    .url('http://www.google.com')
    .getTitle().should.eventually.equal("Google"); // true
    .end();
```

See the [full list of options](#options) you can pass to `.remote(options)`.

## Options

### desiredCapabilities
Type: `Object`<br>

**Example:**

```js
browserName: 'chrome',  // options: firefox, chrome, opera, safari
version: '27.0',        // browser version
platform: 'XP',         // OS platform
tags: ['tag1','tag2'],  // specify some tags (e.g. if you use Sauce Labs)
name: 'my test'         // set name for test (e.g. if you use Sauce Labs)
```

See the [Selenium documentation](https://code.google.com/p/selenium/wiki/DesiredCapabilities) for a list of the available `capabilities`.

### logLevel
Type: `String`

Default: *silent*

Options: *verbose* | *silent* | *command* | *data* | *result*

### screenshotPath
Saves a screenshot to a given path if Selenium driver crashes

Type: `String`|`null`

Default: *null*

### singleton

Type: `Boolean`

Default: *false*

Set to true if you always want to reuse the same remote

## Selector API

The JsonWireProtocol provides several strategies to query an element. WebdriverIO simplifies these
to make it more familiar with the common existing selector libraries like [Sizzle](http://sizzlejs.com/).
The following selector types are supported:

- **CSS query selector**<br>
  e.g. `client.click('h2.subheading a', function(err,res) {...})` etc.
- **link text**<br>
  To get an anchor element with a specific text in it (f.i. `<a href="http://webdriver.io">WebdriverIO</a>`)
  query the text starting with an equal (=) sign. In this example use `=WebdriverIO`
- **partial link text**<br>
  To find a anchor element whose visible text partially matches your search value, query it by using `*=`
  in front of the query string (e.g. `*=driver`)
- **tag name**<br>
  To query an element with a specific tag name use `<tag>` or `<tag />`
- **name attribute**<br>
  For quering elements with a specific name attribute you can eather use a normal CSS3 selector or the
  provided name strategy from the JsonWireProtocol by passing something like `[name="some-name"]` as
  selector parameter
- **xPath**<br>
  It is also possible to query elements via a specific xPath. The selector has to have a format like
  for example `//BODY/DIV[6]/DIV[1]/SPAN[1]`

In near future WebdriverIO will cover more selector features like form selector (e.g. `:password`,`:file` etc)
or positional selectors like `:first` or `:nth`.

## Eventhandling

WebdriverIO inherits several function from the NodeJS [EventEmitter](http://nodejs.org/api/events.html) object.
Additionally it provides an experimental way to register events on browser side (like click,
focus, keypress etc.).

#### Eventhandling

The following functions are supported: `on`,`once`,`emit`,`removeListener`,`removeAllListeners`.
They behave exactly as described in the official NodeJS [docs](http://nodejs.org/api/events.html).
There are some predefined events (`error`,`init`,`end`, `command`) which cover important
WebdriverIO events.

**Example:**

```js
client.on('error', function(e) {
    // will be executed everytime an error occured
    // e.g. when element couldn't be found
    console.log(e.body.value.class);   // -> "org.openqa.selenium.NoSuchElementException"
    console.log(e.body.value.message); // -> "no such element ..."
})
```

All commands are chainable, so you can use them while chaining your commands

```js
var cnt;

client
    .init()
    .once('countme', function(e) {
        console.log(e.elements.length, 'elements were found');
    })
    .elements('.myElem', function(err,res) {
        cnt = res.value;
    })
    .emit('countme', cnt)
    .end();
```

**Note:** make sure you check out the [Browserevent](https://github.com/webdriverio/eventlistener) side project
that enables event-handling on client side (Yes, in the browser!! ;-).

## Adding custom commands

If you want to extend the client with your own set of commands there is a
method called `addCommand` available from the client object:

```js
var client = require("webdriverio").remote();

// example: create a command the returns the current url and title as one result
// last parameter has to be a callback function that needs to be called
// when the command has finished (otherwise the queue stops)
client.addCommand("getUrlAndTitle", function(customVar, cb) {
    this.url(function(err,urlResult) {
        this.getTitle(function(err,titleResult) {
            var specialResult = {url: urlResult.value, title: titleResult};
            cb(err,specialResult);
            console.log(customVar); // "a custom variable"
        })
    });
});

client
    .init()
    .url('http://www.github.com')
    .getUrlAndTitle('a custom variable', function(err,result){
        assert.equal(null, err)
        assert.strictEqual(result.url,'https://github.com/');
        assert.strictEqual(result.title,'GitHub · Build software better, together.');
    })
    .end();
```

## Local testing

If you want to help us in developing WebdriverIO, you can easily add
[mocha](https://github.com/visionmedia/mocha) [tests](test/) and run them locally:

```sh
npm install -g selenium-standalone http-server phantomjs

# start a local Selenium instances
start-selenium

# serves the test directory holding the test files
http-server

# runs tests !
npm test
```

## Selenium cloud providers

WebdriverIO supports

* <img src="https://pbs.twimg.com/profile_images/794342508/Logo_Square_bigger.png" width="48" /> [Sauce Labs](https://saucelabs.com/)
* <img src="https://pbs.twimg.com/profile_images/1440403042/logo-separate-big_bigger.png" width="48" /> [BrowserStack](http://www.browserstack.com/)
* <img src="https://pbs.twimg.com/profile_images/1647337797/testingbot1_bigger.png" width="48" /> [TestingBot](https://testingbot.com/)

See the corresponding [examples](examples/).

## List of current commands methods
These are the current implemented command methods. All methods take from 0
to a couple of parameters. Also all methods accept a callback so that we
can assert values or have more logic when the callback is called. WebdriverIO
has all [JSONWire protocol](https://code.google.com/p/selenium/wiki/JsonWireProtocol)
commands implemented and even a whole bunch of undocumented [Appium](http://appium.io/)
commands of the Selenium.

- **addValue(`String` selector, `String|String[]` value, `Function` callback)**<br>adds a value to an object found by a selector. You can also use unicode characters like `Left arrow` or `Back space`. You'll find all supported characters [here](https://code.google.com/p/selenium/wiki/JsonWireProtocol#/session/:sessionId/element/:id/value). To do that, the value has to correspond to a key from the table.
- **call(callback)**<br>call given function in async order of current command queue
- **chooseFile(`String` selector, `String` localFilePath, `Function` callback)**<br>Given a selector corresponding to an `<input type=file>`, will upload the local file to the browser machine and fill the form accordingly. It does not submit the form for you.
- **clearElement(`String` selector, `Function` callback)**<br>clear an element of text
- **click(`String` selector, `Function` callback)**<br>Clicks on an element based on a selector.
- **close([`String` tab ID to focus on,] `Function` callback)**<br>Close the current window (optional: and switch focus to opended tab)
- **deleteCookie(`String` name, `Function` callback)**<br>Delete a cookie for current page.
- **doubleClick(`String` selector, `Function` callback)**<br>Clicks on an element based on a selector
- **drag(`String` selector, `Number` startX, `Number` startY, `Number` endX, `Number` endY, `Number` touchCount, `Number` duration, `Function` callback)**<br>Perform a drag on the screen or an element (works only on [Appium](https://github.com/appium/appium/blob/master/docs/gestures.md))
- **dragAndDrop(`String` sourceCssSelector, `String` destinationCssSelector, `Function` callback)**<br>Drags an item to a destination
- **dragDown(`String` selector, `Number` touchCount, `Number` duration, `Function` callback)**<br>Perform a drag down on an element (works only on [Appium](https://github.com/appium/appium/blob/master/docs/gestures.md))
- **dragLeft(`String` selector, `Number` touchCount, `Number` duration, `Function` callback)**<br>Perform a drag left on an element (works only on [Appium](https://github.com/appium/appium/blob/master/docs/gestures.md))
- **dragRight(`String` selector, `Number` touchCount, `Number` duration, `Function` callback)**<br>Perform a drag right on an element (works only on [Appium](https://github.com/appium/appium/blob/master/docs/gestures.md))
- **dragUp(`String` selector, `Number` touchCount, `Number` duration, `Function` callback)**<br>Perform a drag up on an element (works only on [Appium](https://github.com/appium/appium/blob/master/docs/gestures.md))
- **emit(`String` eventName, [arg1], [arg2], [...])**<br>Execute each event listeners in order with the supplied arguments.
- **end(`Function` callback)**<br>Ends a sessions (closes the browser)
- **endAll(`Function` callback)**<br>Ends all sessions (closes the browser)
- **execute(`String` or `Function` script, `Array` arguments, `Function` callback)**<br>Inject a snippet of JavaScript into the page for execution in the context of the currently selected frame. If script is a `Function`, arguments is required.
- **flick(`String` selector, `Number` startX, `Number` startY, `Number` endX, `Number` endY, `Number` touchCount, `Function` callback)**<br>Perform a flick on the screen or an element (works only on [Appium](https://github.com/appium/appium/blob/master/docs/gestures.md))
- **getAttribute(`String` selector, `String` attribute name, `Function` callback)**<br>Get an attribute from an dom obj based on the selector and attribute name
- **getCookie(`String` name, `Function` callback)**<br>Get cookie for the current page. If no cookie name is specified the command will return all cookies.
- **getCssProperty(`String` selector, `String` css property name, `Function` callback)**<br>Gets a css property from a dom object selected with a selector
- **getCurrentTabId(`Function` callback)**<br>Retrieve the current window handle.
- **getElementSize(`String` selector, `Function` callback)**<br>Gets the width and height for an object based on the selector
- **getLocation(`String` selector, `Function` callback)**<br>Gets the x and y coordinate for an object based on the selector
- **getLocationInView(`String` selector, `Function` callback)**<br>Gets the x and y coordinate for an object based on the selector in the view
- **getOrientation(`Function` callback)**<br>Get the current browser orientation.
- **getSource(`Function` callback)**<br>Gets source code of the page
- **getTabIds(`Function` callback)**<br>Retrieve the list of all window handles available to the session.
- **getTagName(`String` selector, `Function` callback)**<br>Gets the tag name of a dom obj found by the selector
- **getText(`String` selector, `Function` callback)**<br>Gets the text content from a dom obj found by the selector
- **getTitle(`Function` callback)**<br>Gets the title of the page
- **getValue(`String` selector, `Function` callback)**<br>Gets the value of a dom obj found by selector
- **isSelected(`String` selector, `Function` callback)**<br>Return true or false if an OPTION element, or an INPUT element of type checkbox or radiobutton is currently selected (found by selector).
- **isVisible(`String` selector, `Function` callback)**<br>Return true or false if the selected dom obj is visible (found by selector)
- **leftClick(`String` selector, `Function` callback)**<br>Apply left click at an element. If selector is not provided, click at the last moved-to location.
- **hold(`String` selector,`Function` callback)**<br>Long press on an element using finger motion events.
- **middleClick(`String` selector, `Function` callback)**<br>Apply middle click at an element. If selector is not provided, click at the last moved-to location.
- **moveToObject(`String` selector, `Function` callback)**<br>Moves the page to the selected dom object
- **newWindow(`String` url, `String` name for the new window, `String` new window features (e.g. size, position, scrollbars, etc.), `Function` callback)**<br>equivalent function to `Window.open()` in a browser
- **on(`String` eventName, `Function` fn)**<br>Register event listener on specific event (the following are already defined: `init`,`command`,`end`,`error`)
- **once(`String` eventName, `Function` fn)**<br>Adds a one time listener for the event (the following are already defined: `init`,`command`,`end`,`error`)
- **pause(`Integer` milliseconds, `Function` callback)**<br>Pauses the commands by the provided milliseconds
- **refresh(`Function` callback)**<br>Refresh the current page
- **release(`String` selector, `Function` callback)**<br>Finger up on an element
- **removeListener(`String` eventName, `Function` fn)**<br>Remove a listener from the listener array for the specified event
- **removeAllListeners([`String` eventName])**<br>Removes all listeners, or those of the specified event
- **rightClick(`String` selector, `Function` callback)**<br>Apply right click at an element. If selector is not provided, click at the last moved-to location.
- **saveScreenshot(`String` path to file, `Function` callback)**<br>Saves a screenshot as a png from the current state of the browser
- **scroll(`String` selector, `Function`callback)**<br>Scroll to a specific element. You can also pass two offset values as parameter to scroll to a specific position (e.g. `scroll(xoffset,yoffset,callback)`).
- **setCookie(`Object` cookie, `Function` callback)**<br>Sets a [cookie](http://code.google.com/p/selenium/wiki/JsonWireProtocol#Cookie_JSON_Object) for current page.
- **setOrientation(`String` orientation, `Function` callback)**<br>Set the current browser orientation.
- **setValue(`String` selector, `String|String[]` value, `Function` callback)**<br>Sets a value to an object found by a selector (clears value before). You can also use unicode characters like `Left arrow` or `Back space`. You'll find all supported characters [here](https://code.google.com/p/selenium/wiki/JsonWireProtocol#/session/:sessionId/element/:id/value). To do that, the value has to correspond to a key from the table.
- **submitForm(`String` selector, `Function` callback)**<br>Submits a form found by the selector
- **switchTab(`String` tab ID)**<br>switch focus to a particular tab/window
- **tap(`String` selector,`Number` x,`Number` y,`Number` tapCount,`Number` touchCount,`Number` duration,`Function` callback)**<br>Perform a tap on the screen or an element (works only on [Appium](https://github.com/appium/appium/blob/master/docs/gestures.md))
- **touch(`String` selector, `Function` callback)**<br>Finger down on an element.
- **waitFor(`String` selector, `Integer` milliseconds, `Function` callback)**<br>Waits for an object in the dom (selected by selector) for the amount of milliseconds provided. the callback is called with false if the object isnt found.

## More on Selenium and its protocol
- [Latest standalone server](http://www.seleniumhq.org/download/)
- [The protocol](http://code.google.com/p/selenium/wiki/JsonWireProtocol)
- [Some useful Selenium resources](https://github.com/christian-bromann/awesome-selenium)

## NPM Maintainers

The npm module for this library is maintained by:

* [Camilo Tapia](http://github.com/Camme)
* [Dan Jenkins](http://github.com/danjenkins)
* [Christian Bromann](https://github.com/christian-bromann)
* [Vincent Voyer](https://github.com/vvo)
