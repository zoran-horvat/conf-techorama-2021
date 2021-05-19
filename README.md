# Techorama 2021: Securing Multitenant Databases with Entity Framework Core

Demos were developed using Visual Studio 2019. These are prerequisites to build and run them:
  - .NET Core 3.1
  - Database server - By default, application will connect to MS SQL LocalDB provider; if other provider is to be used, please modify connection string in [Startup.cs](https://github.com/zoran-horvat/conf-techorama-2021/blob/master/01%20Initial/Demo/Startup.cs "Startup.cs")

Before running demos, please refresh the database by dropping it and updating again:

    Drop-Database -context EntireContext
    Update-Database -context EntireContext

This will ensure that all seed data are present which are required to demonstrate security concepts.

## Q&A

**Q:** For the write security, why not just override `SaveChanges` and inspect the `ChangeTracker`? In this way you can see any operations update or delete that are not allowed and throw a security error before anything is committed. You can also add an owner on insert.

**A:** You are right. There was even a version of this presentation with that done, but unfortunately I had to cut it out to fit 60m slot. On the other hand, catching forbidden operations early (i.e. before `SaveChanges`) gives you access to call stack. Call stack is important in this case, because any call to Add or Remove with unauthenticated object is indicative of a bug. There appears that `ChangeTracker` should be used to verify updates, but modified objects must have come through either a query or an Add, and therefore they must have already passed verification.

&nbsp;

**Q:** What happens in more complicated queries where you combine multiple entities in a query and all of them have query filters. Will it create a lot of overheads in the filter clause?

**A:** All query filters will add complexity to the SQL statement, sometimes causing slowdown. However, keep in mind that all query filters, either EF query filters or their custom substitutes, are there because application requirements asked for them. Therefore, it is often impossible to remove them (e.g. to postpone verification on materialized objects) without risking security issues later.

&nbsp;

**Q:** What would you recommend using in a URL instead of an auto incremented ID then? A GUID, or something else?

**A:** GUID is useful, though a bit cumbersome due to its length. It is possible to generate secure character strings of smaller length which are still unique. On top of that, custom character strings are easy to read.

&nbsp;

**Q:** What is the advantage of using this wrapper approach around the `DbContext`, instead of implementing the repository pattern?

**A:** Wrapper presented in this talk is not changing the way we perceive a `DbContext`. All interfaces are simply repeating method names and signatures already present in the `DbContext`. From that point of view, there is no substantial difference between this approach and implementing Repository and Unit of Work patterns. Still, it must be emphasized that the two wrappers presented here are producing queries with different structure (use of `IN` operator, as opposed to using plain = operator). Therefore, even with repository implemented, it could turn out impossible to implement all requirements with only a single repository implementation.

&nbsp;

**Q:** `IContentReadContext` exposes an `IQueryable`, not a `DbSet`, hence the `Include()` method of `DbSet` (for eager loading relations) is not available anymore. How to solve this?

**A:** This talk is implicitly applying concepts of DDD. It is a pity that I forgot to mention that. In that light, I expect a `DbContext` to expose full aggregates, not just entities. In my designs that works well. Not only that there is no measurable performance penalty, but it makes consuming code uniform and, not least important, simple. If there is still need to use a single `DbContext` with different Include clauses, I may suggest exposing several property getters for the affected entity, each hiding a different combination of Includes behind.

&nbsp;

**Q:** Aren't these kind of situation supposed to be solved in the business service? A combination of role/claims checking with a business check whether or not an action is valid.

**A:** Demo is making decisions in Razor pages only because there is a limitation to amount of code that can be shown in a 60m presentation. In a proper, large application, business services would do what pages did in the demo application. However, that does not mean that infrastructure layer is relieved from its part of work. In any application, it is of critical importance to restrict number of rows returned from queries, or otherwise database would have to serve too much data, only for them to be discarded after objects have been materialized. Therefore, the right answer is that infrastructure layer must include security restrictions. Business services may or may not repeat the checks where necessary (and possible).

&nbsp;

**Q:** What are the advantages over using JWT for authentication?

**A:** One should separate the question of authentication from the question of authorization. Speaking of authentication alone, this demo is using cookie authentication only for the sake of simplicity. On the other hand, authorization can be set up in several ways today, like using Azure AD and similar services, in addition to custom authorization as it was done in this demo.