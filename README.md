# waveform-data.js [![Build Status](https://travis-ci.org/bbc/waveform-data.js.svg?branch=master)](https://travis-ci.org/bbc/waveform-data.js)

[![browser support](https://ci.testling.com/bbcrd/waveform-data.js.png)](https://ci.testling.com/bbcrd/waveform-data.js)

**waveform-data.js** is a JavaScript library for creating __zoomable__,
__browsable__ and __segmentable__ representations of audio waveforms.

**waveform-data.js** is part of a [BBC R&D Browser-based audio waveform visualisation software family](http://waveform.prototyping.bbc.co.uk):

- [audiowaveform](https://github.com/bbc/audiowaveform): C++ program that generates waveform data files from MP3 or WAV format audio.
- [audio_waveform-ruby](https://github.com/bbc/audio_waveform-ruby): A Ruby gem that can read and write waveform data files.
- **waveform-data.js**: JavaScript library that provides access to precomputed waveform data files, or can generate waveform data using the Web Audio API.
- [peaks.js](https://github.com/bbc/peaks.js): JavaScript UI component for interacting with waveforms.

We use these projects daily in applications such as
[BBC Radio Archive](http://worldservice.prototyping.bbc.co.uk) and __browser editing and sharing__ tools for BBC content editors.

![Example of what it helps to build](waveform-example.png)

# Install

## npm

You can use `npm` to install `waveform-data`, both for Node.js or your frontend needs:

```bash
npm install --save waveform-data
```

## Bower Component

If you already use `bower` to manage your frontend dependencies, you can then install `waveform-data` with it:

```bash
bower install --save waveform-data
```

# Usage and Examples

Simply add `waveform-data.min.js` in a `script` tag in your HTML page.
Additional and detailed examples are showcased below and in the [documentation pages](doc/README.md).

```html
<!DOCTYPE html>
<html>
<body>
<script src="/path/to/waveform-data.min.js"></script>
<script>
var waveform = new WaveformData(...);
</script>
</body>
</html>
```

You can use any of`dist/waveform-data.min.js` or `dist/waveform-data.js` files.
They are delivered as UMD module so they can be used as:

- Vanilla JavaScript (available as `window.WaveformData`)
- RequireJS module (available as `define(['WaveformData'], function(WaveformData){ ... })`)
- CommonJS module (available as `var WaveformData = require('waveform-data');`)

## Receiving the data from an AJAX request

```javascript
var xhr = new XMLHttpRequest();

// .dat file generated by audiowaveform program
xhr.responseType = "arraybuffer";
xhr.open("GET", "http://example.com/waveforms/track.dat");
xhr.addEventListener("load", function onResponse(progressEvent){
  var waveform = WaveformData.create(progressEvent.target);

  console.log(waveform.duration);
});
xhr.send();
```

## Drawing in canvas

```javascript
var waveform = WaveformData.create(raw_data);
var interpolateHeight = function interpolateHeightGenerator (total_height){
  var amplitude = 256;
  return function interpolateHeight (size){
    return total_height - ((size + 128) * total_height) / amplitude;
  };
};
var y = interpolateHeight(canvas.height);
var ctx = canvas.getContext();
ctx.beginPath();

// from 0 to 100
waveform.min.forEach(function(val, x){
  ctx.lineTo(x + 0.5, y(val) + 0.5);
});

// then looping back from 100 to 0
waveform.max.reverse().forEach(function(val, x){
  ctx.lineTo((waveform.offset_length - x) + 0.5, y(val) + 0.5);
});

ctx.closePath();
canvas.fillStroke();
```

## Drawing in D3

```javascript
var waveform = WaveformData.create(raw_data);
var layout = d3.select(this).select("svg");
var x = d3.scale.linear();
var y = d3.scale.linear();
var offsetX = 100;

x.domain([0, waveform.adapter.length]).rangeRound([0, 1024]);
y.domain([d3.min(waveform.min), d3.max(waveform.max)]).rangeRound([offsetX, -offsetX]);

var area = d3.svg.area()
  .x(function(d, i){ return x(i) })
  .y0(function(d, i){ return y(waveform.min[i]) })
  .y1(function(d, i){ return y(d) });

graph.select("path")
  .datum(waveform.max)
  .attr("transform", function(){ return "translate(0, "+offsetX+")"; })
  .attr("d", area);
```

## In Node.js

You can use the library to both consume the data on the frontend and emitting them from a Node.js HTTP server, for example.

```javascript
// app.js
var WaveformData = require("waveform-data");
var express = require("express");
var app = express();

// ...

app.get("/waveforms/:id.json", function(req, res){
  var data = require("path/to/"+ req.params.id +".json");

  res.json(data);
});
```

You could even self-consume the data from another application:

```javascript
#!/usr/bin/env node
// app/bin/cli-resampler.js
// called like `./app/bin/cli-resampler.js --wid=1337`

var WaveformData = require("waveform-data");
var request = require("request");
var args = require("optimist").argv;

request.get("http://api.myapp.com/waveforms/"+ arvg.wid +".json", function(err, response, body){
  var resampled_waveform = WaveformData.create(body).resample(2000);

  process.stdout.write(JSON.stringify({ min: resampled_waveform.min, max: resampled_waveform.max }));
});
```


# Data format

The [file format](https://github.com/bbc/audiowaveform/blob/master/doc/DataFormat.md) used and consumed by `WaveformData` is documented as part of the [**audiowaveform** project](http://waveform.prototyping.bbc.co.uk).

We basically have **headers** containing:

 * the `version number` of the data format
 * the `number of bits` used to encode the waveform data points
 * the expected `length` of samples to render
 * the `sample rate` of the original audio file used to compute the data
 * the `samples per pixel` which specifies the time resolution of the waveform data

The body contains a *single range* of *minumum* and *maximum* audio peaks.
Which means if we have a `length` of 100, it means we have *200* elements in the body.

[Waveform Data Format Documentation](https://github.com/bbc/audiowaveform/blob/master/doc/DataFormat.md)

# JavaScript API

This section describes the `WaveformData` API.

## `WaveformData`

This is the main object you use to interact with the waveform data. It helps you to:

* access the whole dataset
* iterate easily on an *offset* (visible subset of data, for example)
* generate one or several **resampled views**, e.g., to display the waveform at different zoom levels
* convert positions (in pixels, in seconds, in the offset)
* create and manage segments of waveform data, e.g., to represent different music tracks, or speakers, etc.

[`WaveformData` API Documentation](doc/WaveformData.md)

## `WaveformDataSegment`

Each segment of data is independent and can overlap other existing ones.
Segments allow you to keep track of portions of sound you would be interested to highlight.

[`WaveformDataSegment` API Documentation](doc/WaveformDataSegment.md)

## `WaveformDataAdapter`

This interface provides a backend abstraction for a `WaveformData` instance.
You should not manipulate this data directly.

[`WaveformDataArrayBufferAdapter` API Documentation](doc/WaveformDataArrayBufferAdapter.md)
[`WaveformDataObjectAdapter` API Documentation](doc/WaveformDataObjectAdapter.md)

# Browser Support

Any browser supporting **ECMAScript 5** will be enough to use the library -
think [`Array.forEach`](http://kangax.github.io/es5-compat-table/#Array.prototype.forEach):

 * IE9+, Firefox Stable, Chrome Stable, Safari 6+ are fully supported;
 * IE10+ is required for the [TypedArray](http://caniuse.com/#feat=typedarrays) Adapter;
 * Firefox 23+ and Webkit/Blink browsers are required for the experimental [Web Audio](http://caniuse.com/#feat=audio-api) Builder.

# Development

To develop the code, install [Node.js](http://nodejs.org/) and [npm](https://npmjs.org/). After obtaining the waveform-data.js source code, run `npm install` to install Node.js package dependencies.

# Credits

This program contains code adapted from [Audacity](http://audacity.sourceforge.net/), used with permission.

# License

See COPYING for details.

# Contributing

Every contribution is welcomed, either it's code, idea or a *merci*!

[Guidelines are provided](CONTRIBUTING.md) and every commit is tested against unit tests
using [Karma runner](http://karma-runner.github.io) and the [Chai assertion library](http://chaijs.com/).

# Authors

This software was written by

* [Thomas Parisot](https://github.com/oncletom), thomas.parisot at bbc.co.uk.

## Copyright

Copyright 2014 British Broadcasting Corporation
