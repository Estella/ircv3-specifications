---
title: IRCv3.1 SASL Authentication
layout: spec
updated-by:
  - sasl-3.2
copyrights:
  -
    name: "Jilles Tjoelker"
    period: "2009-2012"
    email: "jilles@stack.nl"
  -
    name: "William Pitcock"
    period: "2009-2012"
    email: "nenolod@dereferenced.org"
  -
    name: "Mantas Mikulėnas"
    period: "2017"
    email: "grawity@nullroute.eu.org"
---
This document describes the client protocol for SASL authentication.

The SASL protocol in general is documented in [RFC 4422][], along with the
'EXTERNAL' mechanism. The most commonly used 'PLAIN' mechanism is documented in
[RFC 4616][].

SASL authentication relies on the [CAP client capability framework][cap-3.1].

Support for SASL authentication is indicated with the "sasl" capability.
The client MUST enable the sasl capability before using the AUTHENTICATE
command defined by this specification.

## The AUTHENTICATE command

The AUTHENTICATE command MUST be used before registration is complete and
with the `sasl` capability enabled. To enforce the former, it is RECOMMENDED
to only send CAP END when the SASL exchange is completed or needs to be
aborted. Clients SHOULD be prepared for timeouts at all times during the SASL
authentication.

There are two forms of the AUTHENTICATE command: initial client message
(mechanism selection) and later messages (mechanism challenge/response data).

### Mechanism selection

The initial client message specifies the SASL mechanism to be used:

    initial-authenticate = "AUTHENTICATE" SP mechanism CRLF

When this is received, the server (or services) will try to start the
mechanism. If this fails (e.g. if the mechanism is unknown), a 904 numeric will
be sent and the session state remains unchanged; the client MAY try another
mechanism.

On success, the server sends one or more regular AUTHENTICATE commands
containing the initial challenge (possibly empty). The server and client keep
exchanging the regular AUTHENTICATE commands until authentication succeeds,
fails, or is aborted.

### Authentication exchange

