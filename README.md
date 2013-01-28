# Script Content Integrity Proposal

This is a proposal for a `<script>` attribute that helps a HTML page's author
make sure that the script downloaded and executed by a user agent is the script
that the author intended to have executed in that page.

This proposal was submitted to the W3C's HTML WG as
[HTML bug 20789](https://www.w3.org/Bugs/Public/show_bug.cgi?id=20789). This
document will be updated to reflect the feedback received on the bug.


## Example

The following example shows a `<script>` that downloads jQuery from a Content
Delivery Network (CDN) and follows this proposal to ensure that user agents
will only execute the JavaScript returned by the CDN if it exactly matches the
jQuery version that the author expects.

```html
<script type="text/javascript"
    src="https://cdnjs.cloudflare.com/ajax/libs/jquery/1.9.0/jquery.min.js"
    digest="sha256:f6DVw/U4x2+HjgEqw5BZf67Kq/5vudRZuRkljnbF344=">
</script>
```


## Motivation

Many Web sites import popular scripts from CDNs (content distribution networks)
to improve the user experience by increasing cache hit ratios. Unfortunately,
this requires full trusting CDNs, which receive the power to execute arbitrary
JavaScript with the credentials of the sites that use them.

If `<script>` supports content integrity checking, the CDNs can at most perform
a denial of service attack by returning the wrong data. CDNs can perform the
same attack simply be refusing to return any content.

Note that using the https: scheme does not solve the problem mentioned in this
scheme, because it only protects the script content while it is in transit
between the server and the client.


## Proposed Syntax

This proposal introduces an optional `digest` attribute to the `<script>` tag.

The proposed syntax of `digest` values is as follows:

```
digest-value     = algorithm-name ":" crypto-hash
algorithm-name   = 1*( ALPHA / DIGIT / "_" )
crypto-hash      = 1*( ALPHA / DIGIT / "/" / "+" / "=" )
```

The algorithm name is made up of the characters that match the `/\w/`
JavaScript regular expression. User agents must check the algorithm name
against a list of algorithms that they support, and do not need to do any
further processing of the algorithm name.

The crypto hash is made up of the characters that may be used in Base64
encoding, as specified in Section 6.8 of RFC2045 [1].

The `digest` attribute is silently ignored if the script does not have a
'src' attribute, if its value does not match the syntax described above, or
if the user agent does not support the specified algorithm. User agents can
optionally log a warning when ignoring the 'digest' attribute, to help HTML
authors with debugging.

The term "digest" is also used to refer to cryptographic hashing in the HTTP
protocol specification for authentication methods [3], in the context of the
Digest authentication method. (Note that the methods proposed here differ from
RFC 2617.)


## Script Fetching

`<script>` is not subject to the same-origin policy, and the `digest` attribute
can be used to check if a resource's cryptographic hash matches a
pre-determined value. This section makes sure that the `digest` attribute obeys
the spirit of the Same-Origin Policy [4].

If the user agent does not ignore the `digest` attribute of a `<script>`
element (implying that the element has both a `src` attribute and a `digest`
attribute whose value has valid syntax and specifies an algorithm supported by
the user agent), the resource referenced by the `<script>` element shall be
fetched according to the CORS specification [5], with the _omit credentials
flag_ set.

The performance impact of the CORS requirement should be minimal. No CORS
preflight requests should be necessary, since scripts are fetched using the
`GET` method and no custom headers. A CDN server can satisfy the CORS checks by
adding an `Access-Control-Allow-Origin: *` HTTP header.

If the response of the HTTP request used to fetch the script satisfies the
resource sharing check in Section 7.2 of the CORS specification, script
fetching is considered successful. Otherwise, the process in the sub-section
below is followed.

### CORS Fallback

This section provides an alternative method for scripts to opt into content
integrity checking, for the situations where the script author is not able to
set the 'Access-Control-Allow-Origin' HTTP header on the script hosting server.
Providing alternatives to setting HTTP headers is consistent with other W3C
standards, such as the `<meta charset>` [6] work-around for not being able to
set the `Content-Type` header, and the `<meta http-equiv>`-based fallback for
not being able to set the `Content-Security-Policy` header [7].

If the response of the HTTP request used to fetch the script does not satisfy
the CORS resource sharing check, the user agent shall perform a case-sensitive
search for the magic comment `//@ scriptDigest` in the script file. This magic
comment should be at the beginning of the file, or be preceded by an end of
line character. This search can be performed using the JavaScript regular
expression `/(^|\r\n)\/\/\@ scriptDigest/`.

If the magic comment search fails, the user agent will consider that the script
fetching resulted in a network error, and will not proceed to content integrity
checking.


## Content Integrity Checking

The content integrity check is performed after the user agent successfully
retrieves a script according to the steps above and before the script is
executed. The check can be considered to be an extension of the script fetching
process. If the check fails, the user agent will follow the same steps as if
the network request for fetching the script had failed. Most importantly, the
user agent shall not execute the script if the content integrity check fails.

The content integrity check is performed by executing the cryptographic hashing
algorithm identified by the `algorithm-name` field in the `digest` attribute
value and comparing its output to the `crypto-hash` field in the `digest`
attribute value.

The cryptographic algorithms used by this proposal must take binary data as
input and output a small string that only contains the characters used by the
Base64 encoding, as specified in Section 6.8 of RFC2045 [1].

The input to the hashing algorithm identified by `algorithm-name` shall be the
binary representation of the script in the body of the HTTP request used to
fetch the script. This is purposefully dependent on the character encoding in
the HTTP response.

### The sha256 Algorithm

This proposal introduces one algorithm named `sha256` that operates as follows.

1. Let `src` be the binary representation of the input script.
1. Let `sha` be the output of executing the SHA-256 algorithm specified in FIPS
   180-4 [2] with `src` as input.
1. Let `out` be the output of executing the Base-64 encoding algorithm
   specified in Section 6.8 of RFC 2045 [1].
1. Produce `out` as the output.

On a system that has the `curl` [8] and `openssl` [9] command-line tools
installed, the output of the `sha256` algorithm for a script can be computed by
the following command.

```bash
curl -s http://cdn.com/script.js | openssl dgst -sha256 -binary | openssl enc -base64
```


## Fallback Mechanisms

No fallback markup is required for user agents that do not implement this
proposal. User agents that do not understand the 'digest' attribute will
silently ignore it and execute the scripts referenced by `<script>` tags
without content integrity checking.


## Contributors

* Victor Costan `victor at costan.us`
* Xi Wang `xi at mit.edu`
* Nickolai Zeldovich `nickolai at csail.mit.edu`
* Emily Stark `estark at mit.edu`


## References

[1] [Base64 encoding in RFC 2045](https://tools.ietf.org/html/rfc2045#section-6.8)

[2] [SHA-256 in FIPS 180-4](http://csrc.nist.gov/publications/fips/fips180-4/fips-180-4.pdf)

[4] [Same-Origin Policy](http://www.w3.org/Security/wiki/Same_Origin_Policy)

[5] [Cross-Origin Resource Sharing](http://www.w3.org/TR/cors/)

[6] [HTML: The Markup Language](http://www.w3.org/TR/html-markup/meta.charset.html)

[7] [Content Security Policy](https://dvcs.w3.org/hg/content-security-policy/raw-file/tip/csp-specification.dev.html#policy-delivery)

[8] [curl](http://curl.haxx.se/)

[9] [OpenSSL](http://www.openssl.org/)
