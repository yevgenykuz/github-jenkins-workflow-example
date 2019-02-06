GitHub + Jenkins Workflow Example
#################################

A complete example for GitHub (code management) + Jenkins (CI, automatic release) workflow, with configuration.

-----

.. contents::

.. section-numbering::

Workflow
********
Git flow
========
Branching strategy
------------------
* ``master`` is the base branch, there is no ``develop`` branch
* ``master`` is considered as stable, tested code
* The are no parallel support for parallel releases - all releases are made from the ``master`` branch
* For every task, a separate branch is checked out from the ``master`` branch (named ``"_ISSUE_KEY_ - _ISSUE_TITLE_"``), and then merged back to the ``master`` branch using a pull request
* Hot fixes created on a separate branch (named ``hotfix-vX.Y.Z - list of relevant _ISSUE_KEYS_``), and then merged back to ``master`` using a pull request
* To prevent complicated merges when a task is complete, programmers are encouraged to pull ``master`` and merge locally to their task branch on a daily basis 
* Programmers can collaborate on a single task by pushing a task branch to remote and sharing it
* Planned releases **and** hot fixes are marked on the ``master`` branch by adding a tag on it (see *Tags*). This is done automatically using a Jenkins job (see *Jenkins release job*)
* All branches are deleted from remote (GitHub) after the relevant pull request is merged

Tags
----
* All releases are marked with tags. These tags are then used by CI and other Jenkins jobs (see *Jenkins*)
* Major releases **and** hot fixes are tagged on the ``master`` branch, by running release Jenkins job (see *Jenkins release job*)
* All tags must be annotated and contain in its description the release version this tag represents, at minimum 


Configuration
*************
Jenkins
=======
Install ngrok on your Jenkins master (if not exposed to the internet)
---------------------------------------------------------------------
| In case your Jenkins server is not exposed to the internet, you can use a tunneling services like `ngrok <https://ngrok.com/>`_ to allow GitHub to communicate with it. ngrok provides a free plan that is sufficient for GitHub <-> communication.
| In any case, exposing your Jenkins to the internet is the preferred way and should be prioritized.

To install ngrok on your Jenkins master see the following configuration example (used on CentOS 7, all commands are done as root):

* Create a free account @ `ngrok <https://ngrok.com/>`_ 
* On your Jenkins master, create a directory for ngrok

.. code-block:: bash

  mkdir /opt/ngrok

* Download ngrok and unzip it into your newly created folder

.. code-block:: bash

  cd /opt/ngrok
  wget https://bin.equinox.io/c/4VmDzA7iaHb/ngrok-stable-linux-amd64.zip
  unzip ngrok-stable-linux-amd64.zip
  rm ngrok-stable-linux-amd64.zip

* Create a configuration file for ngrok at ``/opt/ngrok/ngrok.yml`` with the following:

.. code-block:: bash

  authtoken: your authentication token from ngrok dashboard
  log: /opt/ngrok/ngrok.log
  tunnels:
    jenkins:
      addr: 8080
      proto: http

* Create a systemd service file at ``/etc/systemd/system/ngrok.service`` with the following:

.. code-block:: bash

  [Unit]
  Description=ngrok
  After=network.target

  [Service]
  ExecStart=/opt/ngrok/ngrok start --all --config /opt/ngrok/ngrok.yml
  ExecReload=/bin/kill -HUP $MAINPID
  KillMode=process
  Restart=on-failure
  Type=simple

  [Install]
  WantedBy=multi-user.target

* Update systemd, enable the new service, and start it:

.. code-block:: bash

  sudo systemctl daemon-reload
  sudo systemctl enable ngrok.service
  sudo systemctl start ngrok.service

* Use the generated URL from `ngrok status page <https://dashboard.ngrok.com/status/>`_ to access Jenkins from the internet

Install and configure Jenkins plugins
-------------------------------------
GitHub pull request builder plugin
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* Go to "Manage Jenkins" -> "Manage Plugins" -> "Available" -> install "GitHub Pull Request Builder"
* Go to "Manage Jenkins" -> "Configure System" -> "GitHub Pull Request Builder" section
* Add your GitHub credentials (user should have admin rights), leave other configuration as is

Create Jenkins CI job for every pull request
--------------------------------------------
This job will be triggered every time a pull request is opened against the ``master`` branch.

