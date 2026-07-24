# Question 1 – Duplicate Submission Detector & Backup Script

## Shell Script

```bash
#!/bin/bash

SOURCE_DIR="./submissions"
BACKUP_DIR="./unique_backup"
REPORT_FILE="./report.txt"
ERROR_FILE="./errors.txt"

mkdir -p "$BACKUP_DIR"

total=0
duplicates=0
backed_up=0

declare -A seen_hashes

echo "Processing started at $(date)" > "$REPORT_FILE"
echo "Error log started at $(date)" > "$ERROR_FILE"

for file in "$SOURCE_DIR"/*; do
    if [ ! -f "$file" ]; then
        echo "Warning: $file is not a regular file" >> "$ERROR_FILE" 2>&1
        continue
    fi

    total=$((total + 1))

    hash=$(md5sum "$file" 2>>"$ERROR_FILE" | awk '{print $1}')

    if [ -z "$hash" ]; then
        echo "Error: Could not compute hash for $file" >> "$ERROR_FILE"
        continue
    fi

    if [ "${seen_hashes[$hash]+isset}" ]; then
        duplicates=$((duplicates + 1))
        echo "Duplicate found: $file (matches ${seen_hashes[$hash]})" >> "$REPORT_FILE"
    else
        seen_hashes[$hash]="$file"
        cp "$file" "$BACKUP_DIR/" 2>>"$ERROR_FILE"
        backed_up=$((backed_up + 1))
    fi
done

echo "" >> "$REPORT_FILE"
echo "Total files processed : $total" >> "$REPORT_FILE"
echo "Duplicate files found  : $duplicates" >> "$REPORT_FILE"
echo "Unique files backed up : $backed_up" >> "$REPORT_FILE"
echo "Processing completed at $(date)" >> "$REPORT_FILE"

cat "$REPORT_FILE"
```


## Commands and Explanations

**Command: mkdir -p "$BACKUP_DIR"**

I used mkdir with the -p flag to create the backup directory. The -p option makes sure the command does not throw an error if the directory already exists, which keeps the script safe to run multiple times.


**Command: md5sum "$file" | awk '{print $1}'**

md5sum generates a unique hash for each file based on its content, not its name. If two files have the same hash they are identical. I piped the output into awk to extract only the hash string and ignore the filename that md5sum also prints.


**Command: cp "$file" "$BACKUP_DIR/" 2>>"$ERROR_FILE"**

cp copies the unique file into the backup directory. The 2>> operator appends any error messages (like permission denied) to the error log without interrupting the rest of the script.


**Command: echo "..." >> "$REPORT_FILE"**

The >> operator appends text to the report file without overwriting what was already written. Using > would erase previous entries, so >> is essential for building the report line by line across the loop.


## Sample Output

```
Processing started at Thu Jul 24 21:00:00 IST 2025
Duplicate found: ./submissions/raj_assign.txt (matches ./submissions/student_assign.txt)

Total files processed : 10
Duplicate files found  : 2
Unique files backed up : 8
Processing completed at Thu Jul 24 21:00:01 IST 2025
```


## Justification

The script uses md5sum for content-based comparison rather than filename comparison because students can rename a copied file. A bash associative array (declare -A) stores each hash and its original file path, making lookup fast and simple. All standard error output is redirected with 2>> into a separate errors.txt file so that the report remains clean. The backup uses cp instead of mv to preserve the originals in the source directory. This approach combines hashing, redirection, and conditionals in a straightforward pipeline that is easy to audit and extend.
