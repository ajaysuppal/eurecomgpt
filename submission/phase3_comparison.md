# Phase 3 — Comparison Writeup

Copy this file to `submission/phase3_comparison.md`, fill in every answer slot, and commit it:

```bash
cp phase-3-mapreduce-spark/comparison_template.md submission/phase3_comparison.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers in section 1 must **match `submission/phase3_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Facts from your runs

Numbers only (e.g. `12`, `20.3`, `0.041`).

| | Value |
|---|---|
| Map tasks in the cloud job (`mapreduce.num_splits`) | <!--answer:num_map_tasks-->12<!--/answer--> |
| Reduce tasks in the cloud job (`mapreduce.num_reducers`) | <!--answer:num_reduce_tasks-->4<!--/answer--> |
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->29.1<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.01<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->4.0<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->4<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
Fastest to slowest: local 0.01 s, Spark 4.0 s, cloud 29.1 s. Local and cloud are the like-for-like pair, because they run the same word count on the same 60 documents. The real counting work is only about 10 ms, so almost all of the 29.1 s in the cloud is overhead that starts 16 function calls, reads and writes many small files in Cloud Storage, and the workflow schedules the tasks (plus waiting before retries). Spark does more work (TF-IDF) on one machine in 4.0 s, which shows the cloud job is slow because of coordination and network, not computation.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
4 stages means 3 shuffles. They come from the reduceByKey in term_freq, the reduceByKey in doc_freq, and the join in build_tfidf_rdd. I saw 4 retries, so 16 tasks needed 20 attempts: 4/20 = 20% of attempts failed. 

With --chaos 0.3 I expected around 30% (6-7 retries), but with so few attempts, 4 is a normal random result.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
Transformations: parallelize, flatMap, reduceByKey (twice), map (three times) and join. They are lazy and only build the plan. 

The action is collect. toDebugString only prints the plan. 

- When I ran build_tfidf_rdd it returned at once and nothing was computed.
- 2b just showed the lineage. 
- When I ran collect, Spark ran job 0 (4 stages, 12 tasks) in 4.0 s and returned 5975 rows.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
The first reduceByKey gives ShuffledRDD[3], which stages 1 and 2 read. If a worker with one of its partitions died, Spark would only rebuild that one partition by reading the stage 0 shuffle output again (PairwiseRDD[2]) and redoing the reduce. The other partitions are reused. 

Recovery starts at the shuffle boundary between stage 0 and stage 1 (reduceByKey, line 33).
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows (the mr-wordcount workflow from workflow.yaml).<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->The Cloud Run function mr-worker (main.py), running as mr-runner.<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Cloud Storage (the eurecomgpt-ajay-suppal-pid-007-eurecomgpt bucket).<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->partition() in mapreduce.py (crc32(word) % number of reducers).<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
The barrier is the map_phase parallel step. A parallel step only finishes when all its branches have finished (retries included), so reduce_phase can only start after the last map task is done.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
A map task always reads the same input split and writes to the same file names, and upload_from_string overwrites the old file. So running a task again gives exactly the same result. If run_map appended instead, a task that was retried after already writing would add its pairs twice, the reducers would count those words twice, and the result would no longer match local.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
With --chaos 0.3 some calls failed and the workflow retried them. In Logs Explorer, filtered on mr-worker, I found 4 POST requests with status 503 (the simulated crashes), each only a few ms long because the crash happens before any work. The job still finished with SUCCEEDED in 29.1 s, and the result was identical to the local one (2221 words, identical: True).
<!--/answer-->
