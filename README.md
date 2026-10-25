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
- `details` (optional): extra facts read from a movie or series page: `plot`, `year`, `genres`, a `tmdb` link and an `imdb` link, and the page's own `poster` (used when the title was opened without one). Each can be one recipe or a list of recipes; Sahra uses the first that finds something. Akwam reads the story from the page's JSON and falls back to its `meta` description.
- `play`: how to get from a movie or episode to a video.
- `test`: a word to search for when checking the plugin works.
- `titleCleanup` (optional, on unless you write `false`): Sahra tidies every title a plugin reads. It takes words like "مشاهدة فيلم" off the start and "مترجم", "HD" or the year off the end, and fixes titles written all in small or all in capital letters ("the lord of the rings" becomes "The Lord of the Rings"). It never removes a word from the middle of a title.
- `titleStrip` (optional): more words for Sahra to take off the start or end of titles on this site, like `["حصريا"]`.

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
- `unpack`: read a packed script (the kind that starts `eval(function(p,a,c,k,e,d)`) as plain text, without running it. Use `{ "unpack": true }`. If the script is packed some other way, the step fails and says so.
- `rot13`: undo ROT13, where every letter is moved 13 places. `{ "rot13": true }`.
- `reverse`: turn the text back to front. `{ "reverse": true }`.
- `shift`: move every character by a number, like `{ "shift": -3 }`. Some players hide addresses this way.

In a list, `each` picks one block per title, and `fields` reads the details from inside that block. `scope: "page"` means a field reads the whole page, not only its block.

## Lists: `each` and `fields`

Every list (`list`, `search.list`, `series.episodes`, `series.otherSeasons`, `play.qualities`, `play.sources`) has the same shape: `each` finds one block per item, and `fields` is a set of named recipes read from inside that block.

Each list needs certain fields:

- Titles (`list`, `search.list`, `series.episodes`, `series.otherSeasons`): `title` and `link`. Optional: `poster`, `year`, `quality`, `genres`, `season`, `episode`.
- `play.qualities`: `label` and `link`. Optional: `size`.
- `play.sources`: `url`. Optional: `label`.

An item missing a required field is skipped.

When a series page lists every season at once (Fajer Show does), give each episode a `season` field and leave out `otherSeasons`: Sahra shows that page one season at a time (the first one listed to start with) and offers the others as its other seasons.

A field can use `{name}` in its `select`, where `name` is another field of the same item. Sahra reads that other field first and puts its value in. In Akwam, the `tab` field holds the id of a quality tab, and `link` uses `div#{tab} a.link-show` to find that tab's watch link. Because the tab lives outside the quality's own block, those fields also say `"scope": "page"`, which means "read the whole page, not just this block".

## The other keys, in plain words

- `kindRule` (in `list`): decides if a title is a movie or a series by looking at its link. `{ "linkContains": "/series/", "kind": "series" }` means a link with `/series/` in it is a series. The first rule that matches wins.
- `nextPage` (in `list`): a recipe that finds the "next page" link. If it finds nothing, Sahra knows there are no more pages.
- `within` (inside a recipe): first read some text with the recipe inside `within`, then run this recipe on that text. Akwam uses it to pull the story out of the JSON hidden in a `script` tag: `within` finds the script, and `using: json` with `path: description` reads the story.
- `qualitiesPath` (in `play`, optional): the page the servers are listed on, when it is not the title's own page. Arabseed lists them on the title's watch page: `"qualitiesPath": "{link}watch/"`.
- `qualitiesSteps` (in `play`, optional): when reaching the server list takes more than one page, the steps to get there, written like `resolve` steps; the servers are read from the last one. AsiaTVDrama opens the episode, reads its form's address and episode number, then asks the site's API for the servers.
- `[*]` in a `json` path: every item of a list, or every entry of an object. An entry has a `key` (its name) and a `value`, so `"path": "servers[*]"` with `label` read from `key` lists `{"servers": {"Ok.ru": "…", "VEO": "…"}}` as two servers, in name order.
- `resolve` (in `play`): extra pages to open between the chosen quality and the video. Akwam needs none, so it is `[]`. A step has a `request` (with a `path`) and may have an `extract` and an `as` name to pass its result on as `{name}`. To keep more than one value from the same page, add `keep`: `"keep": { "token": { "using": "regex", "pattern": "token=(\\w+)" } }` makes `{token}` (any name except the ones Sahra fills itself: `link`, `origin`, `address`, `previous`, `page`, `words`, `now`). `{now}` is the current time in milliseconds, for addresses that carry it.
- `sources` (in `play`): how to find the video files on the final page.
- `hoster` (in `play`): how the video is hosted. Use `direct` when the page holds the video file itself.
- `playHeaders` (in `play`): extra details the video player must send when it loads the video. Leave `{}` if the video server needs none.

