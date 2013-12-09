---
layout: post
title: "Careers Localization, part 2: The API"
date: 2013-02-28 15:52
comments: true
categories: [careers, localization]
---

[Previously](/blog/2013/02/27/careers-localization-part-1-why-roll-our-own/) we discussed our design requirements and solution for localization. Here we will continue with a description of the API. This means the way in which we would craft strings to be localized.

## Markup

We chose to use gettext-style translation strings (i.e., not resource files where strings are identified somehow, instead we just indicate the English string right where it will be shown). We chose a syntax that allows dynamic variable replacement and easy pluralization, and can work similarly in JavaScript and C#. Here's what we came up with:

``` cpp C#
_s("Unicorns prance") // "Unicorns prance"
_s("$name$ have horns", new { name = "Unicorns" }) // "Unicorns have horns"
_s("There are $__count$ unicorns", new { __count = 1 }) // "There is 1 unicorn"
_s("There is a unicorn", new { __count = 3 }) // "There are some unicorns"
_s(@"
	Multi
	line
	supported.") // "Multi line supported." (whitespace collapsed)
```

``` js JavaScript
_s('Unicorns prance') // "Unicorns prance" (single or double quotes accepted - any valid JavaScript string)
_s("$name$ have horns", { name: "Unicorns" }) // "Unicorns have horns"
_s('There are $__count$ unicorns', { __count: 1 }) // "There is 1 unicorn"
_s("There is a unicorn", { __count: 3 }) // "There are some unicorns"
```

We chose a function named `_s` that takes a string and an optional values object used for variable replacement. There is a special `__count` member of the values object which, if it exists, indicates to our translation engine that this is a pluralizeable string. Calls to `_s` could be in our C# files anywhere in the tree. They could also be in our razor views:

``` cpp Razor
<div>
	@_s("translate this")
</div>
```

Finally, it also supports existing objects:

``` cpp C#
class User {
	public string Name { get; set; }
}

var user = new User { Name = "Jimmy the Unicorn" };
_s("$Name$ is here!", user);
```

## Markdown

Requirement 5 from part 1 is to support markdown formatting. The purpose of this requirement was so that we would never send HTML to translators. We do not trust someone else writing our HTML. Thus, we needed support for simplified markdown (only bold, italics, and links). Out of this grew the `_m` function (m for markdown, in contrast to s for string in `_s`). This puts certain tweets in perspective.

{% blockquote @kevinmontrose https://twitter.com/kevinmontrose/status/233392144193294336 %}
And so begins the drive to redefine "S&M" in the company lexicon.
{% endblockquote %}

With `_m`, we can do things like:

``` cpp
_m("Hi, **we hate fun** here at [stack overflow](stackoverflow.com).")
```

## Limitations

The above API has some problems, or doesn't support the following cases:

### Best-practice DOM handling

Due to the use of markdown instead of HTML, there are some annoyances about how we must treat DOM elements. Let's say that, before translation, we had something like this:

```
<div>Hello, <span id="click">click here</span> for magic.</div>
<script>
	$("#click").click(cb);
</script>
```

In (incorrect) translated form, we would have this:

```
<div>@_s("Hello,") <span id="click">@_s("click here")</span> @_s("for magic.")</div>
```

This is incorrect because the sentence has now been fragmented into three. Translators will then get these three fragments and must translate each on its own, with no power to reorder them if needed in the target language. But, how can we retain the JavaScript action? The `<span>` in there is not supported since we don't allow HTML in translated strings (or, more accurately, we encode all output strings so they are HTML-safe). The solution is to use our markdown formatting and do funky stuff with the DOM. In markdown, we have access to `<strong>` and `<em>`, thus:

```
<div id="click">@_m("Hello, **click here** for magic.")</div>
<script>
	$("#click strong").click(cb);
</script>
```

The `**click here**` will render as `<strong>click here</strong>`, which we can use in a selector. You must then also apply CSS to disable the normal bolding. Annoying, but it works.

### Gender variants

Some languages vary articles by the gender (or pluralization) of its object. This is difficult to do because sometimes we don't know the gender, and so it's impossible to tell the translation engine which gender to use. Consider:

```
_s("I'm going to $location$.", new { location = "Brazil" })
```

In Portuguese, "to" varies based on `$location$`. We need to know the gender and pluralization of `$location$` (for example, the United States is masculine plural, and so "to" would change to "aos"). But we allow any location to be entered, so we would have to have that data for each of our target languages for all (or at least many) possible places. We're not sure where to get that data, so we're not supporting this yet.

### Multiple pluralized words

We support exactly one `__count` instance, which means you can't have two pluralized words in a sentence. Let's consider the following case, assuming we wish to support any number of pluralizable words:

```
_s("I have $count1$ rainbows and $count2$ pandas.", new { count1 = 2, count2 = 1 })
```

English, German, French (and many others) have two plural forms (singular and other). This means we would have to generate `2 ^ 2 = 4` combinations of sentences to be pluralized:

```
I have 1 rainbow and 1 panda.
I have 1 rainbow and N pandas.
I have N rainbows and 1 panda.
I have N rainbows and N pandas.
```

There are languages that have [up to six](http://unicode.org/repos/cldr-tmp/trunk/diff/supplemental/language_plural_rules.html) plural forms: zero, one, two, few, many, other. So, for the above example string, we would have to generate `6 ^ 2 = 36` combinations:

```
I have 0 rainbows and 0 pandas.
I have 1 rainbows and 0 pandas.
I have 2 rainbows and 0 pandas.
I have few rainbows and 0 pandas.
I have many rainbows and 0 pandas.
I have other rainbows and 0 pandas.
// Etc., with all combinations:
I have [0, 1, 2, few, many, other] rainbows and [0, 1, 2, few, many, other] pandas.
```

These must all be generated for translation because it is possible that in the target language, the surrounding words could change. For instance, maybe "have" changes based on the object, instead of the subject, as in English. Due to this complication, we capped pluralization at one item per string.

### Other things we don't know about

There are many languages, and we're sure some of them have constructs that the above API doesn't support at all.

## Conclusion

The API allows us to do almost everything we need. The limitations ended up being truly limiting in only a few cases, and those could be worked around be restructuring the sentence. The next problem: how do we actually find the usages of `_s` and send them off to translators?

[Part 3: Extraction](/blog/2013/03/01/careers-localization-part-3-extraction/)
