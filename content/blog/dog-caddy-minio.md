---
title: "Putting pictures of my dog on the internet "
description: "How to serve static files to the internet with Cloudflare, Caddy & MinIO"
summary: "A cool picture of my dog, plus how to overcomplicate hosting static sites"
date: 2023-06-23
---

Yesterday I found out that the `.dog` TLD was a thing ([ICANN Wiki](https://icannwiki.org/.dog)). Thinking this was pretty funny, I went straight to namecheap and bought [joeyh.dog](https://joeyh.dog).

Currently, the site is just one picture of my dog, Dexter.

![Dexter the beagle, smiling](https://joeyh.dog "He’s on holiday in Wales at the moment and having a great time at the beach")

I think that this is an excellent use of a domain name. I mean, just look at him! But, as lovely as this picture is, the interesting bit is how it got from my server to your device.

## Overcomplicating serving a single static file

Since I finished my exams about 2 weeks ago, I have gone Full Homelab and set up a Raspberry Pi 4 for hosting ✨ stuff ✨. Among other things, I’m running:

- [MinIO](https://min.io/) - an S3-compatible self-hosted object storage server
- [Caddy](https://caddyserver.com/) - a cool web server that is a lot easier to configure than Nginx
- [cloudflared](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps) - for exposing web services to the internet.

The dog image(s) (more coming soon:tm:) are in a MinIO bucket. A request to [joeyh.dog](https://joeyh.dog) goes to Cloudflare's network, which sends your request down a tunnel to `cloudflared` running on the Pi. `cloudflared` proxies the traffic to Caddy, which does some URL rewriting and then proxies the request to MinIO also running on the Pi, where the image resides.

Why would I want to overcomplicate something as simple as this?

- I wanted to play with S3 because it’s not something I’ve used before and seems like a good skill to have
  - Hosting it myself is cooler and cheaper then paying Amazon for it
- I then need some way to serve the files from it because MinIO does not support [serving a static site like Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteHosting.html) - enter Caddy
- I then need some way for web traffic to reach Caddy - enter Cloudflare
  - I don't want to fuck around with NAT/port forwarding on my shitty Virgin Media router, nor expose my public IP if I can avoid it.
  - I'm not happy about my over-reliance on Cloudflare for all this and will probably switch to a self-hosted tunnelling proxy like [frp](https://github.com/fatedier/frp) or [rathole](https://github.com/rapiz1/rathole) in future.
  - I already use Cloudflare for DNS so this was the easiest solution

I could have just put it on Cloudflare pages or some other free hosting service or something but that’s boring. Caddy also has a [file server directive](https://caddyserver.com/docs/caddyfile/directives/file_server) that's _objectively_ a better solution, but that's also boring.

## Overcomplicating serving an entire static site

The same setup is used for this site, and for my notes site. The only reason I overcomplicated the whole dog thing is because this setup already existed for those and all the stuff was already running so it was easy. They were until recently on GitHub Pages, but hosting stuff yourself is Cool and Fun (despite the sometimes extended downtimes).

You can access the raw HTML file for this page in MinIO [here](https://s3.joeyh.dev/blog/blog/dog-caddy-minio/index.html), but that’s not much use. Caddy is set up to rewrite the paths for incoming URLs to:

- Include the bucket name
- Serve index files (joeyh.dev/blog/ returns joeyh.dev/blog/index.html)
- Return the site’s custom 404 page on error

The Caddyfile for this looks something like:

```
# the site on port 8080
:8080 {

	# send it to the right bucket
	rewrite * /blog{path}

	# rewrite requests with trailing slashes to redirect to index.html
	uri path_regexp ^(.*)/$ $1/index.html

	# reverse proxy to bucket
	reverse_proxy http://minio:9000 {

		# if 404 then use 404 page from bucket
		@error status 404
		handle_response @error {
			rewrite * http://minio:9000/blog/404.html
			reverse_proxy http://minio:9000
		}
	}
}
```

`minio:9000` is MinIO running in Docker on the Pi. Both Caddy and MiniIO are running in containers, so Docker’s [funky container-name DNS stuff](https://docs.docker.com/network/drivers/bridge/#differences-between-user-defined-bridges-and-the-default-bridge) can be used to refer to them. `cloudflared` also runs in a container, and is configured to send all traffic from joeyh.dev to `caddy:8080`.

Easy, but not quite simple, and definitely not necessary. But, hosting my own stuff makes me an [Internet LandChad](https://landchad.net/), and the less traffic goes to Google/Amazon/Microsoft[^1], the better.

[^1]: Yes, Cloudflare isn't much better, either ethically or just purely as an internet monopoly. See above for my alternative plans.
