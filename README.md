UKWA Documentation
==================

Technical documentation for the UK Web Archive.

We use [JupyterBook](https://jupyterbook.org/) to generate this site, allowing example Jupyter notebooks to be mixed with Markdown documentation.

## Making Changes

To update this site, fork it, make your edits, and test them as follows (required Python 3).

Install the dependencies:

```
   virtualenv venv
   source venv/bin/activate
   pip install -r requirements.txt
```

Then build the site:

```
   jb build content
```

The final `jb build content` tells JupyterBook to build the site defined in the `content` folder.

See the returned console information for details of how to view the results. See the [JupyterBook documentation](https://jupyterbook.org/) for more information.

When the changes are ready, open a pull request!

## Official Deployment

Currently, the master branch of this repository gets deployed on [GitHub Pages](https://pages.github.com/) by [this GitHub Action](.github/workflows/book.yml).