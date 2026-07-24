# Question 3 – Secure File Processing Using Linux System Calls

## C Program

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>

#define RECORD_SIZE 64
#define FILE_NAME "employees.dat"

typedef struct {
    int  id;
    char name[40];
    char department[20];
} Employee;

void write_employee(int fd, Employee *emp) {
    ssize_t bytes = write(fd, emp, sizeof(Employee));
    if (bytes != sizeof(Employee)) {
        perror("write failed");
        exit(1);
    }
}

void read_employee(int fd, int index, Employee *emp) {
    off_t offset = (off_t)index * sizeof(Employee);
    if (lseek(fd, offset, SEEK_SET) == -1) {
        perror("lseek failed");
        exit(1);
    }
    ssize_t bytes = read(fd, emp, sizeof(Employee));
    if (bytes != sizeof(Employee)) {
        perror("read failed");
        exit(1);
    }
}

void update_employee(int fd, int index, Employee *emp) {
    off_t offset = (off_t)index * sizeof(Employee);
    if (lseek(fd, offset, SEEK_SET) == -1) {
        perror("lseek failed");
        exit(1);
    }
    ssize_t bytes = write(fd, emp, sizeof(Employee));
    if (bytes != sizeof(Employee)) {
        perror("update write failed");
        exit(1);
    }
}

int main() {
    int fd = open(FILE_NAME, O_CREAT | O_RDWR | O_TRUNC, 0600);
    if (fd == -1) {
        perror("open failed");
        return 1;
    }
    printf("File '%s' created successfully (fd=%d)\n", FILE_NAME, fd);

    Employee records[] = {
        {101, "Amit Sharma",  "Engineering"},
        {102, "Priya Menon",  "HR"},
        {103, "Ravi Kumar",   "Finance"},
        {104, "Sunita Rao",   "Marketing"},
        {105, "Arjun Nair",   "Engineering"}
    };
    int count = sizeof(records) / sizeof(records[0]);

    for (int i = 0; i < count; i++) {
        write_employee(fd, &records[i]);
    }
    printf("Written %d employee records to file.\n", count);

    Employee updated = {103, "Ravi Kumar Pillai", "Accounts"};
    update_employee(fd, 2, &updated);
    printf("Record at index 2 updated.\n");

    Employee fetched;
    for (int i = 0; i < count; i++) {
        read_employee(fd, i, &fetched);
        printf("Record %d -> ID: %d | Name: %-20s | Dept: %s\n",
               i, fetched.id, fetched.name, fetched.department);
    }

    if (close(fd) == -1) {
        perror("close failed");
        return 1;
    }
    printf("File closed successfully.\n");
    return 0;
}
```


## Commands and Explanations

**Command: open(FILE_NAME, O_CREAT | O_RDWR | O_TRUNC, 0600)**

open() is the system call that creates or opens the file. O_CREAT creates it if it does not exist, O_RDWR allows both reading and writing in a single descriptor, and O_TRUNC clears any previous contents. The mode 0600 sets permissions to owner read and write only, which is important for a file holding employee data.


**Command: write(fd, emp, sizeof(Employee))**

write() sends exactly sizeof(Employee) bytes from the struct into the file at the current file offset. Because the struct size is fixed, every record occupies an identical amount of space, which makes random access by index straightforward later. The return value is checked to catch partial writes caused by disk errors.


**Command: lseek(fd, offset, SEEK_SET)**

lseek() moves the file offset to any byte position without reading or writing data. Multiplying the record index by sizeof(Employee) gives the exact byte address of that record. SEEK_SET measures from the beginning of the file, so the position is always absolute and predictable regardless of where the offset was before.


**Command: read(fd, emp, sizeof(Employee))**

read() pulls sizeof(Employee) bytes from the current offset into the struct. Because lseek was called first, the read begins exactly at the desired record. The kernel handles buffering internally so the call is both efficient and consistent.


**Command: close(fd)**

close() releases the file descriptor and flushes any kernel-level write buffers. Failing to call close is a resource leak. On busy servers with many files open, unclosed descriptors can exhaust the per-process file descriptor limit and cause later open() calls to fail.


## Sample Output

```
File 'employees.dat' created successfully (fd=3)
Written 5 employee records to file.
Record at index 2 updated.
Record 0 -> ID: 101 | Name: Amit Sharma          | Dept: Engineering
Record 1 -> ID: 102 | Name: Priya Menon           | Dept: HR
Record 2 -> ID: 103 | Name: Ravi Kumar Pillai      | Dept: Accounts
Record 3 -> ID: 104 | Name: Sunita Rao             | Dept: Marketing
Record 4 -> ID: 105 | Name: Arjun Nair             | Dept: Engineering
File closed successfully.
```


## Explanation

Using system calls directly instead of fopen and fprintf gives precise control over file permissions, descriptor reuse, and I/O boundaries. The fixed-size struct layout is what makes lseek-based random access reliable. If records had variable lengths, a seek calculation based on index alone would not work. The combination of open for creation, write for insertion, lseek for positioning, and read for retrieval replicates what a simple embedded database does at its lowest layer. Checking return values at every step ensures the program fails loudly rather than silently corrupting data.
