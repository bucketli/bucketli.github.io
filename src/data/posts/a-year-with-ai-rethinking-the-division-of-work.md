---
title: How Coding with AI Changed What I Value in Work—and in People
description: After a year of coding with AI, execution is easier—but judgment, context, prioritization, and knowing when to stop matter more than ever.
publishedAt: 2026-10-23
draft: false
---

Recently, I have been improving PostgreSQL DDL synchronization in BladePipe. Put simply, when someone adds a column, changes a default value, or creates an index in the source database, the target database needs the corresponding change. In this round of testing, I focused on PostgreSQL → MySQL.

The work spanned several projects: the parser, schema conversion, data synchronization, and finally tests and documentation. AI helped me explore the code, work through possible approaches, change the implementation, and add tests. I ran SQL against the databases, then brought the results and logs back into the discussion.

I could have let AI execute all the SQL, run the checks, and fix the failures automatically. I chose not to for two reasons: I wanted to understand where the problems were, and some fixes simply were not worth making. AI does not necessarily know that. :)

There was a lot of back and forth. We fixed some problems as soon as we found them. We implemented some approaches and then deleted them. We decided to leave other problems alone for now.

What took the most effort on my part was deciding where to go next—and when to stop.

## Start by Filling the Gaps in the Tests

At first, I did not ask AI to start testing the synchronization pipeline. Instead, I asked it to fill the gaps in the PostgreSQL statement-splitting tests in the parser project.

A robust parser is the foundation of DDL synchronization. It also helps narrow down failures in this scenario to differences between the databases themselves.

We added tests by category: constraints, partitions, sequences, types, functions, triggers, indexes, and so on. Add a round of tests, fix a round of failures. We also tested edge cases separately: semicolons inside strings, function bodies, and nested comments. This kind of work suits AI well. It can quickly map out syntax variations and trace failing tests back to the relevant code.

But when we later prepared tests for actual DDL synchronization, I made another explicit request: do not use the parser's existing DDL implementation as the basis for generating test cases.

Our product focuses on synchronizing data across different database systems, and those systems vary widely in how they handle schema changes. So I wanted the tests organized around two things: operations that affect data, and common business scenarios. That would help keep the scope and complexity of DDL conversion under control.

Neither instruction was technically complicated. Both shaped the order and quality of the work that followed.

## A Column Comment Leads to a Discussion About Schema State

During testing, I ran a statement like this in PostgreSQL:

```sql
COMMENT ON COLUMN orders.note IS 'Order notes';
```

It is a simple statement, but it does not translate directly to MySQL.

Suppose the target column is defined as `varchar(80) NOT NULL DEFAULT 'base'`. To change its comment, we generally need to restate those attributes:

```sql
ALTER TABLE orders
MODIFY COLUMN note varchar(80)
NOT NULL DEFAULT 'base'
COMMENT 'Order notes';
```

The original PostgreSQL statement does not contain that information.

A natural next step would be to query the target's metadata and fill in the missing definition. But we did not have the infrastructure to query the target schema at that point in the synchronization process, and implementing and validating it would have been a substantial amount of work. I chose to leave this unsupported for now. Users could work around it by changing the schema directly on the target.

Changing a column's type or nullability leaves similar gaps in the information available. We did not try to fill those gaps in this round. Setting and dropping default values, on the other hand, have corresponding statements that change just that attribute, so we added support for those.

AI can help us explore an approach and even produce the code very quickly. But whether it is worth pursuing, and whether it belongs in the current scope, depends on the goal. The fact that code can be generated does not mean the resulting solution is worth maintaining.

I also supply context about earlier implementations and how people use the product: that tasks subscribe to an explicit allowlist of tables, which settings remain for compatibility with older tasks, or why a particular conversion has always worked a certain way. That background may not be visible in a few snippets of current code, but it can directly affect the design.

## Please Undo the Code We Just Added

The `USING` clause is one example.

PostgreSQL lets you specify a conversion expression when changing a column's type:

```sql
ALTER TABLE orders
ALTER COLUMN amount TYPE bigint
USING amount::bigint + 100;
```

This statement changes the existing data too. If the target only changes the column type without adding 100 to the values, the synchronized result is still wrong.

We discussed a more comprehensive way to represent this, but I later asked AI to narrow the scope: recognize only a direct cast of the same column to the declared target type, skip complex calculations for now, and remove the extra implementation from the previous round.

This narrower scope mainly addresses extracting schema changes from the source, along with routes such as PostgreSQL → PostgreSQL, where the target has matching capabilities. Whether a type change can actually run on a PostgreSQL → MySQL route still depends on the target conversion logic. Recognizing the source statement is not enough to let it through.

