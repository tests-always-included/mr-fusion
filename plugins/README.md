Mr. Fusion Plugins
==================

Here is a listing of the officially maintained plugins.  If you have needs that are not fulfilled by Mr. Fusion directly, perhaps a plugin can satisfy your requirements.

Loading plugins is really easy when you let Mr. Fusion do it for you.

    mr-fusion -p plugin-name http://github.com/your_name_here/your_project.git

This will download the plugin right from the Mr. Fusion repository.

If you would prefer to write your own, just make sure you check out the `mr-fusion` branch in your repository and put plugins into the `plugins/` folder.  They are simple shell scripts containing a function.  The script is sourced into Mr. Fusion and attaches to hooks so it can run at specific times.  Learn more about [writing plugins](../docs/plugins.md).


branch-lists-match
------------------

When the list of branches from git does not match the list from the `branches.ini` file, this plugin reports an error and will not allow you to continue with the merge.


require-author
--------------

After loading the `branches.ini` file, this plugin makes sure that each branch is configured with an author.  You are allowed to skip branches by setting the `REQUIRE_AUTHOR_SKIP` array in your `config` file.  For example, you can skip the author for `master` and `develop` this way.

    REQUIRE_AUTHOR_SKIP=(master develop)


require-parent-or-root
----------------------

Enforces the rule that all configured branches have a `parent=...` or `root=true`.  This check is performed just after the `branches.ini` file is loaded.  The intent is to avoid typos, such as in this sample `branches.ini`

    # branches.ini example - THIS SHOWS THE PROBLEM
    [valid-parent]
    parent=another-branch-name
    
    [valid-root]
    root=true

    [invalid-due-to-typo]
    paretn=This is a misspelled "parent" key