An empty message is sent as `AUTHENTICATE +`. Otherwise, the message is encoded
in Base64 ([RFC 4648][], the encoded output is split to 400-byte chunks, and
each chunk is sent as a separate `AUTHENTICATE` command.

If the last chunk was exactly 400 bytes long, it must also be followed by an
empty chunk (i.e. `AUTHENTICATE +`) to signal end of message.

The client MUST buffer received data, and act on the whole message only after
receiving the final chunk (i.e. shorter than 400 bytes), but SHOULD impose
limits on the total length of buffered data and abort authentication when the
limit is reached. (A reasonable limit might be 4 kB.)

The server MAY intersperse AUTHENTICATE messages with other IRC protocol
messages.

    regular-authenticate-set = *(partial-authenticate) final-authenticate

        partial-authenticate = "AUTHENTICATE" SP 400BASE64 CRLF

          final-authenticate = "AUTHENTICATE" SP (1*399BASE64 / "+") CRLF

### Success, failure, and edge cases

If authentication succeeds, RPL\_LOGGEDIN and RPL\_SASLSUCCESS will be sent.

If authentication fails, ERR\_SASLFAIL or ERR\_SASLTOOLONG will be sent and the
client MAY start authentication again, beginning with mechanism selection.

(On failure, the server SHOULD additionally send RPL\_SASLMECHS advising the
client which mechanisms are supported. If this numeric is supported, it MUST be
sent before ERR\_SASLFAIL.)

The client can abort an authentication by sending an asterisk as the data. The
server may send ERR\_SASLFAIL or ERR\_SASLABORTED.

    authenticate-abort = "AUTHENTICATE" SP "*" CRLF

If the client attempts to issue the AUTHENTICATE command after already
authenticating successfully, the server MUST reject it with ERR\_SASLALREADY.
(Note that this requirement is removed in `sasl-3.2`.)

If the client completes registration (with CAP END, NICK, USER and any other
necessary messages) while the SASL authentication is still in progress, the
server SHOULD abort it and send ERR\_SASLABORTED, then register the client
without authentication.

This document does not specify use of the AUTHENTICATE command in registered
(person) state.

## SASL overview

### Conversation

Each SASL "conversation" consists of one or more "steps", in which the server
sends a challenge and the client provides a response.

          CLIENT        SERVER

       mechanism -->
                    <-- challenge
        response -->
                    <-- challenge
        response -->
                    <-- success/failure

Each challenge and response are paired. If the server sends a challenge, it
must not report success/failure in the same step – it must wait for a client
response before doing so.

For some mechanisms (known as "client-first") the server's initial challenge
will be empty. In this case, the server MUST still send an empty challenge
message (which is formatted as `AUTHENTICATE +`). For example, PLAIN works this
way.

Similarly, some mechanisms finish when the server sends a final message. (For
example, [SCRAM][RFC 5802] ends with the server sending the "verifier".) In
this case, the client MUST still send an empty response and the server MUST
wait for it before indicating success.

In the IRC protocol, a large challenge or response may be split across several
`AUTHENTICATE` commands; in that case, the entire group of commands still
belongs to a single "step".

### Authorization

Instead of a single username, SASL defines two separate "authentication" and
"authorization" identities:

 * The _authentication_ identity (sometimes abbreviated "authcid") is used for
   verifying credentials – in short, it's the username you log in with.

 * The _authorization_ identity (abbreviated "authzid") is used to determine
   what access or privileges you obtain – i.e. to impersonate another user, as
   if using `su` or `sudo`.

If the client doesn't support specifying a separate authorization identity, it
SHOULD leave the field empty. (Setting both identities to the same value also
works in practice, although is not guaranteed by the specification.)

## Example protocol exchange

C: indicates lines sent by the client, S: indicates lines sent by the server.

The client is using the PLAIN SASL mechanism with authentication identity
`jilles`, authorization identity `jilles` and password `sesame`.

    C: CAP REQ :sasl
    C: NICK jilles
    C: USER jilles cheetah.stack.nl 1 :Jilles Tjoelker
    S: NOTICE AUTH :*** Processing connection to jaguar.test
    S: NOTICE AUTH :*** Looking up your hostname...
    S: NOTICE AUTH :*** Checking Ident
    S: NOTICE AUTH :*** No Ident response
    S: NOTICE AUTH :*** Found your hostname
    S: :jaguar.test CAP jilles ACK :sasl
    C: AUTHENTICATE PLAIN
    S: AUTHENTICATE +
    C: AUTHENTICATE amlsbGVzAGppbGxlcwBzZXNhbWU=
    S: :jaguar.test 900 jilles jilles!jilles@localhost.stack.nl jilles :You are now logged in as jilles
    S: :jaguar.test 903 jilles :SASL authentication successful
    C: CAP END
    S: :jaguar.test 001 jilles :Welcome to the jillestest Internet Relay Chat Network jilles
    (usual welcome messages)

The client is using the SCRAM-SHA-1 mechanism with the same credentials.

    C: CAP REQ :sasl
    C: NICK jilles
    C: USER jilles cheetah.stack.nl 1 :Jilles Tjoelker
    S: NOTICE AUTH :*** Processing connection to jaguar.test
    S: NOTICE AUTH :*** Looking up your hostname...
    S: NOTICE AUTH :*** Checking Ident
    S: NOTICE AUTH :*** No Ident response
    S: NOTICE AUTH :*** Found your hostname
    S: :jaguar.test CAP jilles ACK :sasl
    C: AUTHENTICATE SCRAM-SHA-1
    S: AUTHENTICATE +
    C: AUTHENTICATE bixhPWppbGxlcyxuPWppbGxlcyxyPWM1UnFMQ1p5MEw0ZkdrS0FaMGh1akZCcw==
    S: AUTHENTICATE cj1jNVJxTENaeTBMNGZHa0tBWjBodWpGQnNYUW9LY2l2cUN3OWlEWlBTcGIscz01bUpPNmQ0cmpDbnNCVTFYLGk9NDA5Ng==
    C: AUTHENTICATE Yz1iaXhoUFdwcGJHeGxjeXc9LHI9YzVScUxDWnkwTDRmR2tLQVowaHVqRkJzWFFvS2NpdnFDdzlpRFpQU3BiLHA9T1ZVaGdQdTh3RW0yY0RvVkxmYUh6VlVZUFdVPQ==
    S: AUTHENTICATE dj1aV1IyM2M5TUppcjBaZ2ZHZjVqRXRMT242Tmc9
    C: AUTHENTICATE +
    S: :jaguar.test 900 jilles jilles!jilles@localhost.stack.nl jilles :You are now logged in as jilles
    S: :jaguar.test 903 jilles :SASL authentication successful
    C: CAP END
    S: :jaguar.test 001 jilles :Welcome to the jillestest Internet Relay Chat Network jilles
    (usual welcome messages)

Alternatively the client could request the list of capabilities and enable
an additional capability.

    C: CAP LS
    C: NICK jilles
    C: USER jilles cheetah.stack.nl 1 :Jilles Tjoelker
    S: NOTICE AUTH :*** Processing connection to jaguar.test
    S: NOTICE AUTH :*** Looking up your hostname...
    S: NOTICE AUTH :*** Checking Ident
    S: NOTICE AUTH :*** No Ident response
    S: NOTICE AUTH :*** Found your hostname
    S: :jaguar.test CAP * LS :multi-prefix sasl
    C: CAP REQ :multi-prefix sasl
    S: :jaguar.test CAP jilles ACK :multi-prefix sasl
    C: AUTHENTICATE PLAIN
    S: AUTHENTICATE +
    C: AUTHENTICATE amlsbGVzAGppbGxlcwBzZXNhbWU=
    S: :jaguar.test 900 jilles jilles!jilles@localhost.stack.nl jilles :You are now logged in as jilles
    S: :jaguar.test 903 jilles :SASL authentication successful
    C: CAP END
    S: :jaguar.test 001 jilles :Welcome to the jillestest Internet Relay Chat Network jilles
    (usual welcome messages)

## Numerics used by this extension

`900` aka `RPL_LOGGEDIN` is sent when the user's account name is set (whether by SASL or otherwise).

    :server 900 <nick> <nick>!<ident>@<host> <account> :You are now logged in as <user>

`901` aka `RPL_LOGGEDOUT` is sent when the user's account name is unset (whether by SASL or otherwise).

    :server 901 <nick> <nick>!<ident>@<host> :You are now logged out

`902` aka `ERR_NICKLOCKED` is sent when the SASL authentication fails because the account is currently locked out, held, or otherwise administratively made unavailable.

    :server 902 <nick> :You must use a nick assigned to you

`903` aka `RPL_SASLSUCCESS` is sent when the SASL authentication finishes successfully. It usually goes along with `900`.

    :server 903 <nick> :SASL authentication successful

`904` aka `ERR_SASLFAIL` is sent when the SASL authentication fails because of invalid credentials or other errors not explicitly mentioned by other numerics.

    :server 904 <nick> :SASL authentication failed

`905` aka `ERR_SASLTOOLONG` is sent when credentials are valid, but the SASL authentication fails because the client-sent `AUTHENTICATE` command was too long (i.e. the parameter longer than 400 bytes).

    :server 905 <nick> :SASL message too long

`906` aka `ERR_SASLABORTED` is sent when the SASL authentication is aborted because the client sent an `AUTHENTICATE` command with `*` as the parameter.

    :server 906 <nick> :SASL authentication aborted

`907` aka `ERR_SASLALREADY` is sent when the client attempts to initiate SASL authentication after it has already finished successfully for that connection.

    :server 907 <nick> :You have already authenticated using SASL

`908` aka `RPL_SASLMECHS` MAY be sent in reply to an `AUTHENTICATE` command which requests an unsupported mechanism. The numeric contains a comma-separated list of mechanisms supported by the server (or network, services).

    :server 908 <nick> <mechanisms> :are available SASL mechanisms

_(The final "text" parameter is not to be machine-parsed, as it tends to vary
between implementations and translations.)_

# Errata

* Previous versions of this specification mentioned that 904 numeric would be used when SASL was aborted. 906 ERR_SASLABORTED should be used when SASL is aborted.
* Previous versions of this specification did not precisely describe when is RPL_SASLMECHS being sent.
* Clarified the language how responses are transmitted.
* Added empty initial server response for client-first mechanisms. This had happened de-facto already.
* Added empty final client response for certain mechanisms. This had happened de-facto already.
* Removed mention of IRCD "SASL agents", as it is a server-side implementation detail.
* Described basic concepts of SASL separately from the low-level protocol description.
* Mentioned that empty authzid is preferred.

[cap-3.1]: ../core/capability-negotiation-3.1.html
[RFC 4422]: https://tools.ietf.org/html/rfc4422 "Simple Authentication and Security Layer (SASL)"
[RFC 4616]: https://tools.ietf.org/html/rfc4616 "The PLAIN Simple Authentication and Security Layer (SASL) Mechanism"
[RFC 4648]: https://tools.ietf.org/html/rfc4648 "The Base16, Base32, and Base64 Data Encodings"
[RFC 5802]: https://tools.ietf.org/html/rfc5802 "Salted Challenge Response Authentication Mechanism (SCRAM)"
