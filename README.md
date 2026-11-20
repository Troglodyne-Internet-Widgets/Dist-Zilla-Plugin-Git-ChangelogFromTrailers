# NAME

Dist::Zilla::Plugin::Git::ChangelogFromTrailers - Build the Changes entry for a release from the Changelog trailers of its commits

# VERSION

version 0.001

# SYNOPSIS

In `dist.ini`, above `[Git::Commit]`:

```
[Git::ChangelogFromTrailers]
format = %v %{yyyy-MM-dd}d %P

[Git::Commit]
allow_dirty = Changes
```

Each commit that is worth a line in `Changes` says so in a trailer:

```
git commit -m 'Handle empty input' --trailer 'Changelog: Fix the frobnicator when the input is empty'
```

# DESCRIPTION

Two branches that each add a line to the top of `Changes` conflict when the
second one merges.  Two branches that each say what they change in their commit
messages do not.

This plugin reads the `Changelog` trailers of the commits since the last
release tag.  A trailer is a `Key: value` line in the last paragraph of a
commit message, as [git-interpret-trailers(1)](http://man.he.net/man1/git-interpret-trailers) describes.  The plugin writes
one entry for the trailers at the top of `Changes` in the build.  When the
release succeeds, it writes the same entry to `Changes` in the repository, so
that `[Git::Commit]` commits it with the release.

The entries that are in `Changes` already stay as they are.  The plugin adds
the new entry above the first line that starts with a version, or at the end
of a `Changes` that has no entry yet.  If the repository has no `Changes`,
the plugin makes one, with the line `Revision history for` and the name of the
distribution.

The last release tag is the highest version among the tags that are reachable
from `HEAD` and match ["version\_regexp"](#version_regexp).  If no tag matches, every commit
counts.

The plugin is a subclass of [Dist::Zilla::Plugin::NextRelease](https://metacpan.org/pod/Dist%3A%3AZilla%3A%3APlugin%3A%3ANextRelease).  It takes the
options `format`, `time_zone`, `filename`, `update_filename` and
`user_stash` from that plugin, with the same meaning.  Use it instead of
`[NextRelease]`, not with it.

## Trailers

The value of each trailer is the text of one item, so a commit can have more
than one.  The key does not depend on case.  A trailer that continues on the
next line is joined into one line.

The items are in the order of their commits, the oldest first.  A value that
is the same as an earlier one is left out, so a merge commit can repeat the
trailers of its branch.  A commit with no trailer adds nothing.

GitLab also reads a `Changelog` trailer, but there the value is a category,
such as `added`, and the subject of the commit is the text.  This plugin uses
the value as the text.

## Order in dist.ini

The `after_release` of this plugin must run before that of `[Git::Commit]`,
or the release commit does not contain the new `Changes`.  Dist::Zilla runs
them in the order of `dist.ini`.  So put this plugin above `[Git::Commit]`.
If it is below, the release stops before it starts.

## The version

[Dist::Zilla::Plugin::Git::NextVersion](https://metacpan.org/pod/Dist%3A%3AZilla%3A%3APlugin%3A%3AGit%3A%3ANextVersion) gives each release the version of the
last release tag plus one.  Remove `version` from `dist.ini` and add:

```
[Git::NextVersion]
first_version  = 0.001
version_regexp = ^v?(\d+\.\d+)$
```

Use the same `version_regexp` for both plugins.

# ATTRIBUTES

## trailer

The key of the trailers to read.  The default is `Changelog`.

## version\_regexp

The pattern that a tag must match to be a release tag.  The first capture group
is the version.  The default matches `1.002` and `v1.002`.

# METHODS

## last\_release\_tag

Return the name of the last release tag, or undef if there is none.

## items

Return the values of the trailers in the commits since the last release tag, in
the order that ["Trailers"](#trailers) describes.

## gather\_files

Add an empty `Changes` to the build if the repository has none.

## munge\_files

Add the entry to `Changes` in the build.  If no commit has a trailer, leave
it as it is.

## before\_release

Stop the release if no commit since the last release has a trailer, or if this
plugin is below `[Git::Commit]` in `dist.ini`.

## after\_release

Write `Changes` of the build to the repository.  If the plugin made it, run
`git add --intent-to-add` on it, so that `[Git::Commit]` commits it.

# SEE ALSO

Please see those modules/websites for more information related to this module.

- [Dist::Zilla::Plugin::NextRelease](https://metacpan.org/pod/Dist%3A%3AZilla%3A%3APlugin%3A%3ANextRelease)
- [Dist::Zilla::Plugin::Git::NextVersion](https://metacpan.org/pod/Dist%3A%3AZilla%3A%3APlugin%3A%3AGit%3A%3ANextVersion)
- [Dist::Zilla::Plugin::Git::Commit](https://metacpan.org/pod/Dist%3A%3AZilla%3A%3APlugin%3A%3AGit%3A%3ACommit)
- [CPAN::Changes::Spec](https://metacpan.org/pod/CPAN%3A%3AChanges%3A%3ASpec)

# BUGS

Please report any bugs or feature requests on the bugtracker website
[https://github.com/Troglodyne-Internet-Widgets/Dist-Zilla-Plugin-Git-ChangelogFromTrailers/issues](https://github.com/Troglodyne-Internet-Widgets/Dist-Zilla-Plugin-Git-ChangelogFromTrailers/issues)

When submitting a bug or request, please include a test-file or a
patch to an existing test-file that illustrates the bug or desired
feature.

# AUTHORS

Current Maintainers:

- George S. Baugh <george@troglodyne.net>

# COPYRIGHT AND LICENSE

Copyright (c) 2026 Troglodyne LLC

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:
The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
