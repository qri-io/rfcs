# HTTP Capture

Part of qri's stated design goals is the ability to make changes to a dataset in a manner that is predictable, auditable, reproducable, and deterministic. Due to dealing with the outside world, there's only so far we can go to achieve this, but we should exercise care to get as close as possible. One particular source of external non-determinism, fetching data over http, provides many opportunities to control these outside influences and get closer to our target.


Although we cannot truly eliminate the unpredictable effects of http fetching, we can limit the consequences of using it. We can do this by capturing http data, and then signing it, verifying it, and replaying it.

## Capture

When a script performs http fetching, if that request is executed without additional support, then important information gets lost. That is, the http request and response don't live longer than the script's execution. Rather, the script requests some url, uses what it needs from the response to generate or modify its dataset, and throws away everything else. Anyone in the future trying to audit that transformation has no way of validating that the code actually did what it claims to have done. All that is observable is the previous dataset version, the script's contents, and the next dataset version that was created. Information about the http request and response is gone.

This is problematic for multiple reasons. Http responses are by nature ephemeral. It's possible that responses can change at any time. It's even possible in the worst case that a dishonest actor may setup their own end-point that sends a response with whatever data they want to receive. Imagine a nefarious actor trying to fake some important result by claiming some http server gave them some data at a certain point in time; it's difficult to disprove this when the http response cannot be inspected after the fact.

The way that qri can solve this is by capturing http responses and saving it locally. The saved version shall contain the text of the response, along with relevant http headers, the timestamp of the request / response, and a cryptographic signature created by the agent that made the request, to ensure authenticity.

## Benefits for development

When developing scraping tools, it is common to request http data from a server, then write code iteratively that processes this data in some manner. It can be very frustrating to rerun a script that has been recently modified and see different results, unsure whether its due to code changes or the http server returning something different. Http responses change often, perhaps because the backend in question has new data, or because the responses contain dynamic content, or because the server is treating the client differently due to making too many requests. Programming against data that is not stable is like trying to paint on a canvas that is moving around.

Once the http response has been captured, it can be replayed automatically. Clients do not need to manually save response text and operate on it on their own. Instead the script environment can transparently return this same result, allowing the programmer to iterate their code as much as needed until development is complete. Once the code is stabilized, replaying can stop, so that future http requests are allowed to be sent as normal to the external endpoint in order to get new data.

## Performance gains

While a script is being developed, but also afterwards while it is being shared and used by others, it very quickly becomes inefficient to rerun the same http operations over and over again. Especially in the case of examples within documentation, certain scripts are going to be executed by huge numbers of users, potentially hammering some remote server. Capturing requests using a centralized service can help to completely eliminate these inefficiencies. In the case of Qri Cloud, we can be even more aggressive, and prefetch urls as soon as they are typed, before a transform is explicitly run. This could make for a very plesant user experience, making http appear to fetch instantaneously.

Of course many scripts will be written over time, and they are going to use unique urls, which do not want to be replayed, as their developers will want live data all of the time. But even in these cases, the captured response allows qri to take advantage of http caching mechanisms like etags and the if-modified-since header. This will make the qri transform engine behave more like a web browser, efficiently working to manage its traffic intake.

## Testing is easier

One final benefit is that it's much easier to test scripts that use HTTP calls. Having tests that require network access is a bad idea: both because it adds an uncontrollable external dependency, and adds new failures modes. Once capture is happening, it becomes nearly trivial to also mock http responses from test code.

## UX considerations

It must be made obvious to users that this http capture is occuring. It would be very frustrating if they knew an http server contained new data, but their scraper code was never able to see it. To this end, the UX needs to be clear in communicating that http capture is happening.

On the command-line, scripts should display informational messages that say what urls are currently captured and being replayed. Users can then use flags to disable capture for the next run, but due to reasons mentioned above, disabling capture should opt-in, not the default.

On a graphical or web UI, some part of the page should display which urls are being used as sources of data. An icon with hover text and help links would declare these endpoints to be captured, and a user should be able to temporarily or permanently disable capture in order to ensure they fetch new data.

## Trust improvements

In the case where no http capture is happening, auditing a script requires a lot of trust: that the http responses were correct, and that no emphemeral instability existed at the time of the response, and that the user that ran the script is not trying to trick them. When http capture exists, the only requirement is trusting the user who owns the private key that signed the capture. This doesn't completely solve the problem of nefarious actors, but there is one additional step that could: when a script makes an http request, that request is proxied through cloud which uses its own private key to generate the signature.

By sending http requests through cloud, the trust problem is simplified down to only trusting cloud's private key. It becomes possible to validate any http request by ensuring the signature on a capture is correct.

Of course, there are cases were a user wouldn't want to send their http traffic through cloud. Perhaps a user doesn't trust cloud or has privacy concerns. Perhaps they are sending secret keys or config information in their request. Perhaps they are operating with private data or talking to internal data sources. For all these reasons and more, it should be easy to turn off this behavior, and the messaging around this feature should make it obvious what is happening. But in cases where users are working with open public data, it's likely they would be okay with cloud capturing and signing these requests.

Another nice side effect of having http requests go through cloud, and get signed, is that it can be used to prove what kind of data a server used to send at some point in the past. Let's say server is hosting some data, but then later takes it down. If any client performed an http request before that happens, they have a verifiable claim, not just from themselves, but from cloud, that they saw the old data. This is an important feature for data rescue situations, where some third-party is concerned about open data dissappearing from the web.

## Retention

Saving all http traffic can become very expensive over time. There are a number of ways to ameliorate this problem. First, once a captured http request is signed by cloud, there is no need to keep it forever on cloud. Users could download the packets themselves, letting them replay and audit their results, while still demonstrating that it was cloud that signed them.

If a user wants cloud to retain and store captured http long term, this would be a strong incentive for creating a paid account. In the cases where users aren't able or don't want to do this, we can create a quota limit for their http capture, and encourage them to self-host as their quota runs out.

Finally, an advanced feature would be using static or runtime analysis of starlark scripts to determine what parts of the http response are actually being used. As a user's quota runs low, we could throw away all http data except that which is used by the script, and sign this structure. Obviously, this loses much of the benefit of saving the entire response, so it should only be used if absolutely needed.