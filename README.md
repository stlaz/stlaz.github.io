## What's this

This repository contains data for my GitHub pages. All content is to be found in the
`gh-pages` branch.

## Creating the pages

I followed the instructions in https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/creating-a-github-pages-site-with-jekyll#creating-your-site

Since I'm not a big Ruby fan, I used the jekyll/jekyll container from DockerHub.

Running it is a bit odd but by following https://github.com/envygeeks/jekyll-docker/blob/master/README.md#quick-start-under-linux--git-bash
I ended up with this:

```bash
docker run -v $(pwd):/srv/jekyll --rm -ti jekyll/jekyll -- bash
# the following commands are run from inside of the container
chown -R jekyll /usr/gem/
cd /srv/jekyll/
jekyll new --skip-bundle .
```

Then, following both the `_config.yml` and the steps around 8 from the aforementioned GitHub guide,
I modified the config a bit in `vim` from a different terminal, and then, again from the container,
run:

```bash
bundle install
# bundle add webrick - this may be needed to get the server started
bundle exec jekyll serve
```
