+++
title = "Building a static serverless website using S3 and CloudFront"
author = "Sander Knape"
date = 2020-02-12T19:25:02+02:00
draft = false
tags = ["serverless", "s3", "cloudfront", "aws"]
categories = []
+++

Hosting static websites is great. As they only contain static assets to be downloaded by the visitor's browser - think HTML, CSS, Javascript, Fonts, images - no server-side code such as Java or PHP needs to be run. They're therefore typically faster to load than dynamic websites, they have a smaller attack surface, and are easier to cache for even better performance.

That is why some time ago I moved this blog from a Wordpress installation hosted on EC2 to a static website. As I was already in AWS, and I knew that S3 + CloudFront was a popular choice for hosting static websites, I decided to host my blog in S3 with CloudFront in front of it as the CDN.

I was however a little disappointed when I started configuring these services. The obvious methods for using S3 and CloudFront had some severe limitations and it took me longer than I liked to find proper solutions for these limitations. It's not very clear from the AWS documentation how to properly host a static, serverless website using S3 and CloudFront.

Therefore, in this blog I'll first explain the (unexpected) challenge and then provide two different solutions to this challenge. Let's dive in!

## The (unexpected) challenge

At first glance it may look easy to host a static website from an S3 bucket. S3 supports [Website Endpoints](https://docs.aws.amazon.com/AmazonS3/latest/dev/WebsiteEndpoints.html), which sounds like exactly what you want to do. However, the biggest downside is that HTTPS is not supported. It's not new that browsers are more and more [demotivating the use of HTTP over HTTPS](https://nakedsecurity.sophos.com/2020/02/10/google-chrome-to-start-blocking-downloads-served-via-http/), and you will get a penalty in your Google ranking when not using HTTPS. The use cases for this feature are therefore rather limited.

The common advice then is to use Amazon CloudFront as a CDN in front of S3. CloudFront supports HTTPS - including for custom domains - and has built-in support for S3 using an [Origin Access Identity](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html) (OAI). The OAI is used to authenticate CloudFront to S3 and ensure that the content is only available through the CloudFront endpoint. However, CloudFront uses the *REST* endpoint when connecting to S3 through the OAI instead of the *website* endpoint. This is severely limiting as the REST endpoint does not support redirecting requests to an index object. For example, when visiting `example.com/about/`, you will typically want to serve the file `example.com/about/index.html` to the visitor. This is supported with the website endpoint, but not with the REST endpoint that CloudFront uses when connecting it through the OAI. More differences between these two endpoints are [documented by AWS](https://docs.aws.amazon.com/AmazonS3/latest/dev/WebsiteEndpoints.html#WebsiteRestEndpointDiff).

Do note that CloudFront does support an index object at the root (so only at `example.com`) through the [Default Root Object](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DefaultRootObject.html), but this will not work for subpages. If you are building a Single Page Application (SPA) that will only serve content from the root, this method may actually work for you.

Concluding: the two obvious choices for hosting a static website in S3 (website endpoint + CloudFront through the OAI) both have severe limitations. So let's look at two solutions that will allow you to properly host a website in S3. The first solution is quite simple but arguably "hacky". The second solution is definitely cleaner, but also more complex.

## The simple solution

The problem with using the OAI to connect CloudFront to the S3 *REST* endpoint is that we lose the features offered by the *website* endpoint. Therefore, it would be easiest to connect CloudFront to the website endpoint. This is possible using a [custom origin](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistS3AndCustomOrigins.html#concept_CustomOrigin) in CloudFront. The custom origin allows us to specify a domain name to which CloudFront forwards requests. Thus, we can set up the custom origin to forward requests to the S3 website endpoint.

However, the S3 website endpoint is publicly available. Anyone who knows this endpoint can therefore also request your content while bypassing CloudFront. If both URLs are crawled by Google, you risk getting a penalty for [duplicate content](https://support.google.com/webmasters/answer/66359). But even worse: the S3 website endpoint URL will bypass any (security) settings you set in CloudFront. For example, using a Web Application Firewall (WAF) attached to your CloudFront Distribution you may whitelist only specific IP addresses that can access your content. Or you reduce data transfer costs by caching the contents on CloudFront's edge servers while improving the latency for your visitors as well.

Good thing then that there is a solution, also [documented by AWS](https://aws.amazon.com/premiumsupport/knowledge-center/cloudfront-serve-static-website/) (see the third option on that page). It can definitely be considered a bit hacky, but it does the job. And it's easy.

So how does it work? We configure CloudFront to forward an additional `Referer` header to the origin endpoint (our S3 website endpoint) as it connects to it. This header is completely invisible to the visitor accessing the website. Let's say we set a value of `MyS3cret!` to this header in the `Origin` configuration in the CloudFront distribution. This would look something like this in the CloudFront user interface:

![CloudFront Referer Header](/images/cloudfront_secret.png)

Do keep in mind that everyone in your organization with access to this CloudFront distribution can view this secret. Thus, if you don't want anyone else to see this, use the proper IAM permissisions to block people from viewing this.

Next we configure the S3 Bucket Policy to only accept requests that contain this header. The following bucket policy will achieve this:

```json
{
  "Version":"2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::example.com/*",
    "Condition": {
      "StringLike": {
        "aws:Referer": ["MyS3cret!"]
      }
    }
  }]
}
```

Again: keep in mind that anyone with access to this bucket can view the bucket policy and therefore the secret value. Ensure you properly block these permissions to anyone else with access to your AWS accounts.

While you may want to try to use a different header key such as `x-origin-secret`, unfortunately the S3 bucket policies don't support using any other headers. Therefore, we are restricted to using specifically this header.

## The purist solution

If the previous solution is too "hacky" for your taste, there is a different, cleaner way that however involves an additional component to set up.

The redirect logic provided by the S3 *website* endpoint can be moved to a [Lambda@Edge function](https://aws.amazon.com/lambda/edge/). This is a piece of code that can run in the request/response cycle from your visitor to CloudFront, to S3, and back. This function can modify the request and/or response, such as modifying a request to `example.com/about/` to actually request the object `example.com/about/index.html` in S3. The function only changes the request that CloudFront makes tot S3. This is therefore not noticed by the visitor, and only a minimal latency in the range of milliseconds is added to the request.

The good thing is that you don't have to write this function yourself. I wrote a function for this which is [available in the Serverless Application Repository](https://serverlessrepo.aws.amazon.com/applications/arn:aws:serverlessrepo:us-east-1:646719517841:applications~cloudfront-s3-origin-website-redirects) (SAR). It mimics the behaviour provided by the S3 website endpoint. For example, when request a URL like `example.com/about` it will return the file at `example.com/about/index.html`. When directly requesting the URL at `example.com/about/index.html`, it will however return a 301 redirect to `example.com/about` to avoid duplicate content. The function has some additional behaviour which is documented in the SAR.

We can now configure CloudFront to [access S3 using the Origin Access Identity (OAI)](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html), ensuring that the bucket can not be accessed directly but must be accessed through CloudFront. Next, deploy the Lambda function to your account through the SAR and configure your distribution's `Origin Request` event to use this Lambda function. Please check the documentation in the SAR to see how to configure this.

## Conclusion

As I migrated this blog to a serverless S3 + CloudFront setup I was suprised that the obvious solutions had some severe limitations. It was also not very clear in the documentation what alternatives there are; it took me a while to find the AWS [knowledge center](https://aws.amazon.com/premiumsupport/knowledge-center/cloudfront-serve-static-website/) article that describes the simple `Referer` header method. I hope that with this blog post it is now easier to find a proper solution for hosting a static, serverless website using S3 + CloudFront.
