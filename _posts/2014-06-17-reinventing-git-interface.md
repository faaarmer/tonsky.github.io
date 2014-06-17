---
layout: post
title: Reinventing GIT interface
category: blog
summary: "A set of recommendations on how GIT UI can be improved not on a cosmetic, but on a very fundamental level. We aim at making GIT more usable, powerful and easier to learn by radically simplifying its interface and keeping compartibility with the implementation"
---

I have a long love-hate relationship with GIT. Despite solving all my problems with version control and having a technically excellent implementation, the whole experience sucks constantly. I’ve tried it all: official CLI, non-official CLIs, gitk, third-party GUIs. It always strikes me&nbsp;— how, while having this brilliant model, the model that’s plain dead simple at its heart, the one about DAG of commits,&nbsp;— how _every other interface to it_ manages to ruin it completely?

And it’s not just me&nbsp;— people are complaining about GIT complexity on the internet _a lot_. I’ve seen my fellow developers needing a crash course into even basic GIT usage, and many were still puzzled by some “advanced” stuff like rebasing months after starting using it. It’s easy to see how wrong it is: basic GIT concepts can be explained in a matter of half an hour _on a whiteboard_, yet actually touching GIT _on a computer_ takes you _weeks_ to get used to. Sure, it feels nice to be accounted for something like a “GIT guru”, but, the thing is, I do not know that much about GIT’s command-line arguments or advanced tricks. I just see that things are much simpler than they seem to be.

<div class="anim_cont">
  <img src="louie_ck_parking_sign@2x.jpg" style="width: 554px; height: 310px;" />
  <div class="label">Typical GIT cheatsheet</div>
</div>

So, what shall we do with it? I suggest we get to the core, remove everything accidental, including any UIs and commands, and keep just fundamental information model. And then we start building from there.

## GIT model

At the very core, GIT is about keeping history of a directory. It can store all of the directory content _as a snapshot_, and it can store a lot of snapshots effectively. So, as you keep changing your project, you are making these snapshots and GIT puts them to its local database. You can later restore any of these snapshots, and it will bring your directory’s content exactly as it was back then. Snapshots are called _commits_, and we will stick to that term too.

Besides directory’s content and some metadata like author and timestamp, each commit also remembers link to its parent commit. This is important because it helps tracking casuality. History in GIT is non-linear: there may be several versions of a project co-exisiting in parallel. History forms directed acyclic graph (DAG), a tree of commits with splits (branching) and joins (merges).

Notice that commits and DAG are self-sufficient. They live without branches or remore repositories or stage index or whatever. It’s also important to remember what GIT is calling “a branch” has nothing to do with branches in graph terms. GIT’s branch is just a pointer to some commit, exactly like a tag is. To avoid confusion, we’ll call them _branch pointers_.

## Grow-only repository

It’s all fun and games, but here comes a chance to make the first serious improvement. You may have noticed that GIT warns you _a lot_. Rebase is dangerous, headless state is dangerous, don’t do <nobr><code>push -f</code></nobr>, are you sure, think twice, you’re not supposed to do that, stuff like that. The idea was, I believe, not to prevent any harm, but to help use GIT as its creators intended it to be used. Because, the fact is, you cannot really destroy anything by doing any “potentially dangerous” operations like `rebase` or <nobr><code>commit --amend</code></nobr>. GIT repo is immutable, it only grows, but never shrinks. You can add new snapshots to the repo, but you cannot remove old ones. You cannot change contents of a file once it has been snapshotted, but you can add a new version of it. When you rebase, you do not “move” any commits, neither are you “overwriting” history. You actually create several _new_ commits, keeping old ones intact. You can still reference them or even checkout them by SHA. Even deleting unmerged branch just removes branch pointer, keeping actual commits exactly where they were.

Understanding there’s no harm to be done eases things a lot. It encourages experimenting, which is one of the most effective ways to learn new things and to get familiar with them. Imagine this: you start rebase, mess things up, hit <nobr><code>Ctrl+Z</code></nobr> several times and try again. Good news is, we can implement Undo/Redo given _just ordinary GIT implementation_. GIT already stores all the possible states our repo was in. All Undo/Redo will have to do is to shuffle some branch pointers here and there.

## Working with working copy

