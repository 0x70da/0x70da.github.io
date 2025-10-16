---
title: "0x70da's Blog"
---

# 👋 Welcome

My name is Mahmoud Waleed, and I’m a security researcher and bug bounty hunter.  
This website is a collection of my technical writeups, vulnerability analyses, and responsible disclosures.  

Each post aims to:
- Document real-world vulnerabilities (legally reported and disclosed)
- Share exploitation techniques
- Help developers and researchers build more secure systems


# Latest writeups

<ul>
  {% for post in site.posts limit:10 %}
    <li>
      <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      <small> — {{ post.date | date: "%Y-%m-%d" }}</small>
      <p>{{ post.excerpt | strip_html | truncate: 160 }}</p>
    </li>
  {% endfor %}
</ul>
