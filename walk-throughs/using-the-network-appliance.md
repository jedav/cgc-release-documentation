# Using the Network Appliance

The network security appliance provided within CGC, cb-proxy, is an application layer inspection and modification engine designed to sit between a client and Challenge Binary (CB).
Cb-proxy is capable of performing bi-directional content inspection and modification akin to a Network Intrusion Detection system, without the large learning curve of handling the communication underneath the application layer.
Inspection and modification is performed via custom domain-specific-language similar in nature to the Snort rules language, with the ability to provide multiple rules for a CB.

# Inspection Process

As a message is read, it is appended to a unique buffer for the side of the session being analyzed, then the buffer is inspected.
The inspection iterates over all of the provided rules until none of the rules match the remaining buffer or the buffer is empty.
When a rule successfully completes, the content inspected by the rule is removed from the buffer.
A rule is successful if all of the inspection options evaluate successfully.

After the rule matching process is complete, the message is sent to the destination.

# Rule Syntax

A rule is made up of a rule type, then rule options specified in within a parenthesis, within a single line.
Rule options are specified as a keyword and value pair, with a unique syntax for the value depending on the keyword.
A rule option is specified by the keyword, a colon, the value, and a semi-colon.

A basic rule looks like the following:

    alert (name:"foo"; match:"bar";)

The supported rule types are:
* alert
* admit
* block

An "alert" rule will log the content for analysis upon completion.  An "admit" rule will pass the inspected content on to its destination without logging it.  A "block" rule will cause prevent any further communication from occurring.

The supported rule options are:

* name
* match
* skip
* regex
* side
* state
* flush

# name

Each rule must have one and only one 'name' option, which is used in associated logs generated by 'block' and 'alert' rules.
The 'name' option must come first in the set of options.

The syntax for a 'name' option is an arbitrary string within double quotes.

An example 'name' option is:

    name:"This is a fancy rule name";

# match

Each rule may have an arbitrary number of 'match' options, which are used to perform string matching in the inspection buffer.

The syntax for a 'match' value is a quoted string optionally followed by a comma and then a positive integer.  
The quoted string must be made up of alphanumeric characters, space, or "\x" encoded 2 byte hex-encoded characters.
The optional comma and positive integer indicate that the specified string must exist within the specified amount of data.

An example 'match' option is:

    match:"foo";

An example 'match' option with the optional integer, is:

    match:"bar", 3;

The first option would look for "foo" anywhere in the inspection buffer, and mark all content up to and including the first instance of "foo" as inspected upon identification of match.
The second option would look for "foo" in the first 3 bytes of the inspection buffer, and the 3 bytes as inspected upon match.

# skip

Each rule may have any number of 'skip' options, which are used to skip over the specified amount of content in the inspection buffer.
The 'skip' option allows a rule to indicate content has been inspected without actually performing any inspection.

The syntax for a 'skip' value is a positive integer.

An example 'skip' option is:

    skip:10;

Any content matched by the 'regex' option is considered inspected, and removed the beginning of the inspection buffer.

# replace

Each rule may have any number of 'replace' options, which are used to modify the matched content prior to its sending to the other side.
A 'replace' option must occur immediately after a 'match' option, and only one 'replace' opton may occur for each 'match' option.

The syntax for a 'replace' option is a quoted string.
The quoted string must be made up of alphanumeric characters, space, or "\x" encoded 2 byte hex-encoded characters.
The quoted string, upon decoding, must be the same length as the preceeding 'match' string.

An example 'replace' option is:

    replace:"foo";

The 'replace' keyword is a best effort replacement.
Only content that is modified by 'replace' that is part of the message that triggered the current inspection will be modified as it is sent to the destination.
Any modified content from previous messages will have already been sent to the destination, therefor the modification is unable to occur.

# regex

Each rule may have any number of 'regex' options, which are used to search for a specified string in the inspection buffer.

The syntax for a 'regex' option is an arbitrary string within double quotes.  The string must be of a valid PCRE syntax.

An example 'regex' option is:

    regex:"foo.*bar";

Any content matched by the 'regex' option is considered inspected, and removed the beginning of the inspection buffer.

