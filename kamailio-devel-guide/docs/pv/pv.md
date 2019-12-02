# Pseudo-variables #

Why this name? Yes, they are kind of variables, but a bit different:

* some are read-only - most of them are read-only, because they
are references inside the original SIP message and that does not change
during config file execution (see **Data Lump** chapter.)

* some are of the type array - even if looks as simple variable name, assigning a value means
to add one more in an array - the case for $avp(name).

So, they were named pseudo-variable. Initially they were introduced in
emphasis role="strong">xlog** module having a simple mechanism behind. There is
marker character ($ in this case) to identify the start of pseudo-variable name from the
rest of the text. To a pseudo-variable name was associated a function that returned a string
value. That value was replacing the pseudo-variable name in the message printed to syslog.

Lately the concept was extended to include AVPs, to have writable pseudo-variables and to
be possible to use them directly in the configuration file. Also, some of them can have dynamic
name and index.

The framework for pseudo-variables is now easy to extend. They can be introduced as core
pseudo-variables or exported by modules (this is preferred option). We will show such case
further in the document.

# Naming Format #

The naming format for a pseudo-variable is described in the next example:

    ...
    marker classname
    marker '(' classname ')'
    marker '(' classname '[' index ']' ')'
    marker '(' classname ( '{' transformation '}' )+ ')'
    marker '(' classname '[' index ']' ( '{' transformation '}' )+ ')'
    marker classname '(' innername ')'
    marker '(' classname '(' innername ')' ')'
    marker '(' classname '(' innername ')' '[' index ']' ')'
    marker '(' classname '(' innername ')' ( '{' transformation '}' )+ ')'
    marker '(' classname '(' innername ')' '[' index ']' ( '{' transformation '}' )+ ')'
    ...

Meaning of the tokens:

* marker - it is the char **$**
* classname - a string specifying the class of pseudo-variables. A class can have on
  pseudo-variable, when the innername is missing. Lot of references to parts of
  SIP message are single PV class, e.g., **$ru**
- request URI.
* Classes with more than one pseudo-variable are AVPs ($avp(name)), references to headers
($hdr(name)), script vars ($var(name)) and several others. 
* innername - the name specifying the pseudo-variable in a class. It can be dynamic
(specified by the value of another pseudo-variable), but depends on the pseudo-variable
class, it is not valid for all classes.
*index - index in the array, when the pseudo-variable can have multiple values at
the same time. Such pseudo-variables are header references and AVPs. Also the index can
have dynamic value, up to pseudo-variable class implementation.
* transformation - kind of function that are applied to the value of the pseudo-variable.
A dedicated chapter is included in this tutorial.

Some examples with pseudo-variable names:

    ...
    $ru - reference to request URI
    $(ru) - same as above - this format can be used when the class name cannot be delimited. It must
        be used if the PV has index or transformations.
    $avp(s:test) - the AVP with the string name 'test'
    $(avp(test)[2]) - the third AVP with the string name 'test'
    ...

## Data structures ##

The prototypes and data structures for pseudo-variables are defined in **pvar.h**.

## Type pv_value_t ##

Is the structure returned by the **get** function
associated to a pseudo-variable. It includes the flags that describe the value. It
can have integer and string value, in most of the case, the string value is all the
time set as PV used to be involved in string operations.

    ...
    typedef struct _pv_value
    {
        str rs;    /* string value */
        int ri;    /* integer value */
        int flags; /* flags about the type of value */
    } pv_value_t, *pv_value_p;
    ...

The type can be a combination of the following flags:

    ...
    #define PV_VAL_NONE			0    // no actual value -- it is so just at initialization
    #define PV_VAL_NULL			1    // the value must be considered NULL
    #define PV_VAL_EMPTY		2    // the value is an empty string (deprecated)
    #define PV_VAL_STR			4    // the value has the string attribute 'rs' set
    #define PV_VAL_INT			8    // the value has the integer attribute 'ri' set
    #define PV_TYPE_INT			16   // the value may have both string and integer attribute set, but type is integer
    #define PV_VAL_PKG			32   // the value was duplicated in pkg memory, free it accordingly at destroy time
    #define PV_VAL_SHM			64   // the value was duplicated in shm memory, free it accordingly at destroy time
    ...

