---
notion_page: https://www.notion.so/wpmedia/Pull-Requests-Code-Reviews-428149161ef5412a964bb4f94329b4ad?pvs=4
title: Pull Requests & Code Reviews
---

# Pull Requests & Code Reviews

## Why do we need code reviews?

Crafting software is complex, and as good as they can be, developers are human beings: they can make mistakes, miss something, or simply not think about the best approach right away.

We are working in teams, where several developers will successively work on the same code. To ease this collaboration, we need to standardize our code style, the code quality and expectations: moving from one codebase to another must be as seamless as possible, and anyone should be able to pick up anyone else code seamlessly.

Finally, reviews are a great way to share knowledge, continuously learn and improve yourself and your team.

Therefore, code reaching production must be passing all automated checks, and be at least approved by one peer. To apply this process, we leverage Pull Requests (PR) on GitHub (or Merge Requests on GitLab).

## Opening a Pull Request

### When to open a Pull Request?

You don’t have to wait for the code to be completed to open a **draft** PR. It can be beneficial to share your work early on to ask for and get feedback. The sooner you get the feedback, the quicker you can improve your code.

Once you consider that your code is ready to be shipped into production, you should open a PR (or remove the draft status of the existing PR).

### Target branch

If your PR is part of a more complex code change and is not useful on its won, not meant to be delivered on its own, or depends on other code changes not merged yet, then the target branch must be a dedicated feature branch. Only when the feature is complete, this branch will be merged back into the `develop` branch.

If your PR covers a complete fix/enhancement/feature on its own, then the target branch should be the `develop` branch.

### Pull Request description

The Pull Request description is the document that will follow your code until it reaches production, through peer reviews, QA testing, release notes and documentation. For all teammates to perform those steps accurately, they need to quickly grasp the context of your PR, the reasons why the implementation has been done this way, what it changes, etc. While this is obvious to you after spending hours working on it, it is likely not the case for teammates ; so make sure to make the Pull Request as self-standing as possible!

