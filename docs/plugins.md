Mr. Fusion's Plugin System
==========================

Plugins are additional shell functions that can run any command-line programs.  They are sourced into the environment and have full access to the hooks and environment variables defined within Mr. Fusion.

For a list of plugins and a description of their function, see [plugins/](plugins/).


Sample Plugin
-------------

Here is a sample plugin that we will dissect.

    #!/bin/bash

    hook-add load-config require-parent-or-root

    require-parent-or-root() {
        local BRANCH PARENT_VAR ROOT_VAR

        echo "Scanning configuration to ensure all branches have parents"
        echo "or that they are marked as 'root' in their INI section."

        for BRANCH in "${INI_BRANCHES[@]}"; do
            # Build variable names
            PARENT_VAR="$(ini-var "$BRANCH" parent)"
            ROOT_VAR="$(ini-var "$BRANCH" root)"

            # Check for an empty/missing "parent" in the INI file
            if [[ -z "${!PARENT_VAR}" ]]; then
                # Check for where "root" is not set to "true"
                if [[ "${!ROOT_VAR}" != "true" ]]; then
                    echo "Branch misconfigured: $BRANCH"

                    return 1
                fi
            fi
        done
    }

Let's break that down.  `hook-add` is how you hook into any of the available hooks that Mr. Fusion provides.  See the hook documentation for a full list.  In this example we are adding into `load-config` and it will call `require-parent-or-root`.

When Mr. Fusion runs and loads the configuration, it will now stop and run `require-parent-or-root`.  This loops through all of the branches that are loaded in the `branches.ini` configuration file.  For each of those branches it does two tests.

First it ensures that there is no parent and secondly it make sure that "root" is not set to "true".  In order to do this it constructs the variable names for both the "parent" setting and the "root" setting for the branch's section in the INI file using `ini-var`.  This will handle escaping oddities, such as slashes in branch names.

If both settings are missing then this plugin will say that the `branches.ini` file is wrong and will refuse to continue with the merge.


Environment and More
--------------------

Plugins will execute their code in hooks.  All hooks will run while within the repository directory (`$WORK_DIR`) automatically.  Plugins have access to the following environment variables.

* `DRY_RUN`: Read only variable set to `true` or `false`.  When in dry run mode, changes are not pushed back to origin.
* `GIT_AUTHOR_NAME`, `GIT_AUTHOR_EMAIL`, `GIT_COMMITTER_NAME`, `GIT_COMMITTER_EMAIL`, etc: git-specific environment variables.
* `GIT_BRANCHES`: List of branches that were found in git.
* `GIT_MERGE_MESSAGE`: Message to add to the commit that merges the branches.
* `INI_BRANCHES`: Read only array of section names (which correspond to branch names) from the `branches.ini` file.
* `INI_BRANCHES__*`: See "Parsing branches.ini" for information.
* `MERGE_LIST`: List of branches to merge in the order they should be merged.
* `MERGES_FAIL`: List of branches that failed to merge.
* `MERGES_GOOD`: List of branches that successfully merged.
* `MR_FUSION`: Name of executable.
* `MR_FUSION_BRANCH`: The special branch name where configuration files are stored.
* `MR_FUSION_DIR`: Where the `mr-fusion` binary is located.
* `MR_FUSION_URL`: URL to the project.
* `WORK_DIR`: Where the repository is cloned.


Parsing branches.ini
--------------------

In the environment is the `branches.ini` file's contents.  It's saved as variables starting with `INI_BRANCHES` and you will see them with a simple `set` command.

Let's assume we have this for the file's contents:

    # Comments are supported
    #
    # This is how you define an empty 'master' section
    [master]

    # This 'develop' section has a single key, 'parent', whose value is 'master'
    [develop]
    parent=master

    # You can add as many keys and values as you would like
    [feature/faster]
    parent=develop
    author=Eric

Environment variables are set to these parsed values, but the section names and keys are hex encoded to make sure that special symbols do not interfere with variable names.  *Do not construct these variables by hand!*  Let a function named `ini-var` help.

All sections in the INI file have a matching variable with the value of `true`.  We can check to see if a section is defined using code like this.

    if [[ ! -z "$(ini-var develop)" ]]; then
        echo "The 'develop' section exists"
    fi

How about looping through all branches and verifying they do not have an "ignore" setting?

    for BRANCH in "${INI_BRANCHES[@]}"; do
        if [[ ! -z "$(ini-var "$BRANCH" ignore)" ]]; then
            echo "Branch $BRANCH has an 'ignore' setting"
        fi
    done


Hooks
-----

Plugins are able to run their own functions at key points in the execution or Mr. Fusion.  These functions can execute any arbitrary commands.  Please note that plugins run with `set -e` enabled, so capture any errors.

Unless noted, hooks that return an error status code will cancel the merge process entirely.

* `clone-repository`: Immediately after the repository is cloned and before anything happens to the repository.
* `git-config`: Additional configuration that should be applied to the git repository before switching branches and performing merges.
* `load-config`: After configuration files are loaded into environment variables.  This can be used to validate that various settings are in place or that the INI file has the necessary keys for specific branches.  Do not assume that the git branches have been loaded.
* `git-branches`: After branches are loaded.  This can be used to make sure that necessary branches exist.  Do not assume that the configuration has been loaded.
* `merge-before`: Right after we switched to a branch and just before we merge in the parent.  Failure with this hook will only fail the merge for this branch and will not kill the program.
* `merge-validate`: Right after we performed the merge and just before we push the changes upstream.  Failure with this hook will only fail the merge for this branch and will not kill the program.
* `merge-fail`: Happens when a failed merge has been added to the list.
* `merge-good`: Happens when a successful merge has been added to the list.
* `merge-start`: Configuration was loaded and the list of branches from git were determined.  This hook runs before any branches are actually merged.
* `merge-finish`: All merging is done.
* `switch-branch`: The branch has been switched to a new branch.  This happens once per branch that needs merges plus once to switch to the `mr-fusion` branch to get the configuration.
