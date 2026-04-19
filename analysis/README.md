# Prompts

## synchronisation.md

Identify the use of synchronisation objects such as locks, semaphores or mutexes.

For each individual sync object we want to know:
*  Its name and where it is defined
*  Whether it is currently best practice to use this kind of synchronisation object or whether it is superceded
*  What data structure or structures it protects in this code base
*  When it should be invoked for proper synchronisation
*  When it should not be invoked because it is not needed
*  How it should be used for best multi process performance

Write your analysis to analysis/synchronisation.md

In this code base there may be cases where synchronisation objects are improperly used or not used where they should be.  The catalogue you are building will help identify these cases in a later step.

## synchronisation-recommendations.md

Using the code and the results in analysis/synchronisation.md, find code that does not perform locking when it should (e.g. accesses protected objects unsafely), or that performs locking that exceeds
requirements (is not needed or includes heavier work).

Create prioritised recommendations, prioritising higher performance/resilience impact and simpler fixes, for opportunities to resolve:
*  Missing locking.
*  Locking that is held too long or that is unnecessary.
*  Race conditions.
*  Sleeping or spin locking in the bottom half.
*  Modernising to RCU.
*  TOCTOU.
*  Dual role use of x25_list_lock.

Each recommendation should include:
*  A reference number, SYNC-xxx.
*  A summary line of the proposed change and reason.
*  Description of the proposed change to locking.
*  Rough estimate of the number of lines of code modified.
*  Affected functions where locking would change.
 
Group similar recommendations if the overall lines of code is small.

Write the analysis to analysis/synchronisation-recommendations.md
