---
title: "Creating this site"
date: 2021-12-15
---

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;I've been floating around the idea of creating a personal site ever since I got started in tech. The issue being I created an unrealistic goal of writing it from scratch with whatever web framework I was using at the time. I'd spend a few hours setting up the backend then stall out when it came time to build out the frontend. Even at the start, I had zero aspirations of doing web development for any duration of time. So honestly I have no idea why I got this thought in my head.

Fast forward to now, where I'm rather far removed from web development which allows me to disregard current trends. When researching a tool I had a general idea of what I wanted but had little idea what was out there. I quickly came across static site generators and that seemed to fit the bill.

A few requirements:
- simple design
- pages/blogs editable in markdown
- no javascript

A rather quick search led me to [Hugo](https://gohugo.io/), an open-source static site generator written in Go. With a simple content management model and a large library of themes to choose from, this seemed like the best choice. Plus, another tool written in Go is always welcome.

Following their [quick start guide](https://gohugo.io/getting-started/quick-start/) I was up and running in a matter of minutes. Now I'll skip over the hours I spent testing out themes and say I ended up going with [etch](https://github.com/LukasJoswiak/etch). It fit all my requirements above and seemed to fit my use case. 

While developing, hugo comes with it's own webserver with live reloading enabled.
{{< highlight html >}}
✗ hugo server
{{< /highlight >}}

Hugo uses a simple toml/yaml config file for configurations.

{{< highlight html >}}
baseURL = 'https://cbull.dev/'
languageCode = 'en-us'
title = 'Christian Bull'
theme = "etch"

[params]
  copyright = "Copyright © 2021 Christian Bull"
  dark = "on"
  highlight = true

[menu]
  [[menu.main]]
    identifier = "about"
    name = "about"
    title = "about"
    url = "/about/"
    weight = 10
...

[permalinks]
  posts = "/:title/"

[markup.goldmark.renderer]
  # Allow HTML in Markdown
  unsafe = true
{{< /highlight >}}

Custom pages go in the root on the content folder. Blog posts get their own file in the posts directory.
```
content
 ┣ posts
 ┃ ┗ creating-this-site.md
 ┣ _index.md
 ┣ about.md
 ┣ contact.md
 ┗ posts.md
```

Outputs the final site to the public folder.
{{< highlight html >}}
✗ hugo --minify
{{< /highlight >}}

Throw it into a simple docker configuration.
{{< highlight html >}}
FROM nginx:alpine
COPY site/public /usr/share/nginx/html
{{< /highlight >}}

{{< highlight html >}}
✗ docker build -t csbull55/cbull-dev:test .
✗ docker images csbull55/cbull-dev:test
REPOSITORY                    TAG              IMAGE ID       CREATED         SIZE
csbull55/cbull-dev            test             52c1460df6ba   3 seconds ago   26.1MB
{{< /highlight >}}

Outputs a tiny ~25MB image. Although I'm sure that could be reduced further but that's not a priority right now.

{{< highlight html >}}
✗ docker run -p 8080:80 csbull55/cbull-dev:test
{{< /highlight >}}
:)