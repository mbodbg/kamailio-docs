# RPC Interface#

Control interfaces are channels to communicate with Kkamailio SIP server for
administrative purposes. At this moment is one control interface, the old 
MI interface has been removed:

* RPC - remote procedure call - a more standardized option to execute commands. It is the recommended control interface.

# RPC Control Interface #

RPC is designed as scanner-printer communication channel - RPC commands will
scan the input for parameters as needed and will print back.

* fifo - [ctl module](https://kamailio.org/docs/modules/5.3.x/modules/ctl.html) - the communication is done via FIFO file using a
        simple text-based, line oriented protocol.
* datagram - [ctl module](https://kamailio.org/docs/modules/5.3.x/modules/ctl.html) - the communication is done via unix socket files or UDP sockets.
* tcp - [ctl module](https://kamailio.org/docs/modules/5.3.x/modules/ctl.html) - the communication is done via unix socket files or TCP sockets.
* xmlrpc - [xmlrpc module](https://kamailio.org/docs/modules/5.3.x/modules/xmlrpc.html) - the communication is done via XMLRPC

The RPC API is very well documented at: [http://www.kamailio.org/docs/docbooks/devel/rpc_api/rpc_api.html](http://www.kamailio.org/docs/docbooks/5.3.x/rpc_api/rpc_api.html).
We will show next just an example of implementing a RPC command:
_pkg.stats_ - dump usage statistics of PKG (private) memory,
implemented in modules_k/kex.

## Example of RPC command - pkg.stats ##

    <![CDATA[
    ...
    /**
     *
     */
    static const char* rpc_pkg_stats_doc[2] = {
        "Private memory (pkg) statistics per process",
    0
    };

    /**
     *
     */
    int pkg_proc_get_pid_index(unsigned int pid)
    {
        int i;
        for(i=0; i<_pkg_proc_stats_no; i++)
        {
            if(_pkg_proc_stats_list[i].pid == pid)
                return i;
        }
        return -1;
    }

    /**
     *
     */
    static void rpc_pkg_stats(rpc_t* rpc, void* ctx)
    {
        int i;
        int limit;
        int cval;
        str cname;
        void* th;
        int mode;
        
        if(_pkg_proc_stats_list==NULL)
        {
            rpc->fault(ctx, 500, "Not initialized");
            return;
        }
        i = 0;
        mode = 0;
        cval = 0;
        limit = _pkg_proc_stats_no;
        if (rpc->scan(ctx, "*S", &cname) == 1)
        {
            if(cname.len==3 && strncmp(cname.s, "pid", 3)==0)
                mode = 1;
            else if(cname.len==4 && strncmp(cname.s, "rank", 4)==0)
                mode = 2;
            else if(cname.len==5 && strncmp(cname.s, "index", 5)==0)
                mode = 3;
            else {
                rpc->fault(ctx, 500, "Invalid filter type");
                return;
            }
            
            if (rpc->scan(ctx, "d", &cval) < 1)
            {
                rpc->fault(ctx, 500, "One more parameter expected");
                return;
            }
            if(mode==1)
            {
                i = pkg_proc_get_pid_index((unsigned int)cval);
                if(i<0)
                {
                    rpc->fault(ctx, 500, "No such pid");
                    return;
                }
                    limit = i + 1;
                else if(mode==3) {
                    i=cval;
                    limit = i + 1;
            }
        }

        for(; i<limit; i++)
        {
            /* add entry node */
            if(mode!=2 || _pkg_proc_stats_list[i].rank==cval)
            {
                if (rpc->add(ctx, "{", &th) < 0)
                {
                    rpc->fault(ctx, 500, "Internal error creating rpc");
                    return;
                }
                if(rpc->struct_add(th, "dddddd",
                                       "entry",     i,
                                       "pid",       _pkg_proc_stats_list[i].pid,
                                       "rank",      _pkg_proc_stats_list[i].rank,
                                       "used",      _pkg_proc_stats_list[i].used,
                                       "free",      _pkg_proc_stats_list[i].available,
                                       "real_used", _pkg_proc_stats_list[i].real_used
                                      )<0)
                        {
                            rpc->fault(ctx, 500, "Internal error creating rpc");
                            return;
                        }
            }
        }
    }

    /**
     *
     */
    rpc_export_t kex_pkg_rpc[] = {
    {"pkg.stats", rpc_pkg_stats,  rpc_pkg_stats_doc,       0},
    {0, 0, 0, 0}
    };

    /**
     *
     */
    int pkg_proc_stats_init_rpc(void)
    {
    if (rpc_register_array(kex_pkg_rpc)!=0)
    {
        LM_ERR("failed to register RPC commands\n");
        return -1;
    }
    return 0;
    }
    ...
    ]]>

To add new RPC commands to control interface, you have to register them. One
option, which is used here, is to build an array with the new commands and
the help messages for each one and the use rpc_register_array(...) function.
You have the register the commands in mod_init() function - in our example
it is done via pkg_proc_stats_init(), which is a wrapper function called from
mod_init().

pkg.stats commands has optional parameters, which can be used to specify the pid,
internal rank or position in internal process table (index) of the application
process for which to dump the private memory statistics. If there is no parameter
given, the the statistics for all processes will be printed.

The command itself is implemented in C function rpc_pkg_stats(...). In the first
part, it reads the parameters. You can see there the search for optional
parameter is done by using '*' in the front of the type of parameter (str):

    rpc->scan(ctx, "*S", &amp;cname)

If this parameter is found, then there has to be another one, which this time
is no longer optional.

    rpc->scan(ctx, "d", &amp;cval)

Once input is read, follows the printing of the result in RPC structures. When
kex module is loaded, one can use pkg.stats command via sercmd tool like:

    sercmd pkg.stats
    sercmd pkg.stats index 2
    sercmd pkg.stats rank 4
    sercmd pkg.stats pid 8492

