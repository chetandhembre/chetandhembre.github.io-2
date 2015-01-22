---
layout: post
title: "How to create dynamic graph on server side?"
date: 2015-01-23 22:35:21 +0530
comments: true
categories:
- phantomjs
- node.js
- stream
- graph
- svg
---

One fine weekend I asked *Sam* one question "How to create dynamic graph on server side?". Question made *Sam*
think little but after sometime *Sam* came back with answer as usual and said "why not use **phantom.js** here ?"..After
noticing my confused face *Sam* started to explain me solution step by step, here is how we solve this problem,

1.Create simple line chart in browser

Creating line chart was simple task. We selected [razorflow](https://razorflow.com/features/) as our charting library
(you can select library of your choice) and we came up with very simple line chart, below is simple code snippet to do it,

```html

//filename : line_graph.html
<script type="text/javascript">
  function chart1(data) {
    StandaloneDashboard(function(db){
      var chart = new ChartComponent(data['caption']);
      chart.setDimensions (7, 7);
      chart.setCaption(data['caption']);
      chart.setLabels (data['labels']);
      chart.addSeries (data["series"]["id"], data["series"]["name"], data["series"]["seriesData"], {
        seriesDisplayType: "line"
      });
      chart.setYAxis(data["yaxis"]["label"], {numberPrefix: data["yaxis"]["numberPrefix"], numberHumanize: true});
      db.addComponent (chart);
    });
  }
</script>

```

Here is full version of above code with output <a class="jsbin-embed" href="http://jsbin.com/laxoka/4/embed?html,output">JS Bin</a><script src="http://static.jsbin.com/js/embed.js"></script>.

We are storing data into ```window.data``` so that we can give it to *razorflow* to render graph. And we done rendering line chart in browser. Most of above code is chart library related.

2.Render Graph using *PhantomJS* on server side and create PNG image

We are using awesome [PhantomJS](http://phantomjs.org/) library to get headless browser at server side. You can install *PhantomJS* on your machine by following instructions [here](http://phantomjs.org/download.html).
But we are not installing *phantomjs* manually, we are using awesome *[phantomjs](https://www.npmjs.com/package/phantomjs)* module from *[npm](https://npmjs.com)*,
Here is code snippet which render graph and create PNG of it

```js
  //filename : phantomjs_render.js
  var page = require('webpage').create();

  page.onLoadFinished   = function(status) {
    if (status === 'success') {
        setTimeout(function () {
          page.render('./static_graph.png')
        }, 0)
        phantom.exit();
    } else {
      phantom.exit();
    }
  };  

  page.open('line_graph.html', function () {
  });
```
lets run above code using following command

```sh
$ phantomjs ./phantomjs_render.js
```

Lets assume that above html code is in ``line_graph.html`` file. We are using  *phantomJS* to open html file (or url) in headless browser
and on loading finish it will create png and store it in ``statis_graph.png``. We are using [render](http://phantomjs.org/api/webpage/method/render.html) api which help
us to create png.

And remember to exit phantomjs browser using ```phantom.exit()``` otherwise your process wont exit.

But remember we want to create dynamic graph images on server side ... so we solve 50% of problem. let solve remaining 50%.

3.Create dynamic graphs

Lets continue with our current example. We are able to generate static graph image.
To make graph true-ly  dynamic we have to some how send data dynamically to ```line_graph.html```. We did modify our phantomjs_render.js file to accomplish this goal.
code snippet is below.

```js
  //filename : phantomjs_render.js
  var page = require('webpage').create();
  var system = require('system');
  var args = system.args;
  var data = JSON.parse(system.args[1])


  page.onLoadFinished   = function(status) {
    if (status === 'success') {
      page.evaluate(function(d) {
        window.chart(d)
      }, data);

      setTimeout(function () {
        page.render('./static_graph.png')
      }, 0)
      phantom.exit();
    } else {
      phantom.exit();
    }
  };  

  page.open('line_graph.html', function () {});
```

We have to run following command to run above code in terminal

```sh
$ phantomjs ./phantomjs_render.js  stringified_data
```

Remember we can not have command arguemnt as json so we have to convert json data to string using ```JSON.stringify```.

We are using in build ```system``` from phantomjs to get argument list provided to *phantomjs* command.

Here we are sending our data as second argument to **phantomjs** command which we can get in our javascript code from ```line_graph.html``` using *[system.args](http://phantomjs.org/api/system/property/args.html)* api.

Another interesting api we are using is *[page.evaluate](http://phantomjs.org/api/webpage/method/evaluate.html)* which help us to run any javascript code in context of webpage in phatomjs
browser. On ```onLoadFinished``` event on page we are calling ``chart`` method and pass our dynamic data to webpage which can be use to create line chart.

4.No!! No command line stuff

I exclaimed !! Sam replied calmly with "Why not use ```child_process``` from node to run above code?", and with that we solve another problem. Here is code how to do it,

```js
//filename : phantomjs_test.js

  var phantomjs = require('phantomjs')
    , path = require('path')
    , childProcess = require('child_process')

  var binPath = phantomjs.path;


  var createImage = function (data, url /*may be file path*/) {
    var childArgs = [
      path.join(__dirname, 'phantomjs_render.js'),
      JSON.stringify(data),
    ];


    var spawn = childProcess.spawn
    spawn(binPath, childArgs)
  }

```
run above code using following command

```sh
node phantomjs_test.js
```

so here we are spawning new child process with  ```phantomjs_render.js``` and  stringify data as arguments. We can also pass other parameters
dynamically. Thats it !! problem has been solved, Isn't it simple ? :)  

5.Advance Usage

I asked this question to Sam because i want to send these dynamically generated graph image in email. Email or preciously email client
has very poor support for SVG and embedded image. So you have to use ```img``` tag. Either you can put these dynamically generated
image to ```s3``` or generate dynamically when request come for it. You can find advance code for it [here](https://github.com/chetandhembre/blog/tree/master/dynamic_graph_image_phantomjs).
