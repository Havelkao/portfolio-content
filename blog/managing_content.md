<!-- # Website one, content zero -->

It is simple really. I thought it was a good idea to have a place where I could show some of my projects and share what I have learned while working on them. Whether I actually will or not is a completely different matter of course. I haven't written any coherent piece of text in ages, much less so any text presenting my own experience. As the bare minimum I intend to post three articles (including this one). That is simply because on the homepage I implemented the `latest` section which only shows the last two articles, as opposed to the `blog` page where every article should be available. I do believe it would be a massive shame not to fully utilize this piece of highly sophisticated code, which is a also a good excuse to see how a javascript formatted code block looks like around here.

```js
import { useStore } from "vue";

const articles = useStore().state.articles.slice(0, 2);
```

Brilliant. Anyways, now that the _why_ is out of the way and the website is finished, here comes the question of how to manage the content. I don't have a ton of experience but I know for sure that there is no way in hell I will be writing HTML. I also know that I do not want to commit content to the same repo where the code is hosted, even though with website of this size it might be acceptable, nor do I want to build a dedicated backend. After a minute or so of googling a nice solution presents itself, which is also a good excuse to see how a formatted quote block looks like around here.

> A headless CMS is any type of back-end content management system where the content repository “body” is separated or decoupled from the presentation layer "head."

Eh, it's alright. A headless CMS seems like great option, at first atleast, but there is the inevitable pain of choosing one (there is way too many) and of course learning the API and implementing the solution. The latter would not be that much of a problem, but after contemplating about the direction I want to take I realize there is in fact no reason for me to use any 3rd party CMS. I am the sole contributor and there is only one "head". I don't need no fancy editors or data models especially if I use `markdown`. What I really need is a place to store some static files which I can fetch on demand.

## Enter Github Pages, again

Heading level 2 looks great. But back to the point. I figured there is no simpler solution for the content/form separation issue then just creating another repository, also deployed to github pages. I guess in this case I could even use a git submodule, but since I'm not too familiar with it and I prefer to keep things as simple as possible, I opted for just including `public/content` in `.gitignore` of the main repo. This is the folder/file structure I ended up with.

```text
portfolio/
├── public/
│   ├── content/
│   │   ├── blog/
│   │   │   ├── articles.json
│   │   │   └── managing_content.md
│   │   ├── images/
│   │   ├── about.md
│   │   └── projects.json
│   └── icons/
└── src/
```

Much cool, very text formatted code block. So how does it work? If it wasn't obvious already I simply fetch whatever I need from the secondary repository. In case I'm offline, I can get the files from the local repo. That's being handled by the following function.

```js
function getURL() {
    let url = "/content";
    if (location.hostname === "havelkao.github.io") {
        url = "https://havelkao.github.io/portfolio-content";
    }
    return url;
}
```

The rest of the process is also fairly simple. On the initial load I fetch whatever is needed from wherever is deemed better as a one time import. That is really just `about.md` for the home page introductory part and `projects.json` with `articles.json` which contain data for the respective pages. Everything else shall be fetched (and cached in vuex store) only when needed. As for the `md` files, those are rendered using [markedjs](https://github.com/markedjs/marked) hooked up to [highlightjs](https://github.com/highlightjs/highlight.js) for code formatting and styled with a modified [github-markdown-css](https://github.com/sindresorhus/github-markdown-css) file.

And that's pretty much it. Congratulations if you made it this far, I most certainly would have given up after reading the first sentence. Anywho, in case you are interested in the actual implementation, you can check out [this very page](https://github.com/Havelkao/havelkao.github.io) together with [this very page content](https://github.com/Havelkao/portfolio-content) ... or don't, that's also allowed! Cheers!

Website one, content one.
