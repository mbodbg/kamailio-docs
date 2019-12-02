Kamailio includes its own implementation of SIP parser. It is known
as **lazy** or **incremental** parser. That means
it parses until it founds the required elements or encounters ends
of SIP message.

All parsing functions and data structures are in the files from the
directory **parser**. The main file
for SIP message parsing is [**parser/msg_parser.c**](https://github.com/kamailio/kamailio/blob/5.3/src/core/parser/msg_parser.c) with the
corresponding header file [**parser/msg_parser.h**](https://github.com/kamailio/kamailio/blob/5.3/src/core/parser/msg_parser.h).

It does not parse entirely the parts of the SIP message. For most of
the SIP headers, it identifies the name and body, it does not parse
the content of header's body. It may happen that a header is malformed
and Kamailio does not report any error as there was no request to
parse that header body. However, most used headers are parsed entirely
by default. Such headers are top most Via, To, CSeq, Content-Lenght.

The parser does not duplicate the values, it makes references inside
the SIP message buffer it parses. For the **parsed** structures it allocates
private memory. It keeps the state of parsing, meaning that it has
an anchor to the beginning of unparsed part of the SIP message and
stores a bitmask of flags with parsed known headers. If the function
to parse a special header is called twice, the second time will return
immediately as it finds in the bitmask that the header was already
parsed.

This chapter in not intended to present all parsing functions, you
have to check the files in the directory **parser**. There is kind of naming
convention, so if you need the function to parse the header
**XYZ**, look for the files **parser/parse_XYZ.{c,h}**. If you
don't find it, second try is to use **ctags** to locate a
**parse_XYZ(...)** function. If no luck, then ask on Kamailio development mailing list
**devel@lists.kamailio.org**. Final solution in case on negative answer is to implement it yourself. For
example, CSeq header parser is in **parser/parse_cseq.{c,h}**.
The next sections will present the parsing functions
that give access to the most important parts of a SIP message.

# parse_uri(...) #

## Prototype ##

    ...
    int parse_uri(char *buf, int len, struct sip_uri* uri);
    ...

## Example of URI parser usage ##
    ...
	char *uri;
	struct sip_uri parsed_uri;

	uri = "sip:test@mydomain.com";

	if(parse_uri(uri, strlen(uri), &amp;parsed_uri)!=0)
	{
		LM_ERR("invalid uri [%s]\n", uri);
	} else {
		LM_DBG("uri user [%.*s], uri domain [%.*s]\n",
			parsed_uri.user.len, parsed_uri.user.s,
			parsed_uri.host.len, parsed_uri.host.s);
	}
    ...

# parse_msg(...) #

developer does not interfere too much with this function as it is called
automatically by Kamailio when a SIP message is received from the network.

You can use it if you load the content of SIP message from a file or database,
or you received on different channels, up to your extension implementation. You
should be aware that it is not enough to call this function and then run the
actions from the configuration file. There are some attributes in the structure
**sip_msg** that are specific to the environment:
received socket, source IP and port, ...

## Prototype ##

    ...
    int parse_msg(char* buf, unsigned int len, struct sip_msg* msg);
    ...

Return 0 if parsing was OK, &gt;0 if error occurred.

## Example of usage ##
    ...
    str msg_buf;
    struct sip_msg msg; 
    ...
    msg_buf.s = "INVITE sip:user@sipserver.com SIP/2.0\r\nVia: ...";
    msg_buf.len = strlen(msg_buf.s);

    if (parse_msg(buf,len, &amp;msg)!=0) {
	LM_ERR("parse_msg failed\n");
	return -1;
    }

    if(msg.first_line.type==SIP_REQUEST)
	LM_DBG("SIP method is [%.*s]\n", msg.first_line.u.request.method.len,
		msg.first_line.u.request.method.s);
    ...

# parse_headers(...) #

Parse the SIP message until the headers specified by parameter flags
**flags** are found. The parameter **next** can be used when a header can
occur many times in a SIP message, to continue parsing until a new header
of that type is found.

The values that can be used for **flags** are defined in **parser/hf.h**.

When searching to get a specific header, all the headers encountered during parsing
are hooked in the structure **sip_msg**.

## Prototype ##

    ...
    int parse_headers(struct sip_msg* msg, hdr_flags_t flags, int next);
    ...

Return 0 if parsing was sucessful, &gt;0 in case of error.

## Example of usage ##

    ...
    if(parse_headers(msg, HDR_CALLID_F, 0)!=0)
    {
	LM_ERR("error parsing CallID header\n");
	return -1;
    }
    ...

# parse_to(...) #

Parse a buffer that contains the body of a To header. The function is
defined in **parser/parse_to.h**.

## Prototype ##

    ...
    char* parse_to(char* buffer, char *end, struct to_body *to_b);
    ...

Return 0 in case of success, &gt;0 in case of error.

The next example shows the **parse_from(...)** function that makes use of **parse_to(...)** to
parse the body of header **From**. The function is located in file **parser/parse_from.c**.

The structure filled at parsing is hooked in the structure
**sip_msg**, inside the attribute **from**, which is the shortcut to the
header **From**.

## Example of usage ##

    ...
    <![CDATA[
    int parse_from_header( struct sip_msg *msg)
    {
	struct to_body* from_b;

	if ( !msg->from && ( parse_headers(msg,HDR_FROM_F,0)==-1 || !msg->from)) {
		LM_ERR("bad msg or missing FROM header\n");
		goto error;
	}

	/* maybe the header is already parsed! */
	if (msg->from->parsed)
		return 0;

	/* first, get some memory */
	from_b = pkg_malloc(sizeof(struct to_body));
	if (from_b == 0) {
		LM_ERR("out of pkg_memory\n");
		goto error;
	}

	/* now parse it!! */
	memset(from_b, 0, sizeof(struct to_body));
	parse_to(msg->from->body.s,msg->from->body.s+msg->from->body.len+1,from_b);
	if (from_b->error == PARSE_ERROR) {
		LM_ERR("bad from header\n");
		pkg_free(from_b);
		set_err_info(OSER_EC_PARSER, OSER_EL_MEDIUM, "error parsing From");
		set_err_reply(400, "bad From header");
		goto error;
	}
	msg->from->parsed = from_b;

	return 0;
    error:
	return -1;
    }
    ]]>
    ...

# Get Message Body #

Next example shows how to get the content of SIP message body in a 'str' variable.

## Example of usage ##

    ...
    int get_msg_body(struct sip_msg *msg, str *body)
    {
	/* 'msg' is a pointer to a valid struct sip_msg */

	/* get message body
	- after that whole SIP MESSAGE is parsed
	- calls internally parse_headers(msg, HDR_EOH_F, 0)
	*/
	body->s = get_body( msg );
	if (body->s==0) 
	{
		LM_ERR("cannot extract body from msg\n");
		return -1;
	}

	body->len = msg->len - (body->s - msg->buf);

	/* content-length (if present) must be already parsed */
	if (!msg->content_length) 
	{
		LM_ERR("no Content-Length header found!\n");
		return -1;
	}
	if(body->len != get_content_length( msg ))
		LM_WARN("Content length header value different than body size\n");
	return 0;
    }
    ...

# Get Header Body #

You can see in the next example how to access the body of header **Call-ID**.

## Example of usage ##

  ...
  void print_callid_header(struct sip_msg *msg)
  {
	if(msg==NULL)
		return;
	if(parse_headers(msg, HDR_CALLID_F, 0)!=0)
	{
		LM_ERR("error parsing CallID header\n");
		return;
	}
		
	if(msg->callid==NULL || msg->callid->body.s==NULL)
	{
		LM_ER("NULL call-id header\n");
		return;
	}
	LM_INFO("Call-ID: %.*s\n", msg->callid->body.len, msg->callid->body.s);
    }
    ...

# New Header Parsing Function #

The section gives the guidelines to add a new function for parsing a header.
* add source and header files in directory **parser** naming them **parse_hdrname.{c,h}**.
* if the header is used very often, consider doing speed optimization by allocating
a header type and flag. That will allow to identify the header via integer comparison
after the header was parsed and added in **headers**
list in the structure **sip_msg**.
* make sure you add the code to properly clean up the header structure when the
structure **sip_msg** is freed.
* make sure that the **tm** module properly clones
the header or it resets the pointers to the header when copying the structure
**sip_msg** in shared memory.
