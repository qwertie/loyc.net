---
title: "How to put a site on GitHub"
toc: false
---

GitHub has made it very easy to get started. If you already have a repo that you want to add web site for, just create a *docs* folder in the repo and start adding .md ([Markdown](https://help.github.com/articles/about-writing-and-formatting-on-github/)) files to it; in particular *index.md* will be recognized as the home page. Then you can select a theme and view your pages as [explained here](https://guides.github.com/features/pages/). Links to other markdown files will automatically be converted to html links (for example if you write `[link](file.md)`, it becomes `<a href="file.html">link</a>`.) You can also write content pages in HTML, but it is rarely necessary. It is possible to insert bits of HTML in your Markdown code if necessary (and [Markdown in your HTML](https://stackoverflow.com/a/50974387/22820)).

Then, if you want a custom theme like you see on this site, my [TypeScript/React Primer](http://typescript-react-primer.loyc.net/) is a good example:
{% raw %}

- [View the site's files](https://github.com/qwertie/learn-react/tree/master/docs). Feel free to make a copy and edit it.
- Make sure you have a *_config.yml* file and remove the `theme` option, if any. If you don't have a custom domain you will need to set the `baseurl` option (for example, if your site is *wxyz.github.io/cool*, use `baseurl: "/cool"`) and delete the CNAME file. The CNAME file is used for custom domains - see [these instructions about that](https://help.github.com/articles/using-a-custom-domain-with-github-pages/).
- Each page on the site has a "layout" in the *_layouts* folder, which is the HTML surrounding the page content. The default layout for pages is *page.html*. You can put a layout "inside" another layout by writing a `layout:` option at the top. For example if *page.html* looks like this:

~~~html
---
layout: "default"
---

<main>
{{ content }}
</main>
~~~

Then the "page" layout is the same as the "default" layout (i.e. *default.html*) except that the content is placed within a `main` element. If you want to have more than one layout (style of page), but you want all pages to share some code, then the code that *all* pages share should go in *default.html*, and code that not all pages share should go in *page.html* or another layout file.

Having multiple layouts is not the only way (or even the best way) to customize pages, though, as we can see by looking at the site's *default.html* file:

~~~html
<!DOCTYPE html>
<html>
{% include head.html %}
<body>
{% include navbar.html %}
{{ content }}
{% if page.toc %}
{% include table-of-contents.html %}
{% endif %}
{% include google-analytics.html %}
</body>
</html>
~~~

This demonstrates two important features:

1. Use `{% include %}` to insert code snippets from the *_includes* folder. For example *head.html* contains the `<head>` element for all pages on the site, and *google-analytics.html* connects to Google Analytics so I can find out if I have any visitors (however, GitHub itself [now gathers its own statistics](https://blog.github.com/2014-01-07-introducing-github-traffic-analytics/)).
2. Use `{% if %}` to insert code conditionally. *table-of-contents.html* adds a table of contents, but it is only included on pages with the `toc: true` option. In order to add the `toc: true` option you need to add something called _front matter_ at the top of each page with a table of contents. Front matter starts and ends with `---`, like this:

~~~
---
layout: "page"
toc: true
tagline: "This is another option"
---

### The page's content appears below the front matter ###
~~~

All features of GitHub Pages are provided by a software package called [Jekyll](https://jekyllrb.com/), and the pages you write are actually [Jekyll templates](https://jekyllrb.com/docs/templates/). Jekyll provides access to the options in front matter via Jekyll's `page` object. For example

    {% if page.toc %}stuff{% endif %}
    
inserts `stuff` if there is a `toc` option, and `{{ page.tagline }}` inserts the contents of the `tagline` string (if any) into the page. Options in the front matter can have any name you want; only a few names (like `layout`) are recognized by Jekyll itself. Generally, quotes around options are optional unless the option contains certain punctuation characters.

You can also put custom options in [*_config.yml*](https://github.com/qwertie/learn-react/blob/master/docs/_config.yml) which are accessible through Jekyll's `site` object. For example, the *google-analytics.html* snippet inserts the `google_analytics` option into itself using `{{ site.google_analytics }}`.
All the [options](https://github.com/qwertie/learn-react/blob/master/docs/_config.yml) on the Primer site are custom options, although the `show_downloads` is used only by GitHub themes (not my custom site) and the `theme` option is recognized by GitHub itself. In addition, there are some options you can put in *_config.yml* [that are recognized by Jekyll](https://jekyllrb.com/docs/configuration/).

Now that I've told you all this, you should be able to understand completely how the Primer site works.

### Other tips ###

- Tired of pushing changes to GitHub just to find out how your site looks? It's possible to set up [Jekyll](https://help.github.com/articles/setting-up-your-github-pages-site-locally-with-jekyll/) on your local PC to match GitHub's configuration.
- Jekyll expressions have extremely limited abilities. The language of Jekyll expressions is called Liquid, which has its [own web site](https://shopify.github.io/liquid/) separate from [Jekyll's web site](https://jekyllrb.com).
- Need a page with _no_ layout? Use `layout: null` in your front matter. Also, an empty `.nojekyll` file will turn off Jekyll for the entire site.
- Wondering how to get syntax highlighting? Be sure to include [syntax css](https://github.com/qwertie/learn-react/blob/master/docs/css/syntax.css) on your site and use `~~~` fences as demonstrated on my site's pages.
- Wondering how to get tables of contents? Use [*table-of-contents.html*](https://github.com/qwertie/learn-react/blob/master/docs/_includes/table-of-contents.html) as shown above; you also need [CSS]((https://github.com/qwertie/learn-react/blob/master/docs/css/styles.css)) for the `sidebox` class.
- Wondering how to make your site look good on mobile devices and small or large browser windows? I use simple `@media` queries in my [*styles.css*](https://github.com/qwertie/learn-react/blob/master/docs/css/styles.css) to subtly alter appearance so that my site's wide code snippets are more readable on phones. Phones lie about their resolution, by the way: 1px is normally 2 or 3 real pixels. Also, use `pre { overflow: auto; }` so code snippets can scroll left and right, and be careful how you use tables!
- Wondering how to add a blog? Create a *_posts* folder for your posts and put a date in each file name, e.g. *_posts/2018-12-31-my-blog-post.md*. Jekyll treats the pages in this folder in a special way, e.g. all blog posts are placed in a special collection called `site.posts`. What do we do with that? Here's [the page I use](https://github.com/qwertie/loyc.net/blob/gh-pages/blog-list.html) to show a [list of all posts](http://loyc.net/blog-list.html) with titles only, and here's [the page I use](https://github.com/qwertie/loyc.net/blob/gh-pages/blog/index.html) to render this blog's [main page](http://loyc.net/blog). And here's [how I generate an Atom (RSS) feed](https://github.com/qwertie/loyc.net/blob/gh-pages/atom.xml) for newsreaders. The main blog page has pagination, relying on the `paginate` and `paginate_path` options as demonstrated in [my blog's *_config.yml*](https://github.com/qwertie/loyc.net/blob/gh-pages/_config.yml). Also, you can edit the [`permalink` option](https://jekyllrb.com/docs/permalinks/) to systematically change the URLs of all blog entries. I use `permalink: "/:categories/:year/:title:output_ext"` (with no month and day) because I only publish a few blog posts per year and I prefer shorter URLs. Notice that the "/:categories" part has no effect on this post because this post doesn't have a `categories:` option. In fact, `permalink` reportedly affects *all* pages, not just blog posts, but non-blog posts don't have a year/month/day so that part of the permalink template is ignored on other pages. If you leave out `:output_ext`, Jekyll gives you "pretty" URLs with no `.html` extension.
- Want to set up a multilingual site? Have a look at [Sylvain Durand's guide](https://www.sylvaindurand.org/making-jekyll-multilingual/) and [example site](https://github.com/sylvaindurand/jekyll-multilingual). I _think_ that's the advice I used to set up my [Ungglish site](http://ungglish.loyc.net).
- Want a custom "404 Not Found" page? Just add a *404.html* or *404.md* page with a `permalink: 404.html` option ([example](https://github.com/qwertie/loyc.net/blob/gh-pages/404.html) / [official instructions](https://help.github.com/articles/creating-a-custom-404-page-for-your-github-pages-site/)).
- Wondering how to support comments? Jekyll sites cannot store user-defined content, but it is possible to [add user comments to pages via GitHub's issues system](http://ivanzuzak.info/2011/02/18/github-hosted-comments-for-github-hosted-blogs.html). 
- See also: my old post about [blogging with GitHub Pages](http://loyc.net/2014/blogging-on-github.html).

{% endraw %}
