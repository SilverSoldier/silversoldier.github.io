---
layout: post
title:  "Presentation using Emacs org-mode"
description: Converting org-mode slides to presentation slides using Org-Tree-Slide.
img:
date: 2018-05-30  +1530
---

# Beautiful and classy presentations using Org-Tree-Slide
Recently, I was on a Windows machine and boy, did I struggle. Everything that I was used to was gone, replaced by GUI. Except Emacs.

The only thing I had used Emacs for, till now, was org-mode.

There came a need to make presentation slides, nothing too formal and I did not want to install either latex (and a bunch of tools associated with it) and was too much of a Computer Science student to even think of using slide-making tools like PowerPoint or LibreOffice Presentation.

And, org-mode came to my rescue. With too many options that in fact, I had to spend quite some time trying out the options to choose the one I needed.

The most helpful link was [How to make non-beamer presentations using org-mode](http://orgmode.org/worg/org-tutorials/non-beamer-presentations.html)

Here are some of the options:
+ [org-reveal](https://github.com/yjwen/org-reveal/):
  + Keeps coming on the top of the list, so I've got to absolutely mention it.
  + Converts org-mode slides to HTML presentations using RevealJS.
  + However, requires a bunch of dependencies that I was not really particular about installing and fixing, so I never ended up trying it.
  + **NEED TO TRY** since it seems to have a bunch of features like slide effects and thumbnails and math equations.
  
+ [org-present](https://github.com/rlister/org-present):
  + Extremely minimalistic presentation slides from org-mode.
  + True to its name, all it does is make the top level headers as the title pages and allows you to move between the slides using left and right arrow keys.
  + Works only in Emacs-GUI mode and is extremely minimalistic.
  
And like Goldilocks, I found the right package: **Org-Tree-Slide** 

+ [org-tree-slide](https://github.com/takaxp/org-tree-slide):
  + Very easy to set up - just package-install org-tree-slide.
  + Also works only in Emacs-GUI mode.
  + The first level headings become titles. Travels through all the nodes of the org-tree uptil the leaf (last level org-mode entry, not the text under it).
  + Shows path in the tree for each slide.
  + Has transition effects for the slides.
  + Extremely classy.
