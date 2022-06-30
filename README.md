# Resource Allocation

## Background
- Assume a system in which many jobs are run, each job has `{priority, start time, end time, memory requested, and other proprietary fields}`.
- A job need memory & compute resources to be available in the system in order to run.
- In case there are none available, Job will fail.

## Task 
Given log of all job executions from the past year, including size of memory used.

Study the concurrency of Jobs, in order to optimize pre-allocated resources / budget

Implement the logical component for calculating concurrency of Jobs, and resources used. And offer ways to deduce how many resources should be pre-allocated in order to sustain proper functioning of the system.

**Note**: aiming for 100% availability may be possible yet costly, so please plan for certain ratio of failure ~10%.

## Solution

### Assumptions

- The log was generated with a "100% availability" system (no failed jobs)
- Log is sorted

### Description

The optimal value(s) will be found by iterating on possible optimal value(s).

Each iteration will check a different optimal value(s) and figure out if the acceptable failure ratio constraint was met.

Basically, we'll re-run the log with different resource constraints each time.

#### Step 1: Extract required data

Before running the "simulations", we need to determine:

- What is the "10%" failure rate constraint (in concrete numbers)
  - How: Find the total number of executed Jobs (log's number of rows)
- The maximum resource(s) usage level - will later be used as baseline (starting point)
  - How: Check current system configuration/setup 
    - Or: run an iteration as describe below just to capture this data and then `SELECT MAX(memory_usage) from <table_name>`

#### Step 2: Setup data store

To run the "simulations" we need to track the resource(s) usage in every point in time.

Let's assume a 1-minute granularity is enough (the implementation below will work with a more granular timeframe).

We'll use MySQL in this example but any other storage type (e.g. memory) may work as well.

Create a table, in which each row represents a minute in time, with the current level of resource(s) usage:
- `start_time`, Datetime
- `memory_usage`, Int (initialized to 0)
- `xxx_usage`, Int (initialized to 0)


### Step 3: Run "simulation" iterations

Basically, we'll parse the log line by line, accumulate/sum the resource usage level in the proper column and count failed jobs, given a specific resource limitation.

**Pseudo code:**

- Define a `memoryStep` - Used to calculate `maxMemoryAvailable` in each iteration
- Initialize `maxMemoryAvailable` to the value found in Step 1 (maximum used) minus `memoryStep`
- Initialize `optimalMemoryLevel` to `maxMemoryAvailable`

Iteration:
  - Parse the log and for each line (executed Job):
    - Check if there are enough resources for it to run
      - If it cannot run
        - Increment a `failedJobsCounter`
          - If `failedJobsCounter` is greater than the allowed ratio
            - This iteration fails => `optimalMemoryLevel` is the optimal value
            - Break
      - Else
        - Add it to the DB (similar to Step 1) and move on to the next line
        - Continue (to the next job/line) 

If an iteration (a full log parse) completed successfully:
  - Set `optimalMemoryLevel` to `maxMemoryAvailable`
  - Set `maxMemoryAvailable` to `maxMemoryAvailable`-`memoryStep`
  - Truncate the table
  - Iterate again

Else
  - Done


Done: `optimalMemoryLevel` is the optimal value

### Optimizations
- Use binary search instead of the algo described in Step 3 - O(log(n))
- Use MySQL query "simulations" instead of reading the log line by line:
  - Populate the table with no resource limitation at all (all jobs get stored in the table)
  - Populate another table with every executed job
    - `start_time`
    - `end_time`
  - Iterate on possible optimal values and run an SQL query to find how many jobs will fail for a defined `maxMemoryAvailable`:
    - Find the timeframes in which the resource usage exceeds `maxMemoryAvailable`.
    - Count the number of jobs that ran in these timeframes:
      - A job ran in a given timeframe if its `start_time` or `end_time` are in the timeframe
