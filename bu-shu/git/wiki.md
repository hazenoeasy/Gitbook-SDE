# wiki

### 1 fork the repository into the individual repository.

Click the button `fork` at the right top of the page. Then it will show up in your individual repositories.

Clone it into the local machine.

`git clone git@github.com: <your github username>/S2022-Team-4-repo.git`

check your branch.

`git branch -a`

### 2 set up upstream

`git remote add upstream https://github.com/gcivil-nyu-org/S2022-Team-4-repo.git`

`git remote -v`

```
origin	git@github.com:hazenoeasy/S2022-Team-4-repo.git (fetch)
origin	git@github.com:hazenoeasy/S2022-Team-4-repo.git (push)
upstream	https://github.com/gcivil-nyu-org/S2022-Team-4-repo.git (fetch)
upstream	https://github.com/gcivil-nyu-org/S2022-Team-4-repo.git (push)
```

### 3 update fork code

To make your code up-to-date with Source Repository, run&#x20;

`git fetch upstream`

&#x20;it will pull the latest code into the upstream branch.

Run`git branch -a` to make sure you are at `develop branch`.

Then run  `git merge upstream/develop`

### 3 develop in local machine

### 4 format code

`pip install black` to install the black format tool.

run `black .` at the root directory.

### 5 push to the individual repository.

`https://www.theserverside.com/video/Follow-these-git-commit-message-guidelines`

commit message should follow the document. e.x. `docs: add develop flows to the wiki`.

### 6 create a pull request

Click `contribute button` in your individual repository Github page. Then click `open pull request`

![](<../../.gitbook/assets/Screen Shot 2022-02-26 at 1.04.39 PM.png>)

Connect Issues

![](<../../.gitbook/assets/Screen Shot 2022-02-26 at 1.21.00 PM.png>)

### 7 Review the request

&#x20;Check `pull requests`, if there is no conflict, click merge pull request.

![](<../../.gitbook/assets/Screen Shot 2022-02-26 at 1.21.24 PM.png>)

Rewrite the commit message, add \` close #{issue number}\`, the github bot would detect this issue, and automatically close it.&#x20;

![](<../../.gitbook/assets/Screen Shot 2022-02-26 at 1.21.41 PM.png>)

![](<../../.gitbook/assets/Screen Shot 2022-02-26 at 1.22.12 PM.png>)
