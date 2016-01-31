# ParseWebsite
This core module takes an HTML string and returns a proper data object. The
following NPM modules are used:

    articleTitle = Npm.require "article-title"
    teaser = Npm.require "teaser"
    summarize = Npm.require "summarizely"
    cheerio = Npm.require "cheerio"
    readability = Npm.require "readabilitySAX"

The API of this module includes a central `run()` method.

    @ParseWebsite = share.ParseWebsite = (html) ->
      cleanResults extractFromDOM(html)

## DOM Parsing

Every bit of valuable data is needed, so let's get dirty and fish for more
stuff by hand. To make scraping a bit more sane, the package
[cheerio](https://www.npmjs.com/package/cheerio) is used. It's kind of
jQuery, but on the server. This way, CSS3 selectors can be leveraged instead
of nasty XPaths or unreadable RegExps.

    extractFromDOM = (html) ->
      $ = cheerio.load html
      data = {}
      data.feeds = findFeeds $
      data.favicon = findFavicon $
      data.tags = findTags $
      data.references = findReferences $
      data.image = findImage $
      data.description = findDescription $
      data.text = findText $
      data.url = findCanonical $
      data.title = findTitle $
      data.siteName = findSiteName $
      return data

    findFeeds = ($) ->
      selector = """
        link[type='application/rss+xml'],
        link[type='application/atom+xml'],
        link[rel='alternate']
      """
      feeds = $(selector).map((i,e) -> $(this).attr("href")).get()
      return _.uniq _.compact feeds

    findFavicon = ($) ->
      selector = """
        link[rel='apple-touch-icon'],
        link[rel='shortcut icon'],
        link[rel='icon']
      """
      favicon = $(selector).attr("href")
      return favicon

    findTags = ($) ->
      str = $("meta[name='keywords']").attr("content")
      tags = []
      if str
        lang = Text.detectLanguage str
        tags = if /;/.test str then str.split ';' else str.split ','
        tags = Yaki(tags).clean()
      return tags

    findReferences = ($) ->
      refs = $("a").map (i, e) -> $(e).attr("href")
      _.uniq _.compact _.filter refs, (r) -> Link.test r

    findImage = ($) ->
      selector = """
        meta[property='og:image'],
        meta[name='twitter:image']
      """
      list = $(selector).map((i,e) -> $(this).attr("content")).get()
      _.compact(list)[0]

    findDescription = ($) ->
      selector = """
        meta[property='og:description'],
        meta[name='twitter:description'],
        meta[name='description']
      """
      list = $(selector).map((i,e) -> $(this).attr("content")).get()
      _.compact(list)[0]

    findText = ($) ->
      $("h1,h2,h3,h4,h5,h6,p,article").map((i,e) -> $(e).text()).get().join("\n\n")

    findCanonical = ($) ->
      url = $("link[rel='canonical']").attr("href")
      if Link.test url then url else ""

    findTitle = ($) ->
      selector = """
        meta[property='og:title'],
        meta[name='title']
      """
      list = $(selector).map((i,e) -> $(this).attr("content")).get()
      _.compact(list)[0]

    findSiteName = ($) ->
      selector = """
        meta[property='og:site_name']
      """
      list = $(selector).map((i,e) -> $(this).attr("content")).get()
      _.compact(list)[0]

## Merge the results

Join the results from the TextParser and DomParser into one uniform
result object. Pick the best results if there is some overlap.

    cleanResults = (dom) ->
      data = {}
      data.url = dom.url if dom.url
      data.siteName = Text.clean dom.siteName
      data.title = Text.clean dom.title
      data.text = Text.clean dom.text
      data.description = Text.clean dom.description
      data.favicon = dom.favicon
      data.references = dom.references
      data.image = dom.image
      data.feeds = dom.feeds
      data.lang = Text.detectLanguage "#{data.title} #{data.text}"
      data.tags = dom.tags
      return data
