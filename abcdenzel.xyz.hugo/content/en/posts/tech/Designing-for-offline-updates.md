+++
date = '2026-03-16T13:53:01-04:00'
draft = true
title = 'Designing for Offline Updates'
+++

Designing for offline updates has come up as a critical aspect of one of the modules we are developing for our management web application at PBO. Users expect to be able to interact with our application seamlessly, regardless of their network connectivity. Here are some key considerations we had to think about for implementing offline updates effectively in our stack using .NET Core and Entity Framework over SQL Server:

## Offline support

- Operations queue in Local storage
- Seamless offline error handling
- Schema changes to support optimistic concurrency and idempotent changes:
  - SQLServer's RowVersion and Guid SyncId

## Concurrency Control

There are three concurrency control strategies available:

- **Do Nothing**: if concurrent users are modifying the same record, let the last commit win (the default behavior)

- **Optimistic Concurrency**: assume that while there may be concurrency conflicts every now and then, the vast majority of the time such conflicts won't arise; therefore, if a conflict does arise, simply inform the user that their changes can't be saved because another user has modified the same data

- **Pessimistic Concurrency**: assume that concurrency conflicts are commonplace and that users won't tolerate being told their changes weren't saved due to another user's concurrent activity; therefore, when one user starts updating a record, lock it, thereby preventing any other users from editing or deleting that record until the user commits their modifications

## References

- [Concurrency - learnentityframeworkcore](https://www.learnentityframeworkcore.com/concurrency)
- [Mycelial's video on CRDTs](https://www.youtube.com/watch?v=gZP2VUmH05A)
- [Jordan has no life's video on CRDTs](https://www.youtube.com/watch?v=FG5Varj1Ows)
