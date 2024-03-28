# Preface

> "Software Engineering Is Art Of Compromise" wiki.c2.com

I have seen many projects that grew out of a simple "MVC" structure.
Developers often explain the MVC pattern: "View is HTML templates, Model is an Active Record class (like Eloquent), and one controller to rule them all!"
Well, not just one, but usually all additional logic is implemented in controller classes.
Controllers often contain a lot of code that implements various logic: image uploads, external API calls, database operations, etc.
Sometimes some logic is extracted into "base" controllers to reduce code duplication, but they grow into thousands of lines.
The same problems arise in both medium-sized projects and large portals with millions of daily visitors.

The MVC pattern was invented in the 1970s for graphical user interfaces (GUI).
Simple CRUD (Create, Read, Update, Delete) web applications are essentially just interface to databases, so the reinvented "MVC for web" pattern became very popular.
However, web applications quickly evolve beyond being mere interfaces to databases.
What does the MVC pattern say about working with files (images, music, videos), external APIs, or caching?
What if entities have behaviors that go beyond simple Create-Update-Delete?
The answer is simple: The Model in terms of MVC is not just an Active Record class. It contains all the logic for working with all the application's data.
Over 90% of the code in a modern complex web application is the Model, which includes not only ORM entities but also a large set of other classes.
The creator of the Symfony framework, Fabien Potencier, once wrote: "I don't like MVC because that's not how the web works. Symfony2 is an HTTP framework; it is a Request/Response framework."
The same is true about Laravel and many other frameworks. The web application's task is simple: receive a request and send a response. The more complex the application, the more beneficial it is for developers to understand this and not think about the project in terms of MVC. Web controller classes, just like console command classes, are just entry points into our application. Describing the rest as Model and View would be incorrect for many applications and only makes it difficult to understand.

Frameworks like Laravel contain many RAD (Rapid Application Development) features that enable efficient application development by cutting corners.
They are handy at the "database interface" stage but often become a source of pain as the project evolves.
I have done a lot of refactoring just to remove such features from applications.
All this convenient magic, like "useful" validations a la "quickly go to the database and check if there is such an email in the table" is good. Still, developers must fully understand how they work and when to avoid them.

On the other hand, advice from experienced developers like "your code should be 100% covered by unit tests," "don't use static methods," and "depend only on abstractions" quickly become cargo cults for some projects.
Blindly following them leads to significant time losses.
I have seen an **IUser** interface with more than 50 properties (fields) and a **User: IUser** class with all those properties copied into it (it was a C# project).
I have seen a huge number of abstractions to achieve the required unit test coverage percentage.
Some of these pieces of advice can be misinterpreted, some are only applicable in specific situations, and some have important exceptions.
Developers must understand the problem that a pattern solves, the conditions in which it is applicable, and most importantly, the conditions in which it's better to avoid it.

Projects come in different shapes and sizes.
Certain patterns and practices fit well for some projects but are unnecessary for others.
A wise person once said, "Software development is always a compromise between short-term and long-term productivity."
If I need one functionality in another part of the project, I can simply copy it there.
It will be very productive but will cause problems in the future.
Almost every decision about refactoring or applying a certain pattern presents the same dilemma.
Sometimes, the decision not to apply a pattern that would make the code "better" is correct because the beneficial effect of the pattern is less than the time spent implementing it.
Balancing patterns, practices, techniques, technologies and choosing the most suitable combination for a specific project is the most important skill for a developer/architect.

In this book, I will show the most common problems that arise during the growth of a project and how developers usually solve them.
The reasons behind these solutions are an important part of the book.
