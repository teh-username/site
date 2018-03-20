---
title: Migrating to Vultr
tags:
  - Vultr
  - Digital Ocean
  - Terraform
  - VPS
date: 2017-11-25 21:32:42
---


I am no stranger to continuous experimentation. In fact, I _&ast;may&ast;_ have rewritten the backend code for my website a bunch of times already:

1. [ninja-corgi](https://github.com/teh-username/ninja-corgi) [Ruby]
2. [samurai-husky](https://github.com/teh-username/samurai-husky) [Javascript]
3. [ronin-pug](https://github.com/teh-username/ronin-pug) (Current) [Javascript]

There are plans to rewrite _again_ in Python but that's besides the point and not what this post is about.

### Enter Digital Ocean

I've been a long time user of [Digital Ocean](https://www.digitalocean.com). For the past 1.5 years, I've never really had any problems with regards to their service except for one thing: [Their $5 tier offering](https://www.digitalocean.com/pricing/) (yes, I'm cheap).

{% asset_img do_5_tier.png Digital Ocean&apos;s $5 Tier Offering %}

Any developer who has tried running

``` bash
  composer install
```

or

``` bash
  npm install
```

on a 512MB server would've **surely** encountered the dreaded "Out of memory" error. The primary solution to this is to *not* be a cheapskate and pay more to increase the machine's memory to >= 1GB. There are other no-money-involved alternatives such as [adding a swap file](https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-ubuntu-16-04) but isn't recommended if you're using SSDs (which everyone should be using by now).

Now, some of you might be thinking:

{% blockquote You, On Being a Smartass %}
  But LA, your argument is invalid since you can dynamically change the memory of a machine temporarily while installing dependencies and then switch back.
{% endblockquote %}

and you are absolutely right. My counter argument is:

{% blockquote Abraham Lincoln, Some quote he totally said %}
  Ain't nobody got time for that
{% endblockquote %}

### Enter Vultr

I recently stumbled on [Vultr](https://www.vultr.com/) while looking for possible substitutes to Digital Ocean and instantly got hooked on their [tier pricing](https://www.vultr.com/pricing/):

{% asset_img vultr_pricing_tier.png Vultr&apos;s Tier Offering %}

See that $5 tier? That's double the RAM and 5GB more SSD space when compared to Digital Ocean! (Digital Ocean's $5 tier is just Vultr's $2.50 tier).

I know it's unfair to compare VPS providers by just specs alone which is why I'm trialing Vultr as a _potential_ substitute to Digital Ocean first. I won't be moving away _just_ yet.

### Stack Orchestration

Before you ask, yes, I've looked into [Linode](https://www.linode.com/) as well but backed off since I'm planning on [Terraforming](https://www.terraform.io) my stack (finally) and failed to find any active [Linode Provider](https://github.com/search?utf8=%E2%9C%93&q=linode+terraform&type=). There is hope for a stable [Vultr Provider](https://github.com/search?utf8=%E2%9C%93&q=vultr+terraform&type=) though.

### Closing

I'd still recommend Digital Ocean for developers out there. My experience was smooth sailing since Day 1 and I never encountered anything weird with respect to their services. They really stick to their slogan _"Cloud computing, designed for developers"_. If you want to give Digital Ocean a try, you can sign up using [my affiliate link](https://m.do.co/c/c50b388b7afb) and get $10 for free!
