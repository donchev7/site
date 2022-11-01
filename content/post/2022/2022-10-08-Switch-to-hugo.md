---
date: 2022-10-08
image: 'post/2022/Switchingfromgatsbytohugo.png'
title: 'Switching from gatsby to hugo'
slug: switching-from-gatsby-to-hugo
toc: false
tags:
  - Cloud
  - Firebase
---

Last year I [switched](https://donchev.is/post/switching-from-flask-to-gatsby/) to [gatsby](https://www.gatsbyjs.com/) and [firebase hosting](https://firebase.google.com/docs/hosting) from a self made flask app.

Altough I enjoyed the experience, I found myself updating dependencies and plugins and fixing bugs more often than I would like. I also found myself spending more time on the website than I would like.

What I started appreciating about hugo is that it is a **tool**, a single binary that helps you build a static website. It is not a framework or a library or plugin. There is a real benefit in that. It is a tool that you can use to build a website, not a framework that you have to use to build a website. Using Gatsby to achieve the same result as hugo requires also using various software dependencies, so I have to keep not only Gatsby but also all those packages maintained, which resulted in a lot of time spent on the website.

I also have to admit that some of the plugins in the Gatsby ecosystem or of low quality. Updating Gatsby sometimes wasn't possible because a plugin was not compatible with the new version.

With hugo I don't have these problems.

## Templating

One remark I constantly hear is that Gatsby is React based and thus creating a website is **easy**. But this is only true if you define easy as "familiar". I wouldn't say that creating a website with Gatsby or React is **simple**. You need to learn React, JSX, GraphQL, Gatsby, and all the plugins. Since I know this stack, I can say that it is not simple it's **easy** for people with prior knowledge but it is a lot of work to learn for somebody who is new.

Hugo on the other hand is based on HTML, CSS (SCSS), JS and go templates. The only thing you need to learn is go templates (which are very similar to Jinja2 templates which I have used for many years in the past). 

The argument of which to choose boils down to:

1. go templates vs react / jsx / graphql
2. no npm dependencies vs fixing npm dependencies 


For now I choose the "simpler" solution even though I choose the "easier" solution in the past =-)
