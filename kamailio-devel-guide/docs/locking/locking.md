# Locking system #

Kamailio provides a custom locking system which has a simple interface for development.
Its root element is a mutex semaphore, that can be set (locked) or unset (unlocked). The rest
of synchronization mechanisms available in SysV and POSIX not being needed.

The locks can be used as simple variables or lock sets (array of simple locks). To improve
the speed, behind the locks is, by default, machine-specific code. If the architecture of the
machine is unknown, Kamailio will use SysV semaphores.

# Simple Locks API #

Basically, to create a lock, you have to define a variable of type
**gen_lock_t**, allocate it in shared memory and initialize it.
Then you can perform set/unset operations, destroying the variable when you finish using it.

To use the locking system in your C code you have to include the headers file:
_locking.h_

Data types and available functions to do operations with locks are described in the next
sections.

## gen_lock_t ##

It is the type to define a lock variable. It must be allocated in shared memory, to be
vailable across Kamailio processes.

## Defining a lock ##

    ...
    #include "locking.h"
    
    gen_lock_t lock;
    ...

## lock_alloc(...) ##

Allocates a lock in shared memory. If your <emphasis role="strong">gen_lock_t</emphasis>
is not part of a structure allocated in shared memory, you have to use this function to
properly create the lock.

# Prototype #

    ...
    gen_lock_t* lock_alloc();
    ...

It returns the pointer to a shared memory structure.

# Example of usage #

    ...
    #include "locking.h"
    ...
    gen_lock_t *lock;
    lock = lock_alloc();
    if(lock==NULL)
    {
        LM_ERR("cannot allocate the lock\n");
        exit;
    }
    ...

Lock allocation can be skipped if the lock is already in shared memory
-- BUT be sure the lock content IS in shared memory.

# Example of lock in shared memory #

    ...
    struct s {
        int a;
        gen_lock_t lock;
    } *x;
    
    x = shm_malloc(sizeof struct s); /* we allocate it in the shared memory */
    if (lock_init(&amp;x->lock)==0){
        /* error initializing the lock */
        ...
    }
    ...

# lock_dealloc(...) #

Free the shared memory allocated for lock.

## Prototype ##

    ...
    void lock_dealloc(gen_lock_t* lock);
    ...

The parameter is a variable returned by <emphasis role="strong">lock_alloc(...)</emphasis>.

## Example of usage ##

    ...
    #include "locking.h"
    ...
    gen_lock_t *lock;
    lock = lock_alloc();
    if(lock==NULL)
    {
        LM_ERR("cannot allocate the lock\n");
        exit;
    }
    /* make use of lock */
    ...
    lock_dealloc(lock);
    ...

# lock_init(...) #

Initialize the lock. You must call this function before the first set operation on the lock.

## Prototype ##

    ...
    gen_lock_t* lock_init(gen_lock_t* lock);
    ...

It returns the parameter if there is no error, <emphasis role="strong">NULL</emphasis>
if error occurred.
## Example of usage ##

    ...
    #include "locking.h"
    ...
    gen_lock_t *lock;
    lock = lock_alloc();
    if(lock==NULL)
    {
        LM_ERR("cannot allocate the lock\n");
    exit;
    }
    if(lock_init(lock)==NULL)
    {
        LM_ERR("cannot init the lock\n");
        lock_dealloc(lock);
        exit;
    }
    /* make use of lock */
    ...
    lock_dealloc(lock);
    ...

# lock_destroy(...) #

Destroy internal attributes of <emphasis role="strong">gen_lock_t</emphasis>. You must
call it before deallocating a lock to ensure proper clean up (if SysV is used, it calls
the functions to remove the semaphores).

## Prototype ##
 
    ...
    void lock_destroy(gen_lock_t* lock);
    ...

The parameter is a lock initialized with <emphasis role="strong">lock_init(...)</emphasis>.

## Example of usage ##

    ...
    #include "locking.h"
    ...
    gen_lock_t *lock;
    lock = lock_alloc();
    if(lock==NULL)
    {
        LM_ERR("cannot allocate the lock\n");
        exit;
    }
    if(lock_init(lock)==NULL)
    {
        LM_ERR("cannot init the lock\n");
        lock_dealloc(lock);
        exit;
    }
    /* make use of lock */
    ...
    lock_destroy(lock);
    lock_dealloc(lock);
    ...

# lock_get(...) #

Perform set operation on a lock. If the lock is already set, the function waits until
the lock is unset.

## Prototype ##
    ...
    void lock_get(gen_lock_t* lock);
    ...

The parameter must be initialized with **lock_init(...)**
before calling this function.

## Example of usage ##

    ...
    #include "locking.h"
    ...
    gen_lock_t *lock;
    lock = lock_alloc();
    if(lock==NULL)
    {
        LM_ERR("cannot allocate the lock\n");
        exit;
    }
    if(lock_init(lock)==NULL)
    {
        LM_ERR("cannot init the lock\n");
        lock_dealloc(lock);
        exit;
    }
    /* make use of lock */
    lock_get(lock);
    /* under lock protection */
    ...
    lock_destroy(lock);
    lock_dealloc(lock);
    ...

