[flex](https://github.com/westes/flex) and [bison](http://www.gnu.org/software/bison/)
are used to parse the configuration file and build the actions tree that are executed at run
time for each SIP message. **Bison** is the GNU implementation
compatible with Yacc (Yet Another Compiler Compiler), but also Yacc or Byacc can be used instead of it.

Extending the configuration file can be done by adding a new core parameter or a new core functions.
Apart of these, one can add new routing blocks, keywords or init and runtime instructions.

Starting with release series 3.0, configuration file language has support for preprocessor
directives. They provide an easy way to define tokens to values or enable/disable parts of
configuration file.

The config file include two main categories of statements:

* **init statements** - this category includes setting the global
parameters, loading modules and setting module parameters. These statements are executed only
one time, at startup.
* **runtime statements** - this category includes the actions
executed many times, after Kamailio initialization. These statements are grouped in route
blocks, there being different route types that get executed for certain events.
    * **route** -  is executed when a SIP request is received
    * **onreply_route** - is executed when a SIP reply is
    * **error_route** - is executed when some errors (mainly related to message parsing) are encountered
    * **failure_route** - is executed when a negative reply
was received for a transaction that armed the handler for the failure event.
    * **branch_route** - is executed for each destination branch
of the transaction that armed the handler for it.
* In the next section we will present how to add a core parameter and add a routing action -- core function.
You need to have knowledge of **flex** and **bison**.
# Adding a core parameter #
Some of the core parameters correspond to global variables in Kamailio sources. Others induce
actions to be taken during statup.

Let's follow step by step the definition and usage of the core parameter
**log_name**. It is a string parameter that specifies the 
value to be printed as application name in syslog messages.

First is the declaration of the variable in the C code. The **log_name**
is defined in [**main.c**](https://github.com/kamailio/kamailio/blob/5.3/src/main.c) and initialized to **0** (when set to **o**,
it is printed the Kamailio name (including path) in the syslog).

    ...
    char *log_name = 0;
    ...

Next is to define the token in the **flex** file: **cfg.lex**.

    ...
    LOGNAME		log_name
    ...

The association of a token ID and extending the grammar of the configuration file is done
in the **bison** file: **cfg.y**.

    ...
    %token LOGNAME
    ...
    assign_stm: ...
        | LOGNAME EQUAL STRING { log_name=$3; }
        | LOGNAME EQUAL error { yyerror("string value expected"); }
    ...

The grammar was extended with a new assign statement, that allows to write in the configuration
file an expression like:

    ...
    log_name = "kamailio123"
    ...

When having a line like above one in the configuration file, the variable
**log_name** in C code will be initialized to the string
in the right side of the equal operator.

# Adding a core function #

To introduce new functions in Kamailio core that are exported to the configuration file
the grammar have to be extended (**bison** and **flex** files), the interpreter must be enhanced to be
able to run the new actions.

Behind each core function resides an action structure. This data type is defined in [**route_struct.h**](https://github.com/kamailio/kamailio/blob/5.3/src/core/route_struct.h):

    ...
    typedef struct {
        action_param_type type;
        union {
            long number;
            char* string;
            struct _str str;
            void* data;
            avp_spec_t* attr;
            select_t* select;
        } u;
    } action_u_t;
    
    /* maximum internal array/params
     * for module function calls val[0] and val[1] store a pointer to the
     * function and the number of params, the rest are the function params 
    */
    #define MAX_ACTIONS (2+6)
    
    struct action{
        int cline;
        char *cfile;
        enum action_type type;  /* forward, drop, log, send ...*/
        int count;
        struct action* next;
        action_u_t val[MAX_ACTIONS];
    };
    ...

Each action is identified by a type. The types of actions are defined in same header
file. For example, **strip(...)** function has the type **STRIP_T**, the functions exported by
modules have the type **MODULE_T**.
To each action may be given a set of parameters, so called action elements. In case of
functions exported by modules, the first element is the pointer to the function, next are the
parameters given to that function in configuration file.

For debugging and error detection, the action keeps the line number in configuration file
where it is used.

Next we discuss how **setflag(...)** config function was implemented.

## Extending the grammar ##

Define the token in **flex** file: **cfg.lex**.

    ...
    SETFLAG			"setflag"
    ...

Assign a token ID and extend the <emphasis role="strong">bison</emphasis> grammar.

    <![CDATA[
    ...
    %token SETFLAG
    ...
    cmd:	...

    | SETFLAG LPAREN NUMBER RPAREN	{
                            if (check_flag($3)==-1)
                                yyerror("bad flag value");
                            $$=mk_action(SETFLAG_T, 1, NUMBER_ST,
                                                    (void*)$3);
                            set_cfg_pos($$);
                                    }
    | SETFLAG LPAREN flag_name RPAREN	{
                            i_tmp=get_flag_no($3, strlen($3));
                            if (i_tmp<0) yyerror("flag not declared");
                            $$=mk_action(SETFLAG_T, 1, NUMBER_ST,
                                        (void*)(long)i_tmp);
                            set_cfg_pos($$);
                                    }
    | SETFLAG error			{ $$=0; yyerror("missing '(' or ')'?"); }

    ...
    ]]>

First grammar specification says that **setflag(...)**
can have one parameter of type **number**. The other
rule for grammar is to detect error cases.

## Extending the interpreter ##
First step is to add a new action type in **route_struct.h**.

Then add a new case in the switch of action types, file
**action.c**, function

    <![CDATA[
    ...
        case SETFLAG_T:
            if (a->val[0].type!=NUMBER_ST) {
                LOG(L_CRIT, "BUG: do_action: bad setflag() type %d\n",
                    a->val[0].type );
                ret=E_BUG;
                goto error;
            }
            if (!flag_in_range( a->val[0].u.number )) {
                ret=E_CFG;
                goto error;
            }
            setflag( msg, a->val[0].u.number );
            ret=1;
            break;
    ...
    ]]>

The C function **setflag(...)** is defined and implemented
in **flags.{c,h}**. It simply sets the bit in **flags**
attribute of **sip_msg** at the position given by the parameter.

    ...
    int setflag( struct sip_msg* msg, flag_t flag ) {
    msg->flags |= 1 &lt;&lt; flag;
    return 1;
    }
    ...

We are not done yet. Kamailio does a checking of the actions tree after all configuration
file was loaded. It does sanity checks and optimization for run time. For our case, it does
a double-check that the parameter is a number and it is in the range of
**0...31** to fit in the bits size of an integer value. See
function **fix_actions(...)** in **route.c**.

Next example is given just to show how such fixup can look like, it is no longer used for
flag operations functions.

    ...
        case SETFLAG_T:
        case RESETFLAG_T:
        case ISFLAGSET_T:
            if (t->elem[0].type!=NUMBER_ST) {
                LM_CRIT("bad xxxflag() type %d\n", t->elem[0].type );
                ret=E_BUG;
                goto error;
            }
            if (!flag_in_range( t->elem[0].u.number )) {
                ret=E_CFG;
                goto error;
            }
            break;
    ...

Last thing you have to add is to complete the function **print(action(...)** with a new case for your action
that will be used to print the actions tree -- for debugging purposes. See it in file 
[**route_struct.c**](https://github.com/kamailio/kamailio/blob/5.3/src/core/route_struct.c).

    ...
        case SETFLAG_T:
                LM_DBG("setflag(");
                break;
    ...

From now on, you can use in your configuration file the function **setflag(_number_)**.
Don't forget to add documentation in [**Kamailio Core Cookbook**](https://www.kamailio.org/wiki/cookbooks/5.3.x/core).
