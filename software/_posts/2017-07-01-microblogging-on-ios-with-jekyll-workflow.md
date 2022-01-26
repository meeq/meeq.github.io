---
layout: post
type: article
title: Microblogging on iOS with Jekyll using `Working Copy.app` and `Workflow.app`
image: /assets/microblogging/micropost-ui.png
tags:
  - Blogging
  - Microblogging
  - Static-Site-Generator
  - Jekyll
  - Workflow-app
  - Working-Copy-app
---
*Editor's Note: This article is outdated and may no longer be relevant. The microblog feature on this site has since been removed.*

This website is generated from [Markdown](https://daringfireball.net/projects/markdown/syntax) using [Jekyll](https://jekyllrb.com/) and [GitHub Pages](https://pages.github.com/), which takes a [Git repository](https://git-scm.com/) and feeds it into a [static site generator](https://davidwalsh.name/introduction-static-site-generators). This approach has numerous advantages which make management and maintenance of this site cheap and simple, but it also has limitations compared to other blogging platforms, especially in terms of post creation. To overcome this shortcoming and facilitate quick posts from my phone, I automated the process of posting using a combination of [`Workflow.app`](https://workflow.is/)([App Store Link \[Free\]](https://itunes.apple.com/us/app/workflow-powerful-automation-made-simple/id915249334?mt=8)) and [`Working Copy.app`](https://workingcopyapp.com/)([App Store Link \[$14.99\]](https://itunes.apple.com/us/app/working-copy-powerful-git-client/id896694807?mt=8)).

![App Icons for `Working Copy.app` and `Workflow.app`]({{ site.url }}/assets/microblogging/app-icons.png)

In order to [publish a post using Jekyll](https://jekyllrb.com/docs/posts/):

* A Markdown (or Textile or HTML) file is created in a `_posts` directory
* Any images or other resources referenced by the post are saved to the `assets` directory
* The new post file and any associated assets are committed to the Git repository
* The commit is pushed to the remote Git server up on GitHub
* The magic unicorns in the sky will ingest the content to produce a static file directory, which they generously serve for free [(within limits)](https://help.github.com/articles/what-is-github-pages/#usage-limits)

### Git on iOS

![`Working Copy.app` screenshots]({{ site.url }}/assets/microblogging/working-copy-ui.png)

It is possible to accomplish all of this on iOS using the awesome `Working Copy.app` by Anders Borum, which is an outstanding Git client for iPhone/iPad with file editing and support for [iOS inter-app communication using `x-callback-url`](http://x-callback-url.com/). You will need unlock the full functionality of the app with the one-time In-App Purchase (or [buy the Enterprise Version](https://itunes.apple.com/us/app/working-copy-enterprise/id965019520?mt=8) which is already unlocked) in order to push commits to Git remotes.

The multi-step process of creating and deploying a new post manually can be somewhat cumbersome and tedious, especially for [microblogging](https://en.wikipedia.org/wiki/Microblogging) when I'm out and about, which typically involves a short bit of text and an optional photo or video. For these "micro"-posts, I would prefer a streamlined interface with minimal input similar to Twitter's tweet UI:

![Twitter's tweet UI]({{ site.url }}/assets/microblogging/tweet-ui.png)

### Automation on iOS

Thanks to a clever automation tool for iOS called `Workflow.app`, it is possible to create a reasonable facsimile of this simple interface for typing a message and attaching a media file, which get run through the workflow to create a post, commit it, and deploy it to the remote server.

![Microblog post UI]({{ site.url }}/assets/microblogging/micropost-ui.png)

Sure, it's not exactly pretty, but it gets the job done without ever opening Xcode.

[`Workflow.app` allows you to string together "action bubbles" to create scripts](https://workflow.is/docs/how-workflows-work). For example, here's a workflow that scans a [QR code](https://en.wikipedia.org/wiki/QR_code) using the camera, copies the parsed text to the clipboard, and checks to see if it's a web address, opening the URL in Safari if so:

![Workflow example for scanning a QR code]({{ site.url }}/assets/microblogging/workflow-ui.png)

There are a ton of [built-in actions](https://workflow.is/docs/exploring-the-actions-pane), plus the ability to reach out to other [apps using x-callback-url](http://x-callback-url.com/apps/) or even [hit web APIs](https://workflow.is/docs/taking-advantage-of-web-apis). `Working Copy.app` does not have any built-in actions for `Workflow.app`, but it offers a [wide selection of callback URL schemes](https://workingcopyapp.com/url-schemes.html) for interacting with the app through automation.

### Building the workflow

Here's a breakdown of the entire workflow:

#### Configuration

* Define the repository name to manipulate with `Working Copy.app`
* Define the `Working Copy.app` callback security key
* Define the base URL of the final deployed page

#### Gathering information

* Ask for text input to populate the post body
* Define the post filename based on the current date/time
* Define the [post's front-matter](https://jekyllrb.com/docs/frontmatter/) based on the current date/time
* Define the base filename for any post attachments based on the current date/time
* Ask for any associated media from the camera, saved photos, or the clipboard:
  * **Take Photo**: Run `Take Photo` action, capturing a single photograph
  * **Record Video**: Run `Take Video` action, capturing a video using the camera
  * **Choose Photo/Video**: Run `Select Photos` action, capturing a single image or video item from Saved Photos
  * **Use Clipboard**: Run `Get Clipboard` action, capturing the clipboard contents
  * **Text-Only**: Do nothing

#### Git housekeeping

* To ensure that the post's commit will push cleanly, pull the latest commits from the Git remote with `Working Copy.app`

#### Media handling

* Inspect the type of media to attach:
  * **Image/Video**:
    * Define the attachment's filename
    * Copy the attachment file to the clipboard (so that it can be transferred to `Working Copy.app`)
    * Write the attachment file to the Git repository staging area with `Working Copy.app`
    * Further inspect the media type of the attachment:
      * **Video**:
        * Make an animated GIF of the video
        * Copy the GIF to the clipboard (so that it can be transferred to `Working Copy.app`)
        * Define the GIF's filename
        * Write the GIF file to the Git repository staging area with `Working Copy.app`
        * Define additional post front-matter to declare the GIF as the post's canonical image
        * Define the Markdown text that embeds the GIF into the post and links to the original video
      * **Image**:
        * Make a scaled version of the image for embedding
        * Copy the scaled image to the clipboard (so that it can be transferred to `Working Copy.app`)
        * Write the scaled image file to the Git repository staging area with `Working Copy.app`
        * Define additional post front-matter to declare the original image as the post's canonical image
        * Define the Markdown text that embeds the scaled image into the post and links to the original image
    * Define the number of files to commit
  * **Other/None**:
    * Define an empty text value for the additional post front-matter
    * Define an empty text value for the attachment embedding
    * Define the number of files to commit

#### Post creation

* Assemble the post file contents from previously-defined variables:
  * front-matter
  * user-inputted post message
  * optional inline image reference
* Write the post file contents to the Git repository staging area with `Working Copy.app`

#### Git finalization

* Define a hard-coded commit message
* Create a commit from all of the post files in staging to the Git repository with `Working Copy.app`
* Push the commit to the Git remote server with `Working Copy.app`

You can see this workflow in action to get a feel for the posting process:

<iframe width="320" height="569" src="https://www.youtube.com/embed/4AAM_kvbzW4"></iframe>

### Future concerns

There are a few details of this Jekyll-based microblogging endeavor that I will have to solve for at some point in the future:

* The slowest part of this process is the push to the remote. Most of this problem can be blamed on my slow upload speed, but the situation could be improved by optimizing or not-including the "original media".
* The `jekyll-paginate` plugin does not support filtering by category, so there is currently no built-in way of paginating these micro-posts separately from long-form posts. [GitHub Pages does not support custom plugins.](https://jekyllrb.com/docs/plugins/)
* [GitHub has a maximum file size of 100MB](https://help.github.com/articles/working-with-large-files/#conditions-for-large-files), so long videos cannot be posted. A viable alternative is to post them to YouTube and link/embed them in a post.
* [GitHub has a recommended repository size limit of 1GB](https://help.github.com/articles/what-is-my-disk-quota/#file-and-repository-size-limitations). Eventually the photos and videos will exhaust this limit. I may have to optimize the "original media" that is linked in the post so that it is not so massively huge. For now, I've got plenty of headroom.

### Wrapping up

If you want to use this workflow for your own microblogging with Jekyll, or modify it for other purposes, you can [download a copy of the workflow]({{ site.url }}/assets/microblogging/μblog-Post.wflow) and import it into `Workflow.app`. During the import process, you will be asked to configure the `Working Copy.app` callback key and repository name, as well as the GitHub Pages domain name for your site.

I hope this was helpful and/or interesting. I made this workflow because we recently got a new kitten around the house and I wanted to optimize my ability to quickly share all of her antics, as well as make quick posts about all of the other goings-on around the farm.

![Cat tax]({{ site.url }}/assets/microblogging/cat-tax.jpg)

Thanks for reading.