# lock_try(...) #

Try to perform set operation on a lock. If the lock is already set, the function
returns -1, otherwise sets the lock and returns 0. This is a non-blocking lock_get().

## Prototype ##

    ...
    int lock_try(gen_lock_t* lock);
    ...

The parameter must be initialized with <emphasis role="strong">lock_init(...)</emphasis>
before calling this function.

## Example of usage ##

    ...
    #include "locking.h"
    ...
    gen_lock_t *lock;
    lock = lock_alloc();
    if(lock==NULL)
    {
        M_ERR("cannot allocate the lock\n");
        exit;
    }
    if(lock_init(lock)==NULL)
    {
        LM_ERR("cannot init the lock\n");
        lock_dealloc(lock);
        exit;
    }
    /* make use of lock */
    if(lock_try(lock)==0) {
    /* under lock protection */
    ...
    } else {
       /* NO lock protection */
    ...
    }
    ...
    lock_destroy(lock);
    lock_dealloc(lock);
    ...

# lock_release(...) #

Perform unset operation on a lock. If the lock is set, it will unblock it. You
should call it after <emphasis role="strong">lock_set(...)</emphasis>, after finishing
the operations that needs synchronization and protection against race conditions.

## Prototype ##

    ...
    void lock_release(gen_lock_t* lock);
    ...

The parameter must be initialized before calling this function.

## Example of usage ##

    ...
    #include "locking.h"
    ...
    gen_lock_t *lock;
    lock = lock_alloc();
    if(lock==NULL)
    {
        LM_ERR("cannot allocate the lock\n");
        exit;
    }
    if(lock_init(lock)==NULL)
    {
        LM_ERR("cannot init the lock\n");
        lock_dealloc(lock);
        exit;
    }
    /* make use of lock */
    lock_get(lock);
    /* under lock protection */
    ...
    lock_release(lock);
    ...
    lock_destroy(lock);
    lock_dealloc(lock);
    ...

# Lock Set API #

The lock set is an array of <emphasis role="strong">gen_lock_t</emphasis>. To create a
lock set, you have to define a variable of type **gen_lock_set_t**, allocate it in shared memory specifying the
number of locks in the set, then initialize it. You can perform set/unset operations,
providing the lock set variable and destroying the variable when you finish using it.
To use the lock sets in your C code you have to include the headers file: **locking.h**.

Data types and available functions to do operations with lock sets are described in the next
sections.

# gen_lock_set_t #

It is the type to define a lock set variable. It must be allocated in shared memory, to be
available across Kamailio processes.

# Defining a lock #

    ...
    #include "locking.h"

    gen_lock_set_t lock_set;
    ...

# lock_set_alloc(...) #

Allocates a lock set in shared memory.

## Prototype ##

    ...
    gen_lock_set_t* lock_set_alloc(int n);
    ...

The parameter <emphasis role="strong">n</emphasis> specifies the number
of locks in the set. Return pointer to the lock set, or NULL if the set couldn't
be allocated.

## Example of usage ##

    ...
    #include "locking.h"
    ...
    gen_lock_set_t *set;
    set = lock_set_alloc(16);
    if(set==NULL)
    {
        LM_ERR("cannot allocate the lock set\n");
        exit;
    }
    ...

# lock_set_dealloc(...) #

Free the memory allocated for a lock set.

## Prototype ##

    ...
    void lock_set_dealloc(gen_lock_set_t* set);
    ...

The parameter has to be a lock set allocated with **lock_set_alloc(...)**.

## Example of usage ##
    ...
    #include "locking.h"
    ...
    gen_lock_set_t *set;
    set = lock_set_alloc(16);
    if(set==NULL)
    {
        LM_ERR("cannot allocate the lock set\n");
        exit;
    }
    ...
    lock_set_dealloc(set);
    ...

# lock_set_init(...) #

Initialized the internal attributes of a lock set. The lock set has to be allocated
with <emphasis role="strong">lock_set_alloc(...)</emphasis>. This function must be
called before performing any set/unset operation on lock set.

## Prototype ##

    ...
    gen_lock_set_t* lock_set_init(gen_lock_set_t* set);
    ...

The parameter is an allocated lock set. It returns NULL if the lock set couldn't be
initialized, otherwise returns the pointer to the lock set.

## Example of usage ##

    ...
    #include "locking.h"
    ...
    gen_lock_set_t *set;
    set = lock_set_alloc(16);
    if(set==NULL)
    {
        LM_ERR("cannot allocate the lock set\n");
        exit;
    }
    if(lock_set_init(set)==NULL)
    {
        LM_ERR("cannot initialize the lock set'n");
        lock_set_dealloc(set);
        exit;
    }
    /* make usage of lock set */
    ...
    lock_set_dealloc(set);

    ...