Another example involved rolling back to a savepoint within a request containing multiple SQL statements. Initially, I ran a transaction in psql and saw DDL statements arrive one by one in the logs. I thought the statement-splitting scenario might not arise in practice. Later, I sent the entire SQL block in a single request and found that it did cause mismatches and missed schema changes.

The actual results corrected my judgment.

Once we confirmed the problem, we investigated possible fixes. I ultimately decided to leave the capture mechanism unchanged for now, log the situation clearly, and document the edge case. The problem was real; I just was not going to expand the scope of the changes further at that point.

Using AI has made me more conscious of these trade-offs. It proposes solutions and writes code so quickly that it is easy to think, “We might as well support that too.” But someone will still have to understand and maintain the additional branches, state, and compatibility logic later.

I now often ask myself: how would I have prioritized this problem without AI? Faster implementation today should not make me forget the long-term cost.

## Where Past Experience Helps

While testing indexes and constraints, we ran into another kind of problem.

Suppose the source drops an index:

```sql
DROP INDEX cc_ddl_pg17.ddl_indexes_code_idx;
```

The SQL contains only the schema and index name, with no table name. Our current capture data did not record the owning table either. By the time the index had been dropped, it was too late to look it up.

Names introduce another problem when migrating between database systems. Different databases have different naming rules for indexes and constraints. The initial migration may already have renamed them to avoid conflicts. If the source later drops an object using its original name, that name may no longer map directly to the target.

If you focus only on “How do we translate this SQL statement?”, it is easy to miss those dependencies on what happened before.

I bring experience from earlier schema migrations and customer issues into these discussions. AI also points out situations I have missed, and we check them against the code and test results. Experience helps, but it needs to be tested too. The psql example above is a case in point.

We also reversed course once while writing the documentation. AI initially wrote a thorough explanation of the reasons behind every DDL limitation. After reading it, I felt that level of detail would work in a technical article, but was too much for the documentation for a synchronization route.

Users visiting that page primarily want to know what is unsupported. So I asked AI to consolidate it into a single “DDL Synchronization Limitations” section and list the operations directly underneath. When we found that the capture script was missing `COMMENT`, we added it to the script without adding several paragraphs of explanation.

None of these were major technical decisions. But this is how a product takes shape, one small decision at a time. More content and more features are not always what the work needs.

## This Has Also Changed How I Evaluate People

After this round of work, I still see AI primarily as a more efficient tool for getting things done. It can take on a great deal of information retrieval and synthesis, work out the details of an approach, and implement the code.

It can also contribute ideas and suggest new directions. But someone still has to take responsibility for what we adopt, what we drop, and how we verify the result.

What worries me is that a person's role might gradually shrink to saying “keep going.” Fix every error, add a branch for every new situation, keep changing the code—without anyone stepping back to check whether the original goal has been met.

If AI is doing everything it can while the person contributes no judgment of their own, the product may spend its life patching holes. Each iteration seems to move it forward a little, yet people keep running into problems when they actually use it.

Being fast at execution and knowing your tools used to provide a clear advantage. As more of that work becomes easy to delegate to AI, execution speed alone does less to distinguish people. Understanding the problem, deciding which requirements matter, and weighing risks become more important.

So when I read a résumé now, whether it is from a new graduate or an experienced hire, I want to understand more than how they executed a project.

We still need to talk about technology. A person's technical understanding tells me something about how they learn. Have they really absorbed what they have encountered? Can they connect different pieces of knowledge? How do they make sense of an unfamiliar problem? Thinking needs material to work with. Without enough input, it is hard to come up with good questions out of nowhere.

When discussing work experience, I want to hear about specific choices. Why did you build this project? Which requirements did you decide against? Was there an approach you abandoned halfway through? Once people actually used it, which ideas turned out to be wrong? If the project failed, how did you come to understand that failure?

These details tell me more about a person than a list of frameworks and features. A project that did not succeed can still give us plenty to discuss if the person has thought carefully about why.

I am also interested in life outside work. Is there an activity you have stayed with and explored more deeply over time? What has recently caught your interest? Have you tried something you had never done before? It does not have to be unusual. I just want to hear your own observations and how you felt about the experience.

I want to work with people who understand technology, have their own thoughts about their projects, and take an interest in life. Conversations with those people often bring up perspectives I had not considered.

AI can already help us do a great deal. The longer I use it, the more I feel that my own involvement cannot diminish: provide more context, ask one more question, change course when something is wrong, and stop when something is not worth pursuing.

Ultimately, I still want to meet an interesting person—someone who develops their own ideas from experience and remains willing to engage with and understand the world.
