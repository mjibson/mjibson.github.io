---
layout: post
title: "Careers Localization, part 3: Extraction with Roslyn and Uglify"
date: 2013-03-01 15:52
comments: true
categories: [careers, localization, roslyn, uglifyjs, node]
---

[Previously](/blog/2013/02/28/careers-localization-part-2-api/) we have discussed our reasoning and API for localization. Here, we would like to continue with the next topic: **text extraction from the source code**. That is, after we have indicated the text we would like translated, how we extract that text out of our codebase for translation. (Note: we used [Acclaro](http://www.acclaro.com/) for the actual translation, and have been very happy with them. Recommended.)

The difficulty is to extract all of the strings so that they can be sent to translators. Compounding this difficulty, we also must make sure to send the correct number of **plural forms** to the translators. Look at the example below. `__count` does not appear in the translated string, only in the values object. This means we must, at extraction time, understand these value objects and be able to reason about them---we must know their types.

``` cpp
_s("There is a unicorn", new { __count = 3 }) // "There are some unicorns"
```

There are a few bad ways to do this:

- build a parser by hand to deal with this (possible, but fragile, error-prone, and silently oblivious to things it does not support)
- search or grep
- [regex](http://stackoverflow.com/a/1732454/864236)

Our first solution was the hand-built parser. It worked reasonbly well, but there were known, unfixable bugs. We needed a solution that allowed us to have full syntatic and semantic understanding of source code files. Syntax for knowing the tree of the source code: function calls, argument lists, object access. Semantic for seeing into object types and values.

## C# and Razor extraction with Roslyn

The C# team at Microsoft has been working on a project called [Roslyn](http://msdn.microsoft.com/en-us/vstudio/roslyn.aspx). It is a programmatic way to access the syntax tree and semantic model of C# files. That means that we can hand it a C# file and then search through it looking for certain kinds of things, and act on them. Roslyn comes with a SyntaxWalker class that walks over each node. You can override the one you want. In our case, we want any invocation (function call) named `_s` or `_m`:

``` cpp
private static readonly IEnumerable<string> SINGLE_LINE = new[] { "_s", "_m" };
public override void VisitInvocationExpression(InvocationExpressionSyntax node)
{
	base.VisitInvocationExpression(node);
	var callname = node.Expression.GetText().ToString();
	if (SINGLE_LINE.Any(callname.EndsWith))
	{
		var stringInfo = StringInfo.Create(node.ArgumentList, model);
		if (stringInfo != null)
			// save it somewhere
	}
}
```

Now we're trying to create a new `StringInfo` (just a container class). We need to examine our `node.ArgumentList` and see if the first argument is a string. We use C#'s dynamic feature coupled with various ways to call `StringInfo.Create` to easily support different types of arguments (noted in comments above the functions):

```
// _s("string", object) (object may contain __count)
public static StringInfo Create(ArgumentListSyntax args, ISemanticModel model)
{
	var first = args.Arguments.FirstOrDefault();
	if (first == null || first.Expression.Kind != SyntaxKind.StringLiteralExpression)
		return null;

	var stringInfo = new StringInfo {
		Text = first.Expression.GetFirstToken().ValueText,
	};

	var second = args.Arguments.Skip(1).FirstOrDefault();
	if (second != null)
		stringInfo.Count = HasCount((dynamic)second.Expression, model);

	return stringInfo;
}

// _s("string only") (no object, no __count)
public static StringInfo Create(LiteralExpressionSyntax literal, ISemanticModel model)
{
	if (literal == null)
		return null;

	return new StringInfo {
		Text = literal.Token.ValueText,
		Count = false,
	};
}
```

Ok, now we've got our string. Next we need to figure out if `__count` is part of the object. We've implemented various ways to determine `HasCount()`, as referenced above on line 14. Again, C#'s dynamic proves useful.

```
private const string COUNT = "__count";

// _s("string", new { __count = 1 })
public static bool HasCount(AnonymousObjectCreationExpressionSyntax arg, ISemanticModel model)
{
	var result = arg.Initializers.Any(x =>
		{
			if(x.NameEquals != null)
				return x.NameEquals.Name.Identifier.ValueText == COUNT; // see if __count is part of the object
			return HasCount((dynamic)x.Expression, model); // otherwise throw it back
		});
	return result;
}

// _s("string", obj)
public static bool HasCount(IdentifierNameSyntax arg, ISemanticModel model)
{
	// we already have an object, just see if it has __count
	var symbols = model.LookupSymbols(arg.Span.Start, name: arg.Identifier.ValueText);
	var first = symbols.First();
	TypeSymbol type = ((dynamic)first).Type;
	return type.GetMembers(COUNT).Any();
}

// _s("string", obj.member)
public static bool HasCount(MemberAccessExpressionSyntax arg, ISemanticModel model)
{
	// extract out member...
	var name = arg.Expression as IdentifierNameSyntax;
	if (name != null)
		// ...and try some of our above functions
		return HasCount(name, model);
	return false;
}

// anything else
public static bool HasCount(ExpressionSyntax arg, ISemanticModel model)
{
	return false;
}
```

The above tries all variants (that were in our code) that might contain `__count` somehow. The use of dynamic allows us to not care about the kind of expression, which is determined during runtime, and then the appropriate version of `HasCount()` is called. Here's [a gist](https://gist.github.com/mjibson/5052106#file-extractor-cs) of it all (it's got some other features not listed here).

The above covers our C# controllers. For the Razor views, we cheat a bit. The ASP.NET compiler is invoked which converts the views into C# files, then the same extraction logic is run on those. This is much easier than trying to write a full razor parser.

## JavaScript extraction with UglifyJS

For JavaScript, we use a similar technique. [UglifyJS](https://github.com/mishoo/UglifyJS) is a JavaScript-based JavaScript parser. It also can return a syntax tree, which we can examine for uses of `_s`, from which we can extract strings and objects. It's a pretty hideous function and not worth pasting, but [here it is](https://gist.github.com/mjibson/5052106#file-extractor-js).

Both of these extractors yield JSON, which is processed in node and sent off to our translation service. We also have a dashboard (built with [AngularJS](http://angularjs.org/)) to view all translations, fix them, and reprocess all the files.

## Conclusion

Having two full language parsers allows us to be flexible. Both our C# and JavaScript implementations are equally functional at all levels---API, extraction, display---allowing us to have one unified API in our codebase.

Translation and localization are hard. We spent months refactoring and extending our codebase to support it. But now, we can change, add, and translate strings with little effort, because our tools handle all the hard stuff. If you care about having quality translations (supporting dynamic text insertion and pluralization), whichever solution you use (resource files or gettext-style) must be able to fully understand your code so that it can generate high-quality output to send to translators.
