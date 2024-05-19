%%%
Title = "RDAP Simple Redaction"
area = "Applications and Real-Time Area (ART)"
workgroup = "Registration Protocols Extensions (regext)"
abbrev = "rdap-x"
updates = [7480]
ipr= "trust200902"

[seriesInfo]
name = "Internet-Draft"
value = "draft-newton-regext-rdap-rdap-simple-redaction-00"
stream = "IETF"
status = "standard"
date = 2024-05-19T00:00:00Z

[[author]]
initials="A."
surname="Newton"
fullname="Andy Newton"
organization="ICANN"
[author.address]
email = "andy@hxr.us"

%%%

.# Abstract

This document defines a simple redaction extension for the Registration Data Access Protocol
(RDAP).

{mainmatter}

# Background

[@?RFC9537] defines a redaction extension for the Registration Data Access Protocol (RDAP).
[@?RFC9537] specifies a complex mechanism that has been shown to be difficult to implement.

This document defines a simpler redaction extension, "simple redaction", that achieved through a narrower scope
of features. Simple redaction only defines redaction in JSON strings of RDAP responses. This narrow
scope is justified because the only place in RDAP where there are booleans or integers are used in the
base specification [@!RFC9083] are in the DNSSEC portion of the domain object class, and there 
is unlikely to be policy for redaction such information as it also visible in the public DNS.

In addition, this specification does not require the removal of JSON values or components that
would otherwise make the resulting JSON invalid according to [@!RFC9083] nor semantically invalid according
to [@!RFC9083] or [@!RFC7095].

Additionally, this specification allows for parts of a string to be redacted, which cannot be
done with [@?RFC9537].

Finally, this specification has the benefit that if an RDAP client does not recognize this extension
and simply passes the data onto the user, in many contexts the user may still understand that the
information is redacted.

The following is an example of an RDAP response using simple redaction:

```
{
  "rdapConformance" : [ "Rdap_level_0", “simpleRedaction” ],
  "objectClassName" : "entity",
  "handle":"////REDACTED_HANDLE////",
  "vcardArray":[
    "vcard",
    [
      ["version", {}, "text", "4.0"],
      ["fn", {}, "text", "////REDACTED_FN////"],
        ["email",
        { "type":"work" },
        "text", "redacted_email@redacted.invalid"
      ]
    ]
  ],
  “simpleRedaction”: [
    { “key”: “////REDACTED_HANDLE////” },
    { “key”: “////REDACTED_FN////” },
    { “key”: “redacted_email@redacted.invalid” }
  ]
}

```

# Redaction Keys

Simple redaction allows a server to define a set of keys, each used to signify when data in a string
has been redacted.

## Unstructured Text {#unstructured_keys}

Keys signifying redaction for unstructured text, i.e. free form text, take the form of `////REDACTION_KEY////`.
These keys begin with four forward-slash character ("////"), followed by one or more of the upper and lower case characters 
`A` through `Z`, `0` through `9`, hyphen ("-"), or underbar ("_"), followed by four more forward-slash characters. 

The following example demonstrates redaction of the full name value from a jCard ([@?RFC7095]) array:

```
["fn", {}, "text", "////REDACTED_FN////"]    
```

These keys may be placed in a string with other characters thus allowing for the partial redaction of a string:

```
"Bob ////LAST_NAME_REDACTION////"
```

## TEL URIs {#tel_uri_keys}

Keys for use in "tel" URIs ([@!RFC3966]) follow a form similar to (#unstructured_keys) but with a restricted
set of characters to conform to the "tel" URI syntax. These keys begin and end with four forward-slash characters,
but the set of characters allowed between the slashes is limited to `0` through `9`. For example:
`////0123456789////`.

The following is an example of a "tel" URI used in a jCard array:

```
["tel", {}, "uri", "tel:+////0000000////;ext=////999999////"]    
```

## Email Addresses

Keys for email addresses MUST use a host part that is "redacted.invalid" but may use any local part
allowable in an email address. For example: `redacted_email@redacted.invalid`.

## URIs with Host Names

Keys used in a URI, such as an HTTP URI, MUST use a host name that is "redacted.invalid".
For example: `https://redacted.invalid/redacted_web_page`.

# Explicit Keying

Each defined key MUST be given in the "simpleRedaction" array value. This array contains JSON objects.
Each JSON object has a REQUIRED JSON string name "key", a JSON array named "reasons", and an OPTIONAL
"links" array as defined by [@!RFC9083]. The "reasons" array is OPTIONAL but if present MUST NOT be empty.

The "reasons" array contains JSON objects. These objects SHOULD have a "lang" member as defined
by [@!RFC9083] and MUST have a JSON array named "description". The "description" array contains
only JSON strings.

The following is an example:

```
“simpleRedaction”: [
  {
    “key”: “////REDACTED_FULL_NAME////”,
    “reasons” : [
      {
        “lang”: “en”,
        “description” : [
          “The full name of registrants has not been given.”,
          “This redaction is done according to policy.”
        ]
      },
      {
        “lang”: “ja”,
        “description” : [
          “登録者のフルネームは公表されていない。”,
          “この編集はポリシーに従って行われます。”
        ]
      }
    ],
    "links": [
      {
        "value": "https://example.com/value",
        "rel": "about",
        "href": "https://example.com/href",
        "type": "text/html"
      }
    ]
  }
]
```

A key MUST only appear once, and the "simpleRedaction" JSON value MUST only be in the top-most
object of the RDAP response.

# Keying Strategies

## Unclean Data

While it seems odd that some users would be allowed to give an email address with a host of
"redacted.invalid" or a string that begins and ends with four forward-slashes to an Internet
registration authority, some registries must deal with such data for various reasons.

One strategy servers may use would be to append a set of random digits to each key. For example,
if a registered resource was given as "////I_FOOLED_YOU////" then the server could thwart this
by appending the random digits "90210" to make the key "////I_FOOLED_YOU90210////".

## Handles

For servers operating under policies in which the "handle", as defined by [@!RFC9083] must be
redacted, it would be beneficial to some clients to create unique redaction keys for each handle.
While clients SHOULD use "self" links, as described in [@!RFC9083], to differentiate between
between objects returned in a response, in the absence they may also use the "handle".
Therefore, servers SHOULD create a unique redaction key for each handle that is redacted.

# Security Considerations

TBD.

# IANA Considerations

TBD: registration of "simpleRedaction".

{backmatter}

