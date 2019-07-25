# Community Wiki

## General

As you may have noticed, this wiki is managed using GitHub.

All the pages are written using [`markdown`](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet) syntax, and new pages shall be added via [Pull Requests](https://help.github.com/en/articles/about-pull-requests).

You will need to understand the basics of Git, as the workflow would be something like:

- Create a [fork](https://help.github.com/en/articles/fork-a-repo) of this repository, under your account.

  > This will allow you to create and push new branches, as you may not be able to do it directly to the main one. You may have to do it only ever once.

- Create a [branch](https://help.github.com/en/articles/about-branches) and add commits with your changes (new pages or corrections)

  > This will allow you to store your changes and "push" them to your fork.

- Create a [Pull Request](https://help.github.com/en/articles/about-pull-requests) that will be reviewed.

  > We surely don't want anybody being able to modify this wiki in an uncontrolled way, and [Pull Requests](https://help.github.com/en/articles/about-pull-requests) allow us to take a look at what you want to change before merging in the "master" branch.

I think GitHub even describe this better than I do: https://guides.github.com/introduction/flow/

## Layout

The wiki layout should be rather simple, but at the same time avoid cluttering.

For this, we should go a `sections` folder, refering to different domains of the BGE.

Within each section will be a main `README.md` file, that should describe what the section is about.

Still within each section, new "entries" should be folders, named using [`kebab-case`](https://en.wikipedia.org/wiki/Letter_case#Special_case_styles). The name should be rather verbose, as people should be able to simply look at it and know if it is in their interest to read it.

Any resource (picture, gif, video, ...) should be within the entrie's folder, and be named `.resources`. You will then be able to refer to your resources by inputing a relative path to this folder.

Summarized:

    wiki/
    ┕ sections/
      ┕ python-api/
        ┕ kx-game-object/ # Entry
          ┝ README.md # Content
          ┕ .resources/ # Data
            ┝ example.blend
            ┕ meh.png

Although, please avoid uploading .blend files, since these are binaries, Git will not store them optimally.

If you are wondering about what can be versionned and what cannot, an easy way is to try to open the file using notepad or a similar program:
If you can somewhat read the content like this, then Git might be able to version it.
If you only see random characters, or maybe your editor complained about the size before opening it, then Git may not like that file.

## Why GitHub?

If you aren't using a Version Control System (VCS for short) such as Git already, then you should start now.

GitHub is designed for collaboration, and Git is a fantastic tool for versionning simple text files (such as source files, but not binary files like executables or blends).

Using GitHub's [Pull Requests](https://help.github.com/en/articles/about-pull-requests) ensures that we can look to assist people with their contributions.

The wiki's implementation is the most simple: bare text files, markdown to make it a bit pretty, some images, done. This will guaranty some coherent styling, to some extent.

TL;DR: Git and GitHub are good for you.