Another step towards safe GIT experience is working copy management.
Working copy is just a term for all changes you’ve made to the repo directory and haven’t commited yet, so GIT does not know nothing about them and is not managing them in any way. One sad case where you _may_ lose your work is when you have “dirty” working copy and want to perfom _any other operation_ with GIT. It’s a very common use-case, and what GIT recommends is to create temporary work-in-progress commit or stash current changes and return back to them later. GIT will, in fact, warn you and refuse to do anything before you get your working copy clean. This is very irritating.

As we’re building our (imaginary) brave new GIT interface, let’s make some principles and stick to them. Here’s the first one: never bother user with warnings, and never get in a user’s way. What he wants, he should be able to do. But we cannot lose the user’s data either. So what I propose is to, as soon as you need clean working copy, convert current work in progress into “WIP” commit automatically. It saves the user manual labour of commiting or stashing, and keeps unfinished work safe and accessible. The overall GIT experience should feel much  smoother and hussle-free.

<div class="anim_cont">
  <div class="anim" style="background-image: url(wip@2x.png);"></div>
  <div class="label">Switching branches while having dirty working copy (hover to animate)</div>
</div>

Unification of working copy and commits brings another major win: it simplifies mental model and brings consistency to working copy interactions. GIT is built around commits manipulation, so it’s very logical and consistent to being able to apply all its tools to working copy too. Under the hood, working copy may be treated differently, but for a user there’s no point to be aware of that distinction. From a user’s standpoint, we’ve just removed the concept of working copy altogether, leaving him with the very basic idea&nbsp;— everything is a commit.

This may seem small, but there’re big consequences for GIT usage patterns. For example, commiting becomes much more trivial, as you always have WIP commit ready. <nobr><code>git commit</code></nobr> will become, in fact, just a commit renaming. Instead of staging changes to index, we’ll split WIP commit into two&nbsp;— WIP and STAGED, and then rename STAGED to something official. This brings very important addition to the table&nbsp;— ability to explicitly checkout STAGED version. GIT’s index won’t allow you that.

<div class="anim_cont">
  <div class="anim" style="background-image: url(commit@2x.png);"></div> <div class="anim" style="background-image: url(staging@2x.png);"></div>
  <div class="label">Commit by rename, stage by split (hover to animate)</div>
</div>

By leaving the user with just one concept that everything is a commit, we put him in a very favourable position&nbsp;— all he needs to learn is how to interact with commits. If he knows that, he’ll also be able to manipulate working copy and to stage changes. We simplified things a lot because in traditional GIT these operations are done via completely separate set of commands.

## Delta algebra

As we more or less know how to commit changes, let’s move to the second most essential thing one can do with GIT repo&nbsp;— delta manipulation. Kind of advanced stuff, yet it occurs in everyday GIT usage nonetheless. But before discussing that, let me introduce you to commit’s delta-snapshot duality.

As we’ve already learned, commit is a snapshot of repo directory at some point in time. This is technically correct (this is how GIT stores commits internally), and this is how commits are used for checkouts. But that’s not the only way one can look at commits. Each commit can also be viewed as delta (or diff, change) of parent’s snapthot and its own. If commit B is based on A, then `delta(B) := diff(snapshot(B), snapshot(A))`. Deltas are derived, they are not directly stored by GIT, but rather calculated on the fly when needed. Also, merge commits cannot be directly expressed as deltas because they are based on more than one commit.

So, we can view a GIT repository not only as a series of periodical backups, but also as a series of changes applied on top of one another. Deltas are easier to comprehend and more directly represent “work done” (you usually think in terms of what was changed rather than entire repo state). They also enable a rich set of interactions with repo we’ll call “delta algebra”.

Main operations of delta algebra are: combine deltas (squash), split delta into two, reverse delta. We’ll also need some brigde to/from commits, specifically getting delta from commit and creating commit from delta. Given that, all sort of GIT magic can be expressed: rebase, cherry-pick, revert, commit reordering. Important thing to understand is that they all are based on a very basic delta operations&nbsp;— in essence, we always take _these_ changes, combine/<wbr>split/<wbr>reverse/<wbr>reorder them, and then apply them _there_. You won’t need nothing more than that.

<div class="anim_cont">
  <div class="anim" style="background-image: url(squash@2x.png);"></div>
  <div class="label">Squash operation (hover to animate)</div>
