---
layout: post
title: Using MicroStrategy reports as filters for exists and not exists subqueries
---

Suppose you want to write a report in MSTR to calculate some measure, filtering on patients who had a certain diagnosis and are *not* on a certain medication.
Both Diagnosis and Medication have many-to-many relationships with Patient, and patient count is a metric defined at or below the patient level.

(I'm using the healthcare domain to describe the problem but it applies to all domains where there are multiple relationships to some core entity: students who registered for a course and didn't take one of the prerequisites; customers who ordered a product and have no unresolved support tickets, etc.)

If you do the immediate, obvious thing – create an attribute qualification on Medication = M and Diagnosis <> D, you will get the wrong answer.  This will result in a query equivalent to this:

```
select ...
from fact
join patient on fact
join medication on patient
join diagnosis on patient
where medication = M and diagnosis <> D
```

This is wrong for two reasons: diagnosis doesn't return a single row per patient even with the filter applied; and if a patient has any other diagnosis that is not D, it will still match the condition.

In straight SQL the solution is to use EXISTS / NOT EXISTS subqueries:

```
select ...
from fact
join patient on fact
where
EXISTS (select 1 from medication where medication = M and patient = medication.patient)
AND NOT EXISTS (select 1 from diagnosis where diagnosis = D and patient = diagnosis.patient)
```

The way to get this kind of exists/not exists behavior in MSTR is making separate reports for each subquery and then using the report result as a filter.

You would make one report with Patients on the template and Medication = M on the report filter, and another with Patients on the template and Diagnosis = D on the filter.  Each report would return a list of patients which can then be used as a filter for the main report.

Then you would incorporate these reports in your main report's filter: "(Patients with M report) AND NOT (Patients with D report)"

This will give you the behavior you want.  

Note that you have to structure the diagnosis report as matching patients with Diagnosis = D and then including the output of that with "AND NOT" for the reason described above: you need to exclude patients that have any diagnosis = D, even if there are other diagnosis codes.