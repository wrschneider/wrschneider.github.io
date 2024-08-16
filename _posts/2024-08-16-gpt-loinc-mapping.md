---
layout: post
title: Experiment with OpenAI for lab mapping to LOINC codes
description: OpenAI did a pretty good first pass at mapping lab descriptions to LOINC codes with minimal effort
---

I did an experiment using Azure OpenAI to map free-text short descriptions of lab tests along with their units ("hemoglobin g/dL") to
industry-standard LOINC codes.  I was surprised to see I got a decent result with minimal effort: somewhere in the ballpark of 
70% match against known results, with only a few hours playing around with the prompt.

Sharing some of my observations and takeaways:

*GPT and LLMs are exciting because they are accessible without writing code.*  The interface is free text natural language, 
so you can start using GPT right away without engineering effort.  I started out with a prompt like:

> You are helpful assistant and an expert in medical coding, and you are helping me map these descriptions of lab tests to standard
> LOINC codes.  The input will be formatted like this... and your output should be formatted like this.... 
> here are some examples: ... 

within a GPT-4o playground, then tweaked the prompt a little bit for a few hours until I seemed to hit a plateau.

I didn't have to write a single line of code just to see if I could get reasonable results.  Of course I'd need code with something like
langchain to integrate into an end-to-end solution.

*GPT makes the same kind of mistakes I could imagine a novice human making.*  Often times when OpenAI picked the wrong LOINC code for
a particular lab, I found that it was in the right ballpark but missing slight variations between similar results.  Some examples of
mistakes were like, not realizing that "in blood" and "in serum" are not interchangeable, or not distinguishing between 
percentages (e.g. lymphocytes per 100 leukocytes, vs. absolute count).

*GPT still hallucinates.*  Sometimes I would see codes returned that weren't even valid LOINC codes.  I didn't investigate enough
to see how this happened.

*A feedback loop providing context from human judgement, helped with other results.*  For example, when I added the correct code for 
"% Neutrophils" to the examples, the next iteration provided the correct code for "% Lymphocytes" as well.

All of this leads me to this conclusion:

*GPT could work for mapping, in a copilot-style workflow.*  GPT works well for a task like code lookups, because it is a
text-oriented pattern matching process at its core.  Take some text, and find a good match against some other body of text, given some
example results as illustration.  A human expert will then have to validate and accept or reject the results, much like with Github
Copilot.

Generative AI will make human experts more efficient and productive, effectively giving them a mega-autocomplete, but AI cannot replace
human experts.  Rather, AI is more like a non-expert that does a web search, eyeballs the results a copy-pastes the most likely 
answer. Thus AI output must be treated like a first draft rather than a final word.