# Type pv_name_t #

The structure to store the specifications for innername. Can be integer or string (e.g.,
for AVPs). It can be a pointer to another pseudo-variable specifier or something else,
up to implementation.

    ...
    typedef struct _pv_name
    {
        int type;             /* type of name */
        union {
		struct {
			int type;     /* type of int_str name - compatibility with AVPs */
			int_str name; /* the value of the name */
		} isname;
		void *dname;      /* PV value - dynamic name */
	} u;
    } pv_name_t, *pv_name_p;
    ...

Type can be:

    ...
    #define PV_NAME_INTSTR	0 // the name is constant, an integer or string
    #define PV_NAME_PVAR	1 // the name is dynamic, a pseudo-variable
    #define PV_NAME_OTHER	2 // the name is specific per implementation
    ...

Type for **isname** can be:

    ...
    0                                       // the name is integer
    #define AVP_NAME_STR     (1&lt;&lt;0)   // 1 - the name is string
    ...

## Type pv_index_t ##

The structure holding index specifications.

    ...
    typedef struct _pv_index
    {
        int type; /* type of PV index */
        union {
            int ival;   /* integer value */
            void *dval; /* PV value - dynamic index */
        } u;
    } pv_index_t, *pv_index_p;
    ...

Type can be:

    ...
    #define PV_IDX_INT	0  // the index is integer value
    #define PV_IDX_PVAR	1  // the index is dynamic, a pseudo-variable
    #define PV_IDX_ALL	2  // the index specifies to return all values for that pseudo-variables
                       // - this is up to implementation of pseudo-variable class
    ...

   ## Type pv_param_t ##
   The structure groups the name and the index to be easy to give them as parameter
   of the functions that requires them.

    ...
    typedef struct _pv_param
    {
        pv_name_t    pvn; /* PV name */
        pv_index_t   pvi; /* PV index */
    } pv_param_t, *pv_param_p;
    ...

## Type pv_spec_t ##

The structure that describes the pseudo-variable - the PV spec. It includes
a type of pseudo-variable, the functions to **get** and
**set** the pseudo-variable value, the parameter with
name and index specifiers and the list to transformations associated for that specific
instance of the pseudo-variable.

    ...
    typedef int (*pv_getf_t) (struct sip_msg*,  pv_param_t*, pv_value_t*);
    typedef int (*pv_setf_t) (struct sip_msg*,  pv_param_t*, int, pv_value_t*);

    typedef struct _pv_spec {
        pv_type_t    type;   /* type of PV */
        pv_getf_t    getf;   /* get PV value function */
        pv_setf_t    setf;   /* set PV value function */
        pv_param_t   pvp;    /* parameter to be given to get/set functions */
        void         *trans; /* transformations */
    } pv_spec_t, *pv_spec_p;
    ...

The types are defined in **pvar.h** and are used
to detect the type of core variables. Sometime is useful to filter out pseudo-variables,
for example when you want to allow only some type as parameter to a function or module,
e.g., only AVPs.

Such structure resides behind each occurrence of a pseudo-variable in configuration file.

## Type pv_export_t ##

The structure that one has to fill in order to add a new pseudo-variable. There is an array
of such objects in the core, in file **pvar.c**, and it
is possible to export such structure with the module interface.

    ...
    typedef int (*pv_parse_name_f)(pv_spec_p sp, str *in);
    typedef int (*pv_parse_index_f)(pv_spec_p sp, str *in);
    typedef int (*pv_init_param_f)(pv_spec_p sp, int param);

    typedef struct _pv_export {
        str name;                      /* class name of PV */
        pv_type_t type;                /* type of PV */
        pv_getf_t  getf;               /* function to get the value */
        pv_setf_t  setf;               /* function to set the value */
        pv_parse_name_f parse_name;    /* function to parse the inner name */
        pv_parse_index_f parse_index;  /* function to parse the index of PV */
        pv_init_param_f init_param;    /* function to init the PV spec */
        int iparam;                    /* parameter for the init function */
    } pv_export_t;
    ...

