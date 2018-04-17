---
title: "Testing CockroachDB: Generating Random (Valid) SQL"
date: 2016-10-19
slug: "random-sql"
aliases:
  - /blog/2016/10/19/testing-cockroachdb-generating-random-valid-sql/
---

_(Also published on the [Cockroach Labs Blog](https://www.cockroachlabs.com/blog/testing-random-valid-sql-in-cockroachdb/).)_

Some months ago I started work on a way to test random SQL statements with CockroachDB. This is important to expose unintended behavior in our server. For example, we want to prevent valid SQL statements from unexpectedly crashing the server or using all of the CPU or memory. We have already performed some small-scale fuzz testing, but fuzz testing often produces un-parseable input since it modifies bytes (although some fuzzers like AFL do attempt to produce clean input). The goal here was to produce **valid** SQL statements that the parser would accept and the system would then execute. These statements would essentially attempt to try various combinations of valid SQL to panic or otherwise render the system unusable (like consuming all CPU).

## Generation

There are a few ([sqllogictest](https://www.sqlite.org/sqllogictest/doc/trunk/about.wiki), [sqlsmith](https://github.com/anse1/sqlsmith)) programs that generate valid random SQL. CockroachDB has used sqllogictest for many tests, but as our SQL semantics differ a bit from the other SQL implementations, the test results increasingly differ from our outputs, and so its usefulness has decreased. The sqlsmith program generates more random SQL, but CockroachDB is a young database and as such we don’t yet support some of the more obscure corners of SQL. In addition, we support some additional statements that aren’t generated by this tool. Modifying sqlsmith to produce CockroachDB grammar is possible, but would be an ongoing, manual process.

CockroachDB’s SQL grammar is defined in a [YACC grammar](https://github.com/cockroachdb/cockroach/blob/develop/sql/parser/sql.y). YACC is a language that generates parsers from a grammar. A YACC grammar contains instructions based on input. Any unrecognized input is a parse error. Thus, a YACC grammar can enumerate all possible inputs. We already have a [YACC parser](https://www.cockroachlabs.com/blog/efficient-documentation-using-sql-grammar-diagrams/). With that we were able to convert the YACC grammar into an [AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree), which allows for programmatic inspection of the YACC grammar itself.

With the AST in memory [^1], a program was created that would take some top-level statement and choose one valid possibility for it to become, then repeat itself until there was nothing to change. This works by looking for [non-terminals](https://en.wikipedia.org/wiki/Terminal_and_nonterminal_symbols#Nonterminal_symbols) in the grammar and replacing them with any branch the non-terminal represents. For example, say we start with a [grant statement](https://www.cockroachlabs.com/docs/sql-grammar.html#grant_stmt). The non-terminals are `privileges`, `privilege_target`, and `grantee_list`.

<img src="/images/grant.png" alt="grant grammar diagram">

We can replace `privileges` with one of [its possibilities](https://www.cockroachlabs.com/docs/sql-grammar.html#privileges): `ALL`, or `privilege_list`.

<img src="/images/privileges.png" alt="privileges grammar diagram">

One of those is randomly chosen, and the loop continues. Some tokens can loop back into themselves, so there is a restriction to prevent too many replacements before giving up. The statement is done when there are no tokens left, only literals (keywords, punctuation), identifiers (table or database names), or things like string or float constants.

An example that’s a simplification of this might have this progression:

1. `stmt`
1. `grant_stmt`
1. `GRANT privileges ON privilege_target TO grantee_list`
1. `GRANT ALL ON TABLE table_pattern_list TO name`
1. `GRANT ALL ON TABLE table_pattern TO unreserved_keyword`
1. `GRANT ALL ON TABLE * TO COVERING`

In the end we get a weird variety of statements:

- `ALTER INDEX CHAR @ YEAR RENAME TO ident`
- `DEALLOCATE CHARACTERISTICS`
- `SAVEPOINT SAVEPOINT NEXT`
- `COPY ROLLUP ( ) FROM STDIN`
- `ALTER INDEX TIME @ TIMESTAMP RENAME TO SIMPLE`
- `START TRANSACTION PRIORITY HIGH , ISOLATION LEVEL READ COMMITTED`
- `GRANT DROP , DROP , DROP ON BYTES . REAL , * TO RENAME , ident , ident , ident`
- `REVOKE DELETE ON TABLE ZONE , SMALLINT . NULLS FROM TEXT`
- `ALTER TABLE ONLY ( EXTRACT_DURATION . COLLATION . * . * . * . * ) ALTER COLUMN COPY`
- `TRUNCATE TABLE ONLY ( NUMERIC ) , ONLY ( TIMESTAMP . TRIM ) , ONLY COALESCE CASCADE`
- `ALTER TABLE IF EXISTS BIT . * * RENAME CONSTRAINT UPSERT TO BYTES`

A separate function generates random valid function calls by inspecting our builtin function list and generating random arguments for them based on the type of argument.

## Results

This technique found [over a dozen unique panics](https://github.com/cockroachdb/cockroach/issues?utf8=%E2%9C%93&q=is%3Aissue%20RSG), many of which were various uses of `.*` where a specific table name instead of a `*` was expected. The discovery of these panics led to a [large refactor](https://www.cockroachlabs.com/blog/squashing-a-schroedinbug-with-strong-typing/), where again, these tests found a small bug. By running many queries at once, a race condition was found where a lock was misused. While testing functions we were able to consume all memory or CPU by passing very high values into some math and string generation functions. Last week we added some new SQL syntax and another panic was found.

These tests have found enough bugs that we now run them [every night](https://teamcity.cockroachdb.com/viewType.html?buildTypeId=Cockroach_Nightlies_RandomSyntaxTests&branch_Cockroach_Nightlies=develop&tab=buildTypeStatusDiv&guest=true) to proactively prevent new problems from happening. Since the tests are generated from our YACC grammar, as we continue to add new SQL syntax they will automatically incorporate the new features into the random testing. By continuing to run these tests nightly, our intent is to eliminate unexpected crashes and hangs in CockroachDB.

[1] There are other approaches here that also would have worked to produce the same result. For example, we could use [guru](https://docs.google.com/document/d/1_Y9xCEMj5S-7rv2ooHpZNH15JgRT5iM742gJkw5LtmQ/edit) to find concrete types that implement the [Statement interface](https://godoc.org/github.com/cockroachdb/cockroach/pkg/sql/parser#Statement) and fill in its fields with a similar method. At the end we can produce a string with its `String()` method. This has an added benefit that we do not depend on YACC at all and could thus write a custom parser. Also, it avoids some annoying cases where an invalid statement is generated with the YACC method, for example when the parser validates that an [interval is parseable](https://github.com/cockroachdb/cockroach/blob/511ded9ae509bb42438021a2427e283f7a8b5d09/pkg/sql/parser/sql.y#L1403).