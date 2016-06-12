---
layout: post
title: ASP.NET Core with MVC, React and Webpack -- from the command line
---

[.NET Core](https://www.microsoft.com/net/core#windows) along with [ASP.NET Core](https://docs.asp.net/en/latest/), looks like a promising alternative to build ASP.NET web applications, without a dependency on Windows and Visual Studio.  This squarely addresses  the biggest downsides of ASP.NET -- namely, that it only runs on Windows, and you can't just fire up an application from a command line like you can with Node/Ruby/etc.  

You can still deploy apps to IIS on Windows as usual, but you can also run the new lightweight, `libuv`-based Kestrel server.  The experience of starting up Kestrel from `Program.cs` feels a lot like starting Express from a Node runtime, and indeed, ASP.NET core even uses the same ["middleware"](https://docs.asp.net/en/latest/fundamentals/middleware.html) term.  Editing `project.json` and running `dotnet restore` to retrieve dependencies should also feel familiar for `npm` users.

Depsite the promise, some of the scaffolding for setting up a new project is still dependent on Visual Studio.  It is hard to figure out all the right magic on your own to get an MVC project started.  My first attempt, pieced together from [ASP.NET docs](https://docs.asp.net/en/latest/getting-started.html) and various blog posts [here](https://www.exceptionnotfound.net/the-startup-file-in-asp-net-core-1-0-what-does-it-do/) and [here](http://blog.2mas.xyz/asp-net-5-adding-mvc-to-an-application/), failed.  Although I could figure out how to start the Kestrel server and get MVC routing to work, it took me a while to figure out that the [ASP.NET docs for adding Kestrel to `Program.cs`](https://docs.asp.net/en/latest/getting-started.html) and [startup configuration](https://docs.asp.net/en/latest/fundamentals/startup.html) don't mention explicitly the need to call `SetBasePath` to find static resources and Razor templates -- even the ["working with static files"](https://docs.asp.net/en/latest/fundamentals/static-files.html) document, which describes the relative `wwwwroot` path as the default for static content, does not mention that the base path must be set for `wwwroot` to be found.

Also, without Visual Studio, you have to look up the namespace for each individual configuration class/method call by hand, which is tricky for extension methods.  

In the end, I gave up and used the Visual Studio scaffolding and stripped away the parts I didn't need.  My project [is on github here](https://github.com/wrschneider/aspnet-core-webpack).  I added my controllers and views, and then layered in React components built with Webpack, as I had done previously with Node/Express applications.  As far as ASP.NET is concerned, the Webpack bundles (including my React components) are just another static file.  In development I run `webpack -w` from the another console window while `dotnet run` is serving the ASP.NET app through Kestrel.  

Note that you should also be able to do the same things with Webpack and React on an older version of ASP.NET, and run `webpack -w` while serving an application through IIS or IIS Express.  

Things I didn't get to do yet:

* trying out on another platform (Linux, Mac) - I did `dotnet run` from a cygwin terminal on Windows, without Visual Studio open.  
* finding a Razor template extension to parse the Webpack stats JSON and automatically generate the right SCRIPT tag with the generated bundle filename and hash.  You would want that in production for cache busting.  
* production builds - you would need to add a pre-publish step to the `project.json` to run webpack with a different config file for prod minification etc.  The default already has steps to run Grunt/Gulp tasks and Webpack would be treated similarly.  

