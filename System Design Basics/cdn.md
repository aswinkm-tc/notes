 # Content Delivery Network(CDN)
* A content delivery network (CDN) is a globally distributed network of proxy servers, serving content from locations closer to the user.
* Generally, static files such as HTML/CSS/JS, photos, and videos are served from CDN
* Some CDNs such as Amazon's CloudFront support dynamic content.
* The site's DNS resolution will tell clients which server to contact.

Serving content from CDNs can significantly improve performance in two ways:
* Users receive content from data centers close to them
* Your servers do not have to serve requests that the CDN fulfills

## Push CDNs
* Receives new content whenever changes occur on your ser
* You take responsibility for providing content, uploading directly to CDN and rewriting URLs to point to the CDN
* Content is uploaded only when it is new or changed, minimizing traffic, but maximizing storage.
* Sites with a small amount of traffic or sites with content that isn't often updated work well with push CDNs.
* Content is placed on the CDNs once, instead of being re-pulled at regular intervals.

## Pull CDNs
* Pull CDNs grab new content from your server when the first user requests the content. You leave the content on your server and rewrite URLs to point to the CDN. 
* Slower initial request until the content is cached on the CDN.
* TTL decides how long content is cached
* Pull CDNs minimize storage space on the CDN
* It can create redundant traffic if files expire and are pulled before they have actually changed.

## Disadvantages
* CDN costs could be significant depending on traffic, although this should be weighed with additional costs you would incur not using a CDN.
* Content might be stale if it is updated before the TTL expires it.
* CDNs require changing URLs for static content to point to the CDN.

Feature | Pull CDN | Push CDN |
|---|---|---|
| Content Fetching | Automatically pulls from origin server on request | Manually uploaded to CDN servers by the content owner |
| Cache Management | CDN manages caching automatically | Content owner manages the upload and distribution |
| Ease of Setup | Easy to set up with minimal configuration | Requires more initial setup and maintenance |
| Dynamic Content | Handles dynamic content well | Best for static content |
| Latency | Initial request may have higher latency due to fetching from origin | Lower latency as content is pre-distributed |
| Bandwidth Usage | Bandwidth usage is based on actual user demand | Bandwidth usage can be higher due to pre-distribution |
| Cost Structure | Typically pay-as-you-go based on usage | May have higher costs due to storage and distribution overhead |