</div>

The main difference from GIT is that you expose basic operations instead of high-level shortcuts. Working with fundamental mechanics brings power (there’re more ways to combine basic operations), predictability (as each individual operation is simpler) and eases the learning curve (you’ll be learning the correct mental model from the start).

<div class="anim_cont">
  <div class="anim" style="background-image: url(rebase@2x.png);"></div>  <div class="anim" style="background-image: url(cherry-pick@2x.png);"></div>
  <div class="label">Using direct manipulation to imitate GIT rebase and cherry-pick (hover to animate)</div>
</div>

Another important aspect is to make manipulations _direct_ and _visual_. This always bothers me even in best GIT UIs&nbsp;— you see commit tree, you _know_ how you want to rearrange commits, but there’s always that next, very unfortunate step to figure out how to express that in terms of command-line GIT commands, which, in its turn, has no direct mappings to your intentions. What proper UI should enable is to select, drag, copy and shuffle commits and branch pointers, including HEAD, _directly on the DAG tree_. Experience should be much like dragging points in vector graphics editor.


<div class="anim_cont">
  <div class="anim" style="background-image: url(moving-head@2x.png);"></div>
  <div class="label">Moving HEAD pointer to imitate GIT checkout (hover to animate)</div>
</div>


## Staying connected

Of course GIT is a social tool. While there’s (usually small) point in using it just for yourself, most people use it to collaborate. GIT model exposes concept of “remote repositories” and a set of commands to communicate with them: `fetch`, `pull`, `push` and local-remote branches facilities. This is where GIT model is a little bit too flexible. But first let’s talk about power of decisions and introduce one more principle.

In some situations, user’s decision is required. Only a human can bring sense and meaning to work, and computer’s job is to provide tools for that. But there’re situations where software asks user about the matters that do not actually require any decision to make. This is dull and tiresome to do operations which add nothing to the job, but are only required for software to continue to run. Sometimes these situations can be easily spotted (like annoying UAC popups), sometimes people are so used to them they think there’s nothing wrong with it (like manual install of bugfix releases), but the criteria is always simple: do I really have a choice? Will my work really, in a qualitative sense, depend on decision I’d make?

So, our next UI principle will be: do not ask user when his decision is not required. Applying this to GIT, we should remove any manual branch syncing stuff. Our GIT client will always stay online, and will always sync local branches state with remotes. When you create a branch, it gets immediately pushed and visible for all other clients connected to the same repository. If someone else created or advanced any of the branches, it’ll become immediately visible for you. Not a single button click required. Sure, situations when you’ve done changes to the branch and someone else’ve done the same will occur&nbsp;— there’s nothing wrong with that&nbsp;— but these are exceptional cases and in that case two branches will be displayed, local and remote.

There’s a solid foundation for such “always online” experience. <nobr><code>git fetch</code></nobr> is already always safe to do. As we remember, GIT never changes or modifies commits, but always creates new ones. So there’s no reason this couldn’t be done automatically (by some GIT clients it actually _is_ done automatically already). Creating local branch to track remote one is purely mechanical thing to do&nbsp;— this is required by current GIT implementation, but a human has nothing _to decide_ here. The only non-trivial situation is missynced branches, which we leave for user’s decision, because they _do require_ user’s decision.

This is another huge win for us: we’ve removed a whole school of manual syncing commands besides just setting up remote’s URL. No questions to ask, nothing to learn, no place to make a mistake, it just works. And we haven’t really lost anything here because we’ve removed only mechanical operations, ones which do not require user’s decision. From now on, you’ll just commit locally and, magically, everybody else will be able to see your changes. Feels like Dropbox, and this is a big thing. You’ll love that ”always online” experience, I promise.

Remember that network connection is _not_ required for our client to work properly. Offline experience is more or less the same, but when you’re connected, you’ll have less buttons to click.

## Looks matter

I do believe that the best way to observe and understand such a complex thing as DAG of commits is a visual representation. Command-line just don’t cut it. There’s a bad reputation about lousy GIT GUIs in a programmer’s world, and they may have a thing. Current GIT clients do limit their users to small subset of basic GIT operations and rely on a manual command-line mode for all non-trivial cases. In traditional GUIs, you don’t win that much, therefore there’s little motivation to go for GUI in the first place. We, on the other hand, are talking about full-featured GIT client able to cover _all_ use-cases that were previously possible only in a command-line. We also bring direct manipulation and Undo/Redo to the picture which is a significant improvement over what we could _ever_ have in console.

