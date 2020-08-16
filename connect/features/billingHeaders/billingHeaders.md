# Sources of custom headers

Custom billing headers is a tool for sending of information from RC server system to customers in SIP custom headers (SIP requests and responses) and in TLVs parameters of outgoing SMPP messages.
Custom billing headers are originated from:

1. A set of Restcomm built-in headers that represent RC details like "X-RestComm-AccountSid" / "X-RestComm-OrganizationSid"
2. Billing headers from a new DB table restcomm_tags
3. Subscriber profile based entries ("multiProvider" - "outbound-voice" - "headers") (for voice only) 
4. HTTP headers / POST parameters from REST API requests
5. HTTP headers from HTTP responses from RCML loading
6. RCML Dial - SIP URI header parameters
7. To field SIP URI header parameters from REST API requests

> **WARNING**: The previous list order defines the priority in case of collision. This means Restcomm built-in headers can't be overridden.

## Restcomm Built-in headers 

Here is a list of such headers.

|Header                |Value |
|----------------------|------|
|X-RestComm-ApiVersion |Api version |
|X-RestComm-AccountSid |Account Sid |
|X-RestComm-OrganizationSid |OrganizationSid |
|X-RestComm-ApplicationSid |ApplicationSid |
|X-RestComm-ApplicationUrl |ApplicationUrl |
|X-RestComm-CallSid |CallSid or MessageSid |

## Billing tags from DB

CSP Administrator can create tags for enterprise accounts (max 5 tags per a customer). Tags are put in a DB table "restcomm_tags".
Example of a DB record:

| sid                                | name        | value           | type    | owner_sid                          | target_sid                         | date_created        | date_updated        |
|------------------------------------|-------------|-----------------|---------|------------------------------------|------------------------------------|---------------------|---------------------|
| TGf7df9fafcfb047178d0ca78a8274d2db | MSISDN      | `302108257600`    | Billing | ACae6e420f425248d6a26948c17a9e2acf | AC48ac1aac2ea07b345e778aa923c1854a | 2020-07-11 07:39:40 | 2020-07-11 07:39:40 |

A description of a REST API for adding and managing of Tags is here: [Tags REST API](https://bitbucket.org/telestax/documentation/src/tags_api_docs/modules/ROOT/pages/api/tags-api.adoc)

When RC processes a voice call or a message against an account to which any billing tags are added, then RC will include the tags as it is in next SIP requests (and outgoing SMPP as TLVs).
For an example above a SIP header "MSISDN" with a value `302108257600` will be generated.


## Subscriber profile based entries 

A customer may add headers in a subscriber profile.

 ```
 {"multiProvider":{
   "outbound-voice": [
     { "name":"np02","proxy":[ {
       "uri":"10.0.0.32:5064",
       "headers":[
         {"headerName":"X-SomeHeader","value":"44444"}
       ]
     }]
   }]
 }} 
 ```

This example will add a header "X-SomeHeader" with value "44444"

## Custom headers from REST API

When a customer sends REST-API requests for initiating of calls and messages he can add a "P-" header into an HTTP request.

If we use a curl command then the syntax may be:

* `-H "P-Asserted-Service:urn:urn-7:2FA"` - providing a parameter as a HTTP header.
* `-d "P-Asserted-Service=urn:urn-7:2FA"` - providing a parameter as a POST parameter.

Note that an HTTP header name is case-insensitive (but a value is still case-sensitive). If we provide a header like '-H "P-Some-Header:value"' then a header name will be "p-some-header".

A SIP header "P-Asserted-Service" has a special meaning. It presents a service that initiates a call/message and reused by internal Restcomm applications. It should have a specific format like "urn:urn-7:2FA". Values may be "RVD", "2FA", "NM", "AA".
Here is a [The P-Asserted-Service Header definition](https://tools.ietf.org/html/rfc6050#section-4.1).

## Custom headers from RCML applications

A customer may supply custom headers by adding custom headers in the HTTP response when downloading RCML application.
Again only "P-" prefixed http headers will be processed.


## SIP URI header parameters
SIP Spec allows to specify headers to be included in the outgoing request from the SIP URI.
Only "P-" or "X-" headers are allowed. If not properly prefixed, then
the header will be automatically prefixed with "X-".
 

It can be sent as a parameter in API To field:

* `-d 'To=client:alice?X-Custom-Header1=1234'`

Or part of SIP URI defined in RCML Dial/Sms:

````
<?xml version="1.0" encoding="UTF-8"?>
<Response>
    <Dial>
        <Sip>
        sip:alice@example.com?mycustomheader=tata&myotherheader=toto
        </Sip>
    </Dial>
</Response>
````



# Header translation map

Before sending any custom headers / tags a special header translation map is applied. This functionality allows a CSP provider to rename names of custom headers to other names.
A header translation map can be configured in a subscriber profile. 


## Examples of configuring

An example for a voice part:

```
{"multiProvider": {
  "outbound-voice":[
    {
      "proxy":[
        {
          "uri": "replacement1.com",
          "username": "user1",
          "password": "pass1",
          "headerTranslationMap": {
            "P-Asserted-Service": "X-SomeCustomHeader"
          },
        },
      ]
    }
  ]
}
```

An example for messaging:

```
{"multiProvider": {
  "outbound-sms": [ 
    {
      "destination_network_id":"101",
      "headerTranslationMap": {
        "P-Asserted-Service":"X-TLV-COCTET-3000", "MSIDN":"X-TLV-COCTET-3001"
      }
    }
  ]
}}
```


## Special meaning for TLV headers for outgoing SMPP messages 

Translated headers with names starting with "X-TLV-" have a special meaning. Headers with such names will be converted into SMPP TLVs before of sending by SMPP protocol.
The name contains data in a following format: "X-TLV-<TLV type>-<TLV tag (a decimal value)>". A below table contains a list of possible TLV types.

|TLV type |Description | Name Example |Value example
|---------|------------|--------------|------------|
|1BYTE  | A digital 1-byte value (0..255)                   |X-TLV-1BYTE-3001 | 150
|2BYTE  | A digital 2-bytes value (-32768..32767)           |X-TLV-2BYTE-3002 | -1001
|4BYTE  | A digital 4-bytes value (-2147483648..2147483647 )|X-TLV-4BYTE-3003 | 2147483
|COCTET | A string value (ASCII only available)             |X-TLV-COCTET-3004 | Some text
|OCTET  | A binary value byte per byte hex encoded          |X-TLV-OCTET-3005 | 0199FEAB3A5B

When RC sends an SMPP message then an encoded TLV will be added into it.
When RC send a SIP message such header will be added as a SIP header. Then SMSC GW will convert such header into a TLV field. 
Only renamed custom tags that fits a TLV naming convention will be put as a TLV into a message. All other headers will be dropped. 


