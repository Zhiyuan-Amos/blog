---
title: Database Concurrency Summarised
tags: [concurrency, entityframeworkcore]
readtime: true
---

For starters, [synchronization primitives](https://docs.microsoft.com/en-us/dotnet/standard/threading/overview-of-synchronization-primitives) generally should not be used for Database Concurrency because they only synchronize **within** the Application Server, while generally there's more than 1 instance of Application Server running at any point in time due to the need to scale horizontally.

I will be highlighting 2 ways of implementing Database Concurrency using Entity Framework Core (EF Core).

### Optimistic Concurrency Control (OCC)

The key feature of OCC is the usage of [concurrency tokens](https://docs.microsoft.com/en-us/ef/core/modeling/concurrency) to detect concurrency conflicts:

1. A property that uniquely identifies the state of the model is specified to be the concurrency token. For example, a hash of all the model's values (it is however, simpler to use something like timestamp or version number).

1. Whenever an update or delete operation is performed, EF Core compares the value of the concurrency token on the database against the original value read by EF Core. If the values don't match (because another operation has modified the same row), EF Core aborts the operation.

What happens under the hood can be found [here](https://docs.microsoft.com/en-us/ef/core/saving/concurrency#how-concurrency-control-works-in-ef-core).

Implementing this is trivial as it only involves specifying a property to be the concurrency token by adding the `[ConcurrencyCheck]` attribute, though it involves more code if you are [handling the concurrency conflict](https://docs.microsoft.com/en-us/ef/core/saving/concurrency#resolving-concurrency-conflicts).

### Pessimistic Concurrency Control (PCC)

Also otherwise known as database locking, this implementation works similar to synchronisation primitives. It is however, more complex than using a mutex or semaphore (see [code sample](https://docs.microsoft.com/en-us/ef/core/saving/transactions#using-systemtransactions)). Quite a bit of the complexity comes from specifying the [transaction's isolation level](https://docs.microsoft.com/en-us/dotnet/api/system.transactions.isolationlevel), which determines the degree of read-write access allowed by other transactions.

Care has to be taken when implementing PCC, because:

1. Specifying a higher-than-required isolation level incurs performance cost, while specifying a lower-than-required isolation level can lead to concurrency conflicts.

1. Locking additional tables and rows unnecessarily incurs performance cost.

## OCC > PCC?

While OCC is simpler, easier to implement, and has lesser pitfalls compared to PCC, it doesn't therefore mean that OCC is the superior or default option for Database Concurrency. Both have scenarios that they are better suited for, and here are the considerations:

1. Dependencies on Other Tables

    Suppose the following application logic:

    1. A Course can have multiple Students
    1. Course and Student are separate database tables
    1. A User can add Students into the Course
    1. However, when a Course is set to the completed state, User can no longer add Students into the Course

    So, `AddStudent()`'s logic looks like:

    ```cs
    if (!course.IsCompleted)
    {
        // Adds student
    }
    ```

    The following sequence of events could happen:

    1. `AddStudent()` -> Checks that Course is not in completed state
    1. `SetCompleted()` -> Sets Course to completed state
    1. `AddStudent()` -> Adds Student

    As OCC does not verify that the Course isn't modified between steps 1 and 3 (also known as TOCTOU), a concurrency conflict occurs.

    Using PCC, you can lock the Student table and the corresponding Course row prior to calling `AddStudent()`, which prevents the above sequence of events from occurring.

1. Number of Database Table Rows Required

    Suppose OCC is implemented for booking meeting rooms, where the timing of the booking can be specified to the minute. As OCC requires database rows to already exist, that means 1440 rows (24 hours * 60 minutes) have to be created per day per meeting room, which can be space-inefficient.

    Contrast this with the PCC's implementation, where the number of database rows created per day per meeting room = number of bookings made on a particular day for that particular meeting room, which would reasonably be much lesser than 1440 rows.

    OCC is more suited for the scenario of purchasing movie tickets, where each movie screening is expected to be fully booked. So, the number of database rows created to implement OCC or PCC is similar.

1. Chance of Conflict Occurrence

    From a [StackOverflow answer](https://stackoverflow.com/a/129380/8828382):

    > Optimistic locking is used when you don't expect many collisions. It costs less to do a normal operation [than PCC] but if the collision DOES occur you would pay a higher price to resolve it as the transaction is aborted.

    That is, when the number of collisions are large, the cost of rolling back transactions becomes more than the cost of blocking concurrent operations by locking.

    So, for example, an Administrator updating a User's details is highly unlikely to cause a concurrency conflict, so OCC should be used.

    Whereas purchasing a highly popular and limited edition item online is highly likely to cause concurrency conflicts (suppose a database row stores the quantity of item remaining, and the application verifies that this value is > 0 to ensure that the purchase is successful), so PCC should be used instead.
