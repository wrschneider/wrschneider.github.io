---
layout: post
title: Combining coverage from unit and functional tests
---
You can combine test coverage reports from both unit and end-to-end functional (e.g., Selenium) tests.  

So why would you want to do that?

A combined coverage report across all forms of testing drives accountability across the whole team for functional tests.  Otherwise, if only unit tests count towards coverage, a development team will tend to do one or both of the following:

* ignore functional testing altogether - it's "not my problem" (tm) 
* struggle with mocks and stubs to write tests of controller classes, and the resulting tests won't provide much value

If, on the other hand, a development team sees functional tests contributing to code coverage, such that writing a functional test means not needing to write a unit test, functional tests and unit tests can be on more equal footing.  Then the whole team work together and pick the kind of testing most appropriate to the task at hand. I define "most appropriate" as, the tests are covering some meaningful behavior (given an input, I got the right result) with the least effort.  A good combination might be using functional tests for a pass through the UI, combined with unit tests or back-end integration tests to go deeper on business logic or database query results. 
 
*How does it work?*

I tried this with a simple ASP.NET application on my workstation, using Visual Studio 2013 Professional and OpenCover.

For this example I will focus on a single controller.

```csharp
namespace BillTest.Controllers
{
    public class HomeController : Controller
    {
        private BillContext db = new BillContext();
        public ActionResult Index()
        {
            Debug.WriteLine("Hey!");
            return View();
        }
        public ActionResult About()
        {
            ViewBag.Message = "Your application description page.";
            Debug.WriteLine("Hey!");
            return View();
        }
        public ActionResult Contact()
        {
            ViewBag.Message = "Your contact page.";
            Debug.WriteLine("Hey!");
            return View();
        }
        public ActionResult Bill()
        {
            ViewBag.Junk = db.BillTestEntities.Where(x => x.ID % 2 == 0).ToList();
            return View();
        }
        public ActionResult Foobar()
        {
            var data = db.BillTestEntities.Where(x => x.ID % 2 != 0).ToList();
            return Json(new { status = "OK", results = data }, JsonRequestBehavior.AllowGet);
        }
    }
}
```

I created two separate test projects - one with unit tests and one with a Selenium test.  Having two separate projects is convenient to isolate the groups of tests, but not strictly required. 
 The unit test will only cover the Index, About and Contact methods. I'm using Microsoft's unit testing framework, but NUnit would provide the same outcome.
 
```csharp
namespace BillTest.Tests.Controllers
{
    [TestClass]
    public class HomeControllerTest
    {
        [TestMethod]
        public void Index()
        {
            // Arrange
            HomeController controller = new HomeController();
            // Act
            ViewResult result = controller.Index() as ViewResult;
            // Assert
            Assert.IsNotNull(result);
        }
        [TestMethod]
        public void About()
        {
            // Arrange
            HomeController controller = new HomeController();
            // Act
            ViewResult result = controller.About() as ViewResult;
            // Assert
            Assert.AreEqual("Your application description page.", result.ViewBag.Message);
        }
        [TestMethod]
        public void Contact()
        {
            // Arrange
            HomeController controller = new HomeController();
            // Act
            ViewResult result = controller.Contact() as ViewResult;
            // Assert
            Assert.IsNotNull(result);
        }
    }
}
```

To get a coverage report with OpenCover, you would go into the path for your test project

```
cd C:\_workspaces\Demo\Demo.Tests\bin\Debug   # your DLLs and PDB files are here
OpenCover.Console.exe -target:VSTest.Console.exe -targetargs:Demo.Tests.dll -register:user
```

At this point you will have a file named results.xml in your directory.  That has the coverage from your unit tests.  You can use the ReportGenerator tool to turn that into an HTML report:

```
ReportGenerator.exe -reports:results.xml -targetdir:coverage
```

And this will show that only the unit tests were covered:

![coverage report from unit tests](/assets/images/combining-coverage-unit-only.png)

Now, to get coverage from your functional test, you need to launch IIS or IIS Express with OpenCover.  I am using iisexpress.exe on my workstation to be consistent with what Visual Studio does.

```
cd C:\_workspaces\Demo\Demo\bin\   # your DLLs and PDB files are here
OpenCover.Console.exe "-target:iisexpress.exe" -targetargs:/site:Demo  -register:user
    # use the same site as you use to launch in Visual Studio
```

Now run your Selenium tests, or just hit the site by hand.  When you shut down IIS you will get another results.xml file that only covers your functional tests.  My functional test hit the "/Home/Bill" URL, which covers the Bill method and indirectly the Foobar method, which is called via AJAX from the view.  

![coverage report from functional tests](/assets/images/combining-coverage-functional-only.png){: width="80px"}

Once you have both results.xml files, ReportGenerator can combine them into a single report:
```
ReportGenerator.exe -reports:Demo\bin\results.xml;Demo.Tests\bin\debug\results.xml -targetdir:coverage
```
Now you can see all methods are covered - some only from unit tests, some only from functional tests:

![coverage report from all tests combined](/assets/images/combining-coverage-both.png)