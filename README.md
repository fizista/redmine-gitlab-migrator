Redmine to Gitlab migrator
==========================

[![Build Status](https://travis-ci.org/redmine-gitlab-migrator/redmine-gitlab-migrator.svg?branch=master)](https://travis-ci.org/redmine-gitlab-migrator/redmine-gitlab-migrator)

Migrate code projects from Redmine to Gitlab, keeping issues/milestones/metadata

Does
----

- Per-project migrations
- Migration of issues, keeping as much metadata as possible:
  - redmine trackers become tags
  - redmine categories become tags
  - issues comments are kept and assigned to the right users
  - issues final status (open/closed) are kept along with open/close date (not
    detailed status history)
  - issues assignments are kept
  - issues numbers (ex: `#123`)
  - issues/notes authors
  - issues/notes original dates, but as comments
  - issue attachements
  - issue related changesets
  - issues custom fields (if specified)
  - relations including children and parent (although gitlab model for relations is simpler)
  - keep creation/edit dates as metadata
  - remember who closed the issue
  - convert Redmine's textile format issues to GitLab's markdown
  - possible to map to different users in GitLab
- Migration of Versions/Roadmaps keeping:
  - issues composing the version
  - statuses & due dates
- Migration of wiki pages including history:
  - versions become older commits
  - author names (without email addresses!) are the author/committer names

Does not
--------

- Migrate users, groups, and permissions (redmine ACL model is complex and
  cannot be transposed 1-1 to gitlab ACL)
- Migrate repositories (piece of cake to do by hand, + redmine allows multiple
  repositories per project where gitlab does not)
- Migrate the whole redmine installation at once, because namespacing is different in
  redmine and gitlab
- Archive the redmine project for you
- Keep "watchers" on tickets (gitlab API v3 does not expose it)
- Keep dates/times as metadata
- Keep track of issue relations orientation (no such notion on gitlab)
- Migrate tags ([redmine_tags](https://www.redmine.org/plugins/redmine_tags)
  plugin), as they are not exposed in gitlab API

Requires
--------

- Python >= 3.4
- gitlab >= 7.0
- redmine >= 1.3
- pandoc >= 1.17.0.0
- API token on redmine
- API token on gitlab
- No preexisting issues on gitlab project
- Already synced users (those required in the project you are migrating)

(Original version was developed/tested around redmine 2.5.2, gitlab 8.2.0, python 3.4)
(Updated version was developed/tested around redmine 2.4.3, gitlab 9.0.4, python 3.6)


Let's go
--------

You can or can not use
[virtualenvs](http://docs.python-guide.org/en/latest/dev/virtualenvs/), that's
up to you.

Install it:

    pip install redmine-gitlab-migrator


(or if you cloned the git: `python setup.py install`)

You can then give it a check without touching anything:

    migrate-rg issues --redmine-key xxxx --gitlab-key xxxx \
      <redmine project url> <gitlab project url> --check

The `--check` here prevents any writing , it's available on all
commands.

    migrate-rg --help

Migration process
-----------------

This process is for each project, **order matters**.

### Create the gitlab project

It doesn't neet to be named the same, you just have to record it's URL (eg:
*https://git.example.com/mygroup/myproject*).

### Create users

Manual operation, project members in gitlab need to have the same username as
members in redmine. If you can't use same username in gitlab, e.g. migrating to
gitlab.com, when migrating issues you can create a mappings file with yaml format,
mapping redmine login to gitlab login, with

    --user-dict <user dict file>

Every member that interacted with the redmine project should be added to the
gitlab project. If a corresponding user can't be found in gitlab, the issue/comment
will be assigned to the gitlab admin user.

### Migrate Roadmap

If you do use roadmaps, redmine *versions* will be converted to gitlab
*milestones*. If you don't, just skip this step.

    migrate-rg roadmap --redmine-key xxxx --gitlab-key xxxx \
      https://redmine.example.com/projects/myproject \
      http://git.example.com/mygroup/myproject --check

*(remove `--check` to perform it for real, same applies for other commands)*

### Migrate issues

    migrate-rg issues --redmine-key xxxx --gitlab-key xxxx \
      https://redmine.example.com/projects/myproject \
      http://git.example.com/mygroup/myproject --check

Note that your issue titles will be annotated with the original redmine issue
ID, like *-RM-1186-MR-logging*. This annotation will be used (and removed) by
the next step.

If you don't have direct access to the gitlab machine, e.g. migrating to gitlab.com,
and you want to keep redmine id, use --keep-id, it will create and delete issues in
gitlab for each id gap in redmine project, and won't create issues with different title.
If you have many issues in your redmine projects, it will be a slow process.

    --keep-id

At least redmine 2.1.2 has no closed_on field, so you have to specify the names of the states which define closed issues.
defaults to closed,rejected

    --closed-states closed,rejected,wontfix

If you want to migrate redmine custom fields (as description), you can specify

    --custom-fields Customer,ZendeskIssueId

If you're using SSL with self signed cerificates and get an *requests.exceptions.SSLError: [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed (_ssl.c:600)* error, you can disable certificate validation with

    --no-verify

Migrate issues get all users in gitlab. If you have many users in your gitlab, e.g. migrating
to gitlab.com, it will be a slow process. You can use --project-members-only to query
project members instead of all users, if corresponding user can't be found in project
members, the issue/comment will be assigned to the gitlab admin user.

    --project-members-only

### Migrate Issues ID (iid)

You can retain the issues ID from redmine, **this cannot be done via REST
API**, thus it requires **direct access to the gitlab machine**.

So you have to log in the gitlab machine (eg. via SSH), and then issue the
commad with sufficient rights, from there:

    migrate-rg iid --gitlab-key xxxx \
      http://git.example.com/mygroup/myproject --check

### Migrate wiki pages

First, clone the GitLab wiki repository (go to your project's Wiki on GitLab,
click on "Git Access" and copy the URL) somewhere local to your machine. The
conversion process works even if there are pre-existing wiki pages, however
this is NOT recommended.

    migrate-rg pages --redmine-key xxxx --gitlab-wiki xxxx \
      https://redmine.example.com/projects/myproject \

where gitlab-wiki should be the path to the cloned repository (must be local
to your machine). Add "--no-history" if you do not want the old versions of
each page to be converted, too.

After conversion, verify that everything is correct (a copy of the original
wiki page is included in the repo, however not added/commited), and then
simply push it back to GitLab.

### Import git repository

A bare matter of `git remote set-url && git push`, see git documentation.

Note that gitlab does not support multiple repositories per project, you'll have
to reorganize your projects if you were using that feature of Redmine.

### Archive redmine project

If you want to.

You're good to go :).

### Optional: Redirect redmine to gitlab (for apache)

Since redmine has a common *https://redmine.company.tld/issues/{issueid}* url for issues, you can't create a generic redirect in apache.

This command creates redirect rules that you can place in your `.htaccess` file.

    migrate-rg redirect --redmine-key xxxx --gitlab-key xxxx \
      https://redmine.example.com/projects/myproject \
      http://git.example.com/mygroup/myproject > htaccess.example

The content of htaccess.example will be

    # uncomment next line to enable RewriteEngine
    # RewriteEngine On
    # Redirects from https://redmine.example.com/projects/myproject to https://git.example.com/mygroup/myproject
    RedirectMatch 301 ^/issues/1$ https://git.example.com/mygroup/myproject/issues/1
    RedirectMatch 301 ^/issues/2$ https://git.example.com/mygroup/myproject/issues/2
    ...
    RedirectMatch 301 ^/issues/999$ https://git.example.com/mygroup/myproject/999

Known issues
------------

### Using Sudo

Using `sudo` functionality (enabled by default), issues and comments will preserve their author and timestamps.
However, this will only work when the author is marked as 'admin' in Gitlab.
As a workaround, either make everybody admin in Gitlab, or patch Gitlab. See gitlab-patches folder.

After patching, restart Gitlab: `sudo gitlab-ctl restart`

Gitlab issue: https://gitlab.com/gitlab-org/gitlab-ce/issues/23144

### Re-opened issues

This script looks as the `closed_on` field of the Redmine issue, which will be filled when the issue is first closed.
If the issue is re-opened later in Redmine, the migrator will still mark it as closed in Gitlab.

Manual fix: lookup your redmine project id and fix in Redmine DB:

See if you have any re-opened issues:

```
SELECT issues.id, closed_on
FROM issues JOIN issue_statuses on (issues.status_id=issue_statuses.id)
WHERE issue_statuses.is_closed = 0 AND NOT ISNULL(closed_on) AND project_id = <your_id>;
```

To fix:

```
UPDATE issues JOIN issue_statuses ON (issues.status_id=issue_statuses.id)
SET closed_on = NULL
WHERE issue_statuses.is_closed = 0 AND project_id = <your_id>;
```

Develop / improving this tool
-----------------------------

If you want to go back to a clean Gitlab state after migration, use these commands. All issues and uploads of the given
project will be cleaned!

In `sudo gitlab-rails console`:

```
project = Project.find_by_name('your-project')
project.issues.each {|x| x.destroy }
project.uploads.each {|x| x.destroy }
``` 

To clean group milestones:

```
group = Group.find_by_name('your-group')
group.milestones.each {|x| x.destroy }
```

Further cleanup:

* Clean caches (will reset issue count in UI): `sudo gitlab-rake cache:clear`

Unit testing
------------

Use the standard way:

    python setup.py test

Or use whatever test runner you fancy.
