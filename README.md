# Personal Blog

## Dependencies

- [Hugo v0.68.3 or higher, install using homebrew](https://gohugo.io/getting-started/installing/)
- config.toml in root directory (contains gitalk credentials)
- .env in root directory (contains algolia credentials)

## Usage

Add the required submodules:

```
git submodule init
git submodule update
```

Test locally using:

```
npm start
npm run nodraft
```

Deploy using:

```
npm run build-nodraft
npm run build-all
npm run index
```

Then push to the site by cd'ing into the public folder and committing then pushing.
