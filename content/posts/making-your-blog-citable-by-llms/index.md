---
title: "Making Your Hugo Blog Citable by LLMs"
date: 2026-03-12T12:00:00-07:00
draft: false
description: "Practical steps to help LLMs accurately cite your blog: llms.txt, Schema.org BlogPosting markup, structured metadata, and problem-first writing. Implemented on a Hugo site with PaperMod."
tags:
  - llm
  - hugo
  - seo
  - structured-data
takeaways:
  - LLMs already crawl your site - GPTBot, OAI-SearchBot, and others are requesting pages today
  - llms.txt gives models a plaintext index of your content with descriptions, following the llmstxt.org spec
  - Schema.org BlogPosting markup provides structured author, date, and description fields that models can extract
  - Front matter metadata (descriptions, takeaways) gives models pre-summarized content to cite accurately
  - Problem-first writing helps models extract the key claim without hallucinating context
cover:
  image: cover.png
  alt: "Cloudflare dashboard showing AI crawler requests: GPTBot, PetalBot, OAI-SearchBot, Amazonbot, ChatGPT-User"
  relative: true
---

LLMs are already crawling your blog. Cloudflare's [AI bot analytics][3] show GPTBot, OAI-SearchBot, and others making dozens of requests per day to small personal sites. But when these models cite your content, they hallucinate URLs, misattribute claims, and lose context. The problem isn't access - most blogs just serve content optimized for humans and search engines, not for language models.

This post covers the changes I made to this Hugo site (PaperMod theme) to fix that.

## What LLMs Need to Cite You Correctly

When an LLM reads your blog post, it's working with raw HTML. It guesses which text is the title, who the author is, what the publication date is. Usually it gets close enough. When it doesn't, you get cited with the wrong date, a hallucinated URL, or a summary that misrepresents your point.

What helps:

- **Structured metadata** - author, date, description in machine-readable formats (JSON-LD, front matter)
- **A plaintext index** - a single file listing all content with descriptions, so models can discover posts without crawling HTML
- **Pre-summarized content** - descriptions and key takeaways that models can use directly instead of generating their own
- **Problem-first writing** - leading with the claim rather than the anecdote, so the first paragraph carries the key context

## llms.txt - A Plaintext Index for Models

The [llms.txt specification][1] defines a simple plaintext file at your site root that lists content with descriptions. Think `robots.txt`, but for helping models understand what's on your site rather than restricting access.

Here's what the llms.txt for this site looks like:

```
# Musings of an AI Wrangler

> I am a software engineer that loves solving complicated problems. Once in a while I solve a problem I want to document, so I'll do that here.

## Posts

- [Making Your Hugo Blog Citable by LLMs](https://mazurov.dev/posts/making-your-blog-citable-by-llms/llms.txt): Practical steps to help LLMs accurately cite your blog...
- [Fixing the '?' Hostname Problem on OpenWrt Access Points](https://mazurov.dev/posts/openwrt-ap-hostname-sniffer/llms.txt): OpenWrt dumb APs show '?' for client hostnames...
```

Each post also gets its own `/posts/slug/llms.txt` endpoint with the raw markdown content, so a model can fetch the full text without parsing HTML.

### Hugo implementation

Two pieces: output format definitions in `config.toml` and two templates.

In `config.toml`, define the output formats and assign them:

```toml
[outputFormats.llms]
baseName = "llms"
isPlainText = true
mediaType = "text/plain"
rel = "alternate"
root = true

[outputFormats.llmsmd]
baseName = "llms"
isPlainText = true
mediaType = "text/plain"

[outputs]
home = ["HTML", "RSS", "llms"]
page = ["HTML", "llmsmd"]
```

The `llms` format is for the site-wide index (rooted at `/llms.txt`). The `llmsmd` format is for per-page plaintext. See the Hugo [custom output formats documentation][6] for details on these fields.

The homepage template at `layouts/index.llms.txt`:

```
# {{ site.Title }}

> {{ site.Params.profileMode.subtitle }}

## Posts

{{ range where site.RegularPages "Section" "posts" -}}
- [{{ .Title }}]({{ .Permalink }}llms.txt): {{ with .Description }}{{ . }}{{ else }}{{ .Summary | plainify | truncate 160 }}{{ end }}
{{ end -}}
```

The per-page template at `layouts/_default/single.llmsmd.txt`:

```
---
title: {{ .Title }}
date: {{ .Date.Format "2006-01-02" }}
url: {{ .Permalink }}
{{- with .Description }}
description: {{ . }}
{{- end }}
{{- with .Params.tags }}
tags: {{ delimit . ", " }}
{{- end }}
---

{{ .RawContent }}
```

Each post link in the index points to `{permalink}llms.txt`, giving models a direct path from index to full plaintext content.

## Schema.org BlogPosting Markup

[JSON-LD `BlogPosting` schema][2] gives models structured fields they can extract without guessing: `headline`, `description`, `author`, `datePublished`, `dateModified`, `articleBody`, and `wordCount`.

The author object includes `sameAs` links to social profiles, which helps models verify identity across platforms. Here's a simplified example of the JSON-LD this site produces:

