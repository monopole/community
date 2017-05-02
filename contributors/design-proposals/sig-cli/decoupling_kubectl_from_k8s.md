# Decoupling kubectl from core k8s

Status: Pending

Version: Alpha

Implementation Owner: jregan@google.com

## Motivation

The [kubectl program] is used to configure and control k8s (kubernetes)
clusters.  It's a command line client to the k8s API.  The program
lives in the core [k8s repo].

kubectl depends on various parts of k8s and k8s depends on various
parts of kubectl without any particular constraint or intention.  If a
bug is detected in how kubectl interacts with k8s, there's no
procedure to release a fix that doesn't involve a full release of k8s.

![before decoupling][reposBefore]

This document describes how a clear line will be drawn in the code
between kubectl and the remainder of k8s to leave only intentional
dependencies.  This is a precondition to being able to release kubectl
as a product distinct from the rest of k8s.

![after decoupling][reposAfter]
![ultimate state of repos][reposCandide]

## Proposal

 1. Identify, via `BUILD` file rules, existing good and bad
    dependencies. Good dependencies are on intended, public APIs; bad
    dependencies are otherwise.
 1. Create. delete, move or merge packages to remove bad dependencies,
    and make it impossible to _accidentally_ reintroduce bad
    dependencies.
 1. Adapt CICD tools (and miscellaneous shell scripts) to continue to
    function properly.  In particular, adapt end to end (e2e) test
    framework to test an instance of kubectl (previously released or
    HEAD) against an instance of k8s (likewise, previously released or
    HEAD).  This may require its own proposal if existing framework
    deficient.

### Non-goals

 * Changing policy to release kubectl and k8s at a different cadence.

   For purposes of bootstrapping a new style of e2e testing and
   release, it may be convenient to already have a _decoupled_ kubectl
   in the wild, released per the existing process.  Any new process
   will need rollback state it understands.

 * Moving kubectl to its own repo, outside the [k8s repo].

   This decoupling work is a precondition to such a move, but doesn't
   require a new repo.

## User Experience

The user should see no change other than, it's hoped, faster feature
development (and bug fixes) accruing from the kubectl/k8s decoupling.

## Implementation

The identification of bad dependencies, and the eventual prevention of
their re-introduction will be accomplished via _visibility_ rules
in `BUILD` files.

### Background Facts

 * A _package_ is any directory that contains a BUILD file.
 
 * A `package_group` is a BUILD file term that defines a set of
   packages for use in other rules.
 
 * A _visibility rule_ takes a list of package groups as its
   argument - or one of the pre-defined groups `//visibility:private` or
   `//visibility:public`.
 
 * If no visibility is explicitly defined, a package is _private_ by default.
 
 * kubernetes has explicitly labeled its nearly 2000 packages as
   public - there have been no attempts so far to limit visibility.

 * Violations in visibility cause `make bazel-build` to fail, which in
   turn causes the submit queue to fail.


### Clarify dependence on kubectl

When this work in this section is complete, all dependencies on
kubectl code by non-kubectl code will either be removed or
specifically allowed.

 * A new k8s package will be created called `kubernetes/visible_to`.

   It will have only two files: `BUILD` and `OWNERS`.

 * The only rules in `BUILD` will define package groups.

   These groups will exist only to be used in visibility rules in
   other packages throughout k8s.

 * All kubectl code will switch to private by default.

   This means removal of all package-level policies like
   ```
   package(default_visibility = ["//visibility:public"])
   ```

 * Exceptions to this will be per-target.

   A target will let others depend on it like this:
   ```
   visibility = ["//visible_to:kubectl_client"]
   ```
   Once all these changes are made, all dependencies are
   intentional, albeit both good and bad.
 
 * `OWNERS` file imposes special approval for visibility changes.

   Review will require familiarity with visibility usage, since
   small changes can open large holes.  This is intended to stop
   introduction of new bad depenencies.

 * Bad dependencies labelled for removal.

   Bad deps will be allowed and identified with a special package
   group naming convention to call them out for removal.

 * Bad dependencies removed.

   Code moves as needed to allow only intentional dependence.
   
### Clarify dependence on k8s

When this work in this section is complete, any dependence kubectl has
on k8s will be specifically allowed.

_This requires removing all public exposure from the approximately 2000
package in k8s because a package cannot be both public and declare its
visibility to a specific package._ This task will be done via script.

To avoid turning this into the much more difficult task of identifying
good and bad dependencies throughout kubernetes, the focus will remain
on kubectl.

Specifically, every package in ks8 will define a visibility rule that
allows _one or both_ of the following package groups to see it:

 * kubectl

 * everything minus kubectl

So, a package intended for public consumption will now
define its vis

Once complete, the file `visible_to/BUILD` will define all
allowed reverse dependencies, and the state

As new code isolation projects come along, they can introduce 
If a package allow visibility from

This 

This is the other
When this work in this section is complete, all dependencies on
k8s code will be indentified.


Here the goal is to identify the packages that will be vendored from k8s into kubectl, and possibly other clients like kubeadm, etc, and given them visibility rules that set them apart from the visibility rules of remaining packages.


### k8s isolation (optional)

The work in section determines what is allowed to depend on k8s code.

There's still the problem that 


### Client/Server Backwards/Forwards compatibility

These changes have no effect on backward/forward compatibility.

## Alternatives considered

The only alternative is to continue to live
with the intertwined code.

[k8s repo]: https://github.com/kubernetes/kubernetes
[original proposal]: https://docs.google.com/document/d/1i1vISyLhVcc6skMEgu8idxeL4LrieCCEO77ImZQwNjA/edit#
[kubectl program]: https://kubernetes.io/docs/user-guide/kubectl-overview/
[reposBefore]: reposBefore.svg
[reposAfter]: reposAfter.svg
[reposCandide]: reposCandide.svg
