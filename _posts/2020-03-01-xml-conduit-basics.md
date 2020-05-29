---
categories: xml-conduit haskell
date:   2020-03-01
layout: post
title:  "Basic usage of xml-conduit library"
---

# Loading a XML file

# Filtering

# Note the namespace of Name in content filtering!

We filter out elements of a given `Cursor` with `element` function, of which the type description is as following:

```haskell
element :: Name -> Axis
```

The example usage would be (inspired by [this](https://www.yesodweb.com/book/xml) page):

```haskell
{-# LANGUAGE OverloadedStrings #-}

import Prelude hiding (readFile)
import Text.XML (def, readFile)
import Text.XML.Cursor (child, content, cursor, descendant, element, fromDocument)
import qualified Data.Text as T

main :: IO ()
main = do
    xmlContent <- readFile def "test.xml"
    let cursor = fromDocument xmlContent
    print $ T.concat $
            child cursor >>= element "head" >>= descendant >>= content
```

What took me a while to figure out was that when comparing `Name` (the first argument of `element` function) we have to take the namespace into account. Following the aforementioned page:

> According the XML namespace standard, two names are considered equivalent if they have the same localname and namespace. In other words, the prefix is not important. Therefore, xml-types defines Eq and Ord instances that ignore the prefix.

Say we've got the following Maven POM file:

```xml
<project xmlns = "http://maven.apache.org/POM/4.0.0"
	xmlns:xsi = "http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation = "http://maven.apache.org/POM/4.0.0
	http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.test</groupId>
	<artifactId>project_C</artifactId>
	<version>1.0.0-SNAPSHOT</version>

	<dependencies>
		<dependency>
			<groupId>com.test</groupId>
			<artifactId>project_A</artifactId>
			<version>1.0.0-SNAPSHOT</version>
		</dependency>
		<dependency>
			<groupId>com.external</groupId>
			<artifactId>logger</artifactId>
			<version>1.0.0-SNAPSHOT</version>
		</dependency>
	</dependencies>
</project>
```

If we wanted to extract `artifactId` element of the root `project` element, this approach will not work:

```haskell
{-# LANGUAGE OverloadedStrings #-}

import Prelude hiding (readFile)
import Text.XML (def, readFile)
import Text.XML.Cursor (fromDocument)
import qualified Data.Text as T

main :: IO ()
main = do
    xmlContent <- readFile def "test.xml"
    let cursor = fromDocument xmlContent
    print $ T.concat $
            child cursor >>= element "artifactId" >>= descendant >>= content
```

Because the namespace of root element (`project`) is "http://maven.apache.org/POM/4.0.0" (`xmlns` attribute sets this value) - as namespace is inherited, all descendants also will end up being in it. See XML reference documentation [here](https://www.w3.org/TR/xml-names/#scoping) and [here](https://www.w3.org/TR/xml-names/#defaulting) for the dry definition or [this](https://stackoverflow.com/a/25789100/3088888) SO post for a concise explanation.

What we'll have to do instead is to construct Name element explicitly:

```haskell
{-# LANGUAGE OverloadedStrings #-}

import Prelude hiding (readFile)
import Text.XML (def, readFile)
import Text.XML.Cursor (fromDocument)
import qualified Data.Text as T

main :: IO ()
main = do
    xmlContent <- readFile def "test.xml"
    let cursor = fromDocument xmlContent
    print $ T.concat $
            child cursor 
              >>= element Name {nameLocalName = "artifactId", nameNamespace = Just "http://maven.apache.org/POM/4.0.0", namePrefix = Nothing}
              >>= descendant >>= content
```
