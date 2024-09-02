---
notion_page: https://www.notion.so/wpmedia/Version-control-94bd0c31e0514e31a74c00241f103690?pvs=4
title: Version Control
---

# Version Control

## Git

At WP Media, we collaborate on code using [Git](https://git-scm.com/). This is a powerful and standard version control system that allows parallel work and offers codebase history tracking.
Our main platform for this is [GitHub](https://github.com/wp-media), while some infrastructure-related repositories are handled on our internal GitLab.

Git is a powerful system that supports many workflows and ways of using it. To ease collaboration and clarify expectations, we prefer to work with a standardized flow at WP Media. The following chapters describe the main guidelines about our standard Git management.

### Branch management

#### Long-lived branches

Our standard repository configuration is to have two main branches: `trunk` and `develop`:
- `trunk` updates should correspond to releases. The current state of trunk should therefore be the codebase currently in production (for services) or the latest released version (for plugins). Pushes to trunk are related to a release process. For instance, a CI/CD process can be automated based on this action ; or releases can be tagged based on this branch. 
- `develop` should be the main branch from which development branches will start from, and be merged into.

`develop` should always be strictly ahead of `trunk`. If it is not the case, `trunk` must be merged back into `develop`.

Additional long-lived branches might exist for specific needs, for instance for `transifex` ; but this must remain an exception.

#### Development branches

To contribute to a repository, developers should always create a new branch from develop and commit their changes there. Once the implementation is ready to be reviewed and validated, they should open a Pull Request to develop. 

Those development branches should have a lifetime of up to a few days as we are working on small increments. In some cases, we need to implement a complex feature that requires more time, several contributors, and intermediate steps that should not be merged to develop before the whole feature is ready. In those cases, a `feature` branch must be created and used as the baseline for the small increments to start from it and to be merged into it.

During the development and review phases, it is the responsibility of the branch owner to regularly update it with develop (by merging develop into the branch) to stay up-to-date and identify and fix conflicts as early as possible.

#### Branch naming

Apart from the long-lived branches, branch names must be self-explanatory and follow a standard format: `branch_type/issue_number-short_description`. This is valuable for other teammates, reviewers and testers to quickly identify the context of code changes. The following values are recommended, but you can exceptionally use other values if you see fit, as long as the branch name remains self-standing.
- `branch_type` should be one of the following:
    - `feature` for branches introducing a new feature.
    - `fix` for branches that fix a bug.
    - `enhancement` for branches introducing an enhancement of an already-existing feature
    - `chore` for branches handling technical maintenance such as editing documentation, adding tests, cleaning up code, changing version numbers, updating dependencies, etc.
- `issue_number` should be the number identifying the GitHub issue this PR is addressing.
- `short_description` should be a few words, separated by `-`, that quickly explain the topic of the branch.

Here are a few examples:
- `feature/3700-remove-unused-css`
- `enhancement/2773-no-beta`
- `fix/3437-picture-lazyload-exclusion`

#### Branch protections

Branch protections are rules that can prevent some events on specific branches. They can be configured for each repository on GitHub in the Settings of the repository.

Our standard configuration is to protect both the `trunk` and the `develop` branches with the following rules:
- Require a pull request before merging: all commits must be made to a non-protected branch and submitted via a pull request before they can be merged into a branch that matches this rule.
    - Require 1 approval: pull requests targeting a matching branch require a number of approvals and no changes requested before they can be merged.
    - Dismiss stale pull request approvals when new commits are pushed: New reviewable commits pushed to a matching branch will dismiss pull request review approvals.

Those rules enforce the [peer-review process](reviews.md), ensuring the code we produce and released is at least validated by two developers.

#### Closing branches

Branch owners are responsible for ensuring their branches don't end up stale. If they are relevant, they should be merged and deleted, otherwise they can be closed and deleted. To merge a branch, our [review process](reviews.md) must be followed.

Once a branch has been merged (or closed), it should be deleted. This ensures that we keep only branches for ongoing work, easing the identification of in progress development on a repository.

### Commits

Usually, a single developer is working on a branch. Therefore, it could seem sufficient to commit all changes once they are ready and working. However, this can cause troubles if one needs to circle back to the implementation to investigate a bug or understand what has been implemented. More numerous, small and well-scoped commits can help to navigate the changes, test intermediate versions for a bug, and eventually precisely rollback changes. Therefore, commits should rather be done frequently, and only contain a small, well-defined scope. 

Commit messages are critical for teammates to navigate the history and quickly understand the reason of a change. The first line of a commit message is a brief summary of the commit, describing the expected result of the change or what is modified in the code. Then, a longer message can be added to provide more details if needed. This appended message must be separated from the first line by a blank line, to ensure the first line remains the highly visible description.

## Releases

Releases are linked to a push to the `trunk` branch typically for a merge from `develop`, and ideally automated through CI/CD (GitHub actions). The deployment process may vary depending on the repository, and is quite different for our plugins and our services for example. However, we have some common steps. All our releases should be tagged as such on their GitHub repository and an internal release note should be shared internally. Those steps should ideally be automated as they facilitate knowledge sharing and ease investigation in case of bugs or regressions.

Thanks to our branch management, branch protections and CIs, code must have been reviewed and must be passing the CI before getting into a release. Depending on the project, manual tests on the develop branch can be conducted to ensure the version is properly working.

### Version name/numbers

It is important to clearly name each release to ease the discussion about them. Depending on the repository, we can have 2 version naming system.
- **For releases deliverable to users** (typically WordPress plugins), we use [semantic versioning](https://semver.org/), which is also the recommended standard for [PHP](https://www.php.net/manual/en/function.version-compare.php) and [WordPress](https://developer.wordpress.org/plugins/wordpress-org/how-your-readme-txt-works/#readme-header-information). This syntax is well understood by most users and fits well with our release cycle of plugins.
- **For service releases** (websites, apps, back-ends, ...), we use timestamp versioning with the following syntax: _yyyy.MM.dd-hhmm_ . This approach better fits continuous delivery where version bumps are more of a burden than providing actual benefits. The timestamp is automatically picked during the automated delivery.

### Release notes

Keeping all teammates up-to-date and providing them with a consistent way of knowing about the changes is key in ensuring all operations can smoothly be performed and continuously adapted company-wide: this is true for engineering teammates but also support, product, marketing, etc. Standardize release notes are a great way to achieve this goal.

Every release must have a corresponding release note in our corresponding Notion database. Internal release notes (apps, websites, back-ends, ...) should be automated. Release notes can also be manually added or appended if needed.

To be useful and readable, releases notes must:
- Reference the corresponding version number, product and date;
- List all technical changes with a link to the corresponding GitHub issue or PR (for automated release notes, the automated release description from GitHub should be leveraged);
- Explain the impact and changes experienced from a user perspective.