## Resolvers

Many sites send their videos to the same video hosts (mixdrop, doodstream, voe…). A **resolver** teaches Sahra one such host once, so every plugin can use it. It is its own small file in `resolvers/`, listed in `index.json` under `resolvers`:

```json
"resolvers": [ { "id": "mixdrop", "version": "1.0.0", "file": "resolvers/mixdrop.json" } ]
```

A plugin names the resolvers it needs:

```json
"requires": ["mixdrop"]
```

When someone installs the plugin, Sahra installs its resolvers too, from whichever repository has the highest version. Two plugins that need the same resolver share one copy. It is removed when the last plugin that needs it is removed. Resolvers update on their own, like plugins.

When a plugin reaches a link on a host a resolver handles, Sahra hands that link to the resolver. This can be the quality link, a link a `resolve` step found, or a link `sources` found. The resolver turns it into the video.

A resolver file looks like this:

```json
{
  "format": 1,
  "id": "mixdrop",
  "name": "Mixdrop",
  "version": "1.0.0",
  "hosts": ["mixdrop.*", "*.mixdrop.co"],
  "headers": { "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.0 Safari/605.1.15" },
  "steps": [
    { "request": { "path": "{link}", "headers": { "Referer": "{origin}/" } },
      "extract": { "using": "regex",
                   "pattern": "(eval\\(function\\(p,a,c,k,e,d\\).*?\\.split\\('\\|'\\),0,\\{\\}\\)\\))",
                   "cleanup": [ { "unpack": true },
                                { "replace": "^.*MDCore\\.wurl=\"([^\"]+)\".*$", "with": "https:$1" } ] } }
  ],
  "sources": { "each": { "using": "regex", "pattern": "file:\\s*\"[^\"]+\"" },
               "fields": { "url": { "using": "regex", "pattern": "file:\\s*\"([^\"]+)\"" } } },
  "playHeaders": { "Referer": "{origin}/" },
  "test": { "link": "https://mixdrop.co/e/an-embed-that-stays-up" }
}
```

- `hosts`: the hosts this resolver handles. A `*` stands for one part of the name, between dots: `mixdrop.*` matches `mixdrop.co` and `mixdrop.to`, but not `mixdrop.co.uk` (write that one out in full) and never `mixdrop.co.something.else`. A `*.` at the start stands for one or more parts: `*.mixdrop.co` matches `m1.mixdrop.co` and `a.b.mixdrop.co`, but not `mixdrop.co` itself, so list both when the site uses both. Capital letters don't matter.
- `steps`: work like a plugin's `resolve` steps, `keep` and `{now}` included. `{link}` is the link handed over. `{origin}` is the `https://host` of the last page the steps opened (the link's own host at first), so it follows the host when the link sends Sahra on to another one. A path starting with `/` goes to `{origin}`.
- `sources` (optional): read on the last page the steps reached. It needs a `url` field. Leave it out when the steps themselves reach the video, as Mixdrop's do. If a step's answer is already the video, Sahra plays it and `sources` is not used. When the last step finds a web address (as above), Sahra opens it: a video plays, and a page is read with `sources`.
- `playHeaders`: what the video server needs. `{origin}` is filled in.
- `test.link`: a link that should keep working. Sahra tries it when the resolver is installed and whenever a new version arrives. If it fails, every plugin that requires this resolver shows its play check in red, until a version that works is published. If the video behind the link has simply been deleted, Sahra only shows a warning (the plugin stays green): replace the link with a new one.

Bump the resolver's `version` after every change, and in `index.json` too.

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