# lock_set_destroy(...) #

Destroy the internal structure of the lock set. You have to call it once
you finished to use the lock set. The function must be called after **lock_set_init(...)**

## Prototype ##

    ...
    void lock_set_destroy(gen_lock_set_t* set);
    ...

The parameter is an initialized lock set. After calling this function you should not
perform anymore set/unset operations on lock set.

## Example of usage</title> ##

    ...
    #include "locking.h"
    ...
    gen_lock_set_t *set;
    set = lock_set_alloc(16);
    if(set==NULL)
    {
        LM_ERR("cannot allocate the lock set\n");
        exit;
    }
    if(lock_set_init(set)==NULL)
    {
        LM_ERR("cannot initialize the lock set'n");
        lock_set_dealloc(set);
        exit;
    }
    /* make usage of lock set */
    ...
    lock_set_destroy(set);
    lock_set_dealloc(set);
    ...

# lock_set_get(...) #

Set (block) a lock in the lock set. You should call this function after the lock set
has been initialized. If the lock is already set, the function waits until that lock is
unset (unblocked).

## Prototype ##

    ...
    void lock_set_get(gen_lock_set_t* set, int i);
    ...

First parameter is the lock set. The second is the index withing the set of the lock to
e set. First lock in set has index 0. The index parameter must be between **0** and **n-1** 
(see **lock_set_alloc(...)**.

## Example of usage</title> ##

    ...
    #include "locking.h"
    ...
    gen_lock_set_t *set;
    set = lock_set_alloc(16);
    if(set==NULL)
    {
        LM_ERR("cannot allocate the lock set\n");
        exit;
    }
    if(lock_set_init(set)==NULL)
    {
        LM_ERR("cannot initialize the lock set'n");
        lock_set_dealloc(set);
        exit;
    }
    /* make usage of lock set */
    lock_set_get(set, 8);
    /* under lock protection */
    ...
    lock_set_destroy(set);
    lock_set_dealloc(set);
    ...

# lock_set_get(...) #

Try to set (block) a lock in the lock set. You should call this function after the lock set
has been initialized. If the lock is already set, the function returns -1, otherwise it sets
the lock and returns 0. This is a non-blocking lock_set_get().

## Prototype ##

    ...
    int lock_set_try(gen_lock_set_t* set, int i);
    ...
First parameter is the lock set. The second is the index withing the set of the lock to
be set. First lock in set has index 0. The index parameter must be between
**0** and **n-1** (see **lock_set_alloc(...)**.

## Example of usage ##

    ...
    #include "locking.h"
    ...
    gen_lock_set_t *set;
    set = lock_set_alloc(16);
    if(set==NULL)
    {
        LM_ERR("cannot allocate the lock set\n");
        exit;
    }
    if(lock_set_init(set)==NULL)
    {
    LM_ERR("cannot initialize the lock set'n");
    lock_set_dealloc(set);
    exit;
    }
    /* make usage of lock set */
    if(lock_set_try(set, 8)==0) {
        /* under lock protection */
        ...
    } else {
        /* NO lock protection */
        ...
    }
    ...
    lock_set_destroy(set);
    lock_set_dealloc(set);
    ...
#lock_set_release(...)#
Unset (unblock) a lock in the lock set.

## Prototype ##

    ...
    void lock_set_release(gen_lock_set_t* set, int i);
    ...

First parameter is the lock set. The second is the index withing the set of the lock to
be unset. First lock in set has index 0. The index parameter must be between
**0** and **n-1** (see **lock_set_alloc(...)**.

## Example of usage ##

    ...
    #include "locking.h"
    ...
    gen_lock_set_t *set;
    set = lock_set_alloc(16);
    if(set==NULL)
    {
        LM_ERR("cannot allocate the lock set\n");
        exit;
    }
    if(lock_set_init(set)==NULL)
    {
        LM_ERR("cannot initialize the lock set'n");
        lock_set_dealloc(set);
        exit;
    }
    /* make usage of lock set */
    lock_set_get(set, 8);
    /* under lock protection */
    ...
    lock_set_release(set, 8);
    ...
    lock_set_destroy(set);
    lock_set_dealloc(set);
    ...

# Troubleshooting #
A clear sign of issues with the locking is that one or more Kamailio processes eat lot of CPU.
If the traffic load does not justify such behavior and no more SIP messages are processed, the only
solution is to troubleshoot and fix the locking error. The problem is that a lock is set but never
unset. A typical case is when returning due to an error and forgetting to release a previously lock
set.

To troubleshoot a solution is to use **gdb**, attach to the process
that eats lot of CPU and get the backtrace. You need to get the PID of that Kamailio process -
**top** or **ps** tools can be used.

    ...
    # gdb /path/to/kamailio PID
    # gdb&gt; bt
    ...

From the backtrace you should get to the lock that is set and not released. From there you should
start the investigation - what are the cases to set that lock and in which circumstances it does not
get released.
