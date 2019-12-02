# Transformations #

The transformations are strictly related to pseudo-variables. They are properties that
can be associated to instances of pseudo-variables. A transformation produces changes to the
value returned by a pseudo-variable. There can be a chain of transformations assigned to same
instance of a pseudo-variable.

An example of a transformation is to get a substring from the value returned by a
pseudo-variable. Another one is to get the length of the values returned by a pseudo-variable.

The value of a pseudo-variable with a chain of transformations is evaluated as:
* get the value of the pseudo-variable
* apply the operation specified by the first transformation to the value returned by
the pseudo-variable
* apply the operation specified by the current transformation to the value returned by
the previous transformation. Go to next transformation.
* the value returned by the last transformation is returned to script or to calling C code.

Behind each transformation it is a function that does value (pv_value_t) manipulation. It has as input
such a value and as output another value, stored over the input value. The transformation are
implemented mainly in Kamailio modules, data structures and API are in file
**pvar.h**. Many transformations are effectively implemented in **modules_k/pv/pv_trans.{c,h}**.

# Naming Format #

The transformations are given in between the pharantesis around the parenthesis of a pseudo-variable.
The transformation name and parameters are enclosed in between curly brackets
**{ }**. The grammar for a transformation specifier:

    ...
    '{' classname '.' innername ( ',' parameter )* '}'
    ...

The tokens are:

classname - string identifying the class of transformation. For example:
* 's' - string transformations
* 'uri' - URI transformations
* 'param' - parameter transformations
* innername - string identifying the operation within the class of transformations
* parameter - the parameter to the transformation. There can be transformation with no
  parameters, one or more parameters.

Example of existing transformations:

    ...
    {s.substr,1,2} - return the string with second and the third characters (substring of length 2 from the
                 second character)
    {uri.user} -  return the user part of the SIP URI stored in the pseudo-variable on which this transformation
              is applied
    ...

# Data Structures #

Internally, to the classname of a transformation it is associated type, an integer number, and to
the innername a subtype, also an integer number. Constants and definitions are in file
**transformations.h**

    ...
    typedef struct _tr_param {
        int type;        /* type of the parameter value */
        union {
            int n;       /* the integer value of the parameter */
            str s;       /* the string value of the parameter */
            void *data;  /* pseudo-variable spec of the parameter */
    } v;
        struct _tr_param *next; /* link to next parameter */
    } tr_param_t, *tr_param_p;
    
    typedef int (*tr_func_t) (struct sip_msg *, tr_param_t*, int, pv_value_t*);
    
    typedef struct _trans {
        str name;               /* full name of the transformation */
        int type;               /* the id of the transformation class */
        int subtype;            /* the id of the transformation inner name */
        tr_func_t trf;          /* the function to be executed when applying the transformation */
        tr_param_t *params;     /* the parameters for this transformation */
        struct _trans *next;    /* link to next transformation to be applied to same pseudo-variable */
    } trans_t, *trans_p;
    ...

The parameters of the function to be executed for a transformation:

* msg - the SIP message currently processed
* param - the list with the parameters of the transformation
* subtype - the subtype of the transformation
* value - pointer to the value (pv_value_t) to apply transformation on it and store the result

There is one function for each transformation class, the exact operation to be applied is given
via subtype. The structure **trans_t** is what stands behind each
transformation occurrence.

# Adding a Transformation #

The transformation framework keeps for each transformation class a function to parse the innername
and the parameters, plus a function to execute when the transformation need to be applied. To add
new transformations, you have to implement the two functions.

Exemplifying with transformation **s.len** - it returns the length
of the string value of a pseudo-variable. The type **TR_STRING**
and subtype **TR_S_LEN** are added in file
**modules_k/pv/pv_trans.h** - these are internal IDs used for runtime optimizations.

The function **tr_parse_string()** in file
**modules_k/pv/pv_trans.c** is implementing the parser of the class
**s**. In this function, if the inner name is **len** it sets the subtype accordingly.

    ...
    if(name.len==3 &amp;&amp; strncasecmp(name.s, "len", 3)==0)
    {
        t->subtype = TR_S_LEN;
        return p;
    }
    ...
        </programlisting>
        <para>
            The transformation parser is now extended. Next is the interpreter, the function
            **tr_eval_str(...)** - the evaluation function will be
            associated to string transformation by the parser function
            **tr_parse_string()**.
        </para>
    switch(subtype)
    {
        case TR_S_LEN:
            if(!(val->flags&amp;PV_VAL_STR))
                val->rs.s = int2str(val-&gt;ri, &amp;val-&gt;rs.len);
                
                   val-&gt;flags = PV_TYPE_INT|PV_VAL_INT|PV_VAL_STR;
                   val-&gt;ri = val-&gt;rs.len;
                   val-&gt;rs.s = int2str(val-&gt;ri, &amp;val-&gt;rs.len);
                   break;
    ...

The content of the pv_value_t variable is replaced with the length of the string representation
of its initial value. An example of usage is shown next - get the length of request URI:

    ...
    $var(x) = $(ru{s.len});
    ...

To make the transformations available to the core API, you have to register them. You can do it
via **mod_register(...)** function. Note that this function is
executed when the module is loaded, making possible to use the transformation in module
parameters and configuration file operations. See **modules_k/pv/pv.c**:

    ...
    static tr_export_t mod_trans[] = {
        { {"s", sizeof("s")-1}, /* string class */
            tr_parse_string },
        { {"nameaddr", sizeof("nameaddr")-1}, /* nameaddr class */
            tr_parse_nameaddr },
        { {"uri", sizeof("uri")-1}, /* uri class */
            tr_parse_uri },
        { {"param", sizeof("param")-1}, /* param class */
            tr_parse_paramlist },
        { {"tobody", sizeof("tobody")-1}, /* param class */
            tr_parse_tobody },

    { { 0, 0 }, 0 }
    };
    ...
    int mod_register(char *path, int *dlflags, void *p1, void *p2)
    {
       return register_trans_mod(path, mod_trans);
    }
    ...
