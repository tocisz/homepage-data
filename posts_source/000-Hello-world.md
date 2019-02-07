date: 2018-02-22
abstract: Why this blog came to life? Because it was easy to create it...

# Hello,

Welcome to my minimalistic blog! It's a place for gathering ideas and notes for my hobby projects.

I got web hosting from home.pl for 30 cents... Nothing really special - just PHP and mysql there.

For some time I didn't use it at all. Now I got idea I can publish notes from my projects, if not for the others than just for myself.

Recently I published one tutorial on Github as a Readme.md article for my experiments. Markdown is pretty decent way of writing such notes, so I thought I'll use it on this site. I found [some simplistic solution for Markdown based blog](https://github.com/jacobbednarz/markdown_blog). It is not ideal, but it's a good start.

First what I've changed is version of Markdown-to-HTML translator. I upgraded it to newest version from [here](https://github.com/michelf/php-markdown). By the way I found out that nowadays even PHP has [packaging system for modules](https://getcomposer.org/).

Than I did some minor changes in template...

Last but not least, I wrote simple `cron` job to update this blog from [Github](https://github.com/tocisz/homepage). Now I can edit it on Github and `cron` will pull updates automatically.
