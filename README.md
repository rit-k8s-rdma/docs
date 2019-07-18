# docs

The documentation is made with the tool mkdocs which can be found [here](https://www.mkdocs.org/).

## Install
1. (optional) virtualenv for installing mkdocs
2. Install mkdocs
```
pip install mkdocs
```

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