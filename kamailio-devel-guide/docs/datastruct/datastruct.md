In this chapter we focus on most used data structures inside
Kamailio sources. Most of them relate to SIP message structure.
Other important data structures are explained in the chapters
detailing specific components or functionalities -- for example,
see **Database API** or **Pseudo-variables** chapters.

# str #

sip is a text-based protocol, therefore lot of operations
resume to text manipulation. Kamailio uses references in the
sip message body most of the time, doing it via an anchor
pointer and the length. For that it uses the
**str** structure.

The **str** structure is defined in file **str.h**.

## Definition ##

    ...
    struct _str{
    char* s; /* pointer to the beginning of string (char array) */
    int len; /* string length */
    };

    typedef struct _str str;
    ...

## Example of usage ##

    ...
    #include "str.h"
    ...
    str s;
    s.s = "kamailio";
    s.len = strlen(s.s);
    LM_DBG("the string is [%.*s]\n", s.len, s.s);
    ...

# struct sip_uri #

This is the structure holding a parsed SIP URI. You can fill it
by calling **parse_uri(...)** function.

The structure is defined in file [**parser/msg_parser.h**](https://github.com/kamailio/kamailio/blob/5.3/src/core/parser/msg_parser.h).

## Definition ##

    ...	
    struct sip_uri {
    str user;     /* Username */
    str passwd;   /* Password */
    str host;     /* Host name */
    str port;     /* Port number */
    str params;   /* URI Parameters */
    str headers;  /* URI Headers */
    unsigned short port_no; /* Port number r*/
    unsigned short proto; /* Transport protocol */
    uri_type type; /* URI scheme */
    /* parameters */
    str transport;   /* transport parameter */
    str ttl;         /* ttl parameter */
    str user_param;  /* user parameter */
    str maddr;       /* maddr parameter */
    str method;      /* method parameter */
    str lr;          /* lr parameter */
    str r2;          /* specific rr parameter */
    /* values */
    str transport_val;  /* value of transport parameter */
    str ttl_val;        /* value of ttl parameter */
    str user_param_val; /* value of user parameter */
    str maddr_val;      /* value of maddr parameter */
    str method_val;     /* value of method parameter */
    str lr_val;         /* value of lr parameter */
    str r2_val;         /* value of r2 parameter */
    };
    ...

Members of the structure corresponds to a part of a SIP URI. To get details about
the format of SIP URI read **RFC3261**. Example of SIP URI:
**sip:alice@sipserver.org:5060;transport=tcp**

# struct sip_msg #
This is the main structure related to a SIP message. When a SIP message is received from
the network, it is parsed in such structure. The pointer to this structure is given as
parameter to all functions exported by modules to be used in the configuration file.

The structure is defined in the file **parser/msg_parser.h**.
    
#    # Definition ##
     
        ...
     typedef struct sip_msg {
        unsigned int id;               /*!< message id, unique/process*/
        int pid;                       /*!< process id */
        struct timeval tval;           /*!< time value associated to message */
        snd_flags_t fwd_send_flags;    /*!< send flags for forwarding */
        snd_flags_t rpl_send_flags;    /*!< send flags for replies */
        struct msg_start first_line;   /*!< Message first line */
        struct via_body* via1;         /*!< The first via */
        struct via_body* via2;         /*!< The second via */
        struct hdr_field* headers;     /*!< All the parsed headers*/
        struct hdr_field* last_header; /*!< Pointer to the last parsed header*/
        hdr_flags_t parsed_flag;    /*!< Already parsed header field types */
    
        /* Via, To, CSeq, Call-Id, From, end of header*/
        /* pointers to the first occurrences of these headers;
         * everything is also saved in 'headers'
         * (WARNING: do not deallocate them twice!)*/
    
        struct hdr_field* h_via1;
        struct hdr_field* h_via2;
        struct hdr_field* callid;
        struct hdr_field* to;
        struct hdr_field* cseq;
        struct hdr_field* from;
        struct hdr_field* contact;
        struct hdr_field* maxforwards;
        struct hdr_field* route;
        struct hdr_field* record_route;
        struct hdr_field* content_type;
        struct hdr_field* content_length;
        struct hdr_field* authorization;
        struct hdr_field* expires;
        struct hdr_field* proxy_auth;
        struct hdr_field* supported;
        struct hdr_field* require;
        struct hdr_field* proxy_require;
        struct hdr_field* unsupported;
        struct hdr_field* allow;
        struct hdr_field* event;
        struct hdr_field* accept;
        struct hdr_field* accept_language;
        struct hdr_field* organization;
        struct hdr_field* priority;
        struct hdr_field* subject;
        struct hdr_field* user_agent;
        struct hdr_field* server;
        struct hdr_field* content_disposition;
        struct hdr_field* diversion;
        struct hdr_field* rpid;
        struct hdr_field* refer_to;
        struct hdr_field* session_expires;
        struct hdr_field* min_se;
        struct hdr_field* sipifmatch;
        struct hdr_field* subscription_state;
        struct hdr_field* date;
        struct hdr_field* identity;
        struct hdr_field* identity_info;
        struct hdr_field* pai;
        struct hdr_field* ppi;
        struct hdr_field* path;
        struct hdr_field* privacy;
        struct hdr_field* min_expires;
    
        struct msg_body* body;
    
        char* eoh;        /*!< pointer to the end of header (if found) or null */
        char* unparsed;   /*!< here we stopped parsing*/
    
        struct receive_info rcv; /*!< source & dest ip, ports, proto a.s.o*/
    
        char* buf;        /*!< scratch pad, holds a modified message,
                            *  via, etc. point into it */
        unsigned int len; /*!< message len (orig) */
    
        /* modifications */
    
        str new_uri; /*!< changed first line uri, when you change this
                        don't forget to set parsed_uri_ok to 0*/
    
        str dst_uri; /*!< Destination URI, must be forwarded to this URI if len != 0 */
    
        /* current uri */
        int parsed_uri_ok; /*!< 1 if parsed_uri is valid, 0 if not, set if to 0
                            if you modify the uri (e.g change new_uri)*/
        struct sip_uri parsed_uri; /*!< speed-up > keep here the parsed uri*/
        int parsed_orig_ruri_ok; /*!< 1 if parsed_orig_uri is valid, 0 if not, set if to 0
                                    if you modify the uri (e.g change new_uri)*/
        struct sip_uri parsed_orig_ruri; /*!< speed-up > keep here the parsed orig uri*/
    
        struct lump* add_rm;       /*!< used for all the forwarded requests/replies */
        struct lump* body_lumps;     /*!< Lumps that update Content-Length */
        struct lump_rpl *reply_lump; /*!< only for localy generated replies !!!*/
    
        /*! \brief str add_to_branch;
            whatever whoever want to append to Via branch comes here */
        char add_to_branch_s[MAX_BRANCH_PARAM_LEN];
        int add_to_branch_len;
    
        unsigned int  hash_index; /*!< index to TM hash table; stored in core
                                    to avoid unnecessary calculations */
        unsigned int msg_flags; /*!< internal flags used by core */
        flag_t flags; /*!< config flags */
        flag_t xflags[KSR_XFLAGS_SIZE]; /*!< config extended flags */
        str set_global_address;
        str set_global_port;
        struct socket_info* force_send_socket; /*!< force sending on this socket */
        str path_vec;
        str instance;
        unsigned int reg_id;
        str ruid;
        str location_ua;
        int otcpid; /*!< outbound tcp connection id, if known */
    
        /* structure with fields that are needed for local processing
         * - not cloned to shm, reset to 0 in the clone */
        msg_ldata_t ldv;
    
        /* IMPORTANT: when adding new fields in this structure (sip_msg_t),
         * be sure it is freed in free_sip_msg() and it is cloned or reset
         * to shm structure for transaction - see sip_msg_clone.c. In tm
         * module, take care of these fields for faked environemt used for
         * runing failure handlers - see modules/tm/t_reply.c */
}     sip_msg_t;
       ...
    
To fill such structure you can use function **parse_msg(...)** giving a buffer containing raw text of a SIP message. Most of the attributes in this structure
point directly inside the SIP message buffer.
    
#    # Example of a SIP message: ##
    
    ...
    REGISTER sip:sip.test.com SIP/2.0
    Via: SIP/2.0/UDP 192.168.1.3:5061;branch=z9hG4bK-d663b80b
    Max-Forwards: 70
    From: user &lt;sip:u123@sip.test.com&gt;;tag=ea8cef4b108a99bco1
    To: user &lt;sip:u123@sip.test.com&gt;
    Call-ID: b96fead3-f03493d4@xyz
    CSeq: 3720 REGISTER
    Contact: user &lt;sip:u123@192.168.1.3:5061&gt;;expires=3600
    User-Agent: Linksys/RT31P2-2.0.10(LIc)
    Content-Length: 0
    Allow: ACK, BYE, CANCEL, INFO, INVITE, NOTIFY, OPTIONS, REFER
    Supported: x-sipura
    ...

# struct msg_start #

The structure corresponds to a parsed representation of the first line in a SIP message.
It is defined in file [**parser/parse_fline.h**](https://github.com/kamailio/kamailio/blob/5.3/src/core/parser/parse_fline.h).

## Definition ##

    ...
     typedef struct msg_start {
        short type;					/*!< Type of the message - request/response */
        short flags;				/*!< First line flags */
        int len; 					/*!< length including delimiter */
        union {
            struct {
                str method;			/*!< Method string */
                str uri;			/*!< Request URI */
                str version;		/*!< SIP version */
                int method_value;
            } request;
            struct {
                str version;		/*!< SIP version */
                str status;			/*!< Reply status */
                str reason;			/*!< Reply reason phrase */
                unsigned int statuscode;	/*!< Reply status code */
            } reply;
        }u;
    } msg_start_t;
    ...

To parse a buffer containing the first line of a SIP message you have to use
the function **parse_fline(...)**.

# struct hdr_field #

The structure holding a parsed SIP header. It is defined in file [**parser/hf.h**](https://github.com/kamailio/kamailio/blob/5.3/src/core/parser/hf.h).

## Definition ##

    ...
     typedef struct hdr_field {
        hdr_types_t type;       /*!< Header field type */
        str name;               /*!< Header field name */
        str body;               /*!< Header field body (may not include CRLF) */
        int len;                /*!< length from hdr start until EoHF (incl.CRLF) */
        void* parsed;           /*!< Parsed data structures */
        struct hdr_field* next; /*!< Next header field in the list */
    } hdr_field_t;
    ...

To parse specific headers in a SIP message you have to use the function
**parse_headers(...)**. The function takes as parameter
a bitmask flag that can specify what headers you need to be parsed. For example, to parse
the From and To headers: **parse_headers(msg, HDR_FROM_F|HDR_TO_F, 0);**
To optimize the operations with headers, an integer value is assigned to most used headers.
This value is stored in attribute **type**. Here is the list with the values for header type:

    ...
    enum _hdr_types_t {
        HDR_ERROR_T					= -1   /*!< Error while parsing */,
        HDR_OTHER_T					=  0   /*!< Some other header field */,
        HDR_VIA_T					=  1   /*!< Via header field */,
        HDR_VIA1_T					=  1   /*!< First Via header field */,
        HDR_VIA2_T					=  2   /*!< only used as flag */,
        HDR_TO_T					       /*!< To header field */,
        HDR_FROM_T					       /*!< From header field */,
        HDR_CSEQ_T					       /*!< CSeq header field */,
        HDR_CALLID_T				       /*!< Call-Id header field */,
        HDR_CONTACT_T				       /*!< Contact header field */,
        HDR_MAXFORWARDS_T			       /*!< MaxForwards header field */,
        HDR_ROUTE_T					       /*!< Route header field */,
        HDR_RECORDROUTE_T			       /*!< Record-Route header field */,
        HDR_CONTENTTYPE_T			       /*!< Content-Type header field */,
        HDR_CONTENTLENGTH_T			       /*!< Content-Length header field */,
        HDR_AUTHORIZATION_T			       /*!< Authorization header field */,
        HDR_EXPIRES_T				       /*!< Expires header field */,
        HDR_MIN_EXPIRES_T			       /*!< Min-Expires header */,
        HDR_PROXYAUTH_T				       /*!< Proxy-Authorization hdr field */,
        HDR_SUPPORTED_T				       /*!< Supported  header field */,
        HDR_REQUIRE_T				       /*!< Require header */,
        HDR_PROXYREQUIRE_T			       /*!< Proxy-Require header field */,
        HDR_UNSUPPORTED_T			       /*!< Unsupported header field */,
        HDR_ALLOW_T					       /*!< Allow header field */,
        HDR_EVENT_T					       /*!< Event header field */,
        HDR_ACCEPT_T				       /*!< Accept header field */,
        HDR_ACCEPTLANGUAGE_T		       /*!< Accept-Language header field */,
        HDR_ORGANIZATION_T			       /*!< Organization header field */,
        HDR_PRIORITY_T				       /*!< Priority header field */,
        HDR_SUBJECT_T				       /*!< Subject header field */,
        HDR_USERAGENT_T				       /*!< User-Agent header field */,
        HDR_SERVER_T				       /*!< Server header field */,
        HDR_CONTENTDISPOSITION_T	       /*!< Content-Disposition hdr field */,
        HDR_DIVERSION_T				       /*!< Diversion header field */,
        HDR_RPID_T					       /*!< Remote-Party-ID header field */,
        HDR_REFER_TO_T				       /*!< Refer-To header fiels */,
        HDR_SIPIFMATCH_T			       /*!< SIP-If-Match header field */,
        HDR_SESSIONEXPIRES_T		       /*!< Session-Expires header */,
        HDR_MIN_SE_T				       /*!< Min-SE */,
        HDR_SUBSCRIPTION_STATE_T	       /*!< Subscription-State */,
        HDR_ACCEPTCONTACT_T			       /*!< Accept-Contact header */,
        HDR_ALLOWEVENTS_T			       /*!< Allow-Events header */,
        HDR_CONTENTENCODING_T		       /*!< Content-Encoding header */,
        HDR_REFERREDBY_T			       /*!< Referred-By header */,
        HDR_REJECTCONTACT_T			       /*!< Reject-Contact header */,
        HDR_REQUESTDISPOSITION_T	       /*!< Request-Disposition header */,
        HDR_WWW_AUTHENTICATE_T		       /*!< WWW-Authenticate header field */,
        HDR_PROXY_AUTHENTICATE_T	       /*!< Proxy-Authenticate header field */,
        HDR_DATE_T                         /*!< Date header field */,
        HDR_IDENTITY_T                     /*!< Identity header field */,
        HDR_IDENTITY_INFO_T                /*!< Identity-info header field */,
        HDR_RETRY_AFTER_T		           /*!< Retry-After header field */,
        HDR_PPI_T                          /*!< P-Preferred-Identity header field*/,
        HDR_PAI_T                          /*!< P-Asserted-Identity header field*/,
        HDR_PATH_T                         /*!< Path header field */,
        HDR_PRIVACY_T                      /*!< Privacy header field */,
        HDR_REASON_T                       /**< Reason header field */,
        HDR_CALLINFO_T                     /*!< Call-Info header field*/,
        HDR_EOH_T                          /*!< End of message header */
    };
    ...

If the type of hdr_field structure is **HDR_TO_T** it is the parsed **To** header.

The attribute **parsed** may hold the parsed representation
of the header body. For example, for **Content-Lenght** header it contains the content length value as integer.

# struct to_body #

The structure holds a parsed To header. Same structure is used for From header and the other
headers that have same structure conform to IETF RFCs. The structure is defined in file
[**parser/parse_to.h**](https://github.com/kamailio/kamailio/blob/5.3/src/core/parser/parse_to.h).

## Definition ##

    ...
    struct to_body{
    int error;                    /* Error code */
    str body;                     /* The whole header field body */
    str uri;                      /* URI withing the body of the header */
    str display;                  /* Display Name */
    str tag_value;                /* Value of tag parameter*/
    struct sip_uri parsed_uri;    /* Parsed URI */
    struct to_param *param_lst;   /* Linked list of parameters */
    struct to_param *last_param;  /* Last parameter in the list */
    };
    ...

To parse a To header you have to use function **parse_to(...)**.

# struct via_body #

The structure holds a parsed Via header. It is defined in file [**parse_via.h**](https://github.com/kamailio/kamailio/blob/5.3/src/core/parser/parse_via.h).

## Definition ##

    ...
    struct via_body { 
        int error;          /* set if an error occurred during parsing */
        str hdr;        /* header name "Via" or "v" */
        str name;       /* protocol name */
        str version;    /* protocol version */
        str transport;  /* transport protocol */
        str host;       /* host part of Via header */
        unsigned short proto; /* transport protocol as integer*/
        unsigned short port;  /* port number as integer */
        str port_str;         /* port number as string*/
        str params;           /* parameters */
        str comment;          /* comment */
        unsigned int bsize;           /* body size, not including hdr */
        struct via_param* param_lst;  /* list of parameters*/
        struct via_param* last_param; /*last via parameter, internal use*/
        
        /* shortcuts to "important" params*/
        struct via_param* branch;     /* branch parameter */
        str tid;                      /* transaction id, part of branch */
        struct via_param* received;   /* received parameter */
        struct via_param* rport;      /* rport parameter */
        struct via_param* i;          /* i parameter */
        struct via_param* alias;      /* alias see draft-ietf-sip-connect-reuse-00 */
        struct via_param* maddr;      /* maddr parameter */
        struct via_body* next;        /* pointer to next via body string if compact Via or null */
    };
    ...

The str attributes in the structure are referenced to SIP message buffer. To parse a Via
header you have to use the function **parse_via(...)**.
