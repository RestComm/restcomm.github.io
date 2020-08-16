# Telestax solution for messaging

## Overview

A new messaging solution for RC that we are introducing now define a schema when we have a RC cluster in the center and messaging gateway(s) per a messaging protocol.
RC will process only SIP messages (by usage of a SIP protocol) and the gateways will convert messages to a specific protocol and send messages further to operators. 

## Routing of messages to a specific gateway (message outbound)

### Default processing

The default processing (when we have no profiles) is: if we have an SMPP connection (that is true now for all pops) then all PSTN messages will be routed to the SMPP connection and all SIP messages will be sent to a SIP network.

The routing of messages into a concrete messaging gateway is configured by a provider profile per Accounts(s) / Organization. Routing is applied per a protocol scheme.
We use "tel" prefix for pstn directed messages and "sip" for sip directed messages.

### Specifying of a concrete destination
 
We may want to change this behavior for Accounts(s) / Organization when we want to send messages for sms-outbound to a dedicated server/gateway by SIP protocol and this server/gateway will resend messages with a proper protocol to a needed destination.

For applying of this behavior we need to provide a special subsection "channelsRoutingMap" into a section "platformSessionControl". Routing is applied per a protocol scheme. We use "tel" prefix for pstn directed messages and "sip" for sip directed messages.
We may have other protocols for routing when we have implemented them. Important is adding of "bypasslb=true" for we will send messages directly into SMSC GW without a LB. We can use an IP address of SMSC GW like "10.0.0.146" but better to use a domain that is associated with a SMSC GW.

Example:

```json
{
  "featureEnablement": {
     ..............
  },
  "platformSessionControl" : {
    "channelsRoutingMap": {
      "tel": "sip:vpn-smsc-001.staging-us-east-1-restcomm-internals.com:5060;transport=tcp;bypasslb=true"
    }
  }
}
```

## SIP headers / parameters that are reused at a leg RC - messaging gateway

### Trunkgroup parameter

When RC send a SIP message to a messaging gateway RC adds a `tgrp` parameter into a `To` field for specifying a routing info. 
For SMSC GW: it specifies a networkID value where SMSC GW will route a message. 

Example of a `To` header that contains the information of a "101" networkID:

```text
To: sip:11112223333@vpn-smsc-001.staging-us-east-1-restcomm-internals.com:5060;tgrp=101
```

When SMSC GW sends a message to RC then it adds the same header into "From" header. This value is not used now at RC, we can think of using it for some purpose in future.

### Message ID and delivery receipt

When RC sends an outbound message to a messaging gateway (SMSC GW) it adds 3 headers for initiating of a delivery receipt.

- `NS`: specifying of an "imdn" protocol
- `imdn.Message-ID`: a message ID (sid of a message at a RC side) for a further correlation
- `imdn.Disposition-Notification`: requesting of a delivery receipt (`positive-delivery, negative-delivery` that requests both success and failure delivery results) 

This is [Instant Message Disposition Notification (IMDN)](https://tools.ietf.org/html/rfc5438) specification.


A `Content-Type` of a message is `text/plain` and a message body contains a message text

Example:

```text
Content-Type: text/plain;charset=utf-8
imdn.Message-ID: SM55072e8e32804aaeb8b0d2fa74832935
NS: imdn <urn:ietf:params:imdn>
imdn.Disposition-Notification: positive-delivery, negative-delivery
```

When a DLR has come to a messaging gateway (SMCS GW) then the gateway sends a special message with a `Content-Type` header value `message/imdn+xml` and a header `NS` that specifies of an "imdn" protocol. 

Example:

```text
NS: imdn <urn:ietf:params:imdn>
Content-Type: message/imdn+xml
```

A message body contains a IMDN (an xml-encoded body) with a delivery receipt details.

Example:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<imdn xmlns="urn:ietf:params:xml:ns:imdn">
  <message-id>SM55072e8e32804aaeb8b0d2fa74832935</message-id>
  <datetime>2020-07-07T14:32:00+0000</datetime>
  <recipient-uri>sip:491774699123@10.0.0.146:5060;tgrp=101</recipient-uri>
  <delivery-notification>
    <status>
      <delivered/>
    </status>
    <reason-code xmlns="urn:ietf:params:xml:ns:">0</reason-code>
  </delivery-notification>
</imdn>
```

* `message-id` - is a messageID of an original message for correlation
* `datetime` - is a message delivery time from a delivery receipt
* `status` field explains was a message delivered of failed, possible values: "delivered" or "failed".
* `reason-code` corresponds a code from a DLR + translation applied by SMSC GW if it was configured for the operator

### Segments count

SMSC GW adds an extra header for an inbound message that says of a count of segments from which SMSC GW has reassembled a message.
A header name is `X-SEGMENTS-COUNT`.

Example:  

```text
X-SEGMENTS-COUNT: 1
```


