# Timer #
Kamailio provides internal timers with millisecond precision. It offers developer API to:

* register functions to be run every 1 second or interval of multiple seconds
* register functions to run every millisecond or interval of multiple milliseconds
* start a new timer process

The timer API is implemented in the files **timer.{c,h}**. If
You want to extend the timer API you have to start with those files. We focus on how to add
new functions to be run by timer. The timer is available after shared memory initialization.

## Data Types ##

The functions that can be given as callbacks for timer have the following prototypes:

    ...
    typedef unsigned long long utime_t;
    
    typedef void (timer_function)(unsigned int ticks, void* param);
    ...

Parameters are:

* ticks - number of second ticks elapsed at the moment of running the function
* param - parameter given when registering the callback function

## Timer API Functions ##

Register a function to be executed by second-based timer:

    ...
    int register_timer(timer_function f, void* param, unsigned int interval);
    ...

Parameters:

* f - callback function
* param - parameter to callback function
* interval - interval to execute the callback function
* Register a function to start a new timer process:

    ...
    int register_timer_process(timer_function f, void* param, unsigned int interval);
    ...

Parameters:

* f - callback function
* param - parameter to callback function
* interval - interval to execute the callback function

There are two functions that return the number of ticks elapsed since Kamailio started,
one returning time in seconds and the other one returning the number of internal ticks.

    ...
    unsigned int get_ticks(void);
    
    utime_t get_ticks_raw(void);
    ...

## Example of usage ##

Next we will show how the module **msilo** register a timer
function to clean the stored messages. See [**modules/msilo/msilo.c**](https://github.com/kamailio/kamailio/blob/5.3/src/modules/msilo/msilo.c).

    ...
    #include "../../timer.h"
    ...
    void m_clean_silo(unsigned int ticks, void *);
    ...

Registration to the second-based timer is done in function **mod_init()**.

    ...
    register_timer(m_clean_silo, 0, ms_check_time);
    ...

Function implementation is:

    ...
    <![CDATA[
    /**
     * - cleaning up the messages that got reply
     * - delete expired messages from database
    */
    void m_clean_silo(unsigned int ticks, void *param)
    {
        msg_list_el mle = NULL, p;
        db_key_t db_keys[MAX_DEL_KEYS];
        db_val_t db_vals[MAX_DEL_KEYS];
        db_op_t  db_ops[1] = { OP_LEQ };
        int n;
        
        LM_DBG("cleaning stored messages - %d\n", ticks);
        
        msg_list_check(ml);
        mle = p = msg_list_reset(ml);
        n = 0;
        while(p)
        {
            if(p->flag & MS_MSG_DONE)
            {
    
    #ifdef STATISTICS
    
    if(p->flag & MS_MSG_TSND)
        update_stat(ms_dumped_msgs, 1);
            else
        update_stat(ms_dumped_rmds, 1);
    #endif
    
            db_keys[n] = &sc_mid;
            db_vals[n].type = DB1_INT;
            db_vals[n].nul = 0;
            db_vals[n].val.int_val = p->msgid;
            LM_DBG("cleaning sent message [%d]\n", p->msgid);
            n++;
            if(n==MAX_DEL_KEYS)
            {
                if (msilo_dbf.delete(db_con, db_keys, NULL, db_vals, n) < 0) 
                    LM_ERR("failed to clean %d messages.\n",n);
                    n = 0;
            }
            }
            if((p->flag & MS_MSG_ERRO) && (p->flag & MS_MSG_TSND))
                { /* set snd time to 0 */
            ms_reset_stime(p->msgid);
    #ifdef STATISTICS
        update_stat(ms_failed_rmds, 1);
    #endif
    
        }
    #ifdef STATISTICS
        if((p->flag & MS_MSG_ERRO) && !(p->flag & MS_MSG_TSND))
        update_stat(ms_failed_msgs, 1);
    #endif
    = p->next;
    }
    if(n>0)
    {
        if (msilo_dbf.delete(db_con, db_keys, NULL, db_vals, n) < 0) 
             LM_ERR("failed to clean %d messages\n", n);
             n = 0;
    }
    
    msg_list_el_free_all(mle);
    
    /* cleaning expired messages */
        if(ticks%(ms_check_time*ms_clean_period)<ms_check_time)
        {
            LM_DBG("cleaning expired messages\n");
            db_keys[0] = &sc_exp_time;
            db_vals[0].type = DB1_INT;
            db_vals[0].nul = 0;
            db_vals[0].val.int_val = (int)time(NULL);
            if (msilo_dbf.delete(db_con, db_keys, db_ops, db_vals, 1) < 0) 
                LM_DBG("ERROR cleaning expired messages\n");
        }
    }
    ]]>
    ...

The function deletes from database the messages that were succesfully delivered and the
messages that were stored for too long time in database and the recipient was not online
or not able to receive them.

## Dedicated timer process ##

Sometime might be better to have your own timer process, not to disturb the operations
done by other timer function - recommended when you are doing operation that may take a
while on timer basis.

You can create as many timer processes as you need - it takes two steps:

* in mod_init() tell to timer API how many processes you want to create
+ in child_init() for MAIN child, fork the timer processes with the callback function


    
    ...
    /*
    *init module function
    */
    static int mod_init(void)
    {
        ...
        register_dummy_timers(1);
        
        ...
    }
    
    ...
    
    /**
     * init module children
     */
    static int child_init(int rank)
    {
    
    ...
    if (rank==PROC_MAIN)
    {
        if(fork_dummy_timer(PROC_TIMER, "MY MOD TIMER", 1 /*socks flag*/,
            my_timer_callback, my_timer_param, 1 /*sec*/) &lt; 0) {
                LM_ERR("failed to register timer routine as process\n");
                return -1; /* error */
            }
        }
    ...
    }
    ...
