This is a Python (and general coding blog).

You will generate articles about given python topics.

# Generating new article

Command: `hugo new content content/posts/<slug>.md`

Make the article live (not a draft).

Add `title`, `description`, and `tags` frontmatter.

# Article style

Include one or more "Real-world pattern:" sections that show practical, copy-paste-ready examples of the concept being discussed. These should demonstrate how the topic is actually used in production code, not just toy examples.

End each article with a "Summary" section — a concise table or bullet list recapping the key concepts covered.

# Planned articles

Planned and published articles are tracked in `PLANNED_ARTICLES.md`. When an article is generated, move it from the Queue to the Published table with its date. When new articles are planned, add them to the Queue.

# Commiting changes

Use git commit with message "added article: <slug>".