* Go to Jenkins -> "New Item" -> and create a new "Freestyle project"
* Under "General" -> tick "GitHub project" and insert your project url
* Under "Source Code Management" -> tick "Git"
* Under "Git" -> insert your project url and select your credentials
* Under "Git" -> click "Advanced" and under "Refspec" insert ``+refs/pull/${ghprbPullId}/*:refs/remotes/origin/pr/${ghprbPullId}/*``
* Under "Git" -> under "Branches to build" -> "Branch Specifier" insert ``${ghprbActualCommit}``
* Under "Build Triggers" -> tick "GitHub Pull Request Builder"
* Under "GitHub Pull Request Builder" -> tick "Use github hooks for build triggering"
* Under "GitHub Pull Request Builder" -> click "Advanced"
* Under "Advanced" -> "Trigger phrase" -> insert ``.*(re)?run tests.*`` **to allow restarting the CI by commenting "run tests" in the PR**
* Under "Advanced" -> "White list" -> add the github usernames that will be allowed to trigger this build
* Under "Advanced" -> "Whitelist Target Branches:" -> add ``master``
* Under "Advanced" -> click "Trigger Setup" to customize update messages back at GitHub
* Under "Trigger Setup" -> "Commit Status Context" -> insert ``Jenkins``
* Under "Trigger Setup" -> under "Commit Status Build Result" -> click "Add" and add 3 custom messages for every status (success, error, and failure)
* Under "Build" -> create your CI checks using various Jenkins scripts/plugins
* Other customization (like build name) can be also altered if needed

Create Jenkins release job for planned releases
-----------------------------------------------
This job will be triggered manually by a team member when a planned release or a hot fix is due. The following will be done:

* Latest commit from ``master`` will be pulled
* Relevant files will be updated (for example - some .pom file versions) - using a job parameter (``${ReleaseVersion}`` for example)
* Updated files will be committed
* This commit will be tagged (the tag name is inserted manually as a parameter)
* CI checks will be performed
* If CI checks passed, the latest commit and tag will be pushed, without pull request (Jenkins credentials must have admin repository rights)

To accomplish this, do the following:

* Go to Jenkins -> "New Item" -> and create a new "Freestyle project"
* Under "General" -> tick "GitHub project" and insert your project url
* Under "Source Code Management" -> tick "Git"
* Under "Git" -> insert your project url and select your credentials
* Under "Git" -> click "Advanced" and under "Refspec" insert ``+refs/heads/master:refs/remotes/origin/master``
* Under "Git" -> under "Branches to build" -> "Branch Specifier" insert ``refs/heads/master``
* Under "Build" -> create your file updates and CI checks using various Jenkins scripts/plugins, upload artifacts if successful
* Under "Build" -> create a new shell/powershell script and add "git add ." -new line- "git commit -m "Prepare v${ReleaseVersion}" to commit your changes
* Under "Post-build Actions" -> click "Add post-build action" and create a new "Git Publisher" block
* Under "Git Publisher" -> tick "Push Only If Build Succeeds"
* Under "Git Publisher" -> under "Tags" -> click "Add Tag" 
* Under new tag -> "Tag to push" insert "v${ReleaseVersion}"
* Under new tag -> "Tag message" insert "v${ReleaseVersion}, created by Jenkins"
* Under new tag -> tick "Create new tag"
* Under new tag -> "Target remote name" -> "origin"
* Under "Git Publisher" -> under "Branches" -> click "Add Branch" 
* Under new branch -> "Branch to push" -> "master"
* Under new branch -> "Target remote name" -> "origin"

GitHub
======
GitHub team structure
---------------------
| The only limitation here, to force the reviewing process, is that all members should have "Write" permission level.
| The only user with admin rights should be the user used by Jenkins jobs.

Add webhook for automatic CI using Jenkins
-------------------------------------------
| This webhook will start a Jenkins build on every pull request to merge into ``master`` branch.
| To do so, go to github repository -> "Settings" -> "Webhooks" -> "Add webhook", and set the following:

#. "Payload URL" -> ``http://_Your_Jenkins_Public_IP/ghprbhook/`` (use generated ngrok URL if you used their service)
#. "Let me select individual events." -> tick it
#. "Pull requests", "Issue comments" -> tick both (leave out all others)
#. Click "Add webhook"

Protect ``master`` branch
-----------------------
Create branch protection rule for ``master``. This rule will force the following:

* Prevent direct commits to master branch by forcing all merges to go through pull requests
* Force a minimum of X reviewers to approve each pull request (reviewers will be added automatically from the configuration found at ``.github/CODEOWNERS`` file) 
* Force all pull request to go through a status check before merging

To do so, go to github repository -> "Settings" -> "Branches" -> "Add rule", and set the following:

#. "Apply rule to" -> master
#. "Require pull request reviews before merging" -> tick it
#. "Required approving reviews" -> select the minimum number of reviewers (depends on team size. If possible, 2 should be the minimum in my opinion)
#. "Dismiss stale pull request approvals when new commits are pushed" -> tick it
#. "Require status checks to pass before merging" -> tick it
#. "Require branches to be up to date before merging" -> tick it
#. Select your status check from the list (you must run it at least once for it to appear)

Meta
****
Authors
=======
`yevgenykuz <https://github.com/yevgenykuz>`_

License
=======

Creative Commons Attribution 4.0 International - `LICENSE <https://github.com/yevgenykuz//github-jenkins-workflow-example/blob/master/LICENSE>`_

-----
