https://github.com/Starefossen/docker-github-pages/issues/74
Inside `docker run -it -p 4000:4000 -v $(pwd):/src/site gh-pages /bin/bash`:
* `bundle install`
* `bundle exec jekyll serve --watch --force_polling -H 0.0.0.0 -P 4000 --drafts`