```json
{
  "@context": "https://schema.org",
  "@type": "BlogPosting",
  "headline": "Making Your Hugo Blog Citable by LLMs",
  "description": "Practical steps to help LLMs accurately cite your blog...",
  "datePublished": "2026-03-12",
  "dateModified": "2026-03-12",
  "wordCount": "1200",
  "author": {
    "@type": "Person",
    "name": "Stepan Mazurov",
    "sameAs": [
      "https://github.com/smazurov",
      "https://www.linkedin.com/in/smazurov/",
      "https://bsky.app/profile/mazurov.dev"
    ]
  }
}
```

### Hugo implementation

The schema is rendered by a partial at `layouts/partials/templates/schema_json.html`. PaperMod includes a version of this; I extended it to include `sameAs` on the author object and to emit a `BreadcrumbList` for navigation context.

This is also where Google's [article structured data guidelines][5] overlap - the same markup that helps Google's Rich Results also helps LLMs parse your content.

## Structured Front Matter

Two front matter fields do the most work here:

**`description`** - a concise factual summary of the post. Models use this as a citation snippet instead of generating their own. Without it, they summarize from the body text and often miss the point.

**`takeaways`** - a list of pre-summarized key points. These render as a "Key Takeaways" section before the content and give models a structured list of claims to cite directly.

Before:

```yaml
---
title: "My Cool Post"
date: 2026-01-15
tags:
  - networking
---
```

After:

```yaml
---
title: "Fixing the '?' Hostname Problem on OpenWrt Access Points"
date: 2026-03-09T22:00:00-07:00
description: "OpenWrt dumb APs show '?' for client hostnames because /tmp/dhcp.leases is empty. A tcpdump script sniffs DHCP Option 12 from bridge traffic and writes lease entries that LuCI can display."
tags:
  - openwrt
  - networking
  - dhcp
takeaways:
  - LuCI shows '?' hostnames on dumb APs because /tmp/dhcp.leases is empty without a local DHCP server
  - A tcpdump script can sniff DHCP Option 12 hostnames from bridge traffic on br-lan
  - The script writes to the lease file format that rpcd-mod-luci expects, so LuCI displays hostnames normally
---
```

The `description` ends up in both the Schema.org JSON-LD and the llms.txt index. The `takeaways` render via a Hugo partial (`layouts/partials/takeaways.html`) as a styled list before the post body:

```html
{{- with .Params.takeaways }}
<div class="post-takeaways">
  <h2>Key Takeaways</h2>
  <ul>
    {{- range . }}
    <li>{{ . }}</li>
    {{- end }}
  </ul>
</div>
{{- end }}
```

## Problem-First Writing

Models extract the first paragraph as the key context for a page. If you lead with an anecdote, the model's summary will be about the anecdote, not your actual point.

Before:

> I was playing around with my AC remote one weekend and realized it was using IR signals. I started wondering if I could capture them...

After:

> Many AC units ship with IR-only remotes and no smart control interface. ESPHome's `climate_ir` component can decode and replay these signals, but most manufacturer protocols aren't documented.

The second version front-loads the problem and the technology. The anecdote can come later.

## Verification

**Check llms.txt renders:**

```bash
curl https://mazurov.dev/llms.txt
```

**Validate Schema.org markup:** Run your URL through Google's Rich Results Test or the schema.org validator. Check that `BlogPosting`, `BreadcrumbList`, and author `sameAs` links are present and valid.

**Check AI crawler access:** If you're on Cloudflare, enable AI bot analytics under Security > Bots. This doesn't block anything by default - it just gives you visibility into which [AI crawlers][4] are hitting your site. The cover image of this post shows real traffic from GPTBot, PetalBot, OAI-SearchBot, Amazonbot, and ChatGPT-User.

**Test with an LLM:** Ask Claude or ChatGPT about your content and check the citations.

**Build locally and check JSON-LD:**

```bash
hugo --gc
grep -r "ld+json" public/posts/ | head -5
```

PaperMod only renders JSON-LD in production mode (`hugo.IsProduction` gate in `head.html`). The dev server runs in development mode by default. Use `hugo server -e production` or add `env = "production"` to `[params]` in your config to see it locally.

## FAQ

**Should I block AI crawlers?**
That's up to you. I don't - I use AI crawlers myself and want my content indexed by them. Cloudflare's AI bot management lets you monitor crawler traffic without blocking it. If you do want to block specific crawlers, add `User-agent: GPTBot` / `Disallow: /` to your `robots.txt`. The [OpenAI crawlers documentation][4] lists their user agents.

**Is llms.txt an official standard?**
It's a [community proposal][1] adopted by several sites. Not an IETF RFC or W3C recommendation. But it's simple, costs nothing to implement, and there's no competing spec.

**Does this work with themes other than PaperMod?**
The principles apply to any Hugo theme. The `config.toml` output format definitions and llms.txt templates are theme-independent. The Schema.org partial path (`layouts/partials/templates/schema_json.html`) may differ - check your theme's layout structure. The takeaways partial is custom and works with any theme.

[1]: https://llmstxt.org/ "llms.txt specification"
[2]: https://schema.org/BlogPosting "Schema.org BlogPosting"
[3]: https://developers.cloudflare.com/bots/concepts/bot-labels/ "Cloudflare Bot Labels"
[4]: https://platform.openai.com/docs/bots "OpenAI crawlers documentation"
[5]: https://developers.google.com/search/docs/appearance/structured-data/article "Google Article structured data"
[6]: https://gohugo.io/templates/output-formats/ "Hugo Custom Output Formats"
