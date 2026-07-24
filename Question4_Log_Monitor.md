# Question 4 – Real-Time Log Monitoring Pipeline

## Command Pipeline

```bash
tail -f /var/log/app.log | grep --line-buffered "ERROR" | tee -a error_report.txt > /dev/null
```


## Extended Version with Multiple Filters

```bash
tail -f /var/log/app.log \
  | grep --line-buffered -E "ERROR|WARN|CRITICAL" \
  | while IFS= read -r line; do
      timestamp=$(date '+%Y-%m-%d %H:%M:%S')
      echo "[$timestamp] $line"
    done \
  | tee -a error_report.txt > /dev/null
```


## Setup Commands and Explanations

**Command: tail -f /var/log/app.log**

tail -f continuously reads new lines as they are appended to the file, rather than reading the file once and exiting. The -f flag stands for follow. This keeps the pipeline alive indefinitely, making it suitable for real-time monitoring of a server log without polling or cron jobs.


**Command: grep --line-buffered "ERROR"**

grep filters and passes only lines containing the word ERROR. Without --line-buffered, grep accumulates input in a large buffer before writing, which would delay output in a pipe. The --line-buffered flag forces grep to flush after every matched line so the monitoring tool responds in real time and not in batches.


**Command: tee -a error_report.txt**

tee reads from standard input and writes to both standard output and the named file at the same time. The -a flag appends to the file instead of overwriting it, preserving earlier entries across multiple monitoring sessions. This lets the pipeline keep a persistent record while still passing data downstream.


**Command: > /dev/null**

/dev/null is a special file that discards everything written to it. Redirecting standard output here suppresses the terminal echo of matched lines that tee would otherwise print. The only visible output is whatever the pipeline decides to show explicitly, keeping the terminal clean while the report file still captures everything.


## Sample Output (error_report.txt)

```
[2025-07-24 21:05:12] ERROR: Database connection timeout on port 5432
[2025-07-24 21:06:44] ERROR: Failed to authenticate user id=4821
[2025-07-24 21:08:03] ERROR: Disk write failed on /var/data partition
[2025-07-24 21:09:55] CRITICAL: Service worker crashed, restarting...
```


## Running in the Background

```bash
nohup bash monitor.sh >> monitor.log 2>&1 &
echo "Monitor PID: $!"
```

Running the pipeline under nohup with & sends it to the background. nohup prevents the process from being killed when the SSH session ends. The PID is captured so it can be stopped later with kill.


## Explanation

The pipeline works because each program in a Unix pipe runs as a separate process connected through an in-memory buffer. tail produces data continuously, grep filters it by pattern, tee splits the stream to both a file and standard output, and /dev/null absorbs the terminal copy silently. This design separates concerns cleanly. Changing the filter pattern only requires modifying the grep argument. Adding another stage such as sending an email alert on CRITICAL messages means inserting one more pipe segment without rewriting the rest. The --line-buffered flag is the most important tuning choice here because without it the pipeline behaves correctly in batch processing but fails as a real-time tool since matches would appear in unpredictable bursts. Together the combination of tail, grep, tee, and redirection turns four standard utilities into a lightweight but effective log monitoring system with zero external dependencies.
