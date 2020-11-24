# ProSuite Tech Blog

This is a blog of technical articles relating to
ProSuite, ArcGIS, Programming, and GIS in general.
It is mostly a mnemonic device for The ProSuite Authors,
but if anybody else finds it useful, that would please
us even more.

Built with Jekyll and hosted by GitHub Pages.

## Authoring

1. Git clone or pull this repository.
2. Articles are markdown files in the *_posts* (published) or
   *_drafts* directory. Naming convention: use `kebab-case-name.md`
   in *_drafts* and `yyyy-mm-dd-kebab-case-name.md` in *_posts*,
   where `yyyy-mm-dd` the date of first publication.
   Do not change the file name of published articles.
3. Local preview: requires that you installed Ruby and Jekyll.
   Run the *serve.bat* file, which invokes *jekyll* with the
   proper arguments for previewing: it watches the file system
   for changes, automatically rebuilds the site, and publishes
   the site at [localhost:4000](http://127.0.0.1:4000).
4. Once satisfied, `git commit` to the master branch and
   `git push` to the GitHub repo.
5. GitHub Pages should now automatically run the site
   through Jekyll and publish the resulting site.
   Verify at <https://prosuite.github.io/techblog>.
   Remember that posts in *_drafts* will not be published.

Hint: [Visual Studio Code][vscode] has good support for Markdown.
Install the [markdownlint][] extension to help with consistent
formatting (see the *.markdownlint.json* file in this repository).

It is also possible to write articles directly on the GitHub.com site.

[vscode]: https://code.visualstudio.com/
[markdownlint]: https://marketplace.visualstudio.com/items?itemName=DavidAnson.vscode-markdownlint
