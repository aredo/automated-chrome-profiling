Let's say you want to evaluate the performance of some clientside JavaScript and want to automate it. Let's kick off our measurement in Node.js and collect the performance metrics from Chrome. Oh yeah.

We can use the [Chrome debugging protocol](https://developer.chrome.com/devtools/docs/debugger-protocol) and go directly to [how Chrome's JS sampling profiler interacts with V8](https://code.google.com/p/chromium/codesearch#chromium/src/third_party/WebKit/Source/devtools/protocol.json&q=file:protocol.json%20%22Profiler%22,&sq=package:chromium&type=cs).

So much power here, so we'll use [chrome-remote-interface](https://github.com/cyrus-and/chrome-remote-interface) as a nice client in front of the protocol:

    npm install chrome-remote-interface

Run Chrome with an open debugging port:

    # linux
    google-chrome --remote-debugging-port=9222

    # mac
    /Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --remote-debugging-port=9222


## Let's profile!

Here's what we're about to…

* Open `http://localhost:8080/perf-test.html`
* Start profiling
* run `startTest();`
* Stop profiling and retrieve the profiling result
* We can then load the data into Chrome DevTools to view

<img src="http://i.imgur.com/zAZa3iU.jpg" height=150>

#### Code
```js
var fs = require('fs');
var Chrome = require('chrome-remote-interface');

Chrome(function (chrome) {
    with (chrome) {
        Page.enable();
        Page.loadEventFired(function () {
            // on load we'll start profiling, kick off the test, and finish
            // alternatively, Profiler.start(), Profiler.stop() are accessible via chrome-remote-interface
            Runtime.evaluate({ "expression": "console.profile(); startTest(); console.profileEnd();" });
        });

        Profiler.enable();
        
        // 100 microsecond JS profiler sampling resolution, (1000 is default)
        Profiler.setSamplingInterval({ 'interval': 100 }, function () {
            Page.navigate({'url': 'http://localhost:8000/perf-test.html'});
        });

        Profiler.consoleProfileFinished(function (params) {
            // CPUProfile object (params.profile) described here:
            //    https://code.google.com/p/chromium/codesearch#chromium/src/third_party/WebKit/Source/devtools/protocol.json&q=protocol.json%20%22CPUProfile%22,&sq=package:chromium

            // Either:
            // 1. process the data however you wish… or,
            // 2. Use the JSON file, open Chrome DevTools, Profiles tab,
            //    select CPU Profile radio button, click `load` and view the
            //    profile data in the full devtools UI.
            var file = 'profile-' + Date.now() + '.cpuprofile';
            var data = JSON.stringify(params.profile, null, 2);
            fs.writeFileSync(file, data);
            console.log('Done! See ' + file);
            close();
        });
    }
}).on('error', function () {
    console.error('Cannot connect to Chrome');
});
```

For asynchronous tests, the code would be:

```js
Runtime.evaluate({ "expression": "console.profile(); startTest(function() { console.profileEnd(); });" });

// ...

function startTest(cb) {
    foo.on('end', cb);
    foo.startAsync();
}
```

This is just the tip of the iceberg when it comes to using [the protocol](https://developer.chrome.com/devtools/docs/debugger-protocol) to manipulate and measure the browser. Plenty of other projects around this space as well: [Chromium Telemetry](http://www.chromium.org/developers/telemetry), [ChromeDriver frontend for WebDriver](https://sites.google.com/a/chromium.org/chromedriver/), [trace-viewer](http://dev.chromium.org/developers/how-tos/trace-event-profiling-tool), the aforementioned [chrome-remote-interface](https://github.com/cyrus-and/chrome-remote-interface) Node API, and a number of [other debugging protocol client applications collected here](https://developer.chrome.com/devtools/docs/debugging-clients).  [browser-perf](https://github.com/axemclion/browser-perf) and its viewer [perfjankie](https://github.com/axemclion/perfjankie) are definitely worth a look as well.

#### Why profile JS like this?

Well, it started because testing the performance of asynchronous code is difficult. Obviously measuring `endTime - startTime` doesn't work. However, using a profiler gives you the insight of how many microseconds the CPU spent within each script, function and its children, making analysis much more sane.

### Contributors
* paul irish
* [@vladikoff](http://github.com/vladikoff)
* [Andrea Cardaci](https://github.com/cyrus-and)
