# Introduction

[CouchDB](http://couchdb.apache.org) a database management system server
that use HTTP and HTTPS protocols
primarily to deliver `JSON` documents.
[CouchApp](http://docs.couchdb.org/en/latest/couchapp) a client-server application framework based
on a Web browser front-end and a `CouchDB` back-end.

`pry` is a `CouchApp` that manages literate
programs on `CouchDB`. It uses [markover](http://cygnyx.github.io/markover)
to 'tangle' a `markdown` document into a `CouchApp` and 'weave' it into
a `HTML` document.
From the same [README.md](README.md), it produces a [README.json](README.json) 
and a [README.html](README.html). `CouchDB` delivers a [design document](.)
that is very similar to the `README.json` file.

`pry` itself has a small API: for the source README and the derived `JSON` and
`HTML` versions.
In addition it implements an [editor](editor).
The editor optionally takes a `id` argument with a path to the source
document.

`pry` is a rudimentry deployment tool along the lines of
[Kanso](http://kan.so), [couchapp](https://github.com/couchapp/couchapp), or
[capp](https://github.com/nick-thompson/capp).

`pry` doesn't support attachments to documents. It only supports the creation
of a `JSON` object (or document) which is deployed to the `CouchDB`.
By convention, the source document is called `README.md` and it is included
in the `JSON` document.
This might be a little confusing at first.
The important point is that you shouldn't create a `JSON` document with a
field named `README.md` since this will be used by `pry` to store the source
document (but it is possible to use a different field name).

By default, `pry` operates on itself. In the editor, it will download the
source document (in the `README.md` field) into a browser-based editor.
You can modify `pry`, changing its documentation or source code in the editor.
By changing the `_id` of the document, you can upload it to a different location.
You can change the database used as well in the editor.

Using the `id` parameter, `pry` will edit another source document
by downloading its source document (in the `README.md` field) into the editor.

For example, after opening the `pry` editor you can replace its contents with:

    ```#+_id
    foo
    ```
    
    ```#+history
    It is derived from fubar.
    ```

and upload this document. The resulting `JSON` object will have 3 fields: `_id`,
`history`, and `README.md`. In the editor, these fields are visible in the `Source Code`
tab. Then you can edit this document by passing in `?id=example/foo`.
Recall that `markover` uses a `+` (plus sign) to indicate that a field should be quoted.
You can create a new document by renaming the `_id` field to `bar` and uploading.

You can change the source document name using the option `md` to the editor.
For example, `?md=READMEFIRST.md` would look for the `READMEFIRST.md` field for
the source document.

```noweave#+title
pry
```

```noweave#+tag
CouchApp Literate Programming
```

```noweave#+version
0.0.1
```

# CouchDB fields

There is only one field that is require in CouchDB for every document: _id.
This is the identifier that the JSON document is addressed by.
In this case, `pry` is a design document:

```#+_id
_design/pry
```

# CouchApp fields

There are many optional fields used by a `CouchApp`, but
`pry` is only using two fields: `rewrites` and `shows`.

Rewrites are used to translate nice URLs into the URLs used internally.
They are particularly useful when combined with a virtual host.
For example, by creating a DNS for 'example', `CouchDB` can be
configured to use the path '/example/_design/pry/_rewrite'.
Then URLs like 'http://example/README.html' are translated to
'http://example/example/_design/pry/_rewrite/README.html'.
Note that it is difficult to use `pry` with virtually hosts.
You are better off using the real hostname.

This represents the entire API for `pry`.

```#rewrites
[
   {
       "from": "",
       "to": "."
   },
   {
       "from": "/README.html",
       "to": "_show/README.html"
   },
   {
       "from": "/README.md",
       "to": "_show/README.md"
   },
   {
       "from": "/README.json",
       "to": "_show/README.json"
   },
   {
       "from": "editor",
       "to": "_show/editor"
   },
   {
       "from": "*",
       "to": "_show/404"
   }
]
```

The first URL returns a `JSON` object of the `pry` design document.
The other URLs are converted into show function calls.

CouchApps have limited support for javascript.
The `shows` object is one of the places where you can embed
anonymous javascript functions in a string.
Each function has 2 arguments: the document and the request.
Since they are embedded in strings it is easier to implement
the details in another functions.
But the other functions must be contained within a `CommonJS` module.
In this case the module 'main' is used.
These anonymous rewrite functions tend to be one-liners.

```#shows
{
   "404": "function(doc, req){ return {body: '<h1>404 - Document not found</h1>\\n'}, headers: {'Content-Type': 'text/html'}}}",
   "editor": "function(doc, req){ return {body: require('main').edit.call(this, req), headers: {'Content-Type': 'text/html'}}}",
   "README.md": "function(doc, req){ return {body: this['README.md'], headers: {'Content-Type': 'text/markdown'}}}",
   "README.html": "function(doc, req){ return {body: require('main').weave.call(this), headers: {'Content-Type': 'text/html'}}}",
   "README.json": "function(doc, req){ return {body: require('main').tangle.call(this), headers: {'Content-Type': 'text/plain'}}}"
}
```

# Program Structure

`pry` relies on `markover` for document processing, which relies on `marked`
and `highlight.js`. The editor in `pry` is implemented in the Web browser (of course).
So there are some challenges to constructing an application (`pry`) in the CouchDB environment
that will serve up another application (the editor) in the Web browser environment.
The main issues revolve around special characters and strings that are not allowed
in certain situations.

# Editor

The `pry` editor is a meta-level HTML document. That is, in this document
we are writing the code that will delivered to the Web browser that will
provide the editor for the document.

## HTML layout

The editor is an HTML document with 4 tabs.

```js#+editorcontent
<ul class='tabs'>
  <li><a href='#tab1'>Document</a></li>
  <li><a href='#tab2'>Source Code</a></li>
  <li><a href='#tab3'>HTML</a></li>
  <li><a href='#tab4'>Update</a></li>
</ul>

<div class='display'>
<div id='tab1'><input type="text" id='_src' value=''></input><textarea id='_editor'></textarea></div>
<div id='tab2'><div id='_code'></div></div>
<div id='tab3'></div>
<div id='tab4'></div>
</div>
```

## Editor stylesheet

The editor needs its own stylesheet.

First, reset the stylesheet.

```css#+editorstyle
html, body, div, span, object, iframe, h1, h2, h3, h4, h5, h6, p, blockquote, pre,
abbr, address, cite, code, del, dfn, em, img, ins, kbd, q, samp,
small, strong, sub, sup, var, b, i, dl, dt, dd, ol, ul, li, a, textarea, input,
fieldset, form, label, legend, table, caption, tbody, tfoot, thead, tr, th, td,
article, aside, canvas, details, figcaption, figure,
footer, header, hgroup, menu, nav, section, summary, time, mark, audio, video {
  display: block;
  margin: 0;
  padding: 0;
  border: 0;
  outline: 0;
  line-height: 140%;
  font-size: inherit;
  color: black;
  vertical-align: baseline;
  text-decoration: none;
  background: transparent;
}

a, code, span, b, i, em { display: inline; }

body {
  font-size: 100%;
}
```

Highlight active elements in the interface.

```css#+editorstyle
#tab1 #_src, #tab1 #_editor, .tabs li {
  border: solid black 1px;
}

.tabs li:hover, #tab1 #_editor:focus, #tab1 #_src:focus {
  border: solid orange 1px;
}
```

For the coding environment, use a monospace type.
Set up the tabs with the same margins.

```css#+editorstyle
.display, .tabs {
  margin: 0.4em;
}

#tab1 #_editor, #tab1 #_src {
  width: 99%;
  font-family: Courier, monospace;
  padding: 0.5%;
}

#tab1 #_editor {
  resize: none;
  height: 40em;
}
```

Configure the layout of the menu.

```css#+editorstyle
.tabs li {
  list-style: none;
  display: inline;
  padding: 0.4em;
}

.tabs a, .tabs input {
  display: inline-block;
  color: gray;
}

.tabs a.active {
  color: black;
}
```

For the source code tab show the formatted output.

```css#+editorstyle
.display #_code {
  white-space: pre;
  overflow: visible;
  word-wrap: break-word;
}
```

Hide the menu when printing.

```css#+editorstyle

@media print {
  .tabs {
    display: none;
  }
}
```

## The Editor javascript

After the document is loaded, insert the source into the editor
and configure the tabs.
The source location is initialized as an absolute path on
the couchdb host, not a virtual host.

Tangle and weave can take a significant amount of time.
The `longwait` function is called after the document is modified
and the user has stopped typing. In the first pass it handles
the 2nd tab and in the second pass it handles the 3rd tab.
A short timeout is used between passes to allow for user
interactions to update. Perhaps this could be implemented in a
worker process instead of the main thread. The delay time
is estimated from the previous runtime.

```js#+editorscript
var tab2todo = true

function longwait() {
  var st = new Date()
  if (tab2todo) {
    setuptab2()
    this.timeout = setTimeout(longwait, 50)
  } else {
    setuptab3()
  }
  tab2todo = !tab2todo
  var ms = new Date() - st
  if (ms < 500) ms = 500
  this.delay = 2 * ms
}
```

Getting and putting documents happens asynchronously.
Usually the time gaps are not too significant between
the request and the delivery of a result.

When the document is available `putDoc1` initializes
the editor. `putDoc` is called when the document has
been delivered.

```js#+editorscript
function putDoc1(md) {
  $("#_editor").text(md)
  setuptab1()
  this.timeout = setTimeout(longwait, 50)
}

function putDoc(errflag, md) {
  if (errflag)
    md = ''
  else
    md = md[mdref]
  putDoc1(md)
}
```

`makeurl` adds server location information to a path.

```js#+editorscript
function makeurl(path) {
  return location.protocol + '//' + location.host + '/' + path
}
```

This is called when the document is loaded.
It installs a timeout on the keyup event in the editor,
records the path to the source document,
and gets the source document.
The source document is either pre-downloaded or
we have to get it.

```js#+editorscript
$(document).ready(function () {
  $("#_editor").keyup(function() {
    if (this.timeout) clearTimeout(this.timeout)
    tab2todo = true
    this.timeout = setTimeout(longwait, this.delay || 1000)
  })
  $("#_src").val(mddb)
  markover = require('markover')

  if (typeof md == "undefined") {
    var url = makeurl(mddb)
    getDocument(url, putDoc)
  } else
    putDoc1(md)
}) 
```

In HTML documents the processing of '&' and '<' characters is tricky.
This function handles HTML escapes codes.

```js#+editorscript
function escape(str) {
  return str
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
}
```

Tangle the HTML editor element

```js#+editorscript
var markover

function tangleDoc() {
  var src = $('#_editor')[0].value
  var el = $('#_code')
  var res
  if (typeof mdref == "undefined" || mdref == "")
    res = markover.tangleJSON(src)
  else
    res = markover.tangleJSON(src, {sourcetarget: mdref})
  return res
}
```

Functions to set up the tabs.

```js#+editorscript
function setuptab1() {
  $("#_editor")[0].focus()
}

function setuptab2() {
  var el = $('#_code')
  var txt = tangleDoc()
  el.text(txt)
}

function setuptab3() {
  var src = $('#_editor')[0].value
  var el = $('#tab3')
  var txt = markover.weaveJSON(src, {contentonly: true})
  el.html(txt)
}

function setuptab4() {
  var src = $('#_editor')[0].value
  var url = $('#_src')[0].value.split('/')
  url = url[0]
  var txt = tangleDoc()
  var json = JSON.parse(txt)
  var id = json._id
  json[mdref] = src
  url = [url, id].join('/')
  $('#tab4').html('</h1>Updating ' + id + '</h1>')
  url = makeurl(url)
  updatedoc(url, json)
}

var editfunction = {
  tab1: function(){setuptab1()},
  tab4: function(){setuptab4()}
}
```

Set up the menus to show the active tab.
Install the onclick event handler to switch tabs.

```js#+editorscript
$('ul.tabs').each(function() {
  var active, content, links = $(this).find('a'), inputs = $(this).find('input')

  active = $(links[0])
  active.addClass('active')

  content = $(active[0].hash)

  links.not(active).each(function () {
    $(this.hash).hide()
  })

  $(this).on('click', 'a', function(e){
    active.removeClass('active')
    content.hide()

    active = $(this)
    content = $(this.hash)

    active.addClass('active')

    var f = editfunction[this.hash.substring(1)]
    if (typeof f !== "undefined") {
       f()
    }
    content.show()

    e.preventDefault()
  })
})
```

Quick and dirty implementation of `CommonJS` `require` function.
If a `name` has been loaded, it is returned.
The loading of the modules is described below.

```js#+editorscript
var require = function require(name) {
  if (require.loaded[name])
    return require.loaded[name]
  throw new Error("module missing:" + name)
}
require.loaded = {}
```

## Update document

Before a document can be updated, the revision
of the current document is needed.

Perhaps the revision could be reported by the server,
but for now we simply get the current version from the server.

This function gets a document from a CouchDB server
and calls the callback function with the errflag,
the JSON response, the URL and some other data.

```js#+editorscript
function getDocument(url, cb, data) {
  $.ajax({
    url: url,
    dataType: 'json'
  })
  .done(function(d, s, x) {
    cb(false, x.responseJSON, url, data)
  })
  .fail(function(x, s, e) {
    cb(true, x.responseJSON, url, data)
  })
}
```

Similarly, this function puts a document to a CouchDB server,
calling the callback function with the errflag, and the JSON response.

```js#+editorscript
function putDocument(url, doc, cb) {
  $.ajax({
    url: url,
    type: 'PUT',
    data: JSON.stringify(doc),
    contentType: 'application/json',
    dataType: 'json'
  })
  .done(function(d, s, x) {
    cb(false, x.responseJSON)
  })
  .fail(function(x, s, e) {
    cb(true, x.responseJSON)
  })
}
```

Updating a document happens through a series of
asynchronous calls. The first step is to get the
current version of the document.

```js#+editorscript
function updatedoc(url, json) {
  getDocument(url, updatedoc1, json)
}
```

If there isn't a current document, assume that
a new document is being created, otherwise
add the revision number to the new version of
the document.

```js#+editorscript
function updatedoc1(errflag, resp, url, doc) {
  var rev
  var doit = false
  var err
  if (errflag) {
    if (resp) {
      err = resp.error
      doit = resp.error == "not_found"
    } else {
      doit = true
    }
  } else {
    doit = true
    doc._rev = resp._rev
  }
  if (!doit)
    $('#tab4').html('</h1>Update Error: ' + (err || "unknown") + '</h1>')
  else {
    $('#tab4').html('</h1>Updating ' + url + '</h1>')
    putDocument(url, doc, updatedocwithrev)
  }
}
```

Now the put worked or it didn't. Report the result.

```js#+editorscript
function updatedocwithrev(errflag, doc) {
  if (errflag)
    $('#tab4').html('</h1>Update Error: ' + "unknown" + '</h1>')
  else
    $('#tab4').html('</h1>Updated ' + doc.id + ' to revision ' + doc.rev + '</h1>')
}
```

## Main module

The main module implements the functionality of `pry`.

```js#+main
!(function() {

var markover = require('markover')

var main = {
```

First tangle and weave are handled.
These operations happen on the server side
and are unrelated to the editor.

```js#+main
  weave: function () {
    return markover.weaveJSON(this['README.md'])
  },
  tangle: function() {
    return markover.tangleJSON(this['README.md'], {sourcetarget: 'README.md'})
  },
```

The '</script>' tag needs special handling here. It isn't allowed inside a string
within a javascript block. 
The 'fixup' function replaces '</script>' with '<&#' and '47;script>' combined,
but the script block delimiters
in the 'edit' function should *not* be replaced, and they use '</'+'script>'.

```js#+main
  fixup: function(str) {
    return str.replace(/<\/script>/gm, '<&#'+'47;script>')
  },
```

Regular multiline text is encoded in JSON strings.
Special characters are mapped to escape codes.

```js#+main
  quotestring: function(code) {
    return ('"' +
      code
        .replace(/\\/gm, '\\\\')
        .replace(/\"/gm /*"*/, '\\"')
        .replace(/\n/gm , '\\n')
        .replace(/\t/gm , '\\t')
      + '"')
  },
```

Produce the HTML edit document.
First get the encoded source document.
Note that 'fixup' runs on this document,
so any reference to '</script>' must be handled with care.

```js#+main
  edit: function(req) {
    var mdref = 'README.md'
    var p = req.path
    var mddb = p.slice(0,3).join('/')
    var src
    if (req.query.md) {
      mdref = req.query.md
    }
    if (req.query.id) {
      mddb = req.query.id
    } else
      src = main.quotestring(main.fixup(this[mdref]))
    var buf = []
```

The editor header material

```js#+edithead
<!DOCTYPE html><html><head>
<meta charset="utf-8">
<meta http-equiv="Content-Language" content="en">
<meta name="viewport" content="initial-scale=1">
<title>Editor</title>
<style>
```

Push the header material, the style and
additional script references.
Note the handling of the </script> tag.

```js#+main
    buf.push(this.edithead)
    buf.push(this.editorstyle)
    buf.push('</style>'); // syntax help

    ['https://code.jquery.com/jquery-2.1.3.min.js'].forEach(function(e){
      buf.push('<script type="text/javascript" src="'+e+'"></'+'script>')
    })

    buf.push("</head><body>")
    buf.push(this.editorcontent)
    buf.push("</body></html>")
```

After the document the on-the-fly script is included.
First record the location of the source document, its contents
and the basic edit scripts.

```js#+main
    buf.push('<script type="text/javascript">')
    buf.push('var mdref = "' + mdref + '"')
    buf.push('var mddb = "' + mddb + '"')
    buf.push('var md ' + (src ? (' = ' + src) : ''))
    buf.push(this.editorscript)
    var self = this
```

This is my quick and dirty implementation of pre-loaded `CommonJS`
modules.
The modules are assumed to exist in the design document, `self`.
An initial list of modules is set in `incs`.
For each module, check if it hasn't been loaded.
If not, then statically scan the module code and push
any dependent modules onto `incs`.
Module names and contents are pushed into `mods`

```js#+main
    var incs = ["markover"]
    var e
    var r
    var i
    var exports = {}
    var mods = []

    for(i = 0; i < incs.length; i++) {
      e = incs[i]
      if (exports[e] == undefined && self[e] != undefined) {
        main.scanrequire(incs, self[e])
        if (self[e] != null) {
          mods.push([e, self[e]])
          exports[e] = true
        }
      }
    }
```

Process `mods` in reverse order. All dependent modules
should be loaded before the requiring module.
Initialize a empty `module` and `exports`.
Wrap the module code in a function with arguments
of `module`, `exports`, and `require`.
Call the wrap function with the prepared arguments.
Check for non-empty results and store it in `loaded`.
Clean up temporary objects.

This should work for modules that are not mutually dependent.

An alternative implementation would be to put a default value
for all modules that are in `mods` into require.loaded and
then update the values as they get loaded. May be the default
value would be a function that would return the real object
after it has actually been loaded.

```js#+main
    i = mods.length
    while ( (--i) >= 0) {
      e = mods[i][0]
      r = mods[i][1]
      buf.push('require.module = {}')
      buf.push('require.exports = {}')
      buf.push("require.loading = function (module, exports, require) {")
      buf.push(main.fixup(r))
      buf.push("}")
      buf.push("require.loading(require.module, require.exports, require)")
      buf.push('if (typeof require.module.exports == "undefined") require.module.exports = require.exports')
      buf.push('require.loaded["' + e + '"] = require.module.exports')
      buf.push('delete require.module.exports')
    }
```

The final step is some clean and closing the script.
Note the '</script>' handling.

```js#+main
    buf.push('delete require.loading')
    buf.push('delete require.module')
    buf.push('delete require.exports')
    buf.push('</'+'script>')
    buf.push("")
    return buf.join('\n')
  },
```

`scanrequire` statically looks for module names

```js#+main
  scanrequire: function(incs, str) {
    var m = str.match(/require\(['"]([^'"]*)['"]\)/gm)
    if (m != null) {
      m.forEach(function(e) {
        var name = e.substring(9,e.length-2)
        if (incs.indexOf(name) == -1)
          incs.push(name)
      })
    }
  }
```

Now, export the editor module

```js#+main
}

module.exports = main

}).call(function () {
        return this || (typeof window !== 'undefined' ? window : global)
})
```

# HTML layout

This HTML layout is used for the weave document.
Here is the typical start of an HTML document.

```html#+head
<!DOCTYPE html>
<html lang="en" class="">
<head>
```

These meta lines might help.

```html#+head
  <meta charset="utf-8">
  <meta http-equiv="Content-Language" content="en">
  <meta name="viewport" content="initial-scale=1">
```

And the transition from head to body

```html#+transition
</head>
<body>
```

Tail end

```html#+tail
</body>
</html>
```

# HTML stylesheet

Make the formatted README look a little smarter:

## Vanilla style

This first section resets all `HTML` elements to reasonable default values.

```css#+headstylesheet
html, body, div, span, object, iframe, h1, h2, h3, h4, h5, h6, p, blockquote, pre,
abbr, address, cite, code, del, dfn, em, img, ins, kbd, q, samp,
small, strong, sub, sup, var, b, i, dl, dt, dd, ol, ul, li, a,
fieldset, form, label, legend, table, caption, tbody, tfoot, thead, tr, th, td,
article, aside, canvas, details, figcaption, figure,
footer, header, hgroup, menu, nav, section, summary, time, mark, audio, video {
  display: block;
  margin:0;
  padding:0;
  border:0;
  outline:0;
  line-height: 140%;
  font-size: inherit;
  color: black;
  vertical-align:baseline;
  text-decoration: none;
  background:transparent;
}

a, code, span, b, i, em { display: inline; }

body {
  font-size: 100%;
}

@media print {
  body {
    font-size: 80%;
  }
}
```

## Preliminary Material

The following stylesheet operates within the content generated by applying `weave`
to the source document.
`markover` marks up 3 areas before the main document body:
title, tag, and a table of contents.

Title and tag are each in a single id and only appear once in the display document.

```css#+stylesheet
#content {
  font-family: "HelveticaNeue-Light", "Helvetica Neue Light", "Helvetica Neue", Helvetica, Arial, "Lucida Grande", sans-serif; 
}

#content #title {
  clear: both;
  text-align: center;
  font-size: 140%;
  font-weight: bold;
  margin: 2em 0 0 0;
}

#content #tag {
  clear: both;
  text-align: center;
  font-size: 110%;
  margin: 0 0 1em 0;
}
```

The table of contents, in a single `toc` id, is slightly more complex.

```css#+stylesheet
#content #toc {
  padding: 0 0 0 1em;
  max-width: 96%;
  page-break-after: always;
}

@media screen and (min-width: 641px) {
  #content #toc {
    float: left;
    max-width: 26%;
  }
}

#content .tocsn {
  float: left;
}

#content .tocsr {
  overflow: hidden;
}

#content #toc ol {
  list-style-type: none;
}

#content #toc ol ol {
  margin: 0 0 0 1em;
}
```

## Main body

The main document is in a single `doc` id
Make the toc and doc with the same margins.

```css#+stylesheet
#content #doc,
#content #toc {
  margin: 0 1em 0 1em;
}

#content p { text-indent: 1em; }

#content code { font-size: 120%; }

#content pre code {
  font-family: "Courier New", Courier, monospace;
  font-size: 100%;
}

@media screen and (min-width: 641px) {
  #content #doc {
    overflow: hidden;
    max-width: 70%;
  }
}
```

Field names are in a `field` class.

```css#+stylesheet
#content pre .field {
  display:block;
  margin: 1em 1em 0em 0em;
}
 
#content pre .field::after {
  content: " :=";
}
```

The code blocks are in the `code` tag and
have classes that identify their language.

```css#+stylesheet
#content pre code {
  display: block;
  margin: 1em;
  padding: 1em;
  border: solid 1px #aaa;
}

@media screen {
  #content pre code {
    background-color: AliceBlue;
    overflow: auto;
    max-height: 30em;
  }
}

@media print {
  #content pre {
    page-break-inside: avoid;
  }
  #content pre code {
    overflow: visible;
    word-wrap: break-word;
  }
}
```

Each language uses a slightly different light background color.

```css#+stylesheet
@media screen {
  #content .lang-js { background-color: ivory; }
  #content .lang-md { background-color: lightyellow; }
  #content .lang-html { background-color: floralwhite; }
  #content .lang-json { background-color: honeydew; }
  #content .lang-css { background-color: cornsilk; }
  #content .lang-sh { background-color: lemonchiffon; }
}
```

The header font sizes are set.

```css#+stylesheet
#content h1,
#content h2,
#content h3,
#content h4,
#content h5,
#content h6 {
  margin: 0.5em 0em;
  page-break-after: avoid
}

#content h1 { font-size: 140%; }
#content h2 { font-size: 130%; }
#content h3 { font-size: 120%; }
#content h4,
#content h5,
#content h6 { font-size: 110%; }
```

More styling

```css#+stylesheet
#content a:hover { color: #aaa; }

#content a:visited,
#content a { color: #000;}

@media screen {
  #content a {
    text-decoration: underline;
  }
}

@media print {
  #content a {
    text-decoration: none;
  }
  #content #doc a[href]:after {
    content: " (" attr(href) ")";
    font-size: 90%;
  }
}
```

## Highlight markup

`highlight.js` specific styles for code blocks.
Basic type markup:

```css#+stylesheet
.hljs {
    overflow: auto;
    padding: 0.5em;
    color: #333;
}
.hljs-comment, .diff .hljs-header, .hljs-javadoc {
    color: #999;
    font-style: italic;
}
.hljs-keyword, .css .rule .hljs-keyword, .hljs-winutils, .nginx .hljs-title,
.hljs-subst, .hljs-request, .hljs-status {
    color: #333;
    font-weight: bold;
}
.hljs-number, .hljs-hexcolor, .ruby .hljs-constant {
    color: #088;
}
.hljs-string, .hljs-tag .hljs-value, .hljs-phpdoc, .hljs-dartdoc, .tex .hljs-formula {
    color: #d14;
}
.hljs-title, .hljs-id, .scss .hljs-preprocessor {
    color: #900;
    font-weight: bold;
}
.hljs-list .hljs-keyword, .hljs-subst {
    font-weight: normal;
}
.hljs-class .hljs-title, .hljs-type, .vhdl .hljs-literal, .tex .hljs-command {
    color: #458;
    font-weight: bold;
}
.hljs-tag, .hljs-tag .hljs-title, .hljs-rules .hljs-property, .django .hljs-tag .hljs-keyword {
    color: #000080;
    font-weight: normal;
}
.hljs-attribute, .hljs-variable, .lisp .hljs-body {
    color: #008080;
}
.hljs-regexp {
    color: #009926;
}
.hljs-built_in {
    color: #0086b3;
}
```

Language specific markup

```css#+stylesheet
.hljs-symbol, .ruby .hljs-symbol .hljs-string, .lisp .hljs-keyword,
.clojure .hljs-keyword, .scheme .hljs-keyword, .tex .hljs-special,
.hljs-prompt {
    color: #990073;
}
```

Other uncommon specialize markup:

```css#+stylesheet
.hljs-preprocessor, .hljs-pragma, .hljs-pi, .hljs-doctype, .hljs-shebang,
.hljs-cdata {
    color: #999;
    font-weight: bold;
}
.hljs-deletion {
    background: #fdd;
}
.hljs-addition {
    background: #dfd;
}
.diff .hljs-change {
    background: #0086b3;
}
.hljs-chunk {
    color: #aaa;
}
```

Using `CSS` it is possibly to have a separate
layout for print media.
`markover` overrides certain values for print media.

```css#+stylesheet
@media print {
  .hljs {
    overflow: visible;
    word-wrap: break-word;
  }
  .hljs-number, .hljs-hexcolor, .ruby .hljs-constant {
    color: #888;
  }
  .hljs-string, .hljs-tag .hljs-value, .hljs-phpdoc, .hljs-dartdoc, .tex .hljs-formula {
    color: #ddd;
  }
  .hljs-title, .hljs-id, .scss .hljs-preprocessor {
    color: #999;
  }
  .hljs-class .hljs-title, .hljs-type, .vhdl .hljs-literal, .tex .hljs-command {
    color: #888;
  }
  .hljs-tag, .hljs-tag .hljs-title, .hljs-rules .hljs-property, .django .hljs-tag .hljs-keyword {
    color: #888;
  }
  .hljs-attribute, .hljs-variable, .lisp .hljs-body {
    color: #888;
  }
  .hljs-regexp {
    color: #999;
  }
  .hljs-symbol, .ruby .hljs-symbol .hljs-string, .lisp .hljs-keyword,
  .clojure .hljs-keyword, .scheme .hljs-keyword, .tex .hljs-special,
  .hljs-prompt {
    color: #999;
  }
  .hljs-built_in {
    color: #888;
  }
  .hljs-deletion {
    background: #ddd;
  }
  .hljs-addition {
    background: #ddd;
  }
  .diff .hljs-change {
    background: #888;
  }
}
```

# Modules

`pry` makes use of several `CommonJS` modules.
The code is omitted from the `weave` output.

`markover` provides weave and tangle.

```noweave#+markover
!(function() {

var markover = {

"title":"markover",

"tag":"Web Literate Programming",

"version":"0.0.1",

"head":"<!DOCTYPE html>\n<html lang=\"en\" class=\"\">\n<head>\n  <meta charset=\"utf-8\">\n  <meta http-equiv=\"Content-Language\" content=\"en\">\n  <meta name=\"viewport\" content=\"initial-scale=1\">\n  <script type=\"text/x-mathjax-config\">\n    MathJax.Hub.Config({\n      showProcessingMessages: false,\n      tex2jax: { inlineMath: [['\\\\(','\\\\)']] },\n      TeX: { equationNumbers: {autoNumber: \"AMS\"} }\n    });\n  </script>\n  <script type=\"text/javascript\"\n    src=\"http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML\">\n  </script>\n  <script type=\"text/javascript\"\n    src=\"http://code.jquery.com/jquery-2.1.3.min.js\">\n  </script>",

"transition":"</head>\n<body>",

"tail":"</body>\n</html>\n<script type=\"text/javascript\">\n  [\"bash\", \"xml\", \"javascript\", \"json\", \"css\", \"markdown\"].forEach(function(e) {\n    var s = $('<script>')\n    s.type = 'text/javascript'\n    s.src = 'http://cdnjs.cloudflare.com/ajax/libs/highlight.js/8.4/languages/' + e + '.min.js'\n    $('head').append(s)\n  })\n</script>",

"headstylesheet":"html, body, div, span, object, iframe, h1, h2, h3, h4, h5, h6, p, blockquote, pre,\nabbr, address, cite, code, del, dfn, em, img, ins, kbd, q, samp,\nsmall, strong, sub, sup, var, b, i, dl, dt, dd, ol, ul, li, a,\nfieldset, form, label, legend, table, caption, tbody, tfoot, thead, tr, th, td,\narticle, aside, canvas, details, figcaption, figure,\nfooter, header, hgroup, menu, nav, section, summary, time, mark, audio, video {\n  display: block;\n  margin:0;\n  padding:0;\n  border:0;\n  outline:0;\n  line-height: 140%;\n  font-size: 100%;\n  vertical-align:baseline;\n  text-decoration: none;\n  background:transparent;\n}\n\na, code, span, b, i, em { display: inline; }",

"stylesheet":"#content {\n  font-family: \"HelveticaNeue-Light\", \"Helvetica Neue Light\", \"Helvetica Neue\", Helvetica, Arial, \"Lucida Grande\", sans-serif; \n}\n\n#content #title {\n  clear: both;\n  text-align: center;\n  font-size: 140%;\n  font-weight: bold;\n  margin: 2em 0 0 0;\n}\n\n#content #tag {\n  clear: both;\n  text-align: center;\n  font-size: 110%;\n  margin: 0 0 1em 0;\n}\n#content #toc {\n  padding: 0 0 0 1em;\n  max-width: 96%;\n  page-break-after: always;\n}\n\n@media screen and (min-width: 641px) {\n  #content #toc {\n    float: left;\n    max-width: 26%;\n  }\n}\n\n#content .tocsn { float: left; }\n#content .tocsr { overflow: hidden; }\n\n#content #toc ol {\n  list-style-type: none;\n}\n\n#content #toc ol ol {\n  margin: 0 0 0 1em;\n}\n#content #doc,\n#content #toc {\n  margin: 0 1em 0 1em;\n}\n\n#content p { text-indent: 1em; }\n\n#content code { font-size: 120%; }\n\n#content pre code {\n  font-family: \"Courier New\", Courier, monospace;\n  font-size: 100%;\n}\n\n@media screen and (min-width: 641px) {\n  #content #doc {\n    overflow: hidden;\n    max-width: 70%;\n  }\n}\n#content pre .field {\n  display:block;\n  margin: 1em 1em 0em 0em;\n}\n\n#content pre .field::after {\n  content: \" :=\";\n}\n#content pre code {\n  display: block;\n  margin: 1em;\n  padding: 1em;\n  border: solid 1px #aaa;\n}\n\n@media screen {\n  #content pre code {\n    overflow: auto;\n    max-height: 30em;\n  }\n}\n\n@media print {\n  #content pre {\n    page-break-inside: avoid;\n  }\n  #content pre code {\n    overflow: visible;\n    word-wrap: break-word;\n  }\n}\n@media screen {\n  #content .lang-js { background-color: ivory; }\n  #content .lang-md { background-color: lightyellow; }\n  #content .lang-html { background-color: floralwhite; }\n  #content .lang-json { background-color: honeydew; }\n  #content .lang-css { background-color: cornsilk; }\n  #content .lang-sh { background-color: lemonchiffon; }\n}\n#content h1,\n#content h2,\n#content h3,\n#content h4,\n#content h5,\n#content h6 {\n  margin: 0.5em 0em;\n  page-break-after: avoid\n}\n\n#content h1 { font-size: 140%; }\n#content h2 { font-size: 130%; }\n#content h3 { font-size: 120%; }\n#content h4,\n#content h5,\n#content h6 { font-size: 110%; }\n#content a:hover { color: #aaa; }\n\n#content a:visited,\n#content a { color: #000;}\n\n@media screen {\n  #content a {\n    text-decoration: underline;\n  }\n}\n@media print {\n  #content a {\n    text-decoration: none;\n  }\n  #content #doc a[href]:after {\n    content: \" (\" attr(href) \")\";\n    font-size: 90%;\n  }\n}\n.hljs {\n    overflow: auto;\n    padding: 0.5em;\n    color: #333;\n}\n.hljs-comment, .diff .hljs-header, .hljs-javadoc {\n    color: #999;\n    font-style: italic;\n}\n.hljs-keyword, .css .rule .hljs-keyword, .hljs-winutils, .nginx .hljs-title,\n.hljs-subst, .hljs-request, .hljs-status {\n    color: #333;\n    font-weight: bold;\n}\n.hljs-number, .hljs-hexcolor, .ruby .hljs-constant {\n    color: #088;\n}\n.hljs-string, .hljs-tag .hljs-value, .hljs-phpdoc, .hljs-dartdoc, .tex .hljs-formula {\n    color: #d14;\n}\n.hljs-title, .hljs-id, .scss .hljs-preprocessor {\n    color: #900;\n    font-weight: bold;\n}\n.hljs-list .hljs-keyword, .hljs-subst {\n    font-weight: normal;\n}\n.hljs-class .hljs-title, .hljs-type, .vhdl .hljs-literal, .tex .hljs-command {\n    color: #458;\n    font-weight: bold;\n}\n.hljs-tag, .hljs-tag .hljs-title, .hljs-rules .hljs-property, .django .hljs-tag .hljs-keyword {\n    color: #000080;\n    font-weight: normal;\n}\n.hljs-attribute, .hljs-variable, .lisp .hljs-body {\n    color: #008080;\n}\n.hljs-regexp {\n    color: #009926;\n}\n.hljs-built_in {\n    color: #0086b3;\n}\n\n.hljs-symbol, .ruby .hljs-symbol .hljs-string, .lisp .hljs-keyword,\n.clojure .hljs-keyword, .scheme .hljs-keyword, .tex .hljs-special,\n.hljs-prompt {\n    color: #990073;\n}\n.hljs-preprocessor, .hljs-pragma, .hljs-pi, .hljs-doctype, .hljs-shebang,\n.hljs-cdata {\n    color: #999;\n    font-weight: bold;\n}\n.hljs-deletion {\n    background: #fdd;\n}\n.hljs-addition {\n    background: #dfd;\n}\n.diff .hljs-change {\n    background: #0086b3;\n}\n.hljs-chunk {\n    color: #aaa;\n}\n@media print {\n  .hljs {\n    overflow: visible;\n    word-wrap: break-word;\n  }\n  .hljs-number, .hljs-hexcolor, .ruby .hljs-constant {\n    color: #888;\n  }\n  .hljs-string, .hljs-tag .hljs-value, .hljs-phpdoc, .hljs-dartdoc, .tex .hljs-formula {\n    color: #ddd;\n  }\n  .hljs-title, .hljs-id, .scss .hljs-preprocessor {\n    color: #999;\n  }\n  .hljs-class .hljs-title, .hljs-type, .vhdl .hljs-literal, .tex .hljs-command {\n    color: #888;\n  }\n  .hljs-tag, .hljs-tag .hljs-title, .hljs-rules .hljs-property, .django .hljs-tag .hljs-keyword {\n    color: #888;\n  }\n  .hljs-attribute, .hljs-variable, .lisp .hljs-body {\n    color: #888;\n  }\n  .hljs-regexp {\n    color: #999;\n  }\n  .hljs-symbol, .ruby .hljs-symbol .hljs-string, .lisp .hljs-keyword,\n  .clojure .hljs-keyword, .scheme .hljs-keyword, .tex .hljs-special,\n  .hljs-prompt {\n    color: #999;\n  }\n  .hljs-built_in {\n    color: #888;\n  }\n  .hljs-deletion {\n    background: #ddd;\n  }\n  .hljs-addition {\n    background: #ddd;\n  }\n  .diff .hljs-change {\n    background: #888;\n  }\n}",

"pool":function(options) {
  options = options || {}

  var dam = new require('stream').Transform()
  dam.options = options
  dam.options.convert = dam.options.convert || function(string, options) { return string }
  dam.options.self = dam.options.self || this

  dam.lake = []
  dam._transform = function(chunk, encoding, done) {
    dam.lake.push(chunk.toString())
    done()
  }

  dam._flush = function(done) {
    dam.push(dam.options.convert.call(dam.options.self, dam.lake.join(''), dam.options))
    dam.push('\n')
    dam.lake = []
    done()
  }

  return dam
},

"escape":function (html, encode) {
  return html
    .replace(!encode ? /&(?!#?\w+;)/g : /&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#39;');
},

"unescape":function(html) {
  return html.replace(/&([#\w]+);/g, function(_, n) {
    n = n.toLowerCase();
    if (n === 'colon') return ':';
    if (n === 'lt') return '<';
    if (n === 'gt') return '>';
    if (n === 'quot') return '"';
    if (n.charAt(0) === '#') {
      return n.charAt(1) === 'x'
        ? String.fromCharCode(parseInt(n.substring(2), 16))
        : String.fromCharCode(+n.substring(1));
    }
    return '';
  })
},

"tangleFormatPrefix":" {%n%n",

"tangleFormatSuffix":"%n}%n%n",

"tangleFormatDelimiter":",%n%n",

"tangleFormatFieldPrefix":"\"",

"tangleFormatFieldSuffix":"\":",

"tangleFormat":function(ff, fields, lf) {
  var res = [ff, this.tangleFormatPrefix]
  var firsttime = true
  var v
  for (var e in fields) {
    v = fields[e]
    if (firsttime) {
      firsttime = false
    } else {
      res.push(this.tangleFormatDelimiter)
    }
    res.push(this.tangleFormatFieldPrefix + e + this.tangleFormatFieldSuffix)
    res.push(v)
  }
   res.push(this.tangleFormatSuffix)
   res.push(lf)

   return res.join('')
     .replace(/%q/gm, '\"')
     .replace(/%n/gm, '\n')
     .replace(/%t/gm, '\t')
     .replace(/%s/gm, '\\')
     .replace(/%p/gm , '%')
},

"tangleJSON":function (str, options) {
  options = options || {}
  options.exclude = options.exclude || []
  var marked = require('marked')
  var p = new marked.Renderer()
  marked.setOptions({
    renderer: p,
    gfm: true,
    tables: true,
    breaks: false,
    pedantic: false,
    sanitize: true,
    smartLists: true,
    smartypants: false
  })
  p.code = function(code, lang, escaped) {

    if (!lang)
      return ''

    var field = lang.replace(/\S*#[+]?(\S*)/, "$1")
    if (lang == field)
      return ''

    var quote = lang.replace(/\S*#([+]?)\S*/, "$1")
    lang = lang.replace(/(\S*)#[+]?\S*/, "$1")
    if (lang == '') lang = 'json'

    if (quote == '+') {
      code = ('"' +
        code
          .replace(/\\/gm, '\\\\')
          .replace(/\"/gm /*"*/, '\\"')
          .replace(/\n/gm , '\\n')
          .replace(/\t/gm , '\\t')
        + '"')
    }

    var esccode = code
      .replace(/%/gm , '%p')
      .replace(/"/gm /*"*/, '%q')
      .replace(/\n/gm, '%n')
      .replace(/\t/gm, '%t')
      .replace(/\\/gm, '%s')

    return ',["' + field + '", "' + (quote == '+')  + '", "' + esccode + '"]'
  }
  function noop() { return ''}

  p.blockquote = noop
  p.html = noop
  p.heading = noop
  p.hr = noop
  p.list = noop
  p.listitem = noop
  p.paragraph = noop
  p.table = noop
  p.tablerow = noop
  p.tablecell = noop
  p.strong = noop
  p.em = noop
  p.codespan = noop
  p.br = noop
  p.del = noop
  p.link = noop
  p.image = noop
  var doc = '[' + marked(str).substring(1) + ']'
  var obj = JSON.parse(doc)
  var fields = {}
  var ff = ''
  var lf = ''
  var res = obj.forEach(function(e) {
    var k = e[0]
    var q = (e[1] == "true")
    var v = e[2]
    var doit = true
    if (options.exclude.indexOf(k) != -1)
      doit = false
    if (doit) {
      if (fields[k] != undefined) {
        if (q) {
          v = fields[k].substring(0,fields[k].length-2) + '\\n' + v.substring(2)
        } else {
          v = fields[k] + '%n' + v
        }
      }
      fields[k] = v
    }
  })
  if (fields["first"] != undefined) {
    ff = fields["first"]
    delete fields["first"]
  }

  if (fields["last"] != undefined) {
    lf = fields["last"]
    delete fields["last"]
  }

  if (options.sourcetarget) {
    fields[options.sourcetarget] = 
        ('"' +
          str
          .replace(/\\/gm, '\\\\')
          .replace(/\"/gm /*"*/, '\\"')
          .replace(/\n/gm , '\\n')
          .replace(/\t/gm , '\\t')
          .replace(/%/gm , '%p')
        + '"')
  }

  return this.tangleFormat(ff, fields, lf)
},

"tangleStream":function (opts) {
  var options = {convert: this.tangleJSON, self:this}
  for(var i in opts)
    options[i] = opts[i]
  var p = this.pool(options)

  if (!p.options.notstdio) {
    process.stdin.pipe(p).pipe(process.stdout)
    return true
  }

  return p
},

"heading":function(text, level, raw, toc, prefix) {
  var i = raw.toLowerCase().replace(/[^\w]+/g, '-')
  var n = [0, 0, 0, 0, 0, 0, 0]
  if (level > 6) level = 6
  else if (level < 1) level = 1
  toc.push([text, level, i])
  var l
  for (var x = 0; x < toc.length; x++) {
    l = toc[x][1]
    n[l] = n[l] + 1
    while (++l < toc.length)
      n[l] = 0
  }
  var sn = n.slice(1, level+1).join('.') + (level == 1 ? '.' : '')
  toc[toc.length-1].push(sn)
  return '<h'
    + level
    + ' id="'
    + prefix
    + i
    + '">'
    + sn + ' ' + text
    + '</h'
    + level
    + '>\n'
},

"weaveHighlight":function (lang, code) {
  var hljs = require('highlight.js')
  code = hljs.highlight(lang, this.unescape(code), true)
  return code
},

"weaveCode":function(code, lang, escaped, fields, prefix) {
  var c = code
  if (!lang) {
    return '<pre><code>'
      + this.escape(c)
      + '\n</code></pre>'
  }
  var i = lang.indexOf('#')
  if (i == -1) {
    if (lang == "noweave") return ''
    return '<pre><code class="'
      + prefix
      + this.escape(lang, true)
      + '">'
      + this.weaveHighlight(lang, c).value
      + '\n</code></pre>\n'
  }
  var field = lang.substring(i+1)
  lang = lang.substring(0, i)
  if (lang == "") lang = "json"

  if (field.substring(0,1) == '+') {
    field = field.substring(1)
  }

  if (fields[field] == undefined) {
    fields[field] = c
  } else {
    fields[field] = fields[field] + '\n' + c
  }

  if (lang == "noweave") return ''

  return '<pre><div class="field">'
    + field + '</div>'
    + '<code class="'
    + prefix
    + this.escape(lang, true)
    + '">'
    + this.weaveHighlight((lang != "" ? lang : "json"), c).value
    + '\n</code></pre>\n'
},

"weaveMarked":function () {
  var marked = require('marked')
  var p = new marked.Renderer()

  marked.setOptions({
    renderer: p,
    gfm: true,
    tables: true,
    breaks: false,
    pedantic: false,
    sanitize: true,
    smartLists: true,
    smartypants: false
  })
  p.field = []
  p.toc = []
  p.self = this
  p.heading = function (text, level, raw) {
    return this.self.heading.call(this.self, text, level, raw, p.toc, this.options.headerPrefix)
  }
  p.code = function (code, lang, escaped) {
    return this.self.weaveCode.call(this.self, code, lang, escaped, p.field, this.options.langPrefix)
  }

  return [marked, p]
},

"weaveJSON":function (str, options) {
  options = options || {}
  var m = this.weaveMarked()
  var marked = m[0]
  var p = m[1]
  var doc = marked.call(this, str)
  function f(l,d) { return this.unescape(p.field[l] || d) }
  var title = f("title", "Weave Output")
  var tag = f("tag", "from weaving")
  var headstylesheet = f("headstylesheet", "")
  var stylesheet = f("stylesheet", "")

  var headline = f("head", "<!DOCTYPE html><html><head>")
  var transitionline = f("transition", "</head><body>")
  var tailline = f("tail", "</body></html>") + '\n'

  var headtitle = '<title>' + title + ' - ' + tag + '</title>\n'
  var titleline = '<div id="title">' + title + '</div>'
  var tagline = '<div id="tag">' + tag + '</div>'

  var headstyleline = headstylesheet == '' ? '' : '<style>' + headstylesheet + '</style>'
  var styleline = stylesheet == '' ? '' : '<style>' + stylesheet + '</style>'
  var tocitems = p.toc.map(function(e, i, a) {
    var ll = 0
    var o = ''
    if (i > 0) ll = a[i-1][1]
    if (ll < e[1]) {
      while (ll++ < e[1])
        o = o + '<ol>'
    } else if (ll > e[1]) {
      while (ll-- > e[1])
        o = o + '</ol>'
    }
    return (
            o + '<li><div class="tocsn">' + e[3] + '&nbsp;</div><div class="tocsr"><a href="#' + e[2] + '">'
          + e[0] + '</a></div>'
           )
  })
  var toc = ''
  var ll
  var o

  if (p.toc.length > 0) {
    ll = p.toc[p.toc.length-1][1]
    o = ''
    while (ll-- > 0)
      o = o + '</ol>'
    toc = '<div id="toc"><h1>Contents</h1>\n' + tocitems.join('\n') + o + '</div>\n'
  }
  var heading = ''
  var tailing = ''
  if (options.contentonly != true) {
    heading = headline + headtitle + headstyleline + styleline + transitionline
    tailing = tailline
  } else
    heading = styleline

  return (heading +
          '<div id="content">' +
             titleline + tagline + toc +
          '<div id="doc">' + doc + '</div>' +
          '</div>' + tailing)
},

"weaveStream":function (opts) {
  var options = {convert: this.weaveJSON, self:this}
  for(var i in opts)
    options[i] = opts[i]
  var p = this.pool(options)

  if (!p.options.notstdio) {
    process.stdin.pipe(p).pipe(process.stdout)
    return true
  }

  return p
},

"untangleStream":function (opts) {
  opts = opts || {}
  var options
  options = {
    convert: function (str, options) {
      return '```\n' + str + '```\n'
    },
    self: this
  }

  for(var i in opts)
    options[i] = opts[i]

  var p = this.pool(options)

  if (!options.notstdio) {
    process.stdin.pipe(p).pipe(process.stdout)
    return true
  }

  return p
}
}

if (typeof module !== 'undefined' && typeof exports === 'object') {
    module.exports = markover
} else if (typeof define === 'function' && define.amd) {
    define(function() { return markover })
} else {
    this.markover = markover
}

}).call(function () {
        return this || (typeof window !== 'undefined' ? window : global)
})
```

`markover` depends on `marked` for parsing `markdown` documents.

```noweave#+marked
/**
 * marked - a markdown parser
 * Copyright (c) 2011-2014, Christopher Jeffrey. (MIT Licensed)
 * https://github.com/chjj/marked
 */

;(function() {

/**
 * Block-Level Grammar
 */

var block = {
  newline: /^\n+/,
  code: /^( {4}[^\n]+\n*)+/,
  fences: noop,
  hr: /^( *[-*_]){3,} *(?:\n+|$)/,
  heading: /^ *(#{1,6}) *([^\n]+?) *#* *(?:\n+|$)/,
  nptable: noop,
  lheading: /^([^\n]+)\n *(=|-){2,} *(?:\n+|$)/,
  blockquote: /^( *>[^\n]+(\n(?!def)[^\n]+)*\n*)+/,
  list: /^( *)(bull) [\s\S]+?(?:hr|def|\n{2,}(?! )(?!\1bull )\n*|\s*$)/,
  html: /^ *(?:comment|closed|closing) *(?:\n{2,}|\s*$)/,
  def: /^ *\[([^\]]+)\]: *<?([^\s>]+)>?(?: +["(]([^\n]+)[")])? *(?:\n+|$)/,
  table: noop,
  paragraph: /^((?:[^\n]+\n?(?!hr|heading|lheading|blockquote|tag|def))+)\n*/,
  text: /^[^\n]+/
};

block.bullet = /(?:[*+-]|\d+\.)/;
block.item = /^( *)(bull) [^\n]*(?:\n(?!\1bull )[^\n]*)*/;
block.item = replace(block.item, 'gm')
  (/bull/g, block.bullet)
  ();

block.list = replace(block.list)
  (/bull/g, block.bullet)
  ('hr', '\\n+(?=\\1?(?:[-*_] *){3,}(?:\\n+|$))')
  ('def', '\\n+(?=' + block.def.source + ')')
  ();

block.blockquote = replace(block.blockquote)
  ('def', block.def)
  ();

block._tag = '(?!(?:'
  + 'a|em|strong|small|s|cite|q|dfn|abbr|data|time|code'
  + '|var|samp|kbd|sub|sup|i|b|u|mark|ruby|rt|rp|bdi|bdo'
  + '|span|br|wbr|ins|del|img)\\b)\\w+(?!:/|[^\\w\\s@]*@)\\b';

block.html = replace(block.html)
  ('comment', /<!--[\s\S]*?-->/)
  ('closed', /<(tag)[\s\S]+?<\/\1>/)
  ('closing', /<tag(?:"[^"]*"|'[^']*'|[^'">])*?>/)
  (/tag/g, block._tag)
  ();

block.paragraph = replace(block.paragraph)
  ('hr', block.hr)
  ('heading', block.heading)
  ('lheading', block.lheading)
  ('blockquote', block.blockquote)
  ('tag', '<' + block._tag)
  ('def', block.def)
  ();

/**
 * Normal Block Grammar
 */

block.normal = merge({}, block);

/**
 * GFM Block Grammar
 */

block.gfm = merge({}, block.normal, {
  fences: /^ *(`{3,}|~{3,}) *(\S+)? *\n([\s\S]+?)\s*\1 *(?:\n+|$)/,
  paragraph: /^/
});

block.gfm.paragraph = replace(block.paragraph)
  ('(?!', '(?!'
    + block.gfm.fences.source.replace('\\1', '\\2') + '|'
    + block.list.source.replace('\\1', '\\3') + '|')
  ();

/**
 * GFM + Tables Block Grammar
 */

block.tables = merge({}, block.gfm, {
  nptable: /^ *(\S.*\|.*)\n *([-:]+ *\|[-| :]*)\n((?:.*\|.*(?:\n|$))*)\n*/,
  table: /^ *\|(.+)\n *\|( *[-:]+[-| :]*)\n((?: *\|.*(?:\n|$))*)\n*/
});

/**
 * Block Lexer
 */

function Lexer(options) {
  this.tokens = [];
  this.tokens.links = {};
  this.options = options || marked.defaults;
  this.rules = block.normal;

  if (this.options.gfm) {
    if (this.options.tables) {
      this.rules = block.tables;
    } else {
      this.rules = block.gfm;
    }
  }
}

/**
 * Expose Block Rules
 */

Lexer.rules = block;

/**
 * Static Lex Method
 */

Lexer.lex = function(src, options) {
  var lexer = new Lexer(options);
  return lexer.lex(src);
};

/**
 * Preprocessing
 */

Lexer.prototype.lex = function(src) {
  src = src
    .replace(/\r\n|\r/g, '\n')
    .replace(/\t/g, '    ')
    .replace(/\u00a0/g, ' ')
    .replace(/\u2424/g, '\n');

  return this.token(src, true);
};

/**
 * Lexing
 */

Lexer.prototype.token = function(src, top, bq) {
  var src = src.replace(/^ +$/gm, '')
    , next
    , loose
    , cap
    , bull
    , b
    , item
    , space
    , i
    , l;

  while (src) {
    // newline
    if (cap = this.rules.newline.exec(src)) {
      src = src.substring(cap[0].length);
      if (cap[0].length > 1) {
        this.tokens.push({
          type: 'space'
        });
      }
    }

    // code
    if (cap = this.rules.code.exec(src)) {
      src = src.substring(cap[0].length);
      cap = cap[0].replace(/^ {4}/gm, '');
      this.tokens.push({
        type: 'code',
        text: !this.options.pedantic
          ? cap.replace(/\n+$/, '')
          : cap
      });
      continue;
    }

    // fences (gfm)
    if (cap = this.rules.fences.exec(src)) {
      src = src.substring(cap[0].length);
      this.tokens.push({
        type: 'code',
        lang: cap[2],
        text: cap[3]
      });
      continue;
    }

    // heading
    if (cap = this.rules.heading.exec(src)) {
      src = src.substring(cap[0].length);
      this.tokens.push({
        type: 'heading',
        depth: cap[1].length,
        text: cap[2]
      });
      continue;
    }

    // table no leading pipe (gfm)
    if (top && (cap = this.rules.nptable.exec(src))) {
      src = src.substring(cap[0].length);

      item = {
        type: 'table',
        header: cap[1].replace(/^ *| *\| *$/g, '').split(/ *\| */),
        align: cap[2].replace(/^ *|\| *$/g, '').split(/ *\| */),
        cells: cap[3].replace(/\n$/, '').split('\n')
      };

      for (i = 0; i < item.align.length; i++) {
        if (/^ *-+: *$/.test(item.align[i])) {
          item.align[i] = 'right';
        } else if (/^ *:-+: *$/.test(item.align[i])) {
          item.align[i] = 'center';
        } else if (/^ *:-+ *$/.test(item.align[i])) {
          item.align[i] = 'left';
        } else {
          item.align[i] = null;
        }
      }

      for (i = 0; i < item.cells.length; i++) {
        item.cells[i] = item.cells[i].split(/ *\| */);
      }

      this.tokens.push(item);

      continue;
    }

    // lheading
    if (cap = this.rules.lheading.exec(src)) {
      src = src.substring(cap[0].length);
      this.tokens.push({
        type: 'heading',
        depth: cap[2] === '=' ? 1 : 2,
        text: cap[1]
      });
      continue;
    }

    // hr
    if (cap = this.rules.hr.exec(src)) {
      src = src.substring(cap[0].length);
      this.tokens.push({
        type: 'hr'
      });
      continue;
    }

    // blockquote
    if (cap = this.rules.blockquote.exec(src)) {
      src = src.substring(cap[0].length);

      this.tokens.push({
        type: 'blockquote_start'
      });

      cap = cap[0].replace(/^ *> ?/gm, '');

      // Pass `top` to keep the current
      // "toplevel" state. This is exactly
      // how markdown.pl works.
      this.token(cap, top, true);

      this.tokens.push({
        type: 'blockquote_end'
      });

      continue;
    }

    // list
    if (cap = this.rules.list.exec(src)) {
      src = src.substring(cap[0].length);
      bull = cap[2];

      this.tokens.push({
        type: 'list_start',
        ordered: bull.length > 1
      });

      // Get each top-level item.
      cap = cap[0].match(this.rules.item);

      next = false;
      l = cap.length;
      i = 0;

      for (; i < l; i++) {
        item = cap[i];

        // Remove the list item's bullet
        // so it is seen as the next token.
        space = item.length;
        item = item.replace(/^ *([*+-]|\d+\.) +/, '');

        // Outdent whatever the
        // list item contains. Hacky.
        if (~item.indexOf('\n ')) {
          space -= item.length;
          item = !this.options.pedantic
            ? item.replace(new RegExp('^ {1,' + space + '}', 'gm'), '')
            : item.replace(/^ {1,4}/gm, '');
        }

        // Determine whether the next list item belongs here.
        // Backpedal if it does not belong in this list.
        if (this.options.smartLists && i !== l - 1) {
          b = block.bullet.exec(cap[i + 1])[0];
          if (bull !== b && !(bull.length > 1 && b.length > 1)) {
            src = cap.slice(i + 1).join('\n') + src;
            i = l - 1;
          }
        }

        // Determine whether item is loose or not.
        // Use: /(^|\n)(?! )[^\n]+\n\n(?!\s*$)/
        // for discount behavior.
        loose = next || /\n\n(?!\s*$)/.test(item);
        if (i !== l - 1) {
          next = item.charAt(item.length - 1) === '\n';
          if (!loose) loose = next;
        }

        this.tokens.push({
          type: loose
            ? 'loose_item_start'
            : 'list_item_start'
        });

        // Recurse.
        this.token(item, false, bq);

        this.tokens.push({
          type: 'list_item_end'
        });
      }

      this.tokens.push({
        type: 'list_end'
      });

      continue;
    }

    // html
    if (cap = this.rules.html.exec(src)) {
      src = src.substring(cap[0].length);
      this.tokens.push({
        type: this.options.sanitize
          ? 'paragraph'
          : 'html',
        pre: cap[1] === 'pre' || cap[1] === 'script' || cap[1] === 'style',
        text: cap[0]
      });
      continue;
    }

    // def
    if ((!bq && top) && (cap = this.rules.def.exec(src))) {
      src = src.substring(cap[0].length);
      this.tokens.links[cap[1].toLowerCase()] = {
        href: cap[2],
        title: cap[3]
      };
      continue;
    }

    // table (gfm)
    if (top && (cap = this.rules.table.exec(src))) {
      src = src.substring(cap[0].length);

      item = {
        type: 'table',
        header: cap[1].replace(/^ *| *\| *$/g, '').split(/ *\| */),
        align: cap[2].replace(/^ *|\| *$/g, '').split(/ *\| */),
        cells: cap[3].replace(/(?: *\| *)?\n$/, '').split('\n')
      };

      for (i = 0; i < item.align.length; i++) {
        if (/^ *-+: *$/.test(item.align[i])) {
          item.align[i] = 'right';
        } else if (/^ *:-+: *$/.test(item.align[i])) {
          item.align[i] = 'center';
        } else if (/^ *:-+ *$/.test(item.align[i])) {
          item.align[i] = 'left';
        } else {
          item.align[i] = null;
        }
      }

      for (i = 0; i < item.cells.length; i++) {
        item.cells[i] = item.cells[i]
          .replace(/^ *\| *| *\| *$/g, '')
          .split(/ *\| */);
      }

      this.tokens.push(item);

      continue;
    }

    // top-level paragraph
    if (top && (cap = this.rules.paragraph.exec(src))) {
      src = src.substring(cap[0].length);
      this.tokens.push({
        type: 'paragraph',
        text: cap[1].charAt(cap[1].length - 1) === '\n'
          ? cap[1].slice(0, -1)
          : cap[1]
      });
      continue;
    }

    // text
    if (cap = this.rules.text.exec(src)) {
      // Top-level should never reach here.
      src = src.substring(cap[0].length);
      this.tokens.push({
        type: 'text',
        text: cap[0]
      });
      continue;
    }

    if (src) {
      throw new
        Error('Infinite loop on byte: ' + src.charCodeAt(0));
    }
  }

  return this.tokens;
};

/**
 * Inline-Level Grammar
 */

var inline = {
  escape: /^\\([\\`*{}\[\]()#+\-.!_>])/,
  autolink: /^<([^ >]+(@|:\/)[^ >]+)>/,
  url: noop,
  tag: /^<!--[\s\S]*?-->|^<\/?\w+(?:"[^"]*"|'[^']*'|[^'">])*?>/,
  link: /^!?\[(inside)\]\(href\)/,
  reflink: /^!?\[(inside)\]\s*\[([^\]]*)\]/,
  nolink: /^!?\[((?:\[[^\]]*\]|[^\[\]])*)\]/,
  strong: /^__([\s\S]+?)__(?!_)|^\*\*([\s\S]+?)\*\*(?!\*)/,
  em: /^\b_((?:__|[\s\S])+?)_\b|^\*((?:\*\*|[\s\S])+?)\*(?!\*)/,
  code: /^(`+)\s*([\s\S]*?[^`])\s*\1(?!`)/,
  br: /^ {2,}\n(?!\s*$)/,
  del: noop,
  text: /^[\s\S]+?(?=[\\<!\[_*`]| {2,}\n|$)/
};

inline._inside = /(?:\[[^\]]*\]|[^\[\]]|\](?=[^\[]*\]))*/;
inline._href = /\s*<?([\s\S]*?)>?(?:\s+['"]([\s\S]*?)['"])?\s*/;

inline.link = replace(inline.link)
  ('inside', inline._inside)
  ('href', inline._href)
  ();

inline.reflink = replace(inline.reflink)
  ('inside', inline._inside)
  ();

/**
 * Normal Inline Grammar
 */

inline.normal = merge({}, inline);

/**
 * Pedantic Inline Grammar
 */

inline.pedantic = merge({}, inline.normal, {
  strong: /^__(?=\S)([\s\S]*?\S)__(?!_)|^\*\*(?=\S)([\s\S]*?\S)\*\*(?!\*)/,
  em: /^_(?=\S)([\s\S]*?\S)_(?!_)|^\*(?=\S)([\s\S]*?\S)\*(?!\*)/
});

/**
 * GFM Inline Grammar
 */

inline.gfm = merge({}, inline.normal, {
  escape: replace(inline.escape)('])', '~|])')(),
  url: /^(https?:\/\/[^\s<]+[^<.,:;"')\]\s])/,
  del: /^~~(?=\S)([\s\S]*?\S)~~/,
  text: replace(inline.text)
    (']|', '~]|')
    ('|', '|https?://|')
    ()
});

/**
 * GFM + Line Breaks Inline Grammar
 */

inline.breaks = merge({}, inline.gfm, {
  br: replace(inline.br)('{2,}', '*')(),
  text: replace(inline.gfm.text)('{2,}', '*')()
});

/**
 * Inline Lexer & Compiler
 */

function InlineLexer(links, options) {
  this.options = options || marked.defaults;
  this.links = links;
  this.rules = inline.normal;
  this.renderer = this.options.renderer || new Renderer;
  this.renderer.options = this.options;

  if (!this.links) {
    throw new
      Error('Tokens array requires a `links` property.');
  }

  if (this.options.gfm) {
    if (this.options.breaks) {
      this.rules = inline.breaks;
    } else {
      this.rules = inline.gfm;
    }
  } else if (this.options.pedantic) {
    this.rules = inline.pedantic;
  }
}

/**
 * Expose Inline Rules
 */

InlineLexer.rules = inline;

/**
 * Static Lexing/Compiling Method
 */

InlineLexer.output = function(src, links, options) {
  var inline = new InlineLexer(links, options);
  return inline.output(src);
};

/**
 * Lexing/Compiling
 */

InlineLexer.prototype.output = function(src) {
  var out = ''
    , link
    , text
    , href
    , cap;

  while (src) {
    // escape
    if (cap = this.rules.escape.exec(src)) {
      src = src.substring(cap[0].length);
      out += cap[1];
      continue;
    }

    // autolink
    if (cap = this.rules.autolink.exec(src)) {
      src = src.substring(cap[0].length);
      if (cap[2] === '@') {
        text = cap[1].charAt(6) === ':'
          ? this.mangle(cap[1].substring(7))
          : this.mangle(cap[1]);
        href = this.mangle('mailto:') + text;
      } else {
        text = escape(cap[1]);
        href = text;
      }
      out += this.renderer.link(href, null, text);
      continue;
    }

    // url (gfm)
    if (!this.inLink && (cap = this.rules.url.exec(src))) {
      src = src.substring(cap[0].length);
      text = escape(cap[1]);
      href = text;
      out += this.renderer.link(href, null, text);
      continue;
    }

    // tag
    if (cap = this.rules.tag.exec(src)) {
      if (!this.inLink && /^<a /i.test(cap[0])) {
        this.inLink = true;
      } else if (this.inLink && /^<\/a>/i.test(cap[0])) {
        this.inLink = false;
      }
      src = src.substring(cap[0].length);
      out += this.options.sanitize
        ? escape(cap[0])
        : cap[0];
      continue;
    }

    // link
    if (cap = this.rules.link.exec(src)) {
      src = src.substring(cap[0].length);
      this.inLink = true;
      out += this.outputLink(cap, {
        href: cap[2],
        title: cap[3]
      });
      this.inLink = false;
      continue;
    }

    // reflink, nolink
    if ((cap = this.rules.reflink.exec(src))
        || (cap = this.rules.nolink.exec(src))) {
      src = src.substring(cap[0].length);
      link = (cap[2] || cap[1]).replace(/\s+/g, ' ');
      link = this.links[link.toLowerCase()];
      if (!link || !link.href) {
        out += cap[0].charAt(0);
        src = cap[0].substring(1) + src;
        continue;
      }
      this.inLink = true;
      out += this.outputLink(cap, link);
      this.inLink = false;
      continue;
    }

    // strong
    if (cap = this.rules.strong.exec(src)) {
      src = src.substring(cap[0].length);
      out += this.renderer.strong(this.output(cap[2] || cap[1]));
      continue;
    }

    // em
    if (cap = this.rules.em.exec(src)) {
      src = src.substring(cap[0].length);
      out += this.renderer.em(this.output(cap[2] || cap[1]));
      continue;
    }

    // code
    if (cap = this.rules.code.exec(src)) {
      src = src.substring(cap[0].length);
      out += this.renderer.codespan(escape(cap[2], true));
      continue;
    }

    // br
    if (cap = this.rules.br.exec(src)) {
      src = src.substring(cap[0].length);
      out += this.renderer.br();
      continue;
    }

    // del (gfm)
    if (cap = this.rules.del.exec(src)) {
      src = src.substring(cap[0].length);
      out += this.renderer.del(this.output(cap[1]));
      continue;
    }

    // text
    if (cap = this.rules.text.exec(src)) {
      src = src.substring(cap[0].length);
      out += escape(this.smartypants(cap[0]));
      continue;
    }

    if (src) {
      throw new
        Error('Infinite loop on byte: ' + src.charCodeAt(0));
    }
  }

  return out;
};

/**
 * Compile Link
 */

InlineLexer.prototype.outputLink = function(cap, link) {
  var href = escape(link.href)
    , title = link.title ? escape(link.title) : null;

  return cap[0].charAt(0) !== '!'
    ? this.renderer.link(href, title, this.output(cap[1]))
    : this.renderer.image(href, title, escape(cap[1]));
};

/**
 * Smartypants Transformations
 */

InlineLexer.prototype.smartypants = function(text) {
  if (!this.options.smartypants) return text;
  return text
    // em-dashes
    .replace(/--/g, '\u2014')
    // opening singles
    .replace(/(^|[-\u2014/(\[{"\s])'/g, '$1\u2018')
    // closing singles & apostrophes
    .replace(/'/g, '\u2019')
    // opening doubles
    .replace(/(^|[-\u2014/(\[{\u2018\s])"/g, '$1\u201c')
    // closing doubles
    .replace(/"/g, '\u201d')
    // ellipses
    .replace(/\.{3}/g, '\u2026');
};

/**
 * Mangle Links
 */

InlineLexer.prototype.mangle = function(text) {
  var out = ''
    , l = text.length
    , i = 0
    , ch;

  for (; i < l; i++) {
    ch = text.charCodeAt(i);
    if (Math.random() > 0.5) {
      ch = 'x' + ch.toString(16);
    }
    out += '&#' + ch + ';';
  }

  return out;
};

/**
 * Renderer
 */

function Renderer(options) {
  this.options = options || {};
}

Renderer.prototype.code = function(code, lang, escaped) {
  if (this.options.highlight) {
    var out = this.options.highlight(code, lang);
    if (out != null && out !== code) {
      escaped = true;
      code = out;
    }
  }

  if (!lang) {
    return '<pre><code>'
      + (escaped ? code : escape(code, true))
      + '\n</code></pre>';
  }

  return '<pre><code class="'
    + this.options.langPrefix
    + escape(lang, true)
    + '">'
    + (escaped ? code : escape(code, true))
    + '\n</code></pre>\n';
};

Renderer.prototype.blockquote = function(quote) {
  return '<blockquote>\n' + quote + '</blockquote>\n';
};

Renderer.prototype.html = function(html) {
  return html;
};

Renderer.prototype.heading = function(text, level, raw) {
  return '<h'
    + level
    + ' id="'
    + this.options.headerPrefix
    + raw.toLowerCase().replace(/[^\w]+/g, '-')
    + '">'
    + text
    + '</h'
    + level
    + '>\n';
};

Renderer.prototype.hr = function() {
  return this.options.xhtml ? '<hr/>\n' : '<hr>\n';
};

Renderer.prototype.list = function(body, ordered) {
  var type = ordered ? 'ol' : 'ul';
  return '<' + type + '>\n' + body + '</' + type + '>\n';
};

Renderer.prototype.listitem = function(text) {
  return '<li>' + text + '</li>\n';
};

Renderer.prototype.paragraph = function(text) {
  return '<p>' + text + '</p>\n';
};

Renderer.prototype.table = function(header, body) {
  return '<table>\n'
    + '<thead>\n'
    + header
    + '</thead>\n'
    + '<tbody>\n'
    + body
    + '</tbody>\n'
    + '</table>\n';
};

Renderer.prototype.tablerow = function(content) {
  return '<tr>\n' + content + '</tr>\n';
};

Renderer.prototype.tablecell = function(content, flags) {
  var type = flags.header ? 'th' : 'td';
  var tag = flags.align
    ? '<' + type + ' style="text-align:' + flags.align + '">'
    : '<' + type + '>';
  return tag + content + '</' + type + '>\n';
};

// span level renderer
Renderer.prototype.strong = function(text) {
  return '<strong>' + text + '</strong>';
};

Renderer.prototype.em = function(text) {
  return '<em>' + text + '</em>';
};

Renderer.prototype.codespan = function(text) {
  return '<code>' + text + '</code>';
};

Renderer.prototype.br = function() {
  return this.options.xhtml ? '<br/>' : '<br>';
};

Renderer.prototype.del = function(text) {
  return '<del>' + text + '</del>';
};

Renderer.prototype.link = function(href, title, text) {
  if (this.options.sanitize) {
    try {
      var prot = decodeURIComponent(unescape(href))
        .replace(/[^\w:]/g, '')
        .toLowerCase();
    } catch (e) {
      return '';
    }
    if (prot.indexOf('javascript:') === 0) {
      return '';
    }
  }
  var out = '<a href="' + href + '"';
  if (title) {
    out += ' title="' + title + '"';
  }
  out += '>' + text + '</a>';
  return out;
};

Renderer.prototype.image = function(href, title, text) {
  var out = '<img src="' + href + '" alt="' + text + '"';
  if (title) {
    out += ' title="' + title + '"';
  }
  out += this.options.xhtml ? '/>' : '>';
  return out;
};

/**
 * Parsing & Compiling
 */

function Parser(options) {
  this.tokens = [];
  this.token = null;
  this.options = options || marked.defaults;
  this.options.renderer = this.options.renderer || new Renderer;
  this.renderer = this.options.renderer;
  this.renderer.options = this.options;
}

/**
 * Static Parse Method
 */

Parser.parse = function(src, options, renderer) {
  var parser = new Parser(options, renderer);
  return parser.parse(src);
};

/**
 * Parse Loop
 */

Parser.prototype.parse = function(src) {
  this.inline = new InlineLexer(src.links, this.options, this.renderer);
  this.tokens = src.reverse();

  var out = '';
  while (this.next()) {
    out += this.tok();
  }

  return out;
};

/**
 * Next Token
 */

Parser.prototype.next = function() {
  return this.token = this.tokens.pop();
};

/**
 * Preview Next Token
 */

Parser.prototype.peek = function() {
  return this.tokens[this.tokens.length - 1] || 0;
};

/**
 * Parse Text Tokens
 */

Parser.prototype.parseText = function() {
  var body = this.token.text;

  while (this.peek().type === 'text') {
    body += '\n' + this.next().text;
  }

  return this.inline.output(body);
};

/**
 * Parse Current Token
 */

Parser.prototype.tok = function() {
  switch (this.token.type) {
    case 'space': {
      return '';
    }
    case 'hr': {
      return this.renderer.hr();
    }
    case 'heading': {
      return this.renderer.heading(
        this.inline.output(this.token.text),
        this.token.depth,
        this.token.text);
    }
    case 'code': {
      return this.renderer.code(this.token.text,
        this.token.lang,
        this.token.escaped);
    }
    case 'table': {
      var header = ''
        , body = ''
        , i
        , row
        , cell
        , flags
        , j;

      // header
      cell = '';
      for (i = 0; i < this.token.header.length; i++) {
        flags = { header: true, align: this.token.align[i] };
        cell += this.renderer.tablecell(
          this.inline.output(this.token.header[i]),
          { header: true, align: this.token.align[i] }
        );
      }
      header += this.renderer.tablerow(cell);

      for (i = 0; i < this.token.cells.length; i++) {
        row = this.token.cells[i];

        cell = '';
        for (j = 0; j < row.length; j++) {
          cell += this.renderer.tablecell(
            this.inline.output(row[j]),
            { header: false, align: this.token.align[j] }
          );
        }

        body += this.renderer.tablerow(cell);
      }
      return this.renderer.table(header, body);
    }
    case 'blockquote_start': {
      var body = '';

      while (this.next().type !== 'blockquote_end') {
        body += this.tok();
      }

      return this.renderer.blockquote(body);
    }
    case 'list_start': {
      var body = ''
        , ordered = this.token.ordered;

      while (this.next().type !== 'list_end') {
        body += this.tok();
      }

      return this.renderer.list(body, ordered);
    }
    case 'list_item_start': {
      var body = '';

      while (this.next().type !== 'list_item_end') {
        body += this.token.type === 'text'
          ? this.parseText()
          : this.tok();
      }

      return this.renderer.listitem(body);
    }
    case 'loose_item_start': {
      var body = '';

      while (this.next().type !== 'list_item_end') {
        body += this.tok();
      }

      return this.renderer.listitem(body);
    }
    case 'html': {
      var html = !this.token.pre && !this.options.pedantic
        ? this.inline.output(this.token.text)
        : this.token.text;
      return this.renderer.html(html);
    }
    case 'paragraph': {
      return this.renderer.paragraph(this.inline.output(this.token.text));
    }
    case 'text': {
      return this.renderer.paragraph(this.parseText());
    }
  }
};

/**
 * Helpers
 */

function escape(html, encode) {
  return html
    .replace(!encode ? /&(?!#?\w+;)/g : /&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#39;');
}

function unescape(html) {
  return html.replace(/&([#\w]+);/g, function(_, n) {
    n = n.toLowerCase();
    if (n === 'colon') return ':';
    if (n.charAt(0) === '#') {
      return n.charAt(1) === 'x'
        ? String.fromCharCode(parseInt(n.substring(2), 16))
        : String.fromCharCode(+n.substring(1));
    }
    return '';
  });
}

function replace(regex, opt) {
  regex = regex.source;
  opt = opt || '';
  return function self(name, val) {
    if (!name) return new RegExp(regex, opt);
    val = val.source || val;
    val = val.replace(/(^|[^\[])\^/g, '$1');
    regex = regex.replace(name, val);
    return self;
  };
}

function noop() {}
noop.exec = noop;

function merge(obj) {
  var i = 1
    , target
    , key;

  for (; i < arguments.length; i++) {
    target = arguments[i];
    for (key in target) {
      if (Object.prototype.hasOwnProperty.call(target, key)) {
        obj[key] = target[key];
      }
    }
  }

  return obj;
}


/**
 * Marked
 */

function marked(src, opt, callback) {
  if (callback || typeof opt === 'function') {
    if (!callback) {
      callback = opt;
      opt = null;
    }

    opt = merge({}, marked.defaults, opt || {});

    var highlight = opt.highlight
      , tokens
      , pending
      , i = 0;

    try {
      tokens = Lexer.lex(src, opt)
    } catch (e) {
      return callback(e);
    }

    pending = tokens.length;

    var done = function() {
      var out, err;

      try {
        out = Parser.parse(tokens, opt);
      } catch (e) {
        err = e;
      }

      opt.highlight = highlight;

      return err
        ? callback(err)
        : callback(null, out);
    };

    if (!highlight || highlight.length < 3) {
      return done();
    }

    delete opt.highlight;

    if (!pending) return done();

    for (; i < tokens.length; i++) {
      (function(token) {
        if (token.type !== 'code') {
          return --pending || done();
        }
        return highlight(token.text, token.lang, function(err, code) {
          if (code == null || code === token.text) {
            return --pending || done();
          }
          token.text = code;
          token.escaped = true;
          --pending || done();
        });
      })(tokens[i]);
    }

    return;
  }
  try {
    if (opt) opt = merge({}, marked.defaults, opt);
    return Parser.parse(Lexer.lex(src, opt), opt);
  } catch (e) {
    e.message += '\nPlease report this to https://github.com/chjj/marked.';
    if ((opt || marked.defaults).silent) {
      return '<p>An error occured:</p><pre>'
        + escape(e.message + '', true)
        + '</pre>';
    }
    throw e;
  }
}

/**
 * Options
 */

marked.options =
marked.setOptions = function(opt) {
  merge(marked.defaults, opt);
  return marked;
};

marked.defaults = {
  gfm: true,
  tables: true,
  breaks: false,
  pedantic: false,
  sanitize: false,
  smartLists: false,
  silent: false,
  highlight: null,
  langPrefix: 'lang-',
  smartypants: false,
  headerPrefix: '',
  renderer: new Renderer,
  xhtml: false
};

/**
 * Expose
 */

marked.Parser = Parser;
marked.parser = Parser.parse;

marked.Renderer = Renderer;

marked.Lexer = Lexer;
marked.lexer = Lexer.lex;

marked.InlineLexer = InlineLexer;
marked.inlineLexer = InlineLexer.output;

marked.parse = marked;

if (typeof exports === 'object') {
  module.exports = marked;
} else if (typeof define === 'function' && define.amd) {
  define(function() { return marked; });
} else {
  this.marked = marked;
}

}).call(function() {
  return this || (typeof window !== 'undefined' ? window : global);
}());
```

`markover` depends on `highlight.js` for source code syntax
highlighting.

My version of `highlight.js` reimplements the 'index.js' file in the
distribution.
It requires 'highlight_highlight.js' and then requires all the languages
that are used in the document and registers them

This seems to leave require.loaded with 2 versions of highlight: the
raw version without any languages and this one which has the languages
included.

```noweave#+highlight.js
(function() {

var hljs = require('highlight_highlight.js')
hljs.registerLanguage('bash',       require('highlight_bash.js'))
hljs.registerLanguage('xml',        require('highlight_xml.js'))
hljs.registerLanguage('javascript', require('highlight_javascript.js'))
hljs.registerLanguage('json',       require('highlight_json.js'))
hljs.registerLanguage('css',        require('highlight_css.js'))
hljs.registerLanguage('markdown',   require('highlight_markdown.js'))

if (typeof exports === 'object') {
  module.exports = hljs
} else if (typeof define === 'function' && define.amd) {
  define(function() { return hljs; });
} else {
  this.hljs = hljs;
}

}).call(function() {
  return this || (typeof window !== 'undefined' ? window : global);
}())
```

```noweave#+highlight_highlight.js
/*
Syntax highlighting with language autodetection.
https://highlightjs.org/
*/

(function(factory) {

  // Setup highlight.js for different environments. First is Node.js or
  // CommonJS.
  if(typeof exports !== 'undefined') {
    factory(exports);
  } else {
    // Export hljs globally even when using AMD for cases when this script
    // is loaded with others that may still expect a global hljs.
    window.hljs = factory({});

    // Finally register the global hljs with AMD.
    if(typeof define === 'function' && define.amd) {
      define([], function() {
        return window.hljs;
      });
    }
  }

}(function(hljs) {

  /* Utility functions */

  function escape(value) {
    return value.replace(/&/gm, '&amp;').replace(/</gm, '&lt;').replace(/>/gm, '&gt;');
  }

  function tag(node) {
    return node.nodeName.toLowerCase();
  }

  function testRe(re, lexeme) {
    var match = re && re.exec(lexeme);
    return match && match.index == 0;
  }

  function blockLanguage(block) {
    var classes = (block.className + ' ' + (block.parentNode ? block.parentNode.className : '')).split(/\s+/);
    classes = classes.map(function(c) {return c.replace(/^lang(uage)?-/, '');});
    return classes.filter(function(c) {return getLanguage(c) || /no(-?)highlight/.test(c);})[0];
  }

  function inherit(parent, obj) {
    var result = {};
    for (var key in parent)
      result[key] = parent[key];
    if (obj)
      for (var key in obj)
        result[key] = obj[key];
    return result;
  };

  /* Stream merging */

  function nodeStream(node) {
    var result = [];
    (function _nodeStream(node, offset) {
      for (var child = node.firstChild; child; child = child.nextSibling) {
        if (child.nodeType == 3)
          offset += child.nodeValue.length;
        else if (child.nodeType == 1) {
          result.push({
            event: 'start',
            offset: offset,
            node: child
          });
          offset = _nodeStream(child, offset);
          // Prevent void elements from having an end tag that would actually
          // double them in the output. There are more void elements in HTML
          // but we list only those realistically expected in code display.
          if (!tag(child).match(/br|hr|img|input/)) {
            result.push({
              event: 'stop',
              offset: offset,
              node: child
            });
          }
        }
      }
      return offset;
    })(node, 0);
    return result;
  }

  function mergeStreams(original, highlighted, value) {
    var processed = 0;
    var result = '';
    var nodeStack = [];

    function selectStream() {
      if (!original.length || !highlighted.length) {
        return original.length ? original : highlighted;
      }
      if (original[0].offset != highlighted[0].offset) {
        return (original[0].offset < highlighted[0].offset) ? original : highlighted;
      }

      /*
      To avoid starting the stream just before it should stop the order is
      ensured that original always starts first and closes last:

      if (event1 == 'start' && event2 == 'start')
        return original;
      if (event1 == 'start' && event2 == 'stop')
        return highlighted;
      if (event1 == 'stop' && event2 == 'start')
        return original;
      if (event1 == 'stop' && event2 == 'stop')
        return highlighted;

      ... which is collapsed to:
      */
      return highlighted[0].event == 'start' ? original : highlighted;
    }

    function open(node) {
      function attr_str(a) {return ' ' + a.nodeName + '="' + escape(a.value) + '"';}
      result += '<' + tag(node) + Array.prototype.map.call(node.attributes, attr_str).join('') + '>';
    }

    function close(node) {
      result += '</' + tag(node) + '>';
    }

    function render(event) {
      (event.event == 'start' ? open : close)(event.node);
    }

    while (original.length || highlighted.length) {
      var stream = selectStream();
      result += escape(value.substr(processed, stream[0].offset - processed));
      processed = stream[0].offset;
      if (stream == original) {
        /*
        On any opening or closing tag of the original markup we first close
        the entire highlighted node stack, then render the original tag along
        with all the following original tags at the same offset and then
        reopen all the tags on the highlighted stack.
        */
        nodeStack.reverse().forEach(close);
        do {
          render(stream.splice(0, 1)[0]);
          stream = selectStream();
        } while (stream == original && stream.length && stream[0].offset == processed);
        nodeStack.reverse().forEach(open);
      } else {
        if (stream[0].event == 'start') {
          nodeStack.push(stream[0].node);
        } else {
          nodeStack.pop();
        }
        render(stream.splice(0, 1)[0]);
      }
    }
    return result + escape(value.substr(processed));
  }

  /* Initialization */

  function compileLanguage(language) {

    function reStr(re) {
        return (re && re.source) || re;
    }

    function langRe(value, global) {
      return RegExp(
        reStr(value),
        'm' + (language.case_insensitive ? 'i' : '') + (global ? 'g' : '')
      );
    }

    function compileMode(mode, parent) {
      if (mode.compiled)
        return;
      mode.compiled = true;

      mode.keywords = mode.keywords || mode.beginKeywords;
      if (mode.keywords) {
        var compiled_keywords = {};

        var flatten = function(className, str) {
          if (language.case_insensitive) {
            str = str.toLowerCase();
          }
          str.split(' ').forEach(function(kw) {
            var pair = kw.split('|');
            compiled_keywords[pair[0]] = [className, pair[1] ? Number(pair[1]) : 1];
          });
        };

        if (typeof mode.keywords == 'string') { // string
          flatten('keyword', mode.keywords);
        } else {
          Object.keys(mode.keywords).forEach(function (className) {
            flatten(className, mode.keywords[className]);
          });
        }
        mode.keywords = compiled_keywords;
      }
      mode.lexemesRe = langRe(mode.lexemes || /\b[A-Za-z0-9_]+\b/, true);

      if (parent) {
        if (mode.beginKeywords) {
          mode.begin = '\\b(' + mode.beginKeywords.split(' ').join('|') + ')\\b';
        }
        if (!mode.begin)
          mode.begin = /\B|\b/;
        mode.beginRe = langRe(mode.begin);
        if (!mode.end && !mode.endsWithParent)
          mode.end = /\B|\b/;
        if (mode.end)
          mode.endRe = langRe(mode.end);
        mode.terminator_end = reStr(mode.end) || '';
        if (mode.endsWithParent && parent.terminator_end)
          mode.terminator_end += (mode.end ? '|' : '') + parent.terminator_end;
      }
      if (mode.illegal)
        mode.illegalRe = langRe(mode.illegal);
      if (mode.relevance === undefined)
        mode.relevance = 1;
      if (!mode.contains) {
        mode.contains = [];
      }
      var expanded_contains = [];
      mode.contains.forEach(function(c) {
        if (c.variants) {
          c.variants.forEach(function(v) {expanded_contains.push(inherit(c, v));});
        } else {
          expanded_contains.push(c == 'self' ? mode : c);
        }
      });
      mode.contains = expanded_contains;
      mode.contains.forEach(function(c) {compileMode(c, mode);});

      if (mode.starts) {
        compileMode(mode.starts, parent);
      }

      var terminators =
        mode.contains.map(function(c) {
          return c.beginKeywords ? '\\.?(' + c.begin + ')\\.?' : c.begin;
        })
        .concat([mode.terminator_end, mode.illegal])
        .map(reStr)
        .filter(Boolean);
      mode.terminators = terminators.length ? langRe(terminators.join('|'), true) : {exec: function(s) {return null;}};
    }

    compileMode(language);
  }

  /*
  Core highlighting function. Accepts a language name, or an alias, and a
  string with the code to highlight. Returns an object with the following
  properties:

  - relevance (int)
  - value (an HTML string with highlighting markup)

  */
  function highlight(name, value, ignore_illegals, continuation) {

    function subMode(lexeme, mode) {
      for (var i = 0; i < mode.contains.length; i++) {
        if (testRe(mode.contains[i].beginRe, lexeme)) {
          return mode.contains[i];
        }
      }
    }

    function endOfMode(mode, lexeme) {
      if (testRe(mode.endRe, lexeme)) {
        return mode;
      }
      if (mode.endsWithParent) {
        return endOfMode(mode.parent, lexeme);
      }
    }

    function isIllegal(lexeme, mode) {
      return !ignore_illegals && testRe(mode.illegalRe, lexeme);
    }

    function keywordMatch(mode, match) {
      var match_str = language.case_insensitive ? match[0].toLowerCase() : match[0];
      return mode.keywords.hasOwnProperty(match_str) && mode.keywords[match_str];
    }

    function buildSpan(classname, insideSpan, leaveOpen, noPrefix) {
      var classPrefix = noPrefix ? '' : options.classPrefix,
          openSpan    = '<span class="' + classPrefix,
          closeSpan   = leaveOpen ? '' : '</span>';

      openSpan += classname + '">';

      return openSpan + insideSpan + closeSpan;
    }

    function processKeywords() {
      if (!top.keywords)
        return escape(mode_buffer);
      var result = '';
      var last_index = 0;
      top.lexemesRe.lastIndex = 0;
      var match = top.lexemesRe.exec(mode_buffer);
      while (match) {
        result += escape(mode_buffer.substr(last_index, match.index - last_index));
        var keyword_match = keywordMatch(top, match);
        if (keyword_match) {
          relevance += keyword_match[1];
          result += buildSpan(keyword_match[0], escape(match[0]));
        } else {
          result += escape(match[0]);
        }
        last_index = top.lexemesRe.lastIndex;
        match = top.lexemesRe.exec(mode_buffer);
      }
      return result + escape(mode_buffer.substr(last_index));
    }

    function processSubLanguage() {
      if (top.subLanguage && !languages[top.subLanguage]) {
        return escape(mode_buffer);
      }
      var result = top.subLanguage ? highlight(top.subLanguage, mode_buffer, true, continuations[top.subLanguage]) : highlightAuto(mode_buffer);
      // Counting embedded language score towards the host language may be disabled
      // with zeroing the containing mode relevance. Usecase in point is Markdown that
      // allows XML everywhere and makes every XML snippet to have a much larger Markdown
      // score.
      if (top.relevance > 0) {
        relevance += result.relevance;
      }
      if (top.subLanguageMode == 'continuous') {
        continuations[top.subLanguage] = result.top;
      }
      return buildSpan(result.language, result.value, false, true);
    }

    function processBuffer() {
      return top.subLanguage !== undefined ? processSubLanguage() : processKeywords();
    }

    function startNewMode(mode, lexeme) {
      var markup = mode.className? buildSpan(mode.className, '', true): '';
      if (mode.returnBegin) {
        result += markup;
        mode_buffer = '';
      } else if (mode.excludeBegin) {
        result += escape(lexeme) + markup;
        mode_buffer = '';
      } else {
        result += markup;
        mode_buffer = lexeme;
      }
      top = Object.create(mode, {parent: {value: top}});
    }

    function processLexeme(buffer, lexeme) {

      mode_buffer += buffer;
      if (lexeme === undefined) {
        result += processBuffer();
        return 0;
      }

      var new_mode = subMode(lexeme, top);
      if (new_mode) {
        result += processBuffer();
        startNewMode(new_mode, lexeme);
        return new_mode.returnBegin ? 0 : lexeme.length;
      }

      var end_mode = endOfMode(top, lexeme);
      if (end_mode) {
        var origin = top;
        if (!(origin.returnEnd || origin.excludeEnd)) {
          mode_buffer += lexeme;
        }
        result += processBuffer();
        do {
          if (top.className) {
            result += '</span>';
          }
          relevance += top.relevance;
          top = top.parent;
        } while (top != end_mode.parent);
        if (origin.excludeEnd) {
          result += escape(lexeme);
        }
        mode_buffer = '';
        if (end_mode.starts) {
          startNewMode(end_mode.starts, '');
        }
        return origin.returnEnd ? 0 : lexeme.length;
      }

      if (isIllegal(lexeme, top))
        throw new Error('Illegal lexeme "' + lexeme + '" for mode "' + (top.className || '<unnamed>') + '"');

      /*
      Parser should not reach this point as all types of lexemes should be caught
      earlier, but if it does due to some bug make sure it advances at least one
      character forward to prevent infinite looping.
      */
      mode_buffer += lexeme;
      return lexeme.length || 1;
    }

    var language = getLanguage(name);
    if (!language) {
      throw new Error('Unknown language: "' + name + '"');
    }

    compileLanguage(language);
    var top = continuation || language;
    var continuations = {}; // keep continuations for sub-languages
    var result = '';
    for(var current = top; current != language; current = current.parent) {
      if (current.className) {
        result = buildSpan(current.className, '', true) + result;
      }
    }
    var mode_buffer = '';
    var relevance = 0;
    try {
      var match, count, index = 0;
      while (true) {
        top.terminators.lastIndex = index;
        match = top.terminators.exec(value);
        if (!match)
          break;
        count = processLexeme(value.substr(index, match.index - index), match[0]);
        index = match.index + count;
      }
      processLexeme(value.substr(index));
      for(var current = top; current.parent; current = current.parent) { // close dangling modes
        if (current.className) {
          result += '</span>';
        }
      };
      return {
        relevance: relevance,
        value: result,
        language: name,
        top: top
      };
    } catch (e) {
      if (e.message.indexOf('Illegal') != -1) {
        return {
          relevance: 0,
          value: escape(value)
        };
      } else {
        throw e;
      }
    }
  }

  /*
  Highlighting with language detection. Accepts a string with the code to
  highlight. Returns an object with the following properties:

  - language (detected language)
  - relevance (int)
  - value (an HTML string with highlighting markup)
  - second_best (object with the same structure for second-best heuristically
    detected language, may be absent)

  */
  function highlightAuto(text, languageSubset) {
    languageSubset = languageSubset || options.languages || Object.keys(languages);
    var result = {
      relevance: 0,
      value: escape(text)
    };
    var second_best = result;
    languageSubset.forEach(function(name) {
      if (!getLanguage(name)) {
        return;
      }
      var current = highlight(name, text, false);
      current.language = name;
      if (current.relevance > second_best.relevance) {
        second_best = current;
      }
      if (current.relevance > result.relevance) {
        second_best = result;
        result = current;
      }
    });
    if (second_best.language) {
      result.second_best = second_best;
    }
    return result;
  }

  /*
  Post-processing of the highlighted markup:

  - replace TABs with something more useful
  - replace real line-breaks with '<br>' for non-pre containers

  */
  function fixMarkup(value) {
    if (options.tabReplace) {
      value = value.replace(/^((<[^>]+>|\t)+)/gm, function(match, p1, offset, s) {
        return p1.replace(/\t/g, options.tabReplace);
      });
    }
    if (options.useBR) {
      value = value.replace(/\n/g, '<br>');
    }
    return value;
  }

  function buildClassName(prevClassName, currentLang, resultLang) {
    var language = currentLang ? aliases[currentLang] : resultLang,
        result   = [prevClassName.trim()];

    if (!prevClassName.match(/(\s|^)hljs(\s|$)/)) {
      result.push('hljs');
    }

    if (language) {
      result.push(language);
    }

    return result.join(' ').trim();
  }

  /*
  Applies highlighting to a DOM node containing code. Accepts a DOM node and
  two optional parameters for fixMarkup.
  */
  function highlightBlock(block) {
    var language = blockLanguage(block);
    if (/no(-?)highlight/.test(language))
        return;

    var node;
    if (options.useBR) {
      node = document.createElementNS('http://www.w3.org/1999/xhtml', 'div');
      node.innerHTML = block.innerHTML.replace(/\n/g, '').replace(/<br[ \/]*>/g, '\n');
    } else {
      node = block;
    }
    var text = node.textContent;
    var result = language ? highlight(language, text, true) : highlightAuto(text);

    var originalStream = nodeStream(node);
    if (originalStream.length) {
      var resultNode = document.createElementNS('http://www.w3.org/1999/xhtml', 'div');
      resultNode.innerHTML = result.value;
      result.value = mergeStreams(originalStream, nodeStream(resultNode), text);
    }
    result.value = fixMarkup(result.value);

    block.innerHTML = result.value;
    block.className = buildClassName(block.className, language, result.language);
    block.result = {
      language: result.language,
      re: result.relevance
    };
    if (result.second_best) {
      block.second_best = {
        language: result.second_best.language,
        re: result.second_best.relevance
      };
    }
  }

  var options = {
    classPrefix: 'hljs-',
    tabReplace: null,
    useBR: false,
    languages: undefined
  };

  /*
  Updates highlight.js global options with values passed in the form of an object
  */
  function configure(user_options) {
    options = inherit(options, user_options);
  }

  /*
  Applies highlighting to all <pre><code>..</code></pre> blocks on a page.
  */
  function initHighlighting() {
    if (initHighlighting.called)
      return;
    initHighlighting.called = true;

    var blocks = document.querySelectorAll('pre code');
    Array.prototype.forEach.call(blocks, highlightBlock);
  }

  /*
  Attaches highlighting to the page load event.
  */
  function initHighlightingOnLoad() {
    addEventListener('DOMContentLoaded', initHighlighting, false);
    addEventListener('load', initHighlighting, false);
  }

  var languages = {};
  var aliases = {};

  function registerLanguage(name, language) {
    var lang = languages[name] = language(hljs);
    if (lang.aliases) {
      lang.aliases.forEach(function(alias) {aliases[alias] = name;});
    }
  }

  function listLanguages() {
    return Object.keys(languages);
  }

  function getLanguage(name) {
    return languages[name] || languages[aliases[name]];
  }

  /* Interface definition */

  hljs.highlight = highlight;
  hljs.highlightAuto = highlightAuto;
  hljs.fixMarkup = fixMarkup;
  hljs.highlightBlock = highlightBlock;
  hljs.configure = configure;
  hljs.initHighlighting = initHighlighting;
  hljs.initHighlightingOnLoad = initHighlightingOnLoad;
  hljs.registerLanguage = registerLanguage;
  hljs.listLanguages = listLanguages;
  hljs.getLanguage = getLanguage;
  hljs.inherit = inherit;

  // Common regexps
  hljs.IDENT_RE = '[a-zA-Z][a-zA-Z0-9_]*';
  hljs.UNDERSCORE_IDENT_RE = '[a-zA-Z_][a-zA-Z0-9_]*';
  hljs.NUMBER_RE = '\\b\\d+(\\.\\d+)?';
  hljs.C_NUMBER_RE = '(\\b0[xX][a-fA-F0-9]+|(\\b\\d+(\\.\\d*)?|\\.\\d+)([eE][-+]?\\d+)?)'; // 0x..., 0..., decimal, float
  hljs.BINARY_NUMBER_RE = '\\b(0b[01]+)'; // 0b...
  hljs.RE_STARTERS_RE = '!|!=|!==|%|%=|&|&&|&=|\\*|\\*=|\\+|\\+=|,|-|-=|/=|/|:|;|<<|<<=|<=|<|===|==|=|>>>=|>>=|>=|>>>|>>|>|\\?|\\[|\\{|\\(|\\^|\\^=|\\||\\|=|\\|\\||~';

  // Common modes
  hljs.BACKSLASH_ESCAPE = {
    begin: '\\\\[\\s\\S]', relevance: 0
  };
  hljs.APOS_STRING_MODE = {
    className: 'string',
    begin: '\'', end: '\'',
    illegal: '\\n',
    contains: [hljs.BACKSLASH_ESCAPE]
  };
  hljs.QUOTE_STRING_MODE = {
    className: 'string',
    begin: '"', end: '"',
    illegal: '\\n',
    contains: [hljs.BACKSLASH_ESCAPE]
  };
  hljs.PHRASAL_WORDS_MODE = {
    begin: /\b(a|an|the|are|I|I'm|isn't|don't|doesn't|won't|but|just|should|pretty|simply|enough|gonna|going|wtf|so|such)\b/
  };
  hljs.C_LINE_COMMENT_MODE = {
    className: 'comment',
    begin: '//', end: '$',
    contains: [hljs.PHRASAL_WORDS_MODE]
  };
  hljs.C_BLOCK_COMMENT_MODE = {
    className: 'comment',
    begin: '/\\*', end: '\\*/',
    contains: [hljs.PHRASAL_WORDS_MODE]
  };
  hljs.HASH_COMMENT_MODE = {
    className: 'comment',
    begin: '#', end: '$',
    contains: [hljs.PHRASAL_WORDS_MODE]
  };
  hljs.NUMBER_MODE = {
    className: 'number',
    begin: hljs.NUMBER_RE,
    relevance: 0
  };
  hljs.C_NUMBER_MODE = {
    className: 'number',
    begin: hljs.C_NUMBER_RE,
    relevance: 0
  };
  hljs.BINARY_NUMBER_MODE = {
    className: 'number',
    begin: hljs.BINARY_NUMBER_RE,
    relevance: 0
  };
  hljs.CSS_NUMBER_MODE = {
    className: 'number',
    begin: hljs.NUMBER_RE + '(' +
      '%|em|ex|ch|rem'  +
      '|vw|vh|vmin|vmax' +
      '|cm|mm|in|pt|pc|px' +
      '|deg|grad|rad|turn' +
      '|s|ms' +
      '|Hz|kHz' +
      '|dpi|dpcm|dppx' +
      ')?',
    relevance: 0
  };
  hljs.REGEXP_MODE = {
    className: 'regexp',
    begin: /\//, end: /\/[gimuy]*/,
    illegal: /\n/,
    contains: [
      hljs.BACKSLASH_ESCAPE,
      {
        begin: /\[/, end: /\]/,
        relevance: 0,
        contains: [hljs.BACKSLASH_ESCAPE]
      }
    ]
  };
  hljs.TITLE_MODE = {
    className: 'title',
    begin: hljs.IDENT_RE,
    relevance: 0
  };
  hljs.UNDERSCORE_TITLE_MODE = {
    className: 'title',
    begin: hljs.UNDERSCORE_IDENT_RE,
    relevance: 0
  };

  return hljs;
}));
```

```noweave#+highlight_bash.js
module.exports = function(hljs) {
  var VAR = {
    className: 'variable',
    variants: [
      {begin: /\$[\w\d#@][\w\d_]*/},
      {begin: /\$\{(.*?)\}/}
    ]
  };
  var QUOTE_STRING = {
    className: 'string',
    begin: /"/, end: /"/,
    contains: [
      hljs.BACKSLASH_ESCAPE,
      VAR,
      {
        className: 'variable',
        begin: /\$\(/, end: /\)/,
        contains: [hljs.BACKSLASH_ESCAPE]
      }
    ]
  };
  var APOS_STRING = {
    className: 'string',
    begin: /'/, end: /'/
  };

  return {
    aliases: ['sh', 'zsh'],
    lexemes: /-?[a-z\.]+/,
    keywords: {
      keyword:
        'if then else elif fi for while in do done case esac function',
      literal:
        'true false',
      built_in:
        // Shell built-ins
        // http://www.gnu.org/software/bash/manual/html_node/Shell-Builtin-Commands.html
        'break cd continue eval exec exit export getopts hash pwd readonly return shift test times ' +
        'trap umask unset ' +
        // Bash built-ins
        'alias bind builtin caller command declare echo enable help let local logout mapfile printf ' +
        'read readarray source type typeset ulimit unalias ' +
        // Shell modifiers
        'set shopt ' +
        // Zsh built-ins
        'autoload bg bindkey bye cap chdir clone comparguments compcall compctl compdescribe compfiles ' +
        'compgroups compquote comptags comptry compvalues dirs disable disown echotc echoti emulate ' +
        'fc fg float functions getcap getln history integer jobs kill limit log noglob popd print ' +
        'pushd pushln rehash sched setcap setopt stat suspend ttyctl unfunction unhash unlimit ' +
        'unsetopt vared wait whence where which zcompile zformat zftp zle zmodload zparseopts zprof ' +
        'zpty zregexparse zsocket zstyle ztcp',
      operator:
        '-ne -eq -lt -gt -f -d -e -s -l -a' // relevance booster
    },
    contains: [
      {
        className: 'shebang',
        begin: /^#![^\n]+sh\s*$/,
        relevance: 10
      },
      {
        className: 'function',
        begin: /\w[\w\d_]*\s*\(\s*\)\s*\{/,
        returnBegin: true,
        contains: [hljs.inherit(hljs.TITLE_MODE, {begin: /\w[\w\d_]*/})],
        relevance: 0
      },
      hljs.HASH_COMMENT_MODE,
      hljs.NUMBER_MODE,
      QUOTE_STRING,
      APOS_STRING,
      VAR
    ]
  };
};
```

```noweave#+highlight_xml.js
module.exports = function(hljs) {
  var XML_IDENT_RE = '[A-Za-z0-9\\._:-]+';
  var PHP = {
    begin: /<\?(php)?(?!\w)/, end: /\?>/,
    subLanguage: 'php', subLanguageMode: 'continuous'
  };
  var TAG_INTERNALS = {
    endsWithParent: true,
    illegal: /</,
    relevance: 0,
    contains: [
      PHP,
      {
        className: 'attribute',
        begin: XML_IDENT_RE,
        relevance: 0
      },
      {
        begin: '=',
        relevance: 0,
        contains: [
          {
            className: 'value',
            contains: [PHP],
            variants: [
              {begin: /"/, end: /"/},
              {begin: /'/, end: /'/},
              {begin: /[^\s\/>]+/}
            ]
          }
        ]
      }
    ]
  };
  return {
    aliases: ['html', 'xhtml', 'rss', 'atom', 'xsl', 'plist'],
    case_insensitive: true,
    contains: [
      {
        className: 'doctype',
        begin: '<!DOCTYPE', end: '>',
        relevance: 10,
        contains: [{begin: '\\[', end: '\\]'}]
      },
      {
        className: 'comment',
        begin: '<!--', end: '-->',
        relevance: 10
      },
      {
        className: 'cdata',
        begin: '<\\!\\[CDATA\\[', end: '\\]\\]>',
        relevance: 10
      },
      {
        className: 'tag',
        /*
        The lookahead pattern (?=...) ensures that 'begin' only matches
        '<style' as a single word, followed by a whitespace or an
        ending braket. The '$' is needed for the lexeme to be recognized
        by hljs.subMode() that tests lexemes outside the stream.
        */
        begin: '<style(?=\\s|>|$)', end: '>',
        keywords: {title: 'style'},
        contains: [TAG_INTERNALS],
        starts: {
          end: '</style>', returnEnd: true,
          subLanguage: 'css'
        }
      },
      {
        className: 'tag',
        // See the comment in the <style tag about the lookahead pattern
        begin: '<script(?=\\s|>|$)', end: '>',
        keywords: {title: 'script'},
        contains: [TAG_INTERNALS],
        starts: {
          end: '</script>', returnEnd: true,
          subLanguage: 'javascript'
        }
      },
      PHP,
      {
        className: 'pi',
        begin: /<\?\w+/, end: /\?>/,
        relevance: 10
      },
      {
        className: 'tag',
        begin: '</?', end: '/?>',
        contains: [
          {
            className: 'title', begin: /[^ \/><\n\t]+/, relevance: 0
          },
          TAG_INTERNALS
        ]
      }
    ]
  };
};
```

```noweave#+highlight_javascript.js
module.exports = function(hljs) {
  return {
    aliases: ['js'],
    keywords: {
      keyword:
        'in if for while finally var new function do return void else break catch ' +
        'instanceof with throw case default try this switch continue typeof delete ' +
        'let yield const class',
      literal:
        'true false null undefined NaN Infinity',
      built_in:
        'eval isFinite isNaN parseFloat parseInt decodeURI decodeURIComponent ' +
        'encodeURI encodeURIComponent escape unescape Object Function Boolean Error ' +
        'EvalError InternalError RangeError ReferenceError StopIteration SyntaxError ' +
        'TypeError URIError Number Math Date String RegExp Array Float32Array ' +
        'Float64Array Int16Array Int32Array Int8Array Uint16Array Uint32Array ' +
        'Uint8Array Uint8ClampedArray ArrayBuffer DataView JSON Intl arguments require ' +
        'module console window document'
    },
    contains: [
      {
        className: 'pi',
        relevance: 10,
        variants: [
          {begin: /^\s*('|")use strict('|")/},
          {begin: /^\s*('|")use asm('|")/}
        ]
      },
      hljs.APOS_STRING_MODE,
      hljs.QUOTE_STRING_MODE,
      hljs.C_LINE_COMMENT_MODE,
      hljs.C_BLOCK_COMMENT_MODE,
      hljs.C_NUMBER_MODE,
      { // "value" container
        begin: '(' + hljs.RE_STARTERS_RE + '|\\b(case|return|throw)\\b)\\s*',
        keywords: 'return throw case',
        contains: [
          hljs.C_LINE_COMMENT_MODE,
          hljs.C_BLOCK_COMMENT_MODE,
          hljs.REGEXP_MODE,
          { // E4X
            begin: /</, end: />;/,
            relevance: 0,
            subLanguage: 'xml'
          }
        ],
        relevance: 0
      },
      {
        className: 'function',
        beginKeywords: 'function', end: /\{/, excludeEnd: true,
        contains: [
          hljs.inherit(hljs.TITLE_MODE, {begin: /[A-Za-z$_][0-9A-Za-z$_]*/}),
          {
            className: 'params',
            begin: /\(/, end: /\)/,
            contains: [
              hljs.C_LINE_COMMENT_MODE,
              hljs.C_BLOCK_COMMENT_MODE
            ],
            illegal: /["'\(]/
          }
        ],
        illegal: /\[|%/
      },
      {
        begin: /\$[(.]/ // relevance booster for a pattern common to JS libs: `$(something)` and `$.something`
      },
      {
        begin: '\\.' + hljs.IDENT_RE, relevance: 0 // hack: prevents detection of keywords after dots
      }
    ]
  };
};
```

```noweave#+highlight_json.js
module.exports = function(hljs) {
  var LITERALS = {literal: 'true false null'};
  var TYPES = [
    hljs.QUOTE_STRING_MODE,
    hljs.C_NUMBER_MODE
  ];
  var VALUE_CONTAINER = {
    className: 'value',
    end: ',', endsWithParent: true, excludeEnd: true,
    contains: TYPES,
    keywords: LITERALS
  };
  var OBJECT = {
    begin: '{', end: '}',
    contains: [
      {
        className: 'attribute',
        begin: '\\s*"', end: '"\\s*:\\s*', excludeBegin: true, excludeEnd: true,
        contains: [hljs.BACKSLASH_ESCAPE],
        illegal: '\\n',
        starts: VALUE_CONTAINER
      }
    ],
    illegal: '\\S'
  };
  var ARRAY = {
    begin: '\\[', end: '\\]',
    contains: [hljs.inherit(VALUE_CONTAINER, {className: null})], // inherit is also a workaround for a bug that makes shared modes with endsWithParent compile only the ending of one of the parents
    illegal: '\\S'
  };
  TYPES.splice(TYPES.length, 0, OBJECT, ARRAY);
  return {
    contains: TYPES,
    keywords: LITERALS,
    illegal: '\\S'
  };
};
```

```noweave#+highlight_css.js
module.exports = function(hljs) {
  var IDENT_RE = '[a-zA-Z-][a-zA-Z0-9_-]*';
  var FUNCTION = {
    className: 'function',
    begin: IDENT_RE + '\\(',
    returnBegin: true,
    excludeEnd: true,
    end: '\\('
  };
  return {
    case_insensitive: true,
    illegal: '[=/|\']',
    contains: [
      hljs.C_BLOCK_COMMENT_MODE,
      {
        className: 'id', begin: '\\#[A-Za-z0-9_-]+'
      },
      {
        className: 'class', begin: '\\.[A-Za-z0-9_-]+',
        relevance: 0
      },
      {
        className: 'attr_selector',
        begin: '\\[', end: '\\]',
        illegal: '$'
      },
      {
        className: 'pseudo',
        begin: ':(:)?[a-zA-Z0-9\\_\\-\\+\\(\\)\\"\\\']+'
      },
      {
        className: 'at_rule',
        begin: '@(font-face|page)',
        lexemes: '[a-z-]+',
        keywords: 'font-face page'
      },
      {
        className: 'at_rule',
        begin: '@', end: '[{;]', // at_rule eating first "{" is a good thing
                                 // because it doesn’t let it to be parsed as
                                 // a rule set but instead drops parser into
                                 // the default mode which is how it should be.
        contains: [
          {
            className: 'keyword',
            begin: /\S+/
          },
          {
            begin: /\s/, endsWithParent: true, excludeEnd: true,
            relevance: 0,
            contains: [
              FUNCTION,
              hljs.APOS_STRING_MODE, hljs.QUOTE_STRING_MODE,
              hljs.CSS_NUMBER_MODE
            ]
          }
        ]
      },
      {
        className: 'tag', begin: IDENT_RE,
        relevance: 0
      },
      {
        className: 'rules',
        begin: '{', end: '}',
        illegal: '[^\\s]',
        relevance: 0,
        contains: [
          hljs.C_BLOCK_COMMENT_MODE,
          {
            className: 'rule',
            begin: '[^\\s]', returnBegin: true, end: ';', endsWithParent: true,
            contains: [
              {
                className: 'attribute',
                begin: '[A-Z\\_\\.\\-]+', end: ':',
                excludeEnd: true,
                illegal: '[^\\s]',
                starts: {
                  className: 'value',
                  endsWithParent: true, excludeEnd: true,
                  contains: [
                    FUNCTION,
                    hljs.CSS_NUMBER_MODE,
                    hljs.QUOTE_STRING_MODE,
                    hljs.APOS_STRING_MODE,
                    hljs.C_BLOCK_COMMENT_MODE,
                    {
                      className: 'hexcolor', begin: '#[0-9A-Fa-f]+'
                    },
                    {
                      className: 'important', begin: '!important'
                    }
                  ]
                }
              }
            ]
          }
        ]
      }
    ]
  };
};
```

```noweave#+highlight_markdown.js
module.exports = function(hljs) {
  return {
    aliases: ['md', 'mkdown', 'mkd'],
    contains: [
      // highlight headers
      {
        className: 'header',
        variants: [
          { begin: '^#{1,6}', end: '$' },
          { begin: '^.+?\\n[=-]{2,}$' }
        ]
      },
      // inline html
      {
        begin: '<', end: '>',
        subLanguage: 'xml',
        relevance: 0
      },
      // lists (indicators only)
      {
        className: 'bullet',
        begin: '^([*+-]|(\\d+\\.))\\s+'
      },
      // strong segments
      {
        className: 'strong',
        begin: '[*_]{2}.+?[*_]{2}'
      },
      // emphasis segments
      {
        className: 'emphasis',
        variants: [
          { begin: '\\*.+?\\*' },
          { begin: '_.+?_'
          , relevance: 0
          }
        ]
      },
      // blockquotes
      {
        className: 'blockquote',
        begin: '^>\\s+', end: '$'
      },
      // code snippets
      {
        className: 'code',
        variants: [
          { begin: '`.+?`' },
          { begin: '^( {4}|\t)', end: '$'
          , relevance: 0
          }
        ]
      },
      // horizontal rules
      {
        className: 'horizontal_rule',
        begin: '^[-\\*]{3,}', end: '$'
      },
      // using links - title and link
      {
        begin: '\\[.+?\\][\\(\\[].*?[\\)\\]]',
        returnBegin: true,
        contains: [
          {
            className: 'link_label',
            begin: '\\[', end: '\\]',
            excludeBegin: true,
            returnEnd: true,
            relevance: 0
          },
          {
            className: 'link_url',
            begin: '\\]\\(', end: '\\)',
            excludeBegin: true, excludeEnd: true
          },
          {
            className: 'link_reference',
            begin: '\\]\\[', end: '\\]',
            excludeBegin: true, excludeEnd: true
          }
        ],
        relevance: 10
      },
      {
        begin: '^\\[\.+\\]:',
        returnBegin: true,
        contains: [
          {
            className: 'link_reference',
            begin: '\\[', end: '\\]:',
            excludeBegin: true, excludeEnd: true,
            starts: {
              className: 'link_url',
              end: '$'
            }
          }
        ]
      }
    ]
  };
};
```

# Sharing the Code

Using the CouchDB replicator, the JSON version of this document
can be moved from one CouchDB to another.

Or, select the [README.json](README.json) then copy and paste
the document to your CouchDB.

If you have `markover` installed, then use the following to build
a `JSON` object.

```sh
node -e "require('markover').tangleStream({sourcetarget: 'README.md'})" < README.md > pry.json
```

And the following for the `HTML` documentation.

```sh
node -e "require('markover').weaveStream()" < README.md > index.html
```

The display of this 'README.md' file on GitHub is readable, but not
quite right since GitHub doesn't process the field names for the code
blocks.
The generated [index.html](http://cygnyx.github.io/pry) file is better.
Also on GitHub you can copy and paste the pry.json file into
the `CouchDB` database.

# Final Thoughts

Although the existing editor works well enough, a more sophisticated
editor would be useful for large projects.
It should be possible to change to the [ACE](http://ace.c9.io) editor.
I made minor changes in the markdown mode to work with hash tagged field
names, adding json submode, etc. Another interesting approach would be
the [Hallo](http://hallojs.org) editor. 

Clearly tools like `kanso` provide many additional facilities for
deploying a `CouchApp`.
Many of these capabilities could be included in `pry`.
But before implementing more capabilities, I will try using
the existing system for a while.

In order to bootstap `pry` on a `CouchDB` I wrote
a separate `nodejs` utility that used
`markover` to tangle the source document and then
upload the `JSON` to the `CouchDB`.
Run the utility as follows:

```sh
node update.js < README.md
```

To verify that the editor was working properly,
I would make minor changes to the documentation and code.
But the bulk of the editing occured on my local machine.
This brings into question whether `pry` works effectively.

I've excluded the source from the documentation.

```noweave
var markover = require('../markover/markover.js')

var it = {
  gather: function (res, done) {
    var buf = []
    res.on('data', function(chunk) {
      buf.push(chunk.toString())
    })
    res.on('end', function(chunk) {
      done(buf.join(''))
    })
  },

  updateDesignDocument: function(uri, opts) {
    options = opts || {}
    var url = require('url')
    var http = require('http')
    var ddoc = url.parse(uri)

    http.get(uri, function(res) {
      it.gather(res, update)
    }).on('error', function(e) {
      console.log('ERROR: ' + e.message)
    })

    function update(res) {
      var obj = JSON.parse(res)
      it.rev = obj._rev
      it.gather(process.stdin, update0)
    }

    function update0(res) {
      var str = markover.tangleJSON(res)
      var obj = JSON.parse(str)
      obj._rev = it.rev
      obj['README.md'] = res

      var req = http.request({host: ddoc.host,
        port: ddoc.port || '80',
        path: ddoc.path,
        method: 'PUT'
      }, function(res) {
        it.gather(res, update2)
      })

      req.write(JSON.stringify(obj))
      req.end()
    }

    function update2(res) {
      var obj = JSON.parse(res)
      if (!obj.ok) {
        console.log('ERROR: ', JSON.stringify(obj))
      } else {
        console.log('INFO: rev', obj.rev)
      }
    }
  }
}

it.updateDesignDocument('http://localhost/example/_design/pry')
```

Somewhat surprisingly it is possible to edit documents on
mobile devices.
The `pry` document does take some time to tangle and weave
on a mobile device.

I was able to copy and paste the `pry` JSON to deploy it on `Cloudant`.
And it is able to update itself on it.

Although attachments are not consistent with the 'one' in, 'one' out
structure of `pry`, there is a clear utility in supporting attachments.

Being able to insert code could help with the layout of the shows field.
Currently, the embedded anonymous functions are simple, yet hard to
understand. Being able to insert a quoted function into the middle of
the shows structure is certainly a solution. Another possibility is to
break the shows structure apart, like:

    ```#shows
    {
      "404" :
    ```
    
    ```#+shows
    function(doc, req) {
      return {
        body: '<h1>404 - Document not found</h1>\n',
        headers: {'Content-Type': 'text/html'}
      }
    },
    ```
    
    etc.
    
    ```#shows
    }
    ```

This would make the shows functions much more readable.
However, this requires a change to `markover` to handle
appending quoted fields to non-quoted fields.
