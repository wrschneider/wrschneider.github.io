---
layout: post
title: Capital Bikeshare app with React and Webpack
---

I put together [this app](https://github.com/wrschneider/capital-bikeshare-near-metro) to try out a few things together:

* React
* Webpack
* Node.js and Express
* Bootstrap
* Deploying to Heroku
* geocoded locations of DC Metrorail stops

It started primarily as an exercise in React and Webpack.  Originally I had planned for this to be completely client-side, with no server, with the browser directly accessing the Capital Bikeshare data over AJAX.  Unfortunately, I ran into CORS issues, and I decided I would need to address that was through some kind of server-side proxy.  As a bonus, I could filter the responses coming back from Capital Bikeshare so the browser response only gets a subset of the 5 closest bike stations.

I used Node and Express for the server components.  This is the best option for a number of reasons:

* it's lightweight
* the server is doing almost no work of its own -- it's only a proxy for Capital Bikeshare's own API
* the bundling of client-side resources through Webpack can be directly integrated into the Node/Express server during development, rather than having a separate console up with `webpack -w`

Deploying to Heroku (see end result [here](https://bikeshare-near-metro.herokuapp.com)) was a byproduct of needing a server runtime to try out and see how this would behave on my phone.  There is still work to be done in mobile optimizaton, for sure.  If I could have done this all browser-side I could have just accessed it through github.  Once you have a Heroku account set up, though, it turns out deploying to Heroku is just a matter of getting the right `scripts` in `package.json` and pushing to an additional remote.  
