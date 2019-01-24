date: 2019-01-24

# Migrating to AWS

This blog [came to life some time ago](000-Hello-world) hosted on cheap server
by PHP engine. My needs are modest and hosting cost was less than $1 per year.
But after a year has passed, hosting company wanted ~$120 for the next year,
so I decided to move.

(I know I could use some blogging service, but I like to have full control,
so again I built something from scratch. I decided to use AWS to explore
some of their services a bit.)

## S3

I guess S3 stands for Scalable Storage Service. First thing I did was `wget`
and upload to S3. In a few minutes I had my site hosted by AWS by just
following short [howto article](https://docs.aws.amazon.com/AmazonS3/latest/dev/WebsiteHosting.html).

Basically all you need to do is to point to index page and error page.

![S3 static hosting](008-s3.png)

## Route 53

But URL like `http://tomasz-cichocki.pl.s3-website-eu-west-1.amazonaws.com`
is not very nice, so next step is to set up DNS. Route 53 is DNS service
from AWS (and port 53 is...).

AWS won't allow to transfer .pl domain to them, but you can setup hosted zone
in Route 53 anyway and delegate your domain to AWS DNS servers.

![DNS setup](08-dns.png)

When you have your hosted zone, it's easy to create alias records that point
to various other AWS services (S3, CloudFront, API Gateway, ...).
For details see [here](https://docs.aws.amazon.com/AmazonS3/latest/dev/website-hosting-custom-domain-walkthrough.html).

## CloudFront

One problem with S3 static hosting is that it won't handle HTTPS.
There is nothing really secret on my blog site, but I don't want
[Google Chrome to complain that my website is not secure](https://security.googleblog.com/2018/02/a-secure-web-is-here-to-stay.html).

AWS solution for serious content delivery is CloudFront. It can cache and
serve content from S3, ELB (and anything behind it) or media store (for video files).

### Certificate Manager

But to enable HTTPS you first need an SSL certificate.
Public SSL certificates from AWS [are for free](https://aws.amazon.com/certificate-manager/pricing/).
For CloudFront you need a certificate in US East (N. Virginia) region. Yes,
AWS certificates are not global. They need to be created in the region where
they will be used.

If your domain is hosted by Route 53, easiest way to validate a certificate
(prove that you are the owner of a domain) [is by DNS](https://docs.aws.amazon.com/acm/latest/userguide/gs-acm-validate-dns.html).
In this case AWS will automatically create some DNS records for validation purpose.
When certificate is issued you can remove those records.

### Back to CloudFront

Once you have a SSL certificate it's pretty straight forward to create distribution
for S3 bucket. As in S3 static hosting there are options to setup
"default root object" (index.html) and error pages (there can be separate
error page for each HTTP error code).

One catch is that CloudFront needs to have ListBucket permission for S3 bucket
it is service, otherwise it won't handle "404 Not Found" correctly.

Another catch is that once resource is cached in CloudFront it may take 24 hours
to refresh it. To force refresh you can "Create Invalidation" request
(first 1000 requests in a month are for free).

![CloudFront Invalidation](008-invalidate.png)