# side

Each rule may have a 'side' option, which allow a rule to be limited to inspecting content for a specific side of the session.

The syntax for a 'side' value is either the string client or server, to indicate which side of the session the rule applies.

An example 'side' option is:

    side:server;

# state

Each rule may have an arbitrary number 'state' options, which allow for the creation of rudimentary state machines via setting, unsetting, and checking the state of an arbitrary number of named bits that are allocated per session.

The syntax for a 'state' value is an operation, then a comma, then a string.

The operations are as follows:
* set
* unset
* is
* not

The 'set' operation sets the specified bit to True.  The 'unset' operation sets the specified bit to False.  The 'is' operation only continues if the specified bit is True.  The 'not' operation only continues if the specified bit is False.

The string must be one or more characters that are alphanumeric or "_".

An example 'state' option is:

   state:set,foo;

Note, while states may change during the processing of a rule, the states are only saved if the rule completes successfully.  
This means if a state is changed in the rule, but a later option in the rule fails, the values are not saved.

# flush

Each rule may have one 'flush' option, which empties the analysis buffer of the specified side of the communication.
If a 'flush' option is used, it must come last in the set of rule options.

The syntax for a 'flush' option is either the string client or server, to indicate which buffer should be emptied.

An example 'flush' option is:

    flush:server;

# Beyond Simple Inspection

## Order Matters

As discussed previously, rules are processed iteratively until no additional content has been inspected.
As such, the order the rules are defined matters.

Take the following set of rules:

    alert (name:"rule 1"; match:"foo"; replace:"bar";)
    block (name:"rule 2"; match:"foo";)

As is, rule 2 should never fire, as rule 1 will always replace the string "foo" with "bar".  
However, if the rules were in the opposite order, the session would drop as soon as "foo" was seen.

## Non-determinism in underlying communications

CBs within CGC communicate over TCP/IP, which does not guarantee data always arrives in the same size chunks as it is sent.
While testing in isolation a CB may always work in the same way, but often systems tested at load may cause edge conditions to occur.
One such issue, TCP segmentation, can be replicated in challenge binaries with the '--max_send' parameter to cb-test.

This issue impacts the efficacy of traditional packet based network security appliances.
Traditional appliances perform reassembly at a packet layer, including attempting to handle out of order packets.
Cb-proxy, operating at the socket layer, does not have to deal with out of order packets, but it does have to deal with reading a non-deterministic amount of traffic at a time. 

CB authors have no requirement to follow any known specification for messages, and as such cb-proxy must be flexible enough to handle arbitrary protocols.

Cb-proxy handles these non-deterministic issues by using a per-connection and per side inspection buffer, with the ability to flush the buffers without inspection at any point.
The original CADET_00001 challenge binary [0], the author received 128 bytes into a buffer that was only 64 bytes long.

At first glance, a rule that says if more than 64 bytes is sent could be used to stop a POV for CADET_00001 such as the following rule:
    
    block (name:"too much data sent"; side:client; regex:".{65}";)

However, depending on TCP buffering at various locations, at the client and/or the server, could cause more than 64 bytes in a single 'transmit' to not cause the POV to fire.

The server will always respond with upon receiving data.  
Using this knowledge, we can know if the server has completed a receive early due to buffering.

As such, the only reliable way to detect a POV is to look for more than 64 bytes of data, but as soon as the server has sent data, flush the inspection buffer for the client.

    block (name:"too much data sent"; side:client; regex:".{65}";)
    admit (name:"whenever we see server data, flush the client stream"; side:server; regex:".*"; flush:client;)

## Memory & CPU Consumption

There are some drawbacks to this buffering.
There is a performance impact to the buffering.
Data is always appended to the analysis buffer.
The analysis buffer is always stored for the duration of the session until successful inspection of the data by a rule or a 'flush'.

If data is never inspected or flushed, then the analysis buffer could become large, consuming excessive computing resources.

[0] - https://github.com/CyberGrandChallenge/samples/blob/a9cf305ea81f8454daf9d6dcf1758b57383a189e/examples/CADET_00001/src/service.c
