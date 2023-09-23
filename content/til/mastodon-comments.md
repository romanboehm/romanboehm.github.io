---
title: "Use Mastodon as a Comment System"
date: 2023-09-23
tags:
  - hugo
  - blogmeta
  - mastodon
  - html
  - javascript
tootId: "111115895910218624"
draft: false
---

## Problem

I wanted to have a comment system for the posts in this blog. Not because I think there's demand for it, but because I can. 😎

## Solution

The first thing which came to my mind was abusing Mastodon's _toot_ system for my comment system. And of course, I (luckily) wasn't the first person to have that idea. Daniel Pecos provided both a [detailed instructional post](https://danielpecos.com/2022/12/25/mastodon-as-comment-system-for-your-static-blog/) as well as [webcomponent](https://github.com/dpecos/mastodon-comments) to embed Mastodon discussion on one's site.

Going from there was quite easy, fortunately:

### Providing the Supporting Scripts

In order to pull in the JavaScript supporting Daniel's webcomponent, I adjusted my site's `<head>` _partial_:

In layouts/partials/head.html:

```html
...
{{ with .Params.tootId }}
<script src="https://cdnjs.cloudflare.com/ajax/libs/dompurify/3.0.5/purify.min.js" integrity="sha512-uHOKtSfJWScGmyyFr2O2+efpDx2nhwHU2v7MVeptzZoiC7bdF6Ny/CmZhN2AwIK1oCFiVQQ5DA/L9FSzyPNu6Q==" crossorigin="anonymous" referrerpolicy="no-referrer"></script>
<script src="/js/mastodon-comments.js"></script>
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/4.7.0/css/font-awesome.min.css">
{{ end }}
...
```

mastodon-comments.js is Daniel's [script](https://github.com/dpecos/mastodon-comments/blob/master/mastodon-comments.js), which resides in my static/js folder, i.e. it's not subject to any mangling by the blog engine and will be served just like that, as an asset.
I modified it ever so slighty in that I moved the following code from the webcomponent's constructor into the `connectedCallback` function, else it wouldn't work:

```javascript
this.host = this.getAttribute("host");
this.user = this.getAttribute("user");
this.tootId = this.getAttribute("tootId");
```

Mind you, I have no idea of webcomponents, this was just [the first thing I found when searching the web](https://stackoverflow.com/a/70201467), and it worked.

Additionally, I made all scripts required for the comment system conditional so that they will only be loaded if the page you're on has a `tootId` param in the [_front matter_](https://gohugo.io/content-management/front-matter/).

### Embedding the Component

layouts/posts/single.html and layouts/til/single.html both received this addition to make the comment section appear between content and footer:

```html
...
{{ .Content }}

{{ with .Params.tootId }}
<mastodon-comments host="{{ site.Params.mastodonHost }}" user="{{ site.Params.mastodonUser }}" tootId="{{ . }}"></mastodon-comments>
{{ end }}

{{ end }}
```

Again, a conditional include since I only want to provide a comment section where I've prepared one (see explanation below). In that case, I'll add a `tootId` param to the front matter like mentioned above.

### Providing the Configuration

In config.yaml, I then added the values for `mastodonHost` and `mastodonUser`:

```yaml
params:
  ...
  mastodonHost: "mastodon.social"
  mastodonUser: "romanboehm"
```

## How It Works

In case the post in question hasn't been published yet:

1. Prepare the post.
2. Just before publishing, create the accompanying Mastodon toot, and copy its id from the URL.
3. Provide `tootId: "<designated-comment-toot's-id>` in the post's front matter.
4. Publish the post. (In my case commit and push.)

If I want to enable comments for an existing post:

1. Create the accompanying Mastodon toot and copy its id from the URL.
2. Provide `tootId: "<designated-comment-toot's-id>` in the post's front matter.
3. Re-publish the post.
