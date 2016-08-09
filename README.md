#Getting Started

Writing in this book requires familiarity with git and gitbook. Get started below:

1. [Install gitbook](https://github.com/GitbookIO/gitbook/blob/master/docs/setup.md)

2. Clone this repo
 `git clone https://github.com/CompTox/comparing_similarity.git`

3. Install dependencies  
  ```
  > cd ./comparing_similarity
  > gitbook install
  ```
  Read about the included dependencies and what commands they allow in the **commands** section below.
4. serve the book `> gitbook serve`  
Read the book at [localhost:4000](localhost:4000).  Changes made to the repository will trigger a book reload.

## Commands

###[bibtex-citation](https://github.com/manuelmitasch/gitbook-plugin-bibtex-citation)
Manages citations in bibtex format. Stores citations in [literature.bib](./literature.bib) whose first entry is:  
```
1 @ARTICLE{search,
2  author = {Andreas Junghanns and Jonathan Schaeffer},
3  title = {Sokoban: Enhancing general single-agent search methods using domain knowledge},
4  journal = {Artificial Intelligence},
5  year = {2001},
6  volume = {129},
7  pages = {219-251}
8 }
```
Line 1 gives this article the key "**search**" which we can now reference as we write:
```
Search methods are super important{{"search" | cite}}.
```  
This line renders as "Search methods are super important{{"search" | cite}}."

{% references %}{% endreferences %}