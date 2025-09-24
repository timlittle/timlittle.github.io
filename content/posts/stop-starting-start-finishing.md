+++
date = '2025-08-21T04:57:48Z'
draft = false
title = "Stop Starting, Start Finishing: The Power of Flow in Software and DevOps"
subtitle = "How one-piece flow helps software and DevOps teams deliver faster"
featuredImage = '/images/stop-starting/featured.jpg'
categories = ["DevOps", "Productivity"]
+++

Ever feel like you're busy all day but nothing actually gets finished?

In software development and operations, it is common to be working on multiple projects and tasks at once. An engineer will start a task and get blocked. This could be a code review or waiting for another team to do some work. In this scenario, the engineer would pick up another task from the backlog until the original one is unblocked.

Soon, they're juggling multiple projects. Context switching skyrockets, and Work In Progress(WIP) piles up. This context switching makes it difficult to perform Deep Work on a single project, as another project may become unblocked, and you now have two projects to work on simultaneously.

In [The Phoenix Project](https://www.amazon.co.uk/Phoenix-Project-DevOps-Helping-Business/dp/0988262592), this concept is central to the theme of the problems in the IT operations organisation. Multiple projects are underway, all requiring the same resources. To regain control, they implement a 'change freeze' and then gradually increase the work allowed. At first, this feels counterintuitive, it looks like some people will have nothing to do.

A concept that has stuck with me from that book is the focus on *throughput*, rather than *utilisation*. This links back to one of the original books on Lean Manufacturing, [The Goal](https://www.amazon.co.uk/Goal-Process-Ongoing-Improvement/dp/0566086654). This book focuses on the operation of a factory, similar to our example, which initially started with the concept that all machines need to be 100% utilised.

It soon became clear that the entire system could only go as fast as the slowest component. This is the [Theory of Constraints](https://en.wikipedia.org/wiki/Theory_of_constraints). If instead, you focused on alleviating the constraint, the whole system moves faster.

Just like a factory can only move as fast as its slowest machine, software delivery can only move as fast as its biggest bottleneck, often another team, a review step, or a test environment.

## How can we translate that to operations or software development?

It is common for Software Development teams and DevOps teams to adopt Kanban. In this approach, a board is used to make work visible, and cards are added to the board to indicate work moving through the system. This can be a digital board or a physical one. Typically, a single engineer will work on a single task.

The problem comes from when that task is blocked. As busy professionals, we would be tempted to start a new task. However, we could utilise some more techniques from [Lean Manufacturing](https://en.wikipedia.org/wiki/Lean_manufacturing).

A common one I find myself advocating for is [One Piece Flow](https://www.reliableplant.com/Read/14703/one-piece-flow). In Lean Manufacturing, One Piece Flow means moving one item at a time through each step, instead of batching. In software and operations, this means working on one task until it's finished, rather than juggling multiple half-done tasks.

This follows the [*Stop starting, start finishing*](https://kanbanbooks.com/stop-starting-start-finishing-book/) mentality. Instead of starting new tasks whenever a task is blocked, work to get the work unblocked.

For example, if you're blocked on a pull request, don't immediately grab a new ticket.

Instead, work with the reviewer to unblock it, improve documentation, or prepare tests, so when it moves forward, you can finish it faster. When you are suddenly unblocked, the context switch should be small and enable you to start progressing that task quickly.

If this technique is combined with the kanban board, it can lead to a system that focuses on *throughput*, working to get tasks through the system quickly and deliver value to customers faster.

Not all development or operations work can be done in this manner. If you are on-call, you may be jumping from task to task, trying to figure out the root cause. However, the philosophy can still apply in a looser sense. For example, working the most impactful ticket first, to completion, before moving on to the less impactful ones.

## Conclusion

DevOps work has a tendency to accumulate, especially when tasks are blocked. Starting new work feels productive in the moment, but it creates backlogs and more context switching.

By borrowing from Lean manufacturing, we can shift our focus from utilisation to throughput. **One Piece Flow** and the **Stop Starting, Start Finishing** mindset help engineers deliver faster by finishing what's in motion before starting something new.

What strategies help your team finish more and start less?

---

*Also available on [Medium](https://medium.com/devopsengineering/stop-starting-start-finishing-how-to-improve-devops-flow-with-lean-principles-abc123def456)*
