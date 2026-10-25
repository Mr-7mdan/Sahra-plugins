# Sahra plugins

A plugin is one JSON file. It teaches Sahra how to read one website: how to list titles, search, open a series and find the video. You do not need to write code to fix one. You only edit text.

`index.json` lists the plugins. `akwam.json` is the first one.

## What a plugin file holds

- `format`: the version of the file layout. Leave it at `1`.
- `id`: a short name with no spaces. Never change it after release.
- `name`: what people see in Sahra.
- `version`: bump this every time you change the file (for example `1.0.0` to `1.0.1`). Apps only update when the number goes up.
- `language`: the language of the site, like `ar`.
- `icon`: a link to a picture.
- `address`: the main web address of the site.
- `fallbackAddresses`: other addresses to try if the main one stops working.
- `addressFinder`: a last resort. It opens another page (a portal) and reads the new address from it.
- `headers`: extra details sent with every request. The `User-Agent` tells the site we are a normal browser. Some sites refuse requests without it.
- `notes`: anything you want to remember. Sahra ignores it.
- `sections`: the rows on the plugin's page. Each has an `id`, a `title`, an Arabic `titleAr`, and a `path`. `{page}` in the path is the page number. For now Sahra's rows only load the first page, so it becomes 1.
- `search`: the path used for search. `{words}` is what the person typed.
- `list`: how to read a page full of titles (used by sections and search).
- `series`: how to read a series page: title, season, story, other seasons and episodes.
- `details` (optional): extra facts read from a movie or series page: `plot`, `year`, `genres`, a `tmdb` link and an `imdb` link. Each can be one recipe or a list of recipes; Sahra uses the first that finds something. Akwam reads the story from the page's JSON and falls back to its `meta` description.
- `play`: how to get from a movie or episode to a video.
- `test`: a word to search for when checking the plugin works.

## How a step reads a page

Every value is read with a small recipe. A recipe has `using`, and often `take` and `cleanup`.

`using` says how to look:

- `css`: find an element on the page by its CSS selector, like `h3.entry-title a`. This is the one to use most of the time.
- `regex`: find text by a pattern. Use it only when `css` cannot do it.
- `json`: read a value from JSON, like `description`.
- `header`: read a reply header sent by the site.
- `redirect`: use the address the site sent us to.

`take` says what to keep from the element you found. `text` (the default) is the words. Or name an attribute, like `href` (a link) or `src` (a picture). `html` gives the inner HTML.

`cleanup` is a list of small fixes, applied in order:

- `trim`: remove spaces at the start and end.
- `absolute`: turn a short link like `/movie/5` into a full one.
- `replace`: swap text for other text (`with` is the new text; leave it empty to delete).
- `number`: keep only the first number.
- `map`: swap words for other words, like Arabic "الخامس" for "5".
- `decode`: undo an encoding. You must say which kind: `{ "decode": "base64" }`, `{ "decode": "html" }` or `{ "decode": "url" }`.

In a list, `each` picks one block per title, and `fields` reads the details from inside that block. `scope: "page"` means a field reads the whole page, not only its block.

## Lists: `each` and `fields`

Every list (`list`, `search.list`, `series.episodes`, `series.otherSeasons`, `play.qualities`, `play.sources`) has the same shape: `each` finds one block per item, and `fields` is a set of named recipes read from inside that block.

Each list needs certain fields:

- Titles (`list`, `search.list`, `series.episodes`, `series.otherSeasons`): `title` and `link`. Optional: `poster`, `year`, `quality`, `genres`, `season`, `episode`.
- `play.qualities`: `label` and `link`. Optional: `size`.
- `play.sources`: `url`. Optional: `label`.

An item missing a required field is skipped.

A field can use `{name}` in its `select`, where `name` is another field of the same item. Sahra reads that other field first and puts its value in. In Akwam, the `tab` field holds the id of a quality tab, and `link` uses `div#{tab} a.link-show` to find that tab's watch link. Because the tab lives outside the quality's own block, those fields also say `"scope": "page"`, which means "read the whole page, not just this block".

## The other keys, in plain words

- `kindRule` (in `list`): decides if a title is a movie or a series by looking at its link. `{ "linkContains": "/series/", "kind": "series" }` means a link with `/series/` in it is a series. The first rule that matches wins.
- `nextPage` (in `list`): a recipe that finds the "next page" link. If it finds nothing, Sahra knows there are no more pages.
- `within` (inside a recipe): first read some text with the recipe inside `within`, then run this recipe on that text. Akwam uses it to pull the story out of the JSON hidden in a `script` tag: `within` finds the script, and `using: json` with `path: description` reads the story.
- `resolve` (in `play`): extra pages to open between the chosen quality and the video. Akwam needs none, so it is `[]`. A step has a `request` (with a `path`) and may have an `extract` and an `as` name to pass its result on as `{name}`.
- `sources` (in `play`): how to find the video files on the final page.
- `hoster` (in `play`): how the video is hosted. Use `direct` when the page holds the video file itself.
- `playHeaders` (in `play`): extra details the video player must send when it loads the video. Leave `{}` if the video server needs none.

## Common fixes

**The site moved to a new address.** Change `address` to the new one. Put the old one into `fallbackAddresses`. Bump `version`.

**Nothing shows up, or Sahra says "Nothing matched".** The site changed its page. Find the right selector:

1. Open the site in a desktop browser and go to the page that fails.
2. Right-click the thing that is missing (a title, a poster) and choose Inspect or Inspect Element.
3. Look at the highlighted tag. Note its tag name and class, like `h3.entry-title`.
4. Put that into the matching `select` in the plugin file.
5. Bump `version` and save.

Some selectors match Arabic header text on the page, for example `:contains(الحلقات)` finds the box whose header says "Episodes". If the site renames that header, the selector stops matching. Update the Arabic word to the new header.

**Videos do not play.** Open the watch page in the browser, Inspect the video, and check that the selector under `play.sources` still points at it.

**Always bump `version` after any change.** Without it, apps keep the old copy.

Keep the file valid JSON: every `"name": value` pair is followed by a comma except the last one in a group, and quotes must be straight (`"`). Paste the file into any JSON checker before you save it.
