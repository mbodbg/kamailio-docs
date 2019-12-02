# Data Lumps #

topic that won't be involved very much in the development of new features, but important
to understand as it is one of the most discussed issues related to Kamailio.

Many are surprised to discover that even they remove a header from the original SIP message,
in the configuration file, when they test later for header existence it is still then. Why?
Because the data lumps are behind the remove operations.

The modifications to the original SIP message instructed from configuration file are not applied
immediately. They are actually added in a list of operations to be done. Practically, changing
the content of the SIP message from the configuration file translates in creating a diff
(like in diff/patch tools from Unix/Linux systems) and putting it in the list. The diffs
are applied after the configuration file is executed, before sending the SIP message further
to the network.

Note that in config of Kamailio; 3.x, you can use msg_apply_changes() exported by textopsx
module to apply immediately the changes done to the content of the SIP message.

There are two types of diff operations:

* add content - this diff command is specified by the position in the original SIP
message and the value to be added there. The value can be a static string, or a
special marker to be interpreted later, when required information is available. For
the later, it is the case of Record-Route headers where it is needed to set the
IP address of the socket that is used to send the message further. That address
is detected just before sending through the socket.

* remove content - this diff command is specified by the position in the original SIP
message and the length to be deleted.

There are two classes of data lumps:

* message lumps - these lumps are associated to the SIP message currently processed. The
* changes incurred by these lumps are visible when the SIP message is forwarded.

reply lumps - these lumps can be used when the processed SIP message is a request
and you want to add content to the reply to be sent for that request. Here cannot be
lumps that delete content as the SIP reply is to be constructed from the SIP
request.

Maybe future releases will touch deeper this subject, depending on the interest. If you want to
investigate by yourself, start with files **data_lump.{c,h}** and **data_lump_rpl.{c,h}**.
