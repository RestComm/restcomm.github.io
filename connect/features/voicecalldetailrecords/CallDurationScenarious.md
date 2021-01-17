Call durations scenarios
========================

When we have a RCML script in the middle, we can have 2 general scenarios:

* Inbound SIP call to a Number to which RCML is assigned (SIPInbound)
* HTTP REST API request that rings to a SIP Number (HttpOutbound) and, when answered, a call to a Number to which RCML is assigned

A later behavior is same and depends on a RCML contect, so we need to treat a same RCML for 2 scenarios (SIPInbound and HttpOutbound).

What we need to understand how we want to calculate the duration (with a substruction or not Ringing) for a variety of cases. We have a set or outbound legs and it is not clear how will we substruct of ringing.

**Scenario 1a: InboundSIP -> no Dial verbs (for example only Say/Play/Record/Gather)**

* 3 seconds playing / saying

_TestSuite_: CallDurationTest.testSaySipInbound(),testRecordClentRelease(),testRecordRestcommRelease() - **failed**  
_Duration SipInbound_: 3 seconds

**Scenario 1b: OutboundHttp -> no Dial verbs (for example only Say/Play/Record/Gather)**

* 1 seconds ringing to OutboundHttp
* 3 seconds playing / saying

_TestSuite_: CallDurationTest.testSayHttpOutbound() - **failed**  
_Duration HttpOutbound_: 3 seconds (we have now 4 in CallBack, 3 in CDR)

**Scenario 2a: InboundSIP -> Dial sip**

* 1 seconds ringing to SipOutbound 
* 3 seconds talking

_TestSuite_: CallDurationTest.testDialClientSipInbound(), testDialClientWithTimeLimitSipInbound() - passed  
_Duration SipInbound_: 3 seconds  
_Duration SipOutbound_: 3 seconds

**Scenario 2b: OutboundHttp -> Dial sip**

* 1 seconds ringing to HttpOutbound
* 2 seconds ringing to SipOutbound 
* 3 seconds talking

_TestSuite_: CallDurationTest.testDialClientHttpOutbound() - **failed**  
_Duration HttpOutbound_: 3 seconds (we have now 6 in CallBack, 5 in CDR)  
_Duration SipOutbound_: 3 seconds

**Scenario 3a: InboundSIP -> Dial sip -> Say/Record**

* 3 seconds ringing to SipOutbound 
* 10 seconds talking
* 4 seconds Say/Play

_Duration SipInbound_: 14 seconds  
_Duration SipOutbound_: 10 seconds

**Scenario 3b: OutboundHttp -> Dial sip -> Say/Record**

* 2 seconds ringing to HttpOutbound
* 3 seconds ringing to SipOutbound 
* 10 seconds talking
* 4 seconds Say/Play

_Duration HttpOutbound_: 3+10+4=17 seconds  
_Duration SipOutbound_: 10 seconds

**Scenario 4a: InboundSIP -> Say/Record -> Dial sip**

* 4 seconds Say/Play
* 1 seconds ringing to SipOutbound 
* 3 seconds talking

_Duration SipInbound_: 4+1+4=8 seconds  
_Duration SipOutbound_: 3 seconds

**Scenario 4b: OutboundHttp -> Say/Record -> Dial sip**

* 2 seconds ringing to HttpOutbound
* 3 seconds ringing to SipOutbound 
* 4 seconds Say/Play
* 10 seconds talking

_Duration HttpOutbound_: 3+4+10=17 seconds  
_Duration SipOutbound_: 10 seconds

**Scenario 5a: InboundSIP -> Dial sip -> Dial sip2**

* 3 seconds ringing to SipOutbound 
* 10 seconds talking
* 3 seconds ringing to SipOutbound2 
* 10 seconds talking 2

_Duration SipInbound_: 10+10=20 seconds  
_Duration SipOutbound_: 10 seconds  
_Duration SipOutbound2_: 10 seconds

PS: not sure if our algo allows to substruct several Ringing for several legs

**Scenario 5b: OutboundHttp -> Dial sip -> Dial sip2**

* 2 seconds ringing to HttpOutbound
* 3 seconds ringing to SipOutbound 
* 10 seconds talking
* 3 seconds ringing to SipOutbound2 
* 10 seconds talking 2

_Duration HttpOutbound_: 10+10=20 seconds  
_Duration SipOutbound_: 10 seconds  
_Duration SipOutbound2_: 10 seconds

PS: not sure if our algo allows to substruct several Ringing for several legs

**Scenario 6a: InboundSIP -> Dial sip no answer**

* 2 seconds ringing to SipOutbound 

_TestSuite_: CallDurationTest.testDialClientNoAnswerSipInbound() - passed  
_Duration SipInbound_: 0 seconds  
_Duration SipOutbound_: 0 seconds

**Scenario 6b: OutboundHttp -> Dial sip no answer**

* 1 seconds ringing to HttpOutbound
* 2 seconds ringing to SipOutbound 

_TestSuite_: CallDurationTest.testDialClientNoAnswerHttpOutbound() - **failed**  
_Duration HttpOutbound_: 0 seconds (we have now 3 in CallBack, 2 in CDR)  
_Duration SipOutbound_: 0 seconds

**Scenario 7a: InboundSIP -> Dial sip no answer -> Say/Record**

* 3 seconds ringing to SipOutbound 
* 4 seconds Say/Play

_Duration SipInbound_: 4 seconds  
_Duration SipOutbound_: 0 seconds

**Scenario 7b: OutboundHttp -> Dial sip no answer -> Say/Record**

* 2 seconds ringing to HttpOutbound
* 3 seconds ringing to SipOutbound 
* 4 seconds Say/Play

_Duration HttpOutbound_: 4 seconds  
_Duration SipOutbound_: 0 seconds

**Scenario 8a: InboundSIP -> Say/Record -> Dial sip no answer**

* 3 seconds Say/Play
* 2 seconds ringing to SipOutbound 

_TestSuite_: CallDurationTest.testSayDialNoAnswerSipInbound() - **failed**  
_Duration SipInbound_: 3 seconds (we have now 0 in CallBack, 0 in CDR)  
_Duration SipOutbound_: 0 seconds

**Scenario 8b: OutboundHttp -> Say/Record -> Dial sip no answer**

* 1 seconds ringing to HttpOutbound
* 3 seconds Say/Play
* 2 seconds ringing to SipOutbound 

_Duration SipInbound_: 3 seconds  
_Duration SipOutbound_: 0 seconds


Live Call Modification scenarios (LCM) 
======================================

it depends on the LCM Case but:

- a new Dial will create a new leg so inbound will keep following the same rule as ringing time don't count in the duration.
- a Say and Play in an existing conversation will not add anything in terms of duration