To achieve this, a default Pull Request template is automatically applied on new Pull Requests of WP Media owned repositories on GitHub. This template is maintained [here](https://github.com/wp-media/.github/blob/main/.github/PULL_REQUEST_TEMPLATE.md). Specific templates can override this generic one in some specific repositories.

Note: Our CI pipelines on GitHub should contain [this GitHub action](https://github.com/wp-media/pr-checklist-action) that validates proper filling of the Pull Request description.

### Getting reviews

As the PR owner (creator), you are responsible for getting reviews once the PR is ready. A PR is considered ready when the automated CI checks are passing and that you consider your implementation completed. Then, make sure to properly update your PR description, make it visible and, if needed, ask for reviews. 

Many things happen every day within our teams: it is easy to miss an opened PR. To make sure your PR get reviews check that it is visible as "Ready for Review" on your team's board, in the current sprint. Also assign the team tag as reviewer, so that teammates are notified.

Our teams SLA is to handle reviews within 24 hours. If you don't get any reviews after 24 hours, kindly ask for one on Slack specifying what the PR is about and consider pinging relevant teammates. You should also leverage the daily stand-up meeting to find reviewers.

## CI - Automated Continuous Integration Pipeline

Many checks and validations can be done automatically on each PR, without the need for a human to perform any actions. The set of automated checks is called the Automated Continuous Integration Pipeline (CI). While it can differ between each repository, mostly depending on the language, we have baselines that should be common to all our CIs.

On GitHub, we leverage GitHub actions to implement our CIs. For each main task to be performed by the CI, it is preferred to use separate *jobs* so that they can fail independently, facilitating investigation and allowing to retry only the needed steps.

### Pull Request Description validation

The importance of the Pull Request description is emphasized above in this document. We have [an automated check](https://github.com/wp-media/pr-checklist-action) that ensures the standardized PR template is properly filled. This check can help quickly identify Pull Requests not up to standards so that the owner can fix it, and avoid reviewers to begin reviewing while some context is missing.

### Linting, Code Style and Code Standards

Developers have to work on many repositories. To minimize cost of context switching and feel at home in every codebase, it is important to share common style guidelines. Also, keeping code easy and natural to read is key for efficiency in the long run.

All our CI must include steps and tools to:
- Check code style. This ensures conventions and rules about formatting and code structure are applied (indentation, naming conventions, spacing, etc.)
- Check code standards. This ensures general best practices of the language are applied so that the implementation remains safe, organized and robust.
- Check linting. This ensures there are not syntax errors and identify potential bad patterns, useless complexity and potential bugs.

For each language that we use, we leverage different tools. Refer to the section of each language for more details and examples.

### Running automated tests

Automated tests are a key asset of our codebases. They are designed to capture some behaviors of the codebase and check that they can consistently be reproduced across evolutions of the software.

Every time we alter the code (hence, for every PR), we must ensure that those behaviors are still reproducible. So all tests must run and pass as part of the CI.

If tests are not passing, it means a behavior changed:
- if this is expected as a result of the code change, the tests must be adapted to reflect this change;
- if this is not expected, then a regression has been introduced and the code change must be reviewed to fix the issue.

For each language that we use, we leverage different tools. Refer to the section of each language for more details and examples.

### Diff Coverage

To ensure that the changes we introduce will be stable over time and prevent them to be impacted by regressions, it is critical to cover those changes with automated tests. Adding tests to capture the behaviors we introduce is a great way to demonstrate how the code should behave, and to allow future maintainers to easily check they did not alter this expected behavior.

Diff coverage step identifies all lines being modified by your PR, and checks whether they are executed during test execution. This approach allows to only focus on what the current PR is changing, regardless of the overall coverage level of the repository.

When a line is not covered, it means that it could be removed or altered without any tests failing, which is a great risk for regression in the future. On the contrary, a covered line is not totally safe: a line being executed during a test does not mean its logic is fully tested and validated. Therefore, diff coverage must be taken as an indication of untested lines of code for developers to self-review and improve autonomously their coverage.

We globally aim at a minimum of 50% diff coverage. Below this, this step of the pipeline must fail. This should not be a mandatory step to validate, as it depends on the context of the PR and the impacted lines.

We have two ways of computing and reporting diff coverage, depending on the repository we are working on.

#### Diff coverage with Codacy

[Codacy](https://www.codacy.com/login) is a platform that analyses git repositories and additional reports such as code coverage to report metrics. We use it on our public repositories as those features are free for them.

To get diff coverage reported directly on the Pull Requests:
- Add the repository on the Codacy account and check its configuration, especially Integrations & Gates.
- Authorize Codacy app on GitHub to access the repository.
- Instruct the test tools to output a coverage report during the CI. For instance, with `--coverage-php tests/report/unit.cov` for PHPUnit, or `--cov=. --cov-report=xml` for pytest.
- Once the report is available, upload it to Codacy using [the dedicated GitHub action](https://github.com/codacy/codacy-coverage-reporter-action/tree/v1/).
You can check the [CI of apply-filters-typed](https://github.com/wp-media/apply-filters-typed/blob/develop/.github/workflows/test.yml) as a good example. Here are the steps related to Codacy:

```
    - name: Unit/Integration tests
      run: composer run-tests

    - name: Merge Code Coverage Reports
      run: composer report-code-coverage

    - name: Run codacy-coverage-reporter
      uses: codacy/codacy-coverage-reporter-action@v1
      with:
        project-token: ${{ secrets.CODACY_PROJECT_TOKEN }}
        coverage-reports: tests/report/coverage.clover
```

#### Diff coverage with diff-cover

[diff-cover](https://github.com/Bachmann1234/diff_cover) is an open-source library that computes diff coverage based on coverage reports and git diff, for any languages.
We use it for our private repositories for which Codacy is not available.

To get diff coverage reported directly on the Pull Requests:
- Instruct the test tools to output a coverage report during the CI. For instance, with `--coverage-php tests/report/unit.cov` for PHPUnit, or `--cov=. --cov-report=xml` for pytest.
- Once the report is available, add steps in the CI to run diff-cover and send comments and annotations to the PR.
As an example, check out the [TB-TT CI](https://github.com/wp-media/TB-TT/blob/develop/.github/workflows/ci-on_pr_main_bash.yml).

```
    - name: Run pytest
      run: |
        pytest -m "not staging_env" --cov=. --cov-report=xml 
      shell: bash

      - name: Generate diff-coverage report
        if: github.event_name == 'pull_request'
        run: |
          diff-cover coverage.xml --compare-branch=origin/${{ github.base_ref }} --markdown-report diff-cover-report.md --exclude test*.py --fail-under=50 --expand_coverage_report
          echo "DIFF_COVER_EXIT_STATUS=$?" >> $GITHUB_ENV
        shell: bash

      - name: Delete previous diff-cover reports
        uses: actions/github-script@v6
        with:
          script: |
            const { data: comments } = await github.rest.issues.listComments({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number
            });
            
            for (const comment of comments) {
              if (comment.user.login === 'github-actions[bot]' && comment.body.includes('# Diff Coverage')) {
                console.log(`Deleting comment with ID: ${comment.id}`);
                await github.rest.issues.deleteComment({
                  owner: context.repo.owner,
                  repo: context.repo.repo,
                  comment_id: comment.id
                });
              }
            }
        env:
          GITHUB_TOKEN: ${{ secrets.DIFF_COVER_COMMENT }} 

      - name: Post diff-cover report to PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v6
        with:
          script: |
            const fs = require('fs');
            const comment = fs.readFileSync('diff-cover-report.md', 'utf8');
            await github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: comment,
            });
      - name: Fail job if coverage is below threshold
        if: github.event_name == 'pull_request'
        run: |
          if [[ "${{ env.DIFF_COVER_EXIT_STATUS }}" -ne 0 ]]; then
            echo "Coverage below threshold; failing the job."
            exit 1
          fi
        shell: bash
```
