---
layout: post
title: Advice for STM R&D Assignments
description: Some guidelines that will help you
categories: programming
tags:
- programming
- life
- career
---

##### Introduction & Disclaimer

STM (Starter Mission) is the training program for
new employees of
[Works Applications Co., Ltd.](http://www.worksap.com/)
Each employee is  required to implement
several information systems.
Here are some practical guidelines for you
to program and deliver your systems more effectively.

Here we use a simple leave-application system as an illustration.
It has some basic requirements as below:

* There are two roles in the system: manager and employees.
* Managers should see applications of leaves from employees.
* Managers is able to approve or reject an application.
* The system may have other features such as departments.

But the main feature is the application of leaves.

Disclaimer:
this article does not provide solutions to any assignment in STM.
It never guarantees your success in STM.
The author is not responsible for any failure
because of taking the advice in this article.

##### Maintain a good quality of source code
Perhaps you have not programmed for months before STM and
you want to pass STM as soon as possible.
However these are not excuses to sacrifice your
code quality for faster development.
Believe me that you will probably spend more time
due to poor code quality.

##### Think about the five principles

You may have heard of the SOLID principles,
which are

* [Single responsibility](http://en.wikipedia.org/wiki/Single_responsibility_principle)
* [Open/closed](http://en.wikipedia.org/wiki/Open/closed_principle)
* [Liskov substitution](http://en.wikipedia.org/wiki/Liskov_substitution_principle)
* [Interface segregation](http://en.wikipedia.org/wiki/Interface_segregation_principle)
* [Dependency inversion](http://en.wikipedia.org/wiki/Dependency_inversion_principle)

The first two principles helped me a lot
during the design of my own system.
Ask yourself when writing a class or method:

* Does it perform a single task for one responsibility?
* Is it closed for modification and open for extension?

For example,
combining adding and updating methods into one method called
_apply()_ is not a good idea.
They had better be separated.
Another example is to implement public method cautiously.

Remember: if you strictly follow single responsibility principle,
open/closed principle would be natural for you as well.

##### Controller - the key part between view and model

Each controller should be responsible for only one feature.
It should not be defined by roles in the system.
One controller should be responsible for CRUD
(create, read, update and delete)
of exactly one feature, e.g., _UserController_.

In addition,
input validation and page redirection methods
should be separated from controllers as well.
Some people believe that the input validation logic should be bounded to each entity,
such that the data inside an entity is consistent wherever it is used.

##### Build up UI from components

Break down one page into different components and reuse them.
It saves you lots of time and greatly improves the maintainability of your UI.
[This](http://www.tutorialspoint.com/jsf/jsf_facelets_tags.htm)
simple tutorial will help you decompose your UI.

##### SQL builder (optional)

Write a SQL builder (wrapper) to generate SQL strings.
It does not save you lots of time,
but it improves maintainability of your SQL.

##### Dictionary generator (optional)

If you need to implement a dictionary feature,
consider writing a program to automatically generate dictionary insertion SQL.

##### What is a merit?

You need to be very clear on this principle since the beginning of STM:

**Not every feature is a merit.**

Some features are necessary for daily business operation.
Merits are always related to benefits.
They are those features that generate revenue,
improve efficiency,
enhance security,
ensure the stability of the system and operations.

Therefore when you write a catalog or implement a feature,
always ask yourself these questions:

* What merits does it bring to the user?
* What are the benefits of using my system compared to using just paper and pen?
* What makes my client want to buy my system?

Take the leave-application system for example:

* Does it really help the manager to maintain the leaves of his employees?
* Does it ease the process of applying and maintaining leaves?
* What is the advantage of my system over writing leaves on paper?

##### Is the UI intuitive to use?

Since you have stared at your system for a very long time,
you should be very familiar with it.
But the customer who use your system may not be.
Ask yourself these questions on each page (view):

* Is the user able to see the information presented in a clear and organized manner?
* Does the user know what he is doing and what to do next from the UI?
* Does the design and flow of my system effectively reduces clicks and page navigations?

If your answer is negative or not sure,
consider improving your UI.
The user should be able to use your system by
reading as few instructions as possible.

For example, consider using a
[wizard](http://www.primefaces.org/showcase/ui/panel/wizard.xhtml)
if there is a flow.
Give warning or confirmation before irrecoverable operations,
such as deletion and payment.
Do not purposefully choose fanciful (and complicated) UI component
just because it looks nice.
Simplicity has its value.

##### Robust testing using test cases

Your system is expected to be bug free
because nobody wants to buy a buggy product.

Write down test cases to systematically test your system.
Do not just randomly click around on the screen and
wish bugs will pop out by themselves.

If possible, automate the test, for example:
run unit test for Util classes.

Learn to use debug mode instead of printing messages into console.

##### Clear catalog: show how your system flows

My personal suggestion for your catalog is to
write use cases to demonstrate at least one complete flow of your system.

For example, your catalog should present the following work flow:
manager creates an employee profile ->
employee applies for a leave ->
manager approves that application ->
employee takes the leave ->
employee returns to work later than expected ->
manager updates the employee`s leave record

##### Summary

STM is not meant to fail you.
It is designed to equip you with
essential knowledge and skills for future work.
Please fully use the time and resources during STM and
it will surely benefit you in the days to come.
I wish you all the best.
