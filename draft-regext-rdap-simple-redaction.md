%%%
Title = "RDAP Simple Redaction"
area = "Applications and Real-Time Area (ART)"
workgroup = "Registration Protocols Extensions (regext)"
abbrev = "simple-redaction"
updates = [7480]
ipr= "trust200902"

[seriesInfo]
name = "Internet-Draft"
value = "draft-newton-regext-rdap-simple-redaction-00"
stream = "IETF"
status = "standard"
date = 2024-06-17T00:00:00Z

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

This document defines a simple redaction extension, "simple redaction", that is achieved through a narrow scope
of features. Simple redaction only defines redaction in JSON strings of RDAP responses. This narrowed
scope meets the purposes of known redaction policies, such as the 
[ICANN Registration Data Policy](https://www.icann.org/resources/pages/registration-data-policy-2024-02-21-en),
where redaction is applied against "personal data" which is only found in JSON strings in the
RDAP specification ([@!RFC9083]). The only place in RDAP where booleans or integers are used in the
base specification of [@!RFC9083] are in the DNSSEC portion of the domain object class, and there 
is unlikely to be policy for redaction of such information as it also visible in the public DNS.

This specification does not require the removal of JSON values or components that
would otherwise make the resulting JSON invalid according to [@!RFC9083] nor semantically invalid according
to [@!RFC9083] or [@!RFC7095].

Finally, this specification has the benefit that if an RDAP client does not recognize this extension
and simply passes the redaction signals onto the user, in some contexts the user may still understand that the
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

# Redaction Keys {#redaction_keys}

Simple redaction allows a server to define a set of keys, each used to signify when data in a string
has been redacted. Clients use these keys to identify the information being redacted.

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
"Alice ////LAST_NAME_REDACTION////"
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

The above is just an example, but [@?RFC6530], which defines the much of the structures in [@?RFC7095], does
not require the "tel" property to be a URI. So this maybe written as:

```
["tel", {}, "text", "////TELEPHONE_REDACTION////"]    
```

## Email Addresses

Keys for email addresses MUST use a host part that is "redacted.invalid" but may use any local part
allowable in an email address. For example: `redacted_email@redacted.invalid`.

The ".invalid" TLD is a special-use domain defined in [@!RFC6761] and is unuseable on the Internet.

## URIs with Host Names

Keys used in a URI with host names, such as an HTTP URI, MUST use a host name that is "redacted.invalid".
For example: `https://redacted.invalid/redacted_web_page`.

The ".invalid" TLD is a special-use domain defined in [@!RFC6761] and is unuseable on the Internet.

# Explicit Keying {#explicity_keying}

All redaction keys ((#redaction_keys)) are explicitly specified by the server.

## The "simpleRedaction" Array {#simple_redaction_array}

Each defined key MUST be given in the "simpleRedaction" array value. This array contains JSON objects.
Each JSON object has a REQUIRED JSON string named "key", a JSON array named "reasons", and an OPTIONAL
"links" array as defined by [@!RFC9083] (see (#alternates) and (#policy) on usage).
The "reasons" array is OPTIONAL but if present MUST NOT be empty.

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
        "href": "https://example.com/some-policy.html",
        "type": "text/html"
      }
    ]
  }
]
```

A key MAY be used more than once in an RDAP object, but it MUST only appear once in the "simpleRedaction" array
of objects. A client MUST NOT consider any key not found to be the "simpleRedaction" array of objects as a valid
redaction (i.e. do not signal to the user that the information has been redacted). 

The "simpleRedaction" JSON value MUST only be in the top-most object of the RDAP response.

## Alternates {#alternates}

The "simpleRedaction" array described in (#simple_redaction_array) allows each key to be accompanied by
an array of links, as defined by [#RFC9083]. Usage of the links may be used to signal an alternate
usage in cases where the alternate can be expressed as a URI. To do this, servers MUST use the "alternate"
link relation and clients SHOULD signal to users that the "href" value is available for alternate usage.

The following example demonstrates the singaling on a web-based contact form to be used instead of email.

```
{
  "rdapConformance" : [ "Rdap_level_0", “simpleRedaction” ],
  "objectClassName" : "entity",
  "handle":"foo",
  "vcardArray":[
    "vcard",
    [
      ["version", {}, "text", "4.0"],
      ["fn", {}, "text", "Bob Allison"],
      ["email",
        { "type":"work" },
        "text", "redacted_email@redacted.invalid"
      ]
    ]
  ],
  “simpleRedaction”: [
    { 
      “key”: “redacted_email@redacted.invalid”,
      "links": [
        {
          "value": "https://example.com/value",
          "rel": "alternate",
          "href": "https://example.com/contact-form",
          "type": "text/html"
        }
      ]
    }
  ]
}
```

This example demonstrates singaling that an alternate email address is to be used.

```
{
  "rdapConformance" : [ "Rdap_level_0", “simpleRedaction” ],
  "objectClassName" : "entity",
  "handle":"foo",
  "vcardArray":[
    "vcard",
    [
      ["version", {}, "text", "4.0"],
      ["fn", {}, "text", "Bob Allison"],
      ["email",
        { "type":"work" },
        "text", "redacted_email@redacted.invalid"
      ]
    ]
  ],
  “simpleRedaction”: [
    { 
      “key”: “redacted_email@redacted.invalid”,
      "links": [
        {
          "value": "https://example.com/value",
          "rel": "alternate",
          "href": "mailto:proxy-service@example.com"
        }
      ]
    }
  ]
}
```

Clients should consider, when presenting information to a user,
that an alternate use may differ from the form in the RDAP response. For example,
the RDAP response may contain an email address but the alternate usage is a web page.

## Redaction Policy {#policy}

The "links" array described in (#simple_redaction_array) may also be used to link to a web page
describing the redaction policy. When the array is to be used for this purpose, the "about" relationship
MUST be used. The following is an example.

```
{
  "rdapConformance" : [ "Rdap_level_0", “simpleRedaction” ],
  "objectClassName" : "entity",
  "handle":"foo",
  "vcardArray":[
    "vcard",
    [
      ["version", {}, "text", "4.0"],
      ["fn", {}, "text", "Bob Allison"],
      ["email",
        { "type":"work" },
        "text", "redacted_email@redacted.invalid"
      ]
    ]
  ],
  “simpleRedaction”: [
    { 
      “key”: “redacted_email@redacted.invalid”,
      "links": [
        {
          "value": "https://example.com/value",
          "rel": "about",
          "href": "https://example.com/some-policy.html",
          "type": "text/html"
        }
      ]
    }
  ]
}
```

## Keying Strategies

### Unclean Data

While it seems odd that some users would be allowed to give an email address with a host of
"redacted.invalid" or a string that begins and ends with four forward-slashes to an Internet
registration authority, some registries must deal with such data for various reasons.

One strategy servers may use would be to append a set of random digits to each key. For example,
if a registered resource was given as "////I_FOOLED_YOU////" then the server could thwart this
by appending the random digits "90210" to make the key "////I_FOOLED_YOU90210////".

### Handles

For servers operating under policies in which the "handle", as defined by [@!RFC9083] must be
redacted, it would be beneficial to some clients to create unique redaction keys for each handle.
While clients SHOULD use "self" links, as described in [@!RFC9083], to differentiate between
between objects returned in a response, in the absence of "self" links they often use the "handle".
Therefore, servers SHOULD create a unique redaction key for each handle that is redacted.

# Examples

## Unstructured Addresses

[@?RFC7095] allows for the representation of unstructured postal addresses. The following is a simple
example of an RDAP response with simple redactions where the postal address is given as
unstructured.

```
{
  "rdapConformance" : [ "Rdap_level_0", “simpleRedaction” ],
  "objectClassName" : "entity",
  "handle":"foo",
  "vcardArray":[
    "vcard",
    [
      ["version", {}, "text", "4.0"],
      ["fn", {}, "text", "Bob"],
      ["adr",
        {

          "type":"home",
          "label":"////REDACTED_STREET////\nVancouver\nBC\n////REDACTED_POSTAL_CODE////\n"
        },
        "text",
        [
          "", "", "", "", "", "", ""
        ]
      ],
    ]
  ],
  “simpleRedaction”: [
    { “key”: “////REDACTED_STREET////” },
    { “key”: “////REDACTED_POSTAL_CODE////” },
  ]
}
````

## Structured Addresses

[@?RFC7095] allows for the representation of structured postal addresses. The following is a simple
example of an RDAP response with simple redactions where the postal address is given as
structured.

```
{
  "rdapConformance" : [ "Rdap_level_0", “simpleRedaction” ],
  "objectClassName" : "entity",
  "handle":"foo",
  "vcardArray":[
    "vcard",
    [
      ["version", {}, "text", "4.0"],
      ["fn", {}, "text", "Bob"],
      ["adr",
        { "type":"work" },
        "text",
        [
          "",
          "////REDACTED_STREET////",
          "////REDACTED_STREET////",
          "Quebec",
          "QC",
          "////REDACTED_POSTAL_CODE////",
          "Canada"
        ]
      ]
    ]
  ],
  “simpleRedaction”: [
    { “key”: “////REDACTED_STREET////” },
    { “key”: “////REDACTED_POSTAL_CODE////” },
  ]
}
````

## A Complete Example

The following is an example an RDAP response to a domain lookup in which the redactions
specified in 
[ICANN Registration Data Policy](https://www.icann.org/resources/pages/registration-data-policy-2024-02-21-en),
have been applied.

```
{
  "rdapConformance": [
    "rdap_level_0", "simpleRedaction"
  ],
  "objectClassName": "domain",
  "handle": "////REGISTRY_DOMAIN_ID_REDACTION////",
  "ldhName": "example.com",
  "secureDNS": {
    "delegationSigned": false
  },
  "notices": [
    {
      "title": "Terms of Use",
      "description": [
        "Service subject to Terms of Use."
      ],
      "links": [
        {
          "rel": "self",
          "href": "https://www.example.com/terms-of-use",
          "type": "text/html",
          "value": "https://www.example.com/terms-of-use"
        }
      ]
    }
  ],
  "nameservers": [
    {
      "objectClassName": "nameserver",
      "ldhName": "ns1.example.com"
    },
    {
      "objectClassName": "nameserver",
      "ldhName": "ns2.example.com"
    }
  ],
  "entities": [
    {
      "objectClassName": "entity",
      "handle": "123",
      "roles": [
        "registrar"
      ],
      "publicIds": [
        {
          "type": "IANA Registrar ID",
          "identifier": "1"
        }
      ],
      "vcardArray": [
        "vcard",
        [
          [
            "version",
            {},
            "text",
            "4.0"
          ],
          [
            "fn",
            {},
            "text",
            "Example Registrar Inc."
          ],
          [
            "adr",
            {},
            "text",
            [
              "",
              "Suite 100",
              "123 Example Dr.",
              "Dulles",
              "VA",
              "20166-6503",
              "US"
            ]
          ],
          [
            "email",
            {},
            "text",
            "contact@organization.example"
          ],
          [
            "tel",
            {
              "type": "voice"
            },
            "uri",
            "tel:+1.7035555555;ext=1234"
          ],
          [
            "tel",
            {
              "type": "fax"
            },
            "uri",
            "tel:+1.7035555556"
          ]
        ]
      ],
      "entities": [
        {
          "objectClassName": "entity",
          "roles": [
            "abuse"
          ],
          "vcardArray": [
            "vcard",
            [
              [
                "version",
                {},
                "text",
                "4.0"
              ],
              [
                "fn",
                {},
                "text",
                "Abuse Contact"
              ],
              [
                "email",
                {},
                "text",
                "abuse@organization.example"
              ],
              [
                "tel",
                {
                  "type": "voice"
                },
                "uri",
                "tel:+1.7035555555;ext=1234"
              ]
            ]
          ]
        }
      ]
    },
    {
      "objectClassName": "entity",
      "handle": "////REGISTRY_REGISTRANT_ID_REDACTION////",
      "roles": [
        "registrant"
      ],
      "vcardArray": [
        "vcard",
        [
          [
            "version",
            {},
            "text",
            "4.0"
          ],
          [
            "fn",
            {},
            "text",
            "////REGISTRANT_NAME_REDACTION////"
          ],
          [
            "org",
            {},
            "text",
            "////REGISTRANT_ORG_REDACTION////"
          ],
          [
            "adr",
            {},
            "text",
            [
              "",
              "////REGISTRANT_STREET_REDACTION////",
              "////REGISTRANT_STREET_REDACTION////",
              "Quebec",
              "////REGISTRANT_CITY_REDACTION////",
              "////REGISTRANT_POSTAL_CODE_REDACTION////",
              "Canada"
            ]
          ],
          [
            "email",
            {},
            "text",
            "registrant-email-redaction@redacted.invalid"
          ],
          [
            "tel",
            {
              "type": "voice"
            },
            "uri",
            "tel:+////0000000000////;ext=////1111111111////"
          ],
          [
            "tel",
            {
              "type": "fax"
            },
            "uri",
            "tel:+////2222222222////"
          ]
        ]
      ]
    },
    {
      "objectClassName": "entity",
      "handle": "////REGISTRY_TECH_ID_REDACTION////",
      "roles": [
        "technical"
      ],
      "vcardArray": [
        "vcard",
        [
          [
            "version",
            {},
            "text",
            "4.0"
          ],
          [
            "fn",
            {},
            "text",
            "////TECH_NAME_REDACTION////"
          ],
          [
            "org",
            {},
            "text",
            "Example Inc."
          ],
          [
            "adr",
            {},
            "text",
            [
              "",
              "Suite 1234",
              "4321 Rue Somewhere",
              "Quebec",
              "QC",
              "G1V 2M2",
              "Canada"
            ]
          ],
          [
            "email",
            {},
            "text",
            "technical-email-redaction@redacted.invalid"
          ],
          [
            "tel",
            {
              "type": "voice"
            },
            "uri",
            "tel:+////3333333333////;ext=321"
          ],
          [
            "tel",
            {
              "type": "fax"
            },
            "uri",
            "tel:+1-555-555-4321"
          ]
        ]
      ]
    }
  ],
  "events": [
    {
      "eventAction": "registration",
      "eventDate": "1997-06-03T00:00:00Z"
    },
    {
      "eventAction": "last changed",
      "eventDate": "2020-05-28T01:35:00Z"
    },
    {
      "eventAction": "expiration",
      "eventDate": "2021-06-03T04:00:00Z"
    }
  ],
  "status": [
    "server delete prohibited",
    "server update prohibited",
    "server transfer prohibited",
    "client transfer prohibited"
  ],
  “simpleRedaction”: [
    { “key”: “////REGISTRY_DOMAIN_ID_REDACTION////” },
    { “key”: “////REGISTRY_REGISTRANT_ID_REDACTION////” },
    { “key”: “////REGISTRANT_NAME_REDACTION////” },
    { “key”: “////REGISTRANT_STREET_REDACTION////” },
    { “key”: “////REGISTRANT_POSTAL_CODE_REDACTION////” },
    { “key”: “////0000000000////” },
    { “key”: “////1111111111////” },
    { “key”: “////REGISTRY_TECH_ID_REDACTION////” },
    { “key”: “////TECH_NAME_REDACTION////” },
    { “key”: “////1111111111////” },
    { “key”: “////2222222222////” },
    { “key”: “registrant-email-redaction@redacted.invalid” },
    { “key”: “tech-email-redaction@redacted.invalid” },
    { “key”: “////REGISTRANT_ORG_REDACTION////” },
    { “key”: “////REGISTRANT_CITY_REDACTION////” }
  ]
}  
```

# Security Considerations

TBD.

# IANA Considerations

TBD: registration of "simpleRedaction".

{backmatter}

