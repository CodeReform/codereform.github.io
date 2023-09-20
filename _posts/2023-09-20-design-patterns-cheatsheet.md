---
layout: post
image: /assets/2023/09/pexels-karolina-grabowska-4887152.jpg
title: Software design patterns cheatsheet
date: 2023-09-20
type: post
comments: false
published: true
status: publish
categories:
- "Architecture"
tags:
- Design
- Architecture
author:
  display_name: George Dyrrachitis
  first_name: George
  last_name: Dyrrachitis
permalink: "/2023/09/20/design-patterns-cheatsheet"
---

## Introduction
How many times has it been that I had been looking at a software problem and seemed like a neat idea to use a design pattern to simplify the code. Sometimes I may be torn between few patterns or completely forgot that some others exist, as I don't really use all of them everyday. It would be cool if I had a cheatsheet to use. I know, there are tons of them online, but I like to make my own stuff. Besides, what's the point to have my own blog and not create my own cheatsheets to look from time to time?

Initially, I've thought to create just some PDFs, have them uploaded to my drive and reference them whenever I needed. But on the other hand, I thought that these can be shared and others can benefit. Well, I know well that this topic is well understood and tons of good material can be found on the internet. That is true, this content cannot compete with content produced by `refactoring.guru` or others. It is just collections of my notes that I reference day-to-day to solve coding problems. If this can be useful to others, then I'm happy. If looking for very comprehensive guides or tutorials, perhaps you're looking at the wrong resource, as I mentioned, there are plenty of great websites and books that teach you design patterns out there.

## FAQs
### Where's the cheatsheet?
Cheatsheet can be found in this blog at [https://codereform.com/cheatsheets/design_patterns](https://codereform.com/cheatsheets/design_patterns).

> Bookmark this URL to quickly refer to design patterns and sample code

I've actually created an entire cheatsheets section where I plan to add several other cheatsheets, for caching, architecture and many more.

### What I will find in the cheatsheet?
I've listed all 23 GOF patterns on that cheatsheet and separated them by pattern category, that is behavioural, structural and creational.
You will find an 1-sentence summary of the pattern and a UML diagram.
I also list usages, that is when to use the pattern and have links to examples on my github page.

The examples are written in Python. I've run each example on a Jupyter notebook, which I also have linked to each pattern. In each notebook I demonstrate the implementation of the pattern and include execution logs.

Here's an example of the singleton pattern, as I have defined in:
[https://github.com/gdyrrahitis/design-patterns-python/blob/main/singleton/pattern.py](https://github.com/gdyrrahitis/design-patterns-python/blob/main/singleton/pattern.py)
```python
"""Classic singleton"""
class Database:
    def __new__(cls):
        if not hasattr(cls, "instance"):
            cls.instance = super(Database, cls).__new__(cls)
        return cls.instance

    def save(self):
        print("saving to database")
```

And here is how I use the singleton `Database` class in the Jupyter notebook at:
[https://github.com/gdyrrahitis/design-patterns-python/blob/main/mediator/notebook.ipynb](https://github.com/gdyrrahitis/design-patterns-python/blob/main/mediator/notebook.ipynb)

```python
# Classic singleton example
from pattern import Database

db = Database()
db2 = Database()

print(f"is db and db2 same? {db is db2}")
# >> is db and db2 same? True

# calls save method on singleton
db.save()
# >> saving to database
```

## References
To write the cheatsheet, I referenced the [GOF book](https://www.goodreads.com/book/show/85009.Design_Patterns), which includes all 23 patterns.
Although this book is great and I recommend to read it 100%, I would admit that it is hard to read and sometimes it could use simpler language and examples. Few times I had to research online to find better definitions or more examples in order to build better comprehension for a pattern.
Check these other resources too
* [https://www.dofactory.com/net/design-patterns](https://www.dofactory.com/net/design-patterns) provides easy to understand examples on patterns usage, all in C#.
* [https://refactoring.guru/design-patterns](https://refactoring.guru/design-patterns) is one of the best websites on the subject, they offer also premium content and a book with code examples in many languages.