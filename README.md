# NAME

Dist::Zilla::Plugin::Git::ChangelogFromTrailers - Build the Changes entry for a release from the Changelog trailers of its commits

# VERSION

version 0.002

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
commit message, as [git-interpret-trailers(1)](http://man.he.net/man1/git-interpret-trailers) describes, or in any other
paragraph that holds only trailers.  The plugin writes
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

git reads trailers only from the last paragraph, but a squash merge on GitHub
leaves them in an earlier one.  It moves each `Co-authored-by` trailer of the
pull request into a paragraph of its own at the end, and it joins the messages
of the commits of the pull request into one body.  So the plugin also reads
each paragraph whose first line is a trailer, and whose every line is a trailer
or continues the one above it with leading whitespace.  A paragraph of prose
counts for nothing, although one of its lines starts with the key.

The items are in the order of their commits, the oldest first, and in the
order of the message within a commit.  A value that
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

## after\_release

Write `Changes` of the build to the repository, as
[Dist::Zilla::Role::AfterRelease](https://metacpan.org/pod/Dist%3A%3AZilla%3A%3ARole%3A%3AAfterRelease) asks.  If the plugin made it, run
`git add --intent-to-add` on it, so that `[Git::Commit]` commits it.

Returns nothing.

Dies when `Changes` cannot be written, and with a Git::Wrapper::Exception when
git add fails.

```
$plugin->after_release;
```

## before\_release

Check that the release can write its entry, as [Dist::Zilla::Role::BeforeRelease](https://metacpan.org/pod/Dist%3A%3AZilla%3A%3ARole%3A%3ABeforeRelease) asks.

Returns nothing.

Dies by log\_fatal when no commit since the last release has a trailer, or when this plugin is below `[Git::Commit]` in `dist.ini`.

```
$plugin->before_release;
```

## gather\_files

Add an empty `Changes` to the build if the repository has none, as [Dist::Zilla::Role::FileGatherer](https://metacpan.org/pod/Dist%3A%3AZilla%3A%3ARole%3A%3AFileGatherer) asks.

Returns nothing.

```
$plugin->gather_files;
```

## items

The values of the trailers in the commits since the last release tag, in the order that ["Trailers"](#trailers) describes.  Nothing when the repository has no commit.

Returns Str, the values, as a list.

Dies with a Git::Wrapper::Exception when git tag or git log fails.

```perl
my @items = $plugin->items;
```

## last\_release\_tag

The name of the last release tag: of the tags that are reachable from HEAD and match version\_regexp, the one with the highest version.

Returns Str or Undef, the name of the tag, or undef if no tag matches.

Dies with a Git::Wrapper::Exception when git tag fails.

```perl
my $tag = $plugin->last_release_tag;
```

## munge\_files

Add the entry to `Changes` in the build, as [Dist::Zilla::Role::FileMunger](https://metacpan.org/pod/Dist%3A%3AZilla%3A%3ARole%3A%3AFileMunger) asks.  If no commit has a trailer, leave it as it is.

Returns nothing.

Dies by log\_fatal when the build has no `Changes`.

```
$plugin->munge_files;
```

## trailers\_in(message)

The values of the trailers of ["trailer"](#trailer) in each paragraph of the commit message `$message` that is a block of trailers, in their order.  A paragraph is such a block when its first line is a trailer, and each line after it is a trailer or continues the one above it with leading whitespace.  A continued trailer is joined into one line.  git reads trailers only from the last paragraph, so this finds those that a squash merge left above it.

- message - Str - the whole commit message

Returns Str, the values, as a list.

```perl
my @values = $plugin->trailers_in("Subject\n\nChangelog: A change\n\nCo-authored-by: A. Tester <tester\@test.test>");
```

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

# CONTRIBUTORS

- troglodyne-bot <troglodyne-bot@users.noreply.github.com>
- troglodyne-robot <trogbot@troglodyne.net>

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
