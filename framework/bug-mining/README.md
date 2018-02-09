Creating a bug-mining working directory and configuring a project
-------------------------
A Perl module and a wrapper build file are necessary to mine defects for a (new)
project in `Defects4J` -- templates exist for both to ease this task.

1. Suppose the project's name is `my-awesome-project` and the descriptive
*<project id>* is `MyProject`, then create the Perl module and wrapper build file
for the new project with:
  - `./create-project.pl -pMyProject -n"my-new-project" -w bug-mining`

This command initializes the *bug-mining* working directory and creates the
following files:
  - Project Perl module: `bug-mining/framework/core/Project/MyProject.pm`
  - Project build file: `bug-mining/framework/projects/MyProject/MyProject.build.xml`

For example, the project name for the Apache commons-lang project already
included in Defects4J is *commons-lang*, and its project id is *Lang*. The
project id should start with an upper-case letter and should be short yet
descriptive (keep in mind that this id is used for commands such as `defects4j
checkout -p <project_id>`). The project name should be hyphenated and must not
include spaces.

2. Adapt the Perl module and the following properties if necessary:
    - Version control system (default is Git)
    - Location of the version control repository (default is
      `bug-mining/project_repos/<project_name>.git`)
    - Project layout (source and test directories)

3. Adapt the Wrapper build file

Overview of the bug-mining process for a configured project
----------------------------
1. Finding candidate revisions by cross-referencing a development commit log
   with an issue tracker database.

2. Project setup: Making sure that the candidate revisions are compilable and
   conform to path expectations of `Defects4J`.

3. Reproducing faults: Running tests to verify that the bug can be reproduced
   reliably with a test that fails before the fix and passes afterwards.

4. Reviewing revisions and promoting to main database: Manually determine
   whether the bug fix is minimal (i.e., does not include features or
   refactorings). If the bug fix is minimal, promote the bug to the main
   `Defects4J` database!

The candidate commit database `commit-db`
-------------------------
1. Bug mining starts with identifying the issue tracker for the project you are
   interested in. Defects4j's bug-mining framework supports:
    - google (google-code)
    - jira (defaulting to apache)
    - github
    - sourceforge (sf)

2. For trackers jira, github, google, but not sourceforge, create a
   directory `issues` to download the issue numbers:
    - `mkdir bug-mining/issues`

3. For trackers jira, github, google, but not sourceforge, use
   `download-issues.pl` to download the issues (additionally, use
   `merge-issue-numbers.pl` if a project has multiple trackers):
    - `./download-issues.pl jira -p lang -o bug-mining/issues |
      ./merge-issue-numbers.pl -f bug-mining/issues.txt`

4. Obtain the development history (commit logs) for the project:
    - `git --git-dir=bug-mining/project_repos/<project_name>.git/ log > bug-mining/gitlog`

5. Cross-reference the commit log with the issue numbers known to be bugs
   (saved in *issues.txt* in this example) by using `vcs-log-xref.pl`. The
   script requires a Perl regular expression that matches bug-fixing commits
   (e.g, issue numbers, keywords, etc.). Note that the regular expression has to
   capture the issue number. The script `merge-commit-db.pl` enumerates the
   output of `vcs-log-xref.pl`, and outputs or updates the `commit-db`:
    -  `./vcs-log-xref.pl git -b '/(LANG-\d+)/mi' -l bug-mining/gitlog
       -r ./bug-mining/project_repos/<project_name>.git/
       -c './verify-bug-file.sh ./bug-mining/issues.txt' |
       ./merge-commit-db.pl -f bug-mining/framework/projects/<project_id>/commit-db -g ./bug-mining/project_repos/<project_name>.git`


   These are the issue trackers, project IDs, and regular expressions we used
   (Note that we manually built the commit-db for Chart):

   | Defects4J ID | Issue tracker | Project ID        | Regexp                  |
   |--------------|---------------|-------------------|-------------------------|
   | Chart        |               |                   |                         |
   | Closure      | google        | closure-compiler  | `/issue.*(\d+)/mi`        |
   | Lang         | jira          | lang              | `/(LANG-\d+)/mi`          |
   | Math         | jira          | math              | `/(MATH-\d+)/mi`          |
   | Time         | github        | JodaOrg/joda-time | `/Fix(?:es)?\s*#(\d+)/mi` |
   | Time         | sf            | joda-time         | `/\[.*?(\d+)/mi`          |

6. For tracker sourceforge (as we used for Time), note that due to a change in
   sourceforge's structure, bug ids which were once universal became project
   specific. Unfortunately the old, universal IDs are not available publicly for
   bulk query. Fortunately, the old ids web page will still redirect to the new
   IDs web page, allowing individual query.
   Provide `verify-bug-sf.sh <project_id>` as the `-c` option to do this query
   on an individual basis.

Project setup
------------
1. Initialize all project revisions with `initialize-revisions.pl`. This script
   will identify the various directory layouts and run a sanity check on each
   candidate revision in `commit-db`:
    - `./initialize-revisions.pl -p Lang -w bug-mining`

2. Analyze all candidate revisions with `analyze-project.pl`. This will identify suitable
   candidates -- ones that compile and have a non-empty source diff:
    - `./analyze-project.pl -p Lang -w bug-mining`

Reproducing faults
-------------
1. Determine triggering tests with `get-trigger.pl`. This will determine the
   revisions in `commit-db` that have a test that can reproduce a fault:
    - `./get-trigger.pl -p Lang -w bug-mining`

2. Determine the classes modified by the bug fix with `get-class-list.pl`. This
   will determine the list of modified classes for revisions with a reproducible
   fault, and also the list of classes loaded during the execution of the
   triggering test:
    - `./get-class-list.pl -p Lang -w bug-mining`

Reviewing revisions and promoting to main database
------------------
1. Each reproducible fault has an entry in the `trigger_tests` directory:
    - `ls bug-mining/framework/projects/<project_id>/trigger_tests`

2. Manually analyze the stack trace for each fault and make sure this is a real
   fault reproduction, not a configuration issue (e.g., `CLASSPATH` errors or
   missing files). Each file in `trigger_tests` contains the stack trace for a
   reproduced fault:
    - `vim bug-mining/framework/projects/<project_id>/trigger_tests/*`

3. Manually review the diff for each fault and make sure it is minimal. Every
   reproducible fault has an entry with the file name `<id>.src.patch` in the
   `patches` directory:
     - `ls -l Lang/patches/*.src.patch`
     - `ls -l bug-mining/framework/projects/<project_id>/patches/*.src.patch`
     - `./minimize-patch.pl -p <project_id> -v <id> -w bug-mining`

   Note that the patch is the *reverse* patch, i.e., patching the fixed revision
   with this patch will reintroduce the fault.

4. For each fault, if the diff is minimal (i.e., does not include features or
   refactorings), promote the fault to the main `Defects4J` database:
    - `./promote-to-directory.pl -p <project_id> -v <id> -w bug-mining`

   Note: Make sure to specify the `-v` option as the default is to promote all
         found bugs!