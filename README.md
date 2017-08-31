# acu-splitAP-launch: A _Split Activity Provider_ Launch mechanism (xAPI)

## Intro 

acu-splitAP-launch describes a generic launch mechanism for xAPI browser-based content that favors security and privacy over other considerations. While it takes ideas from existing launch mechanisms, it makes explicit a very strong assumption: the browser-based content **alone** does not have enough information to send statements directly to an LRS: it does not know anything about the destination LRSs, and it may not even know the real identity of the user (actor). The browser-based content, then, **needs** the cooperation of a server-side system so the statements can ultimately be sent to one or more LRSs.

Thus, the browser-based content by itself cannot be considered an xAPI Activity Provider. However, if we consider both parts together: the browser-based content and the server-side component or system, then yes, we do have an Activity Provider (it's capable of sending complete and correct statements to an LRS). This AP is _split_ in two parts (client and server), that is why the name of this launch mechanism contains the word _split_.

With this model, not only can we keep all the confidential information on the server side of the AP, but we can do things that would be either impossible, very difficult, or very insecure to do on the client side (like signing statements, for example.).

Obviously, the burden on the server side component is considerable. It must:

    1. Launch the browser-based content
    2. Manage session information about the launched contents
    3. Manage credentials to one or more LRSs
    4. Provide an endpoint where the browser-based content can send statements
    5. Act as proxy to an LRS, and manipulate statements/information than it receives from the browser-based content before forwarding it to the destination LRS(s)
    6. Possibly host the browser-based content
    7. Manage access to the browser-based content

These functionalities might be implemented very simply or in an advanced manner.

This launch mechanism is only concerned with the runtime behavior, not with any packaging of the content.


## Rationale

In an xAPI environment where most of the statements will originate in browser-based content, the launch mechanism is vital, because it:
    1. sets up the 'actor' information that is needed for the statements
    2. tells the browser-based content **where** to send the staments (an endpoint)
    3. provides the browser-based content with _some_ authorization to send statements to the given endpoint

The simplest (and most widely known ) launch mechanism provides all these data on the launch url query string. So the information is visible, easy to get... which is not good for security. However as the creators [clearly explain in this document](https://experienceapi.com/launch/), that launch mechanism was intended as a temporary solution. And more importantly, **the launch is NOT part of xAPI**.

More advanced launch mechanisms ([adl-xapi-launch](https://github.com/adlnet/xapi-launch) and the [CMI5 launch](https://github.com/AICC/CMI-5_Spec_Current/blob/quartz/cmi5_spec.md#content_launch)) necessarily involve a **server** component. In both cases, this _server component_ is not at all trivial. In fact, in CMI5, the _server component_ is the LMS, responsible for launching the content ([_The AU MUST be launched by the LMS_](https://github.com/AICC/CMI-5_Spec_Current/blob/quartz/cmi5_spec.md#81-launch-method). Also, CMI5 assumes that the LRS is part of the LMS (or at least they work together).   In the case of adl-xapi-launch, the server acts as a proxy to the real LRS, and in fact it signs the statements before forwarding them. ADL's xapi-launch avoids placing PII (personally identifying information) in the query string, which is an improvement for privacy. Both systems make assumptions. Both have strengths.

There is also (usually) a need to control access to content (allowing only authorized users to launch it).

One assumption that all the mentioned launching mechanisms make is that there is only one LRS involved in the xAPI solution, and/or, that the LRS is always the same. What if we want to allow content to send the statements to more than one LRS at the same time (not a 'normal' situation, but there are scenarios in which that would be useful or necessary). What if, in addition, we want the content to send the statements to one LRS or another, depending on some conditions that can only be asserted at runtime? what if we just want to improve privacy/security when launching content? What if we want to perform consistent tracking on anonymous users?

For different requirements, different launch mechanism are needed if the existing ones are not a good fit for the situation.

The acu-splitAP-launch  came about to address some present and future needs for things like those mentioned above. It's very open: for the server side it just prescribes a couple of things. Everything else is unspecified: a server system that implements acu-splitAP-launch may implement some of the more advanced capabilities any way it wants, or not at all. Also, existing launch mechanisms provide very useful ideas that can be used, modified or intact, in any other launch mechanism. All or most of the Launch Data items are taken from CMI5.

Really, documenting the acu-splitAP-launch publicly is just a way to stress some things about the launch and to provide ideas:

    - the launch is important (for browser-based content)
    - the launch is not part of xAPI
    - the fact that an authoring tool implements just one launch mechanism, doesn't mean that's the only way to launch the content
    - maybe in your situation you need to come up with a new launch mechanism
    - here are some ideas, hopefully some are useful to you in your situatio and you can take them


## Algorithm

    1- A student requests a launch for a piece of content (Unit, or AU, to use CMI5 terminology).
    2- The acu-splitAP-launch server receives this request (maybe from its own web-based interface, or from a remote system) and determines if that user has permissions to launch that content.
    3- If the user is not authorized, the request is rejected.
    4- If the user is authorized, the acu-splitAP-launch server generates a session with a session id (uuid), stores the user (actor) information in the serverside session along with whatever other session information it needs. It also saves in the session any Launch Data associated with that AU.
    5- The acu-splitAP-launch server serves the content to the user, placing a session cookie whose value is the id of the session. 
    6- The server provides **two** fixed routes to the content:
    /LD: This is a one-time per session use endpoint where the content can retrieve the Launch Data. It will be available for a short time.
    /xAPI: This is a permanent endpoint where the content can send the statements.
    7- When the content 'boots up', it will immediately send a POST request to /LD, to request the Launch Data
    8- The server checks that the session is valid/active. If it is not (no cookie session, or invalid value, or the session timed out), it redirects the client to a predefined route. If the session is ok, it continues processing the request. (The server always behaves like this). If the session/request is ok, it sends the Laund Data in the body, as application/json, and immediately deletes the Launch Data structure from the serverside session. This way, subsequent requests to /LD in this session will fail.
    9- Once the content receives the Launch Data, it verifies that it is correct. If it is not, it informs the user about it and  redirects the user to a predefined route, ending the session. If it is ok, the launch sequence has finished. At this point, the content can send statements to the /xAPI endpoint.


## Server 

The defining characteristic of the acu-splitAP-launch server component is that it is a _manipulating xAPI proxy_, providing one endpoint where the browser-based content can send statements.
This is similar to adl-xapi-launch, with the difference that in adl-xapi-launch the endpoint includes a variable part (different for each session, the launch token) and in acu-splitAP-launch the endpoint is fixed, always the same: `/xAPI`. This makes it easy on the content. No need to get the endpoint from anywhere. It's always the same.

Just like in adl-xapi-launch, in acu-splitAP-launch the browser-based content sends statements to the server without an Authorization header, but with a cookie. The server verifies that that the value of the cookie corresponds to a valid session (otherwise it rejects the request). Aside from this, the /xAPI enpoint is just like a real LRS endpoint, as far as the browser-based content is concerned.

The /LD endpoint is valid just once per session, and it is valid only for a short period of time (2 minutes or less) after launch. This is to increase security somewhat. The idea is similar to CMI5, and the launch data can be fairly rich, and it's almost the same as in CMI5. Obviously, the difference is that in CMI5 the content retrieves the launch data from the LRS, and in acu-splitAP-launch, it retrieves it directly from the launching server. This assumes that the acu-splitAP-launch server (the launching server), has some way to determine the Launch Data that is associated with an AU. It may have it stored in a DB, with other information abouth the AU, or the launch data may come from the remote system that is requesting the launch, or a combination of both. This is an implementation detail.

The server can place fake 'actor' information in the launch data, if it wants to keep the real actor identity concealed. This way, the content will get some formally valid actor information to put in the statements. It could as well generate it randomly.

When the server receives a POST or PUT to the /statements resource, it will manipulate the statement to remove the fake actor and add the real one. It may also check that other info in the statement is unaltered/correct (e.g. the context). 

The server will also need to manipulate query strings to replace the fake actor info with the real one before forwarding requests to the LRS. For example, when loading or saving state, the agent parameter is required, and the server must use the correct one to make the request to the LRS.

The server may make other changes, like overwriting or removing the id, or signing the statement. This all depends on the specific needs.

The server manages the credentials to the LRSs. The real endpoint of the LRSs and the credentials are **never** sent to the browser. The server system should implement the  mechanisms to determine to which LRSs it should send the statements for a given session.

The server must not change the verb and the object parts of the statement. In a split AP scenario, each side will naturally have more (or all) responsibility for some parts of the statement, and the other part should 'respect' what the other side sets (otherwise, the parts would not be collaborating). Since the verb and the object are derived directly from what the user is doing while interacting with the browser-based content, they should be set by the browser-based side, and the server side should not touch them. The result is another part of the statement that would probably be set by the browser side. The context (context template) is initially set by the server and passed to the browser in the Launch Data. The browser-based content _could_ add data to some specific pieces of information (e.g. to the contextActivities), but should not modify what was already there. The server could check of the browser-side has behaved or not. Some elements, like attachments, could be added by either side. The server side has to be careful when reassembling the statement not to ovewrite or delete any 'legitimate' data received from the browser side.


## Launch Data

The first thing that the content must do when launched is to retrieve the Launch Data, which is a json string that, when deserialized, results in an object with various properties. Most of these properties are taken from CMI5, and have the same meaning. Differences with CMI5 are noted. The Launch Data should have:
    
  - `sessionId`: A uuid4 that identifies the session established between the launching server and the browser-based content.
  - `trkCookieName`: The name of the session cookie.
  - `activityId`: A unique id for the activity being launched. Unlike CMI5, there is no requirement that this activityId be different from the publiser's assigned activityId.
  - `registration`: The registration id of the student's enrollment corresponding to the AU being launched.
  - `launchMode`: Same as in CMI5
  - `masteryScore`: If present, must be a decimal value between 0 and 1.
  - `moveOn`: If present, must have one of the values defined in CMI5.
  - `returnURL`: Same as in CMI5. If not present, the browser-based content will use '/' as the return URL (which, most likely, will be the main interface root for the launching server).
  - `bailoutURL`: If not present, the browser-based content will use the return'URL. It's just a way to indicate to the content an alternative route to redirect to when there is a 'forced' exit for some unexpected error condition.
  - `launchParameters`: Same as in CMI5.
  - `contextTemplate`: Same as in CMI5.

Note that some of the elements taken from CMI5 are not necessarily needed per se in the acu-splitAP-launch server component, but might come with the request from a remote system to launch content, so our launching server should just pass them on to the browser-based content.

In general, launch data elements can/will be used by the browser-based content to modify its behavior. 

The context template _could_ be kept on the server side, or at least the contextActivities, but it just seems easier to pass it on to the browser-based content so it can use it to build the statements. The server should, however, check/verify/reconstruct the context if the client has _broken_ it, before sending the statement to the LRS.

Note that there are no entitlement keys of any type passed on to the browser-based content. Authorization to access the content is resolved _before_ the content is launched. In other words, it is the acu-splitAP-launch server the one that determines if the requested AU can be launched. For that, it might use the identity of the user on the launching server, or it might use data that it has received as part of a remote request to launch content, or any combination of data. The browser-based content can be sure that the launch was allowed, and there's nothing else to check.


## Browser-based content

The acu-splitAP-launch is only concerned with the runtime launch algorithm, not with any packaging of content, or with the patterns of statements that the AP should send. It takes many ideas from CMI5, because they make a lot of sense. The launch mechanism specified in CMI5 is just one small part of the scope of CMI5. Most of the concepts/ideas used in CMI5 can be implemented in browser-based content that is launched with acu-splitAP-launch. Obviously, the browser-based content, _on its own_ would **not** be CMI5 compliant, but it could be part of a CMI5 launch.. For example, the prescribed sequence of statements in CMI5, the concept of 'cmi5 allowed' and 'cmi5 defined' statements, are very useful anyway. Why not implement it in the browser-based content? 

So, since the acu-splitap-launch is purposely very limited in scope, for any behaviors that come _after_ the launch, we recommend to use, or at least look at, CMI5 (or other profiles, depending on the particular case).


## Split AP and the advantages of server-side xAPI

Yes, to many it will sound weird to have an activity provider that is itself a client-server app. Somehow there's this impression about xAPI (at least to newcomers) that, outside of the LRSs realm, everything else is meant to be 'client side', and run on the browser. But it's not true. The client to an LRS can be _anything_ that can talk http, be it an app (mobile or desktop), a server, a cluster, a client/server app... _anything_. If an xAPI environment requires security, privacy, flexibility... the odds are high that there will be a need for a server component in that environment. 

It can be advantageus to do xAPI processing in a server environment prior to sending data to LRSs, such as:

  - _Reliable generation of Ids_. It is known that uuid4 generation in the browsers is not very good, causing collisions. This is _bad_ in xAPI, since an LRS MUST reject a statement (or the whole batch) if it already has a statement with the same id. Generating the ids for the statements on the server could be more reliable.
  - _Server-side buffering_ of statements: For an LRS, it's better to receive one request with 500 statements than 500 requests with one statement. A server component can buffer statements based on destination LRS + credentials, and send them in larger batches. A sufficiently smart server system can use large, small or no buffers depending on the rate at which statements arrive from the browser sessions.
  - _Signing of statements_. Aside from managing the data necessary to sign statements, the server-side component can include any logic to determine which statements need to be signed.
  - _Generation of other attachments_. The server-side component can include any logic to determine if an statement should include some specific attachment, for instance a badge, and obtain/generate it, and add it to the stamenet.
  - _Flexibility in destination LRSs_. A server-side component can determine at runtime which LRSs to send the statement to, or even if it should be sent to more than one.
  - _Choice of programming language_. On the browser side, you have no choice. On the server-side, to do more complex tasks like the ones mentioned above, you can use the languages of your choice.
 
