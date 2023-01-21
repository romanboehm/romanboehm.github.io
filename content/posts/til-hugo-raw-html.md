+++
title = "Use Raw HTML in Hugo's Markdown"
date = "2023-01-02"
tags = [
    "TIL",
    "hugo",
    "blogmeta",
    "markdown",
    "html"
]
toc = true
draft = false
+++

## Problem

I want the link to this blog in my Mastodon profile to be ["verified" (green) link](https://snyk.io/blog/verify-and-secure-your-mastodon-account/). This is done by adding a custom anchor element somewhere on the site: 

```html
<a rel='me' href='https://mastodon.social/@romanboehm'>Mastodon</a>
```

By default, however, Hugo, the static site generator of my choice, does not permit this.


## Solution

If you don't like to activate the [renderer's unsafe mode](https://gohugo.io/getting-started/configuration-markup/) in order to be able to insert raw html, you can achieve this with so-called [_shortcodes_](https://gohugo.io/content-management/shortcodes/):

1. Create an HTML file under `layouts/shortcodes`. I called it rawhtml.html.
2. Put just the text `{{.Inner}}` as the file's content.
3. Use it like `{{</* rawhtml >}}<b>foo</b>{{< /rawhtml */>}}`.

Or, for the case of adding the "verifiable" link:

```markdown
{{</* rawhtml >}}<a rel='me' href='https://mastodon.social/@romanboehm'>Mastodon</a>{{< /rawhtml */>}}
```

(Note the single quotes for the attribute values)

You can view the whole commit on [GitHub](https://github.com/romanboehm/romanboehm.github.io/commit/a045300be06e68283c58ec55b4704ccd289d3f53).