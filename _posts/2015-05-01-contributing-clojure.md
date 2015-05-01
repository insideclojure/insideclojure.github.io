---
layout: post
date: 2015-05-01
title: Contributing to Clojure
---

In the spirit of the classic ["How A Bill Becomes a Law"](https://www.youtube.com/watch?v=Otbml6WIQPo), I'd like to give some insight into how a Clojure [JIRA](http://dev.clojure.org/jira/browse/CLJ) ticket becomes a commit in the [Clojure code](https://github.com/clojure/clojure).

## Where do tickets come from?

Clojure (and the associated contrib projects) manage their tickets in a  [JIRA](http://dev.clojure.org/jira/browse/CLJ) issue management system. The particular machine that hosts the system is provided courtesy of <a href="http://contegix.com/">Contegix</a> and we're very thankful for their support of open source software projects. At this point, it's a relatively old version of JIRA and we expect to update it at some point.

Anyone can create an account and file a JIRA issue for Clojure. To create an account, go to the [Signup page](http://dev.clojure.org/jira/secure/Signup!default.jspa), log in, and then use the "Create Issue" button in the upper right. 

*Note: when your account is created you are allowed to create tickets, comment on tickets, and vote on tickets, but not edit their fields after creation. Generally these default permissions work fine and we automatically elevate permissions when someone becomes a contributor (see below). If you need edit permissions for a ticket you just wrote, please ask on the [Clojure mailing list](https://groups.google.com/forum/#!forum/clojure).*

There are three kinds of issues: defects (bugs in the existing code), enhancements (new features or new functionality for existing code), and tasks (which are rarely used).

New tickets will have a number of fields to complete:

* Summary - a concise description of the problem
* Priority - defaults to Major, but usually the development team uses this truly as a priority (not a severity) that may fluctuate during the lifecycle of the ticket. Don't worry about it too much.
* Affects Versions - it's important to specify which version you found the problem in. It's not critical that you check every other possible version, just let us know where you saw it. If you're filing an enhancement for new functionality, just use the latest stable version.
* Environment - if there are important details about your java version, operating system, etc, leave them here.
* Description - this is the place for the real description - more on this in a bit.
* Attachment, Patch - usually don't need any attachments in the initial filing.
* Labels - it's ok to leave this blank and let us apply standard labels but try to use the major component (reader, printing, etc) - DO NOT use "bug", "enhancement", "patch" or things that covered by other existing fields.

The heart of writing a good ticket is of course in the Description and there is a more written about this in [Creating Tickets](http://dev.clojure.org/display/community/Creating+Tickets). Generally we like these descriptions for defects at filing time to include:

* Statement of the problem 
* Example demonstration - show the incorrect behavior and indicate expected behavior if it's not obvious. In 99% of cases, this should be a demonstration at the REPL, NOT a link to a github repo or a gist or anything else.

If you have also worked on a solution, then the description should include:

* Cause - the identified cause of the problem in Clojure
* Approach - how the cause should be fixed and discussion of alternative approaches that were *not* chosen and why
* Patch - name of the "current" patch - it's hard to keep track sometimes!

Often new tickets will not include these sections and they will be added later as people work on the ticket.

Here's an example of what a description might look like (this is not an actual bug of course):

> Adding odd numbers doesn't work. 
> 
> user> (+ 2 2)
> 4
> user> (+ 1 3)
> ClassCastException
> 
> Cause:  Never implemented odd number adding in the Compiler! 
> See the missing branch in FooExpr.
>
> Approach:  Fully implemented the branch for odd numbers to be just like even numbers. 
> Considered just getting rid of addition altogether but I guess people use it.
> 
> Patch: add-odd-3.patch

In general, it's helpful to keep all of these sections as minimal as possible so that later when someone new picks up the ticket, they see exactly the right details to understand the actual/expected behavior and how it's being fixed. Additional information can be added in the comments if you need to provide more background.

## Becoming a Contributor

If you'd like to provide patches or do more work on tickets, you will need to become a Clojure Contributor. Contributors to Clojure are required to jointly assign copyright on their code to Rich Hickey, the creator of Clojure. Having a single copyright owner for the entire project allows it to be defended against legal challenges and also to make later licensing changes without needing to obtain additional permissions from individual contributors. For more on the details, see the [contributors page](http://clojure.org/contributing), the [contributing FAQ](http://dev.clojure.org/display/community/Contributing+FAQ), or the [OCA FAQ](http://oss.oracle.com/oca-faq.pdf) (the Clojure CA is based on the Sun CA which later became the Oracle CA... whew).

To become a contributor, you need to electronically sign the Clojure Contributor Agreement form. Within a few days after that (we batch these up), you will be added as a contributor on the contributors page and you should be able to post on the [clojure-dev](https://groups.google.com/forum/#!forum/clojure-dev) mailing list, and elevated permissions on JIRA and Confluence. Sometimes there are kinks in this process - ask on the mailing lists if something goes awry.

Patches submitted to JIRA without a contributors agreement **will not be reviewed** as we cannot accept them and we may need to write a clean patch.

Speaking of patches .... Clojure and the contrib projects only accept changes via patches attached to JIRA tickets. These projects do not use GitHub issues or accept GitHub pull requests. We realize that this can be jarring for developers used to contributing to other open source projects. 

The pull request model optimizes for ease of contribution. However, Rich has [made the point](https://groups.google.com/forum/#!msg/clojure/jWMaop_eVaQ/3M4gddaXDZoJ) in the past that he prefers to optimize instead for the management/assessment side of the process where his preferred workflow is more efficient working with patches. 

That said, the PR functionality has improved in many ways since these decisions were made and it's possible that things will change in the future. For now though, it's patches - I'll describe the basics below.

## Ticket Workflow

There are a number of ticket "states" defining the ticket workflow, demonstrated in this diagram from the [workflow page](http://dev.clojure.org/display/community/JIRA+workflow):

![Ticket workflow](http://dev.clojure.org/download/attachments/1572892/workflow.png)

As a contributor, the primary place where you get involved is the orange block where patches are being developed.

Tickets generally go through the following processes (these are the colored blocks). As tickets move they land in different states (the rounded corner boxes). Most processes involve some person or group making a decision (the diamonds).

* Triage - screeners review tickets as they arrive and consider the question: "is this a good problem to work on?". If the answer is yes, the ticket is marked as "Triaged" in the Approval field and the ticket moves to the Triaged state. If the answer is no, the ticket is Declined or marked as a duplicate. If the answer is maybe, the ticket is left in the open state. Screeners will often also improve the quality of the ticket at this point - simplifying the description, adding labels, and generally making it easier to consume.
* Vetting - Rich periodically (on the order of months) looks at the Triaged tickets and double-checks the work of the screener, which is nearly always ok. He then marks it as Vetted.
* Release scheduling - Rich then evaluates the vetted tickets and decides which release to target. This is nearly always the "current" release, but occasionally the "next" release. This process is often done at the same time as vetting.

Once a ticket has been through these processes, a screener and Rich have both agreed it's a good problem and have decided when it should be fixed. At this point, things shift to focus on making a patch for the ticket. Sometimes one will come in with the initial ticket, but in the majority of cases, those initial patches are not what ultimately gets applied.

There is a whole page on [developing patches](http://dev.clojure.org/display/community/Developing+Patches) but I'll hit the highlights here. If you want to work on a patch, clone or fork the Clojure github repo and create a branch for the ticket you're working on:

{% highlight bash %}
git clone git@github.com:clojure/clojure.git
cd clojure
git co -b clj-9999
{% endhighlight %}

Then make your changes to address the ticket. You can run the full test suite with `mvn clean test`. You can install a new version of Clojure into your local Maven repository with `mvn install` - this is then accessible for other local builds as a snapshot version. See the [developing patches](http://dev.clojure.org/display/community/Developing+Patches) page for lots more detail.

Once everything is ready, you can create your patch like this:

{% highlight bash %}
git format-patch master --stdout > clj-9999.patch
{% endhighlight %}

Then you can attach that patch to the ticket in work. If I am creating a new version of an existing patch, I often create a fresh branch, `git apply` the old patch, then work from there.

While in development, the processes are:

* Development - the process above where contributors add patches to the ticket
* Screening - screeners review the patch and decide whether it is a good solution to the problem in the ticket. If yes, then it's marked as "Screened". If no, then it's marked as "Incomplete" with comments about what to change.

At this point, the ticket+patch are ready for Rich to review. Periodically (often on Fridays), Rich will review all screened tickets. If the patch is good, it's marked "Ok". If it needs more work, it's marked as "Incomplete" and it goes back into development. Once tickets are marked "Ok", Stu Halloway does a final quick look and applies the patch to the Clojure code base, then closes the ticket.

## More links!

Virtually everything related to the Clojure contribution process (including every link in this post) is linked and categorized on the [contribution page](http://dev.clojure.org/display/community/JIRA+workflow) of the community wiki. For any question, please use that as your starting place! 

## Newbie tickets

That contribution process page includes a link to JIRA reports for every phase in the lifecycle above, plus additional useful reports. One of the reports that might be of interest is a list of tickets tagged as ["newbie" tickets](http://dev.clojure.org/jira/secure/IssueNavigator.jspa?mode=hide&requestId=10617). These tickets are ones that I've found that I think are worth working on, don't have a patch, and seem fairly approachable. Generally there aren't many of these at any given time as patches are supplied by people relatively quickly.

## Voting

Another way to be involved in the ticket process is by voting on existing issues - this helps the core team prioritize which items should be moved forward at a faster clip, due to their interest, impact, etc. The JIRA system has both "vote" and "watch" functionality and we look at them to gauge interest.

You can search for tickets and include the number of votes as a column in the report or sort by them (I use JIRA configured to show these by default). Andy Fingerhut has also created a process that periodically computes a [weighted voting scheme](http://jafingerhut.github.io/clj-ticket-status/CLJ-top-tickets-by-weighted-vote.html). Each person gets 1 "vote" divided across every issue they've voted on, so voting on many things means your vote counts fractionally less on each one.

Both of these voting algorithms has merit and I actually look at both metrics. As we move into the next release it will be time to look at these again as we start to make decisions about what to pull in, so now's a good time to vote!

## Thanks...

Every release contains input from dozens of contributors working on hundreds of issues. Thanks to all the contributors for everything you do!!

If you have questions, please raise them on the [Clojure mailing list](https://groups.google.com/forum/#!forum/clojure) and I'm happy to answer them.



