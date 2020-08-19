# mattias.holmlund.se

Source for public site https://mattias.holmlund.se

## Publishing a new post

- Create a post under content/post
- Make sure that the `date` in the file is correct. hugo will not publish it if the date is in the future
- `hugo server` to test locally
- git push will trigger a netlify deploy

## Gotchas

The `hugo` version is configured in netlify Deploy Settings / Environment.

The theme is a git submodule. When updating it, you may have to edit layouts/partials/head.html. See the same file in the theme for hints.