Practically, to add the pseudo-variable you have to give the classname and 
implement the functions to:

* get the value of the pseudo-variable (required)
* set the value of the pseudo-variable (optional)
* parse the inner name (optional)
* parse the index (optional)
* initialize the pseudo-variable spec (optional)

The optional attributes can be left NULL. **iparam**
is used together with function **init_param**.

## Adding a pseudo-variables ##

We will show how to add a simple pseudo-variable in the core, later will show how to add
a pseudo-variable with a inner name via module interface. **$ru**
(request URI) is taken as example. This pseudo-variable is read/write, so we have to implement
the **get** and **set** functions. These are in file **module_k/pv/pv_core.c**.

    <![CDATA[
    ...
    static int pv_get_ruri(struct sip_msg *msg, pv_param_t *param,
                 pv_value_t *res)
    {
        if(msg==NULL || res==NULL)
            return -1;

        if(msg->first_line.type == SIP_REPLY)	/* REPLY doesn't have a ruri */
            return pv_get_null(msg, param, res);

        if(msg->parsed_uri_ok==0 /* R-URI not parsed*/ && parse_sip_msg_uri(msg)<0)
        {
            LM_ERR("failed to parse the R-URI\n");
            return pv_get_null(msg, param, res);
        }
        
        if (msg->new_uri.s!=NULL)
            return pv_get_strval(msg, param, res, &msg->new_uri);
    return pv_get_strval(msg, param, res, &msg->first_line.u.request.uri);
    }
    ...
    int pv_set_ruri(struct sip_msg* msg, pv_param_t *param,
        int op, pv_value_t *val)
    {
    struct action  act;
    struct run_act_ctx h;
    char backup;
    
    if(msg==NULL || param==NULL || val==NULL || (val->flags&PV_VAL_NULL))
    {
        LM_ERR("bad parameters\n");
        return -1;
    }
    
    if(!(val->flags&PV_VAL_STR))
    {
        LM_ERR("str value required to set R-URI\n");
        goto error;
    }
    
    memset(&act, 0, sizeof(act));
    act.val[0].type = STRING_ST;
    act.val[0].u.string = val->rs.s;
    backup = val->rs.s[val->rs.len];
    val->rs.s[val->rs.len] = '\0';
    act.type = SET_URI_T;
    init_run_actions_ctx(&h);
    if (do_action(&h, &act, msg)<0)
    {
        LM_ERR("do action failed\n");
        val->rs.s[val->rs.len] = backup;
        goto error;
    }
    val->rs.s[val->rs.len] = backup;
    
    return 0;
    error:
        return -1;
    }
    ...
    ]]>

The parameters to the functions are:

* msg - the SIP message currenty processed
* param - the param field from PV spec structure
* op - the assign operation for **set** function
* value - pointer to a pv_value_t structure. It is out parameter for
**get** function and in parameter for
**set** function.

The functions return **0** in case of success and &lt;0 in case of error.

In the **get** function, it checks whether there is a new 
request URI values and return that. If not, returns the URI from the original SIP message.
It takes care that the URI is parsed, so it is valid.

In the **set** function, it checks to be sure that the value
to assign is a string, and then calls the internal SET_URI_T action.

the last step is to add the proper entry in the pseudo-variables table (see
**modules_k/pv/pv.c**) -- remember that this is required only for
pseudo-variables included in core, not for the ones exported by modules.

    ...
    static pv_export_t mod_pvs[] = {

    ...
        {{"ruri", (sizeof("ruri")-1)}, /* */
            PVT_RURI, pv_get_ruri, pv_set_ruri,
            0, 0, 0, 0},
    ...
        {{0,0}, 0, 0, 0, 0, 0, 0, 0}
    };
    ...

... and do not forget to document in the [**Pseudo-Variable Cookbok**](http://www.kamailio.org/wiki/cookbooks/5.3.x/pseudovariables).
