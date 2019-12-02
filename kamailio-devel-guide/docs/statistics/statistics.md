These are integer numbers that collect information about Kamailio internals. They are providing real-time
feedback about health and load of an Kamailio instance. In fact, they are represented by an integer variable or
a function that returns an integer.

The **statistics engine** is implemented in the files
**lib/kcore/statistics..{c,h}** - practically they are part
of internal library **kcore**. If you want to extend it, you have to read and
understand the code in those file. The purpose of this chapter is to teach how to add new statistic
variables.

You have to include the header file **lib/kcore/statistics.h** and declare the
statistic variable. We exemplify with the statistic **stored_messages**
from module **msilo**. In the file
**modules/msilo/msilo.c**.

    ...		
    #include "../../lib/kcore/statistics.h"

    stat_var* ms_stored_msgs;
    ...

Next is to register the statistic to the engine, which can done there via 
register_module_stats(...) function when you have an array of new statistics.

    ...
    stat_export_t msilo_stats[] = {
        {"stored_messages" ,  0,  &amp;ms_stored_msgs  },
    ...
    if (register_module_stats( exports.name, msilo_stats)!=0 ) {
        LM_ERR("failed to register core statistics\n");
        return -1;
    }
    ...

Alternative is to use the function **register_stat(...)** defined in
**lib/kcore/statistics.{c,h}**.

    ...
    int register_stat( char *module, char *name, stat_var **pvar, int flags);
    ...

The parameters are:

module - name of the module exporting the statistic

* name - name of the statistic
* var - where the statistic value will be stored
* flags - flags describing the statistic

Updating the value of the statistic is done in function **m_store(...)**,
once a new message is stored in database.

    ...
    update_stat(ms_stored_msgs, 1);
    ...

# Statistic Macros #

There are three macros that help to deal with statistic values easily.

update_stat (stat, val) - add to the statistic value the **val**.
**val** can be negative as well, resulting in substraction.
* reset_stat (stat) - set the value of the statistic to **0**
* get_stat_val (stat) - return the value of the statistic
