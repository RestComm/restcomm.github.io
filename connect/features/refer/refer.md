# Refer Feature

This feature is about implementing [RFC-3515](https://tools.ietf.org/html/rfc3515).

Related Call Flows maybe found at [RFC-5589](https://tools.ietf.org/html/rfc5589#section-6).

RC will implement blind transfer with Target-Dialog support, and previous Hold. 
Transfer with consultation, or transfer with Dialog reuse will not be implemented.
Under RFC definition, RC will become Transferor role.

# RCML Spec
This will be based on [Twilio spec](https://www.twilio.com/docs/voice/twiml/refer).
Some notes:
- Refer only has "action" attribute. There is no status-callback. This means "action" will be invoked with final status of transfer (whether success or failed).
- Its possible to inject custom headers into the outgoing REFER request using regular SIP URI convention.
- Refer feature may only be invoked for SIP Call legs, whether call is bridged or not. Trying Refer on non-SIP call legs will result in error.
- Refer is a terminating verb. If no action is provided,and RCML doc is not including a subsequent verb, the call will be terminated.

# Impact on RC
- Current VoiceInterpreter contains all RCML verb logic.
- Refer logic needs to be added to VoiceInterpreter.
- RC as potential  participant of transfers, needs to properly advice support through standard Allow and Supported headers.

# Moving FSM refactoring forward
- Separation of RCML verb logic from VoiceInterpreter is the most important point in the solution. It will greatly simplify voice interpreter code, and improve overall testability and maintainability of project.
- Inserting new FSM framework in VI and resulting RCMLVerb FSMs is suggested.
- A protocol between VI FSM and RCMLVerb FSM needs to be created. SUBSCRIBE/NOTIFY or REFER/NOTIFY approach is suggested as a guideline.


# Pending Issues
- For Bridged calls, what happens to the other callLeg?

# VI Refactoring principles
- VI is stateful component that orchestrates service/RCML triggering.
- VI uses an RCMLFactory to bootstrap the corresponding RCMLVerbFSM. This factory may use provisioning/config data to discover the appropiate FSM
- RCML Verbs intercommunication is completely removed. Different Verbs are not aware of each other.
- VI holds the sessionContext, and only VI may modify it.
- VI serializes sessionContext on subscription to service/RCML. Content of sessionContext is mainly URIs to different resources (SipUris,MGCPUris, ProvisioningUris, ChargingUris...)
- VI sends the RCML Tag as it is to RCML Verb on subscription. The RCMLVerb may reject the VI subscription if the RCML Tag is violating some specific verb restriction.
- VI centralizes interaction with HTTP servers. RCMLVerb is freed from this interaction.
- RCML service notifies interim and final status to VI. These notifications may contain special/generic orders for VI to execute (sendstatusCallBackWithParams, retrieveActionWithParams, modifySessionContextWith)
- VI executes one RCML Verb at a time. Nesting verbs is considered a detail of a particular Verb, and VI has no visibility on it.
- VI is a listener of any resource started during the session. Resources will send events to VI. VI needs to provide a basic handling of such events (specially when resource is terminated in the network). 
If a resource event causes VI session to be terminated, VI will send unsubscribe to ongoing RCMLFSM if any.
- RCMLVerb is a listener of resources created during the RCMLVerb logic execution. When RCMLVerb completes, VI observes the resulting resources.
- When VI finds session is completed, it has to release all Resources contained in sessionContext.
- VI centralizes Notifications generation. RCMLVerbs only send notifications to VI,that may spawn a notification.

Concerns:
- VI needs to check whether RCMLVerb is allowed to be executed!! Use XML schema and apply static(this is not enough) Evaluation of sessionContext is required.

![SunnyDay](http://www.plantuml.com/plantuml/png/dL9DRvmm4BtpAoOtDgBx0pXKQQWY8MsgA2ZqF9ZPhLLZe_4uAV--vWD4kcqFkSxxU8-7xxmWI-XC8beCmNFnU88gM3jynG5wTegyr_QIr2Ly-WnrcKFhtgnns8xKtNdXVgDVKXCt2pAI7b29uo479c5Dh_HGFTu7RLfYdzg5VqXsrDaJSdYDo4BT6OvykBsDk692eJ-X77nel6BTKEh7_iuMpafCog13RTUyXRw6eLBK2xNf8KcEnoDCu1FnsN7duUGkMa6yps0P_LXtO9teym1lMdwu8A3Gc0RLpv5ubR2GKprhAq49_l1Flm-OPPFcFcfUda_PDDyJCdOnPTvejQo88nY7ccJ5SasPBi4Wr-MiokhD8DbjKhYkDoIqZl9zBFj5ITpvlrNFsg9PY-B4Vs7pjvgRmyrDbBVaXTqGiSqnz2qALUz-xjv_DTfIuFt3lVhIVOSxictep6y0)

![User Aborted](http://www.plantuml.com/plantuml/png/TP71Ri8m38RlVGfpAsZZ0NgOs92Ga62QLerxcXY8bYR8Tc2y_K8RiL9fjyh_r--NumMB8ecjAyIAG_XSRZTG8xCD7bGJID0KkpKyly1kNO495--2CZTiC3vHqCZyZZ-oGsfoSeDMIakWQmM4GXRFkAgtRz4wWyIbp_oq6A2z4oeufoIZ0-6DXzaivPfG2OwZ2ZWshCasM7A_np9uIKrFq2hhSr_6xsLAQpm9cd9Q5FFv7Bhm0saVg6NOX8FLaEPzk4u-iYtu87P_XOkLOuW2MZdNAFtTWUE639Us_xSM9P5psHSZmL7v0ySJB9EbCtGV_GDtwjVXDOOBHtzV-T_QlVNAvTAnOJ99d44M2jVxKVz6zSChFfeMw4ritIy0)

![Gather Verb](http://www.plantuml.com/plantuml/png/PP9FRzim3CNl_XJipIcy5FjpCh2XNO0E1fAXg57lp4XCAbIM38fgvsy_-SzcTxa9vDFxFOgw3O8iiRMP6B44Zxl37dwWF1F3cv5dgB7FcWe-V8EcAa7xDsSzwa6sAdVi5ONmatvBGtoSEuFe7fLuxajW4ZRqIlVruNiqLXE7tCtwJRQIJfvGmclgpDB9SCwL7E3wEWaK4mfTz4e8yHWKcoFb8QKotksyayGgu3ogRWBsDvt5VW8F4VvHKIdgX7I-oRBjTWjZyu7uvBaDiFTdeo3LiuKtJ7geONo5jfCxftVUMbAIQsbFHM8zFUuBwEnuICWaNWSkmRzChCe9iMd1UJ6dVdn_7mKKMdj4zJ9mBsv3ESlpQJRINpz--lnbdNwNObaI-Xl_L_A5EoN6kyjHccfnF1c7jXQK60h0yS8JssBSp0Cojk4_wEPY6LTNoKvJ4FRBFqgbs6aKgj-jPysRdHOglFzyETCvMkIcce8KIwPDTajEnzR-0G00)

![Dial Verb](http://www.plantuml.com/plantuml/png/VLBDRjim3BxxATYRNNZPtOUXdOS20PgjMB3ipCYCAqoM38fgbhUV52k6uyDyCK1-FnyfFdb1bZ2OGhGOWf_jxXM-QJJ5OGkVli1Xsy38Jf7tt-cl78YFkPEGBwZ4yQK19c5D7_IottqDsdJ4lhGPNnKBwAG93Np4JePcmUGQ-V3u6DkA2OLMdGPyE7h5-aYbZucqiwsp4drgw2BcizTZLMJqx8HAqIZKbLLHR3ORI9dWZVZaUF3mwdgj8DQNl1QDsQ75ddZEachbYTrGYwlXdJ0D6NbLiKH7yeyWlTEsJ-KuJ3FHrwZire0BSaFF4LOgYxcd4RPMYFK0rea-NU2ivJtAW8hO8we_rt6M0lTBJB1dwy_4rDrlgycA9UPs3LvGWc-pSfLVP7zNNUorTSDeIEYssDyn6ZmhqSx-BzKKIcNoAXNCiKxQefcyQy2hwYyszVMOtf1nyELu_MtYLrMhytbuY4eTUM5tYQrPF8mT2ktl3cffTTEP3PECErlZi4kdNJBrmI0VjbE6UeuJgKzakp2OVm00)

# Planning Breakdown - Full Blown approach
1. Create new VI FSM with new protocol: 3
2. Refactor DialFSM: 8
3. Refactor SayFSM: 3
4. Refactor PlayFSM: 3
5. Refactor RedirectFSM: 2
6. Refactor SmsFSM: 2
7. Refactor EmailFSM:2
8. Refactor GatherVoiceFSM: 5
9. Refactor GatherDTMFFSM: 3
10. Refactor HangupFSM: 2
11. Refactor RecordFSM: 3
12. Refactor RejectFSM: 2
13. Refactor PauseFSM: 1
14. Create Refer FSM: 5

# Planning Breakdown - Iterative approach

First Iteration

1. Refactor VI FSM with new protocol: 5
2. Create Refer FSM: 5

Remaining Iterations (any order)

- Refactor DialFSM: 8
- Refactor SayFSM: 3
- Refactor PlayFSM: 3
- Refactor RedirectFSM: 2
- Refactor SmsFSM: 2
- Refactor EmailFSM:2
- Refactor GatherVoiceFSM: 5
- Refactor GatherDTMFFSM: 3
- Refactor HangupFSM: 2
- Refactor RecordFSM: 3
- Refactor RejectFSM: 2
- Refactor PauseFSM: 1

# Misc Planning
- Capacity : 5
- SynthMon : 5
- Doc : 2

# Attended Transfer support
Customer requested to include this scenario. This scenario introduce the additional challenge to consult if the transfer is ok for the taget before doing the actual transfer. So,
RC needs to establish a new call with transfer target, and play an IVR menu to check if target accepts. If target just hangup, RC assumes the transfer was rejected. If target accepts,
RC needs to include additional Replaces header parameter in Refer-To header.

Following sequence diagram depicts how RC will handle this.
![Attended accepted transfer](http://www.plantuml.com/plantuml/png/jLPTRzis5xxNhpZGHLy7KFtXzywGt3gAsmPjMiFnkXP1OJ3IiSr58WMIof1_lqEAbcKjtPO10n948S_tyvn7UgiDKwOkYu3LHegwl4S5ax7z2UFx-TruFFuH1XNPy8nNJ1W85o4m3OlpmpLgaycoXC7bXRjVf-S6AoKVkUPISB5t1gk2cPrKfRtenKflFi6YgWgUCKCUpjmUqjcvPUgkqY184bW05_Uo5Zbah2WWIO85l9wJtmokFt-ztYm7HYOn5rGbbR0wI87pYxlRwzMFO9rQosv1CtY_t0n6pJpqw4zk7pORykjwDBbW1jFn-Cl7GQooilXb_QcrRNBuBbpuayqRM_4jpnFDyyd9mPdsubgNxPwP0wf1tFpTiinEQJAFOlwejYMAUZA_E8OHqY38HrGK9Mgd61DUbHGQvm40GFpUHO8cKXAEShxdHY-5GOLv25nKjKb2NI5c1yXDkGl1y1AbFVr_23IdA-UHX9Em2Ud2thAd-Qf0fX4gXLh31AG2pPxejq0DDBpcQHYFoDONCsggcGVnw5Gc4h2s5p_8HJiHdLUp-5xnV8-tLoiFwGdxHVhHdNYEx2MbvS98kFww9f9qvhE86WSu85CQx1n1LdOW4J2H-mwqBD4V7vF0hBbp8nWGWEkV07u-rXRroIjtdlmJw0-ooBSvPuLbSneT4OipANHT6BSlQq-hbr7W0qKU-UShIj0b0ojIMLO56ipFuEKLrAeWasVIXtFDgrdsDfSbus98-qCOy-g8iKrnZd3mAmBHZ02G0coV0R50WCqEHaBgsC85DvpIY3tGBHZ5XDwXQkdXO5wvA0NkRIJRbmNQut5pImFD3apiCPL2tCWS1vgyluZYwQ93hlNojPSD8bWIiyGQIJ9g6TRV-CMM-1UnH67sw0oxuh8uCV4vod4DGlxUDkwuQWnbnBpjyTWxvX5a2kd1APsRJQz1VvzVkhMuktcVKbgwQKkM3uFJpapdZOMFOEyxhNVjVEjfl_LqsrczjeuP55myR6HJXRUwUBOQxzJkQtz_e5cW7irxwZdzjzAvdNQqS_hVQEVBH3EjjP4bHVq6EoLBiD4gnJN2T-3QuKuh447L5LSfq9A1PHbMvcmzhEfQojULDL3u6sRcxoKts52rRkkwOxT9sPEJJykyq-RrnaLOfbvAsJDKtabuvNLKHoTvTSFxIHT1X-1zmUqnwlENpVMxxuFS6xIawqIG3tRBKfz8jraYs771zU6PgfFa1Qe-_KjLFoEbGeh-CjAjhSkeXrV_mmboYLhypu10suOa79N2mYh7lJG7HoCTj9zk1wRTkPDSkNSWgRA9n0LeLY9erDg--hG4RYY0E4rdqUT3LqpVBTS3r-Ph-zvw4oia5cMeqsPQqs6JSnw1OMgFvihzcyH-cXD_tnDVEyLfvuIWzygrzl5dlGIjXDdAF0ZT04w0-Yft-NqvQDodRVuUqoYGVF2NlI-K_Y55wmVfs5bIReuZRT-w_IyLLwZxAac1ARJmh2w_EK0ZQ07b6cHjGExS6cc-5ypAXlChaJg2YjRUlNc6ms7mbg2ho-8F)

## Call Screening behavior

After creating a comp test suite for call screening testing the following additional findings are shown.
The scenario is call screening procedure initiate "Gather" operation and then we process 2 types of RCML (in 2 tests) empty that leads call bridging and "Hangup" that leads all calls terminating.

SubVI returns two types of response to VI:
1. "End" - that leads of call bridging
2. Several error responses (Hangup, Cancel, Exception) (after Hangup/Cancel RCMP verb, not installing of a second leg, any error) - this leads of both call legs terminating and "Dial-action" is invoked only for reporting, no RCML is processed


None of 1) and 2) action allows to continue call 1 with terminating of call 2 but without of calls bridging, so no possibility to "Say" something to call leg 1.
We need to change 2) behavior to allow Dial.action to be invoked with information about the particular error,while mantaining parentCallLeg alive for potential interaction.

## RCML Spec
### Hold
Hold verb allows to mute a call leg. The mute involves both media and signalling operations. At media level RC needs to tweak the corresponding connection mode to prevent receiving RTP packets into the bridge
. At signalling level, RC needs to send a REINVITE SIP message with SDP "a" attribute set to "sendonly".
```
<Hold mode="sendonly|inactive">
```
THe hold verb support an optional parameter to set the "a" attribute mode in the REINVITE. By default, it will be "sendonly", but RCML may set it to inactive. Hold verb doesn't allow 
any nesting.

### Resume
Resume verb allows to unmute a call leg. This again involves both media and signalling operations. At media level RC needs to tweak the corresponding connection mode to allow receiving RTP packets into the bridge.
At signalling level, RC needs to send a REINVITE SIP message with SDP "a" attibute set to "sendreceive".
```
<Resume>
```
This verb do not support any parameter or nesting.

### Refer
Refer verb is used to initiate a call transfer. VI will take parentCallSid and send SIP REFER to this call leg, assumed to be the trasferee. 
```
<Refer action="url" method="GET|POST">
    <Sip>sipuri</Sip>
<Refer>
```

Refer verb maybe used for both blind and attended transfers. For blind transfer an optional "Sip" subelement is supported indicating the target trasfer sip uri. As usual,sip URIs may contain any URI parameters, or header parameters to be populated in the REFER request.
For an attended transfer, Refer verb wont include any sip subelement, assuming there is previous Dial verb executed with an outgoing call leg.
The optional "action" attribute allows RCMLApp to be invoked with result of Refer operation. Both refer and notify response code will be provided in the request.

## Additional Planning
From previous design there are additional items:
- Hold Verb 3
- Resume Verb 2
- Refer FSM with Replaces 1
- Ensure CallScreening correctness 2