So, it has to be visual, what else? Nothing revolutionary here, just a few tweaks to the current state of affairs:

<div class="anim_cont">
  <img src="overview@2x.png" style="width: 554px; height: 400px; padding: 10px; background-color: white; " />
  <div class="label">Visualisation of DAG tree (static screenshot)</div>
</div>

- Branch always occupies vertical column, never bends right or left;
- Commits are colored not by branch color (we keep branches visually separated by previous rule), but by author. Author matters&nbsp;— in a team, you usually either look for a commits by specific teammate (to review or merge), or you’re trying to understand who wrote specific piece;
- Merge commits have a different, much subtle look, because they are not an effort per se, but a place where two other efforts join;
- Parents of the current HEAD highlighted, effectively slightly hiding future and parallel commits;
- Branch labels on the left help scan what you are interested in: branches or commit messages.


## Remote interface

There’s one single feature that still makes CLI clients look so appealing: ability to work on another machine. Proxying CLI is ubiquotus, proxing GUI is uncommon and still awkward. Can we possibly answer to that?

Thing is, we actually can. There’s one way to proxy GUI applications that is widely used and, if not as smooth as local apps, at least pretty usable. It’s web. Applications controlled via web browser can be run anywhere you can connect to, yet can have almost all the benefits of native GUI apps.

This is the way all modern apps should be written (if “app as a service” model cannot be used): they start web server and open a browser window, providing all UI though JS and HTML. And I’m not talking about node-webkit nonsense here which combines downsides of a web app (limited system integration) with downsides of a native app (cannot be accessed remotely). No, just a regular web server, honest HTTP port, regular browser window. When used locally, experience is the same as with local app, but we get remote execution, literally, for free. With current state of web technology it won’t be any harder (and may, in fact, sometimes be much easier) to develop apps that way. 

## This is awkwardly long, let’s sum it up

We gave a set of recommendations on how GIT UI can be improved not on a cosmetic, but on a very fundamental level. Our concept makes GIT more usable, powerful and easier to learn mainly by unifying redundant concepts and adding some new features. Here they are:

- Remove warnings as GIT repo is immutable and cannot lose data
- Provide undo/redo
- Everything is a commit. Treat working copy and staging area as commits, allow regular commit operations to be applied to them
- Provide basic delta manipulations interface
- Direct interaction with visual DAG tree
- Provide always online experience, automatically sync branches with remotes
- Improve looks of DAG tree by providing better coloring and layout
- Make it web app in order to make it accessible from remote machines via browser

In spite of the fact that all this sounds revolutinary, it’s important that we can keep full compatibility with regular GIT repositories and allow different GIT clients to be used together.

I also ask not to discard all this nonsense right away, but at least give it a fair round of thought. My recommendations, if applied in their entirety, can radically change GIT experience. They can move GIT from advanced, hacky league to the mainstream and enable adoption of VCS by very wide groups of people. While this is still a concept and obviosly requires a hell of a job to become reality, my hope is that somebody working on a GIT interface can borrow from these ideas and we all will get better GIT client someday.

<style type="text/css">
  .anim_cont { width: 1088px; text-align: center; margin: 2em 0 2em -272px; }
  .anim { width: 400px; height: 250px; background-size: 100%; background-color: white; border-radius: 3px; margin: 0 6px; display: inline-block; }
  .label { text-align: center; font-size: 12px; font-style: italic; }
  img { border-radius: 3px; }
</style>

<script type="text/javascript">

  function animate(el) {
    if (el.animated) {
      el.offset = (el.offset || 0) - 250;  
      setTimeout(function() { animate(el); }, 700);
    }
    el.style.backgroundPositionY = el.offset + "px";
  }

  var els = document.getElementsByClassName("anim");  
  for (var i=0; i<els.length; ++i) {
    var el = els[i];
    el.onmouseover = function() {
      if (!this.animated) {
        this.animated = true;
        animate(this);
      }
    }
    el.onmouseout = function() {
      this.animated = false;
      this.offset = 0;
      animate(this);
    }
  }
</script>
