## 1. What is the most complex automation you have built?
## 2. Why did you build it?
## 3. What was the manual process before automation?
## 4. How many servers/workloads did it manage?
## 5. What language did you use and why?
## 6. How did you handle failures?
Answer:

> "I designed automation so that failure will be isolated, retryable, recoverable"

I didn't treat automation as a single or All-or-nothing script.
- validation before execution
- Idempotency
- Retry transient failure
- partial failure handling
- state tracking
- rollback
- Manual intervention
- Monitoring and alerting


> "Suppose the automation has five steps, it fails partially.
> i will not restart the complete process from the first stage. i will persist the states and identify which steps are succeeded and which are failed.
> as the process is transient will retry, the steps which is already succeed will not any affect as idempotent. will keep automation that it will retry
> for some time and then will mark it as a failure."
 

## 7. How did you implement retries?
Answer:

I will retry the automation only if it is transient. i will not retry every error, because retrying every error will make even worse.
```
Worker receives Node-A event
          ↓
       Execute
          ↓
       Failed?
       /     \
     No       Yes
     ↓         ↓
 SUCCESS    Is error retryable?
             /        \
           No          Yes
           ↓            ↓
         FAILED      Retry count?
                       /      \
                     < max      >= max
                       ↓          ↓
                    Backoff     FAILED
                       ↓          ↓
                    Retry      Alert
```
- Exponential backoff
- Retry limit
- Retry only transient error
- persist retry state.
- Idempotency
  
## 8. How did you make it idempotent?

> I made an automation state-based instead of action based. instead of running same automation everytime
> i will check the state of the existing state, and accordingly i will add logic -- do i need to execute this automation or not?

- check before change
- Make each step independently Idempotent
- Avoid non-idempotent operations
- Database query protection

  
## 9. How did you monitor the automation?

> I will monitor automation with metrics, logs and alerts.i will make sure that automation is working fine or not.
> how long it will take time to complete.

- metrics
- Log monitoring
- Alerts
- Dashboard


## 10. How did you test it?
## 11. What happens if the automation partially fails?
Make automation idempotent.
If it is partitally fails i will set alerts. and persist the state of the automation. for that i came to know that which stage is succeed or which is failed.
i will make sure that only  failed one should execute.  do not repeat the succeeded stages with idempotency.

- checkpointing
- retrying
- idempotency
- alerts
- rollback

  
## 12. How did you rollback?
Avoid Saying i always rollback. in infrastructure some changes are reversible and some are not.

Before making changes i capture the state of the current status so it will help in rolling back.

Rollback isn't always possible.

## 13. How did you make it safe for production?

- least privillege -> limited access
- Scope the automation -> dont allow cluster access
- Rate limits
- Circuit breaker
- gurd rails -> Never drain more then 5 nodes
- audit trails -> every action should be recorded

## 14. How much engineering time did it save?

## 15. How would you redesign it if the environment grew 10×?
1. Horizontally scale workers
- Scale workers according to queue lenght with using HPA/KEDA.

2. Partition the queue
3. Add per node locking
4. persistant state
5. Rate limiting
6. priority queue
7. dead-letter queue
8. observability
9. Reconcillation instead of relying only on events.

```
10× infrastructure
       ↓
Horizontal workers
       +
Queue partitioning
       +
Concurrency limits
       +
Rate limiting
       +
Persistent state
       +
Reconciliation
       +
Observability
```


