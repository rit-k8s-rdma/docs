# docs

The documentation is made with the tool mkdocs which can be found [here](https://www.mkdocs.org/).

## Install
1. (optional) virtualenv for installing mkdocs
2. Install mkdocs
```
pip install mkdocs
```

### Files to change
There are a lot of files in this directory for the docs to be correct they had to all be built put in the root. The two things you need to edit in order to update the site are:
1. docs/ - directory that contains the MarkDown pages
2. mkdocs.yml - yaml for the navbar and update theme, etc.

## Modifying and Viewing
1. Startup mkdocs in serve mode
```
mkdocs serve
```
2. Start editing files and it will update live

## Updating Site
1. Run the mkdocs build command to update site
```
mkdocs build
```
2. Copy site files to root directory
```
cp -R site/* .
```
3. Push your changes up the repo
```
git commit -m "Message"
git push
```