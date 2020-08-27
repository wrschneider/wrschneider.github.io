---
layout: post
title: My first impression of Go
description: I had to learn Golang to work on some Terraform modules and I found it annoying
---

I recently learned Golang so I could contribute to some Terraform and AWS-related
projects.

On the plus side, Go is simple, so it's easy to learn, and you can pick up a piece of code
and understand what it's doing without too much trouble.  In contrast, my main complaint
about Scala, which I also learned for a specific purpose (Spark), is that Scala is
overloaded with features and syntax oddities so code can be hard to read.
(I'm talking about you, `implicit`.  And why the @#$ are parens *prohibited* for
some zero-argument function calls?)

I like the `interface` concept, and I see it sort of like
static duck typing -- it doesn't matter how the argument was declared, as long as it supports a
particular set of operations.   

The declare-and-assign `:=` operator is also nice (like `var` in Scala),
as is the concept of all methods acting like extension methods, in the sense that
you can add them anywhere even if you didn't define the type yourself.

The thing that bothers me the most about Go is the parameter passing semantics.
I thought this was largely a settled matter in modern
languages: objects are passed as pointers, primitives are passed by value,
and value types like strings should be immutable so they are no different from
primitives.  This behavior is consistent enough that I don't even
think about it going back and forth between Java, Scala, Python and Javascript.

Working with Go was, for me, like stepping back into a time warp 20 years,
to the last time I worked with C/C++ professionally and had to keep `foo`,
`*foo` and `&foo` straight.  And this is particularly annoying with Terraform
providers, where a fair amount of code seems to be devoted to translating `string`
to `*string` for the AWS API and vice versa.

Lack of exceptions is also annoying.  Copy-pasting `return err, nil` everywhere
gets old.  In this specific case, lack of a language feature actually does impact
readability because of the clutter of copy-paste.  Lack of generics haven't
been a big problem for me so far, though.

Also, I'm not sure I'm seeing what makes Go particularly good for this task.  From
what I read, it sounded like Go was created to be like a better C/C++ -- a compiled language suitable for
lower-level applications where performance matters, but with garbage collection and
faster compile times.  Sort of like Java was in the old pre-generics days, but with first-class
functions, and compiled to native executables.  Go would be a great choice if I was going
to build the next Apache, Nginx, etc.  

On the other hand, the limiting factor
for performance with Terraform is not how fast the Terraform executable itself runs, but how quickly
all the AWS API calls come back.  I can't honestly understand why Python or Java or Node.js wouldn't
have worked equally well for Terraform, besides the convenience of distributing a static-linked
executable.

Maybe I missed something, though?
