Mutex: m

* Mutex unlocked: 0
* Mutex locked: 0b01
* Mutex locked with waiters: 0b11

## Fast path

m value before | m value after | Thread 1     | Thread 2     | Action
---------------|---------------|--------------|--------------|------
0              | 1             | m c+= 1 == 1 |              | Increment Mutex. It is now 1, therefore it was unlocked and is now locked.
1              | 0             | m l= 1 == 1 |              | Set Mutex to 0. It was 1, therefore it was locked without contention.
0              | 0             | [Leave]      | [Enter]      |
0              | 1             |              | m c+= 1 == 1 |
1              | 0             |              | m c-= 1 == 1 |

## One thread contention

m value before | m value after | Thread 1     | Thread 2     | Action
---------------|---------------| -------------|--------------|------
0              | 1             | m c+= 1 == 1 |              | Increment Mutex. It is now 1, therefore it was unlocked and is now locked.
1              | 1             | [Leave]      | [Enter]      | System context switch.
1              | 2             |              | m c+= 1 == 2 | Increment Mutex. It was locked. Enter the slow path. Send a message BlockingScalar(LockMutex, &m) to Mutex server.
1              | 2             | [Enter]      | [Leave]      |
2              | 0             | m l= 0 == 2 |               | Set Mutext to 0. It was 2 (not 1), therefore the slow path was taken. Send a message Scalar(UnlockMutex, &m) to Mutex server.
0              | 0             | [Leave]      | [Enter]      | Mutex server responds to Thread 2's LockMutex call and resumes Thread 2.
0              | 1             |              | m c+= 1 == 1 | Increment Mutex. It is now 1, therefore it was unlocked and is now locked.
1              | 0             |              | m c-= 1 == 1 |

## One thread contention mid-switch

m value before | m value after | Thread 1     | Thread 2     | Action
---------------|---------------| -------------|--------------|------
0              | 1             | m c+= 1 == 1 |              | Increment Mutex. It is now 1, therefore it was unlocked and is now locked.
1              | 1             | [Leave]      | [Enter]      | System context switch.
1              | 2             |              | m c+= 1 == 2 | Increment Mutex. It was locked.
1              | 2             | [Enter]      | [Leave]      | System happens to switch contexts immediately.
2              | 0             | m l= 0 == 2 |               | Set Mutext to 0. It was 2 (not 1), therefore the slow path was taken. Send a message Scalar(UnlockMutex, &m) to Mutex server.
0              | 0             | [Leave]      | [Enter]      | Send a message BlockingScalar(LockMutex, &m) to Mutex server. Mutex server makes a note that the next `LockMutex(&m)` call should return immediately.
0              | 0             |              | [Leave]      |
0              | 0             |              | [Enter]      | Mutex server responds to Thread 2's LockMutex call and resumes Thread 2.
0              | 1             |              | m c+= 1 == 1 | Increment Mutex. It is now 1, therefore it was unlocked and is now locked.
1              | 0             |              | m c-= 1 == 1 |


## One thread contention mid-switch

m value before | m value after | Thread 1     | Thread 2     | Action
---------------|---------------| -------------|--------------|------
0              | 1             | m c+= 1 == 1 |              | Increment Mutex. It is now 1, therefore it was unlocked and is now locked.
1              | 1             | [Leave]      | [Enter]      | System context switch.
1              | 2             |              | m c+= 1 == 2 | Increment Mutex. It was locked.
1              | 2             | [Enter]      | [Leave]      | System happens to switch contexts immediately.
2              | 0             | m l= 0 == 2 |               | Set Mutext to 0. It was 2 (not 1), therefore the slow path was taken. Send a message Scalar(UnlockMutex, &m) to Mutex server.
0              | 0             | [Leave]      | [Enter]      | Send a message BlockingScalar(LockMutex, &m) to Mutex server. Mutex server makes a note that the next `LockMutex(&m)` call should return immediately.
0              | 0             |              | [Leave]      |
0              | 0             |              | [Enter]      | Mutex server responds to Thread 2's LockMutex call and resumes Thread 2.
0              | 0             | [Enter]      | [Leave]      | System happens to switch contexts immediately.
0              | 1             | m c+= 1 == 1 |              | Increment Mutex. It is now 1, therefore it was unlocked and is now locked.
1              | 1             | [Leave]      | [Enter]      | System context switch.
