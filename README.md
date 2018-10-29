# Migrate a project from svn to git

## Generate user file

In order to keep commit history associated to authors, it's important to retrieve the list of users from the svn repo before importing it.

From the local svn repository type :
`$ svn log --quiet | grep "^r" | awk '{print $3}' | sort | uniq > users.txt`

The file users.txt will look like :
```
user1
user2
user3
...
```

Complete the file to use the git format :
```
user1 = Firstname1 LastName1 <email1>
user2 = Firstname2 LastName2 <email2>
user3 = Firstname3 LastName3 <email3>
...
```

## Create a local git repository from svn repository

Into the svn repository, retrieve the url of the remote server typing :
`$ svn info | grep -E "^URL" | awk '{print $2}'`

Then generate a new git repository typing :
`$ git svn clone <svn_url> --authors-file=users.txt --tags=<tags_path> --branches=<branches_path> --trunk=<trunk_path>`

**NOTE:**
- It's not mandatory to specify `--tags`, `--branches` and `--trunk` parameters if the repository uses the standard svn layout, in that case just specify `--stdlayout`.
- Those parameters can be specified several times if the repo places them under multiple paths
- If the git master is a specific branch just specify `--trunk=<branches_path>/<branch_name>`

## Redefine references

Now that a new git repository has been created from svn, all the references should be available by typing :

```
$ git show-ref
refs/heads/master
refs/remotes/origin/<branch1>
refs/remotes/origin/<branch2>
refs/remotes/origin/tags/<tag1>
refs/remotes/origin/tags/<tag2>
refs/remotes/origin/trunk
```
### Renaming trunk to master

First of all, rename trunk to master by typing:
`$ git reset --hard  refs/remotes/origin/trunk`

And then delete the trunk reference
`$ git branch -r -d origin/trunk`

### Clearing references

In order to redefine branches and tags, you can either redefine each reference manually :

```
### For branches ###
$ git branch <branch_name> refs/remotes/origin/<branch_name>
$ git branch -r -d origin/<branch_name>
### For tags ###
$ git tag <tag_name> refs/remotes/origin/tags/<tag_name>
$ git branch -r -d origin/tags/<tag_name>
```

Or use the following command to automate the process :
```
$ for ref in $(git for-each-ref refs/remotes --format='%(refname)' | cut -d "/" -f 4,5); do if [ $(echo $ref | grep tags) ]; then git tag $(echo $ref | cut -d "/" -f 2) refs/remotes/origin/$ref; git branch -r -d origin/$ref; elif [ $(echo $ref | grep @) ]; then git branch -r -d origin/$ref; else git branch $ref refs/remotes/origin/$ref; git branch -r -d origin/$ref; fi; done
```

## Pushing the git repository on remote server

### Create a new remote

Now that the git repository is created and configured it has to be linked to the remote server.

To link the local repo with the remote git server, enter the following command after creating an empty git project on the server :

`$ git remote add origin <server_url>:<remote_repo>`

### Push the project

In order to push modifications on the remote repository and to set it as default type the following commands :

```
$ git push --set-upstream origin --all
$ git push --set-upstream origin --tags
```

# Fetch new changes from SVN repository

If new commits have been added into the SVN repository it may be useful to push them onto the git repository.

Here is the procedure to do it

## Clone the git repository

Retrieve the remote git repository

`$ git clone <server_url>:<remote_repo>`

And configure the SVN repository once into it

`$ git svn init <svn_url> --authors-file=users.txt --tags=<tags_path> --branches=<branches_path> --trunk=<trunk_path>`

**NOTE:**
- Don't forget to specify the authors file otherwise new commits won't be associated to git users (i.e. As mentioned above at step **Create a local git repository from svn repository**)

## Get the changes

Once the git repo has been configured to retrieve changes from the SVN repo, checkout the branch on which the changes have been committed

`git checkout <branch>`

And get the changes 

`git svn rebase`

## Push retrieved changes

After rebasing the branch from SVN commits, push the local modifications to the configured remote repository

`git push origin`

# Fetch new branch from SVN repository

If a new branch have been added into the SVN repository it may be useful to fetch it onto the git repository.

Here is the procedure to do it

## Clone the git repository

As above

**NOTE:**
- As mentioned above at step **Clone the git repository**

## Manually add the remote branch

Once the git repo has been configured to retrieve new branch from the SVN repo, create the references to the remote branch manually.

```
git config --add svn-remote.<branch_name>.url <svn_branch_url>
git config --add svn-remote.<branch_name>.fetch :refs/remotes/<branch_name>
```

## Fetch the new branch

After configuring the remote branch into the git repo, fetch the branch.

`git svn fetch <branch_name>`

## Push the retrieved branch

Once the new branch as been fetched, checkout the branch

`git checkout -b <branch_name> <branch_name>`

And push the new branch to the remote git repository

`git push origin <branch_name>`
