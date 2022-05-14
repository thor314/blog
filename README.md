# Hey what's this
This is the README I wrote to remind myself that I have a blog.

Hey thor. You have a blog.

To add new entries to the blog, I add a file to content. I can check out more deets [https://gohugo.io/getting-started/quick-start/](here).

In a terminal, `hugo new posts/NAME_OF_POST.(org|md)` creates a beautiful new post for me.

Then, `hugo server -D` starts a draft-enabled server where I can check out the build at http://localhost:1313/.

`hugo -D` builds static pages. Output lives in the public directory.

Finally, push to github, and Netlify will pick up my new shtuff. Go manage netlify stuff [[https://app.netlify.com/][here]].

## A couple more details
- archetypes/ contains the template used when I =hugo new blah=.
- content/ contains my draft blog posts.
- data/ contains links to my socials for my social buttons.
- public/ contains built static pages.
- resources/ and /static contain the pretty things.
- themes/ contains blog template related things.
- config.toml is where I'm supposed to modify the blog-related features.
- I added KaTeX math rendering support at the suggestion of [Lucas
  Meier](https://twitter.com/cronokirby), thanks Lucas!
  [This](https://www.simonspavound.com/posts/2020/09/equations-with-katex-in-hugo/)
  blog post describes exactly how I did it.

