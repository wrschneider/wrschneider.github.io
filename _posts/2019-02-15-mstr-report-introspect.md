---
layout: post
title: Introspect MicroStrategy report definition with Web SDK
---

Sometimes it can be useful to introspect MSTR report definitions to make
assertions about them for unit tests, or to extract other metadata about their
structure.

I wrote some code that illustrates how to open a report definition
and check whether a particular attribute is included in the report's template
or not:

```java
final String ATTR = "... MSTR attribute GUID ..."
final String reportId = "... mstr report GUID to look at ..."

WebObjectsFactory wof = WebObjectsFactory.getInstance();
WebIServerSession iSession = wof.getIServerSession();
// set iSession server/port/etc.

System.out.println(iSession.getSessionID());

WebReportSource reptSrc = wof.getReportSource();
reptSrc.setExecutionFlags(EnumDSSXMLExecutionFlags.DssXmlExecutionNoAction);
WebReportInstance inst = reptSrc.getNewInstance(reportId);
@SuppressWarnings("unchecked")
Enumeration<WebTemplateAttribute> attrs = inst.getTemplate().getTemplateAttributes().elements();
if (!Collections.list(attrs).stream().anyMatch(attr -> attr.getAttributeInfo().getName().equals(ATTR))) {
  System.out.println("Expected attribute missing");
}

iSession.closeSession();
```

Two tricks here to point out

* `DssXmlExecutionNoAction` was necessary to avoid breaking on prompted reports.
I wanted to have MSTR pull the report definition like going into design mode, but
without attempting to execute the report.  Similarly, I didn't want jobs
to clutter up the I-server if I'm trying to crack open dozens of reports to
make assertions and move on.

* It takes a few extra steps to use MSTR list/array-like objects with Java 8+ closures.
MSTR collections don't obey the standard Java collection APIs but you can often
get an old-school `java.util.Enumeration` out of them.  From there, you can
get to a `List` and then to a stream.  Lots of hoops to jump through, but worth it
to avoid writing the same old `while (!found)` loop yet one more time.
