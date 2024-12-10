---
notion_page: https://www.notion.so/wpmedia/PHPStan-154ed22a22f080709efed66eeae2d0bf?pvs=4
title: PHPStan
---

# PHPStan

## How PHPStan Works

1. **AST (Abstract Syntax Tree)**: PHPStan analyzes code by converting it into a syntax tree, enabling consistent interpretation regardless of formatting.
2. **Custom Rules**:
    - Rules implement the `\PHPStan\Rules\Rule` interface.
    - **Methods**:
        - `getNodeType()`: Specifies the type of node to analyze (e.g., method calls, new instances).
        - `processNode()`: Contains logic for identifying and reporting errors.

## Custom rules

### Objective

The goal is to enhance knowledge sharing by integrating company-specific standards and solutions directly into the codebase using PHPStan custom rules. This helps bridge the gap between documentation and practice, ensuring that crucial conventions and lessons are not forgotten during development.

**Why Use PHPStan for Knowledge Sharing?**

Traditional documentation requires developers to actively search for information, which may be overlooked or forgotten. PHPStan custom rules embed this knowledge directly into the development process by providing real-time feedback during static code analysis.

1. **Bug Prevention**:
    - Automate solutions for known issues, preventing repeated developer effort.
2. **Enforce Standards**:
    - Ensure compliance with company conventions (e.g., avoiding certain ORM practices).
    - Promotes team-wide adherence to standards.
    - Reduces reliance on developers’ memory or manual checks.
    - Streamlines onboarding and knowledge transfer.
3. **Production Safety**:
    - Flag hard-to-reproduce bugs (e.g., incorrect event subscriber usage).

### Action to Take When Adding a New Rule

When adding a new PHPStan custom rule, it’s essential to prevent errors from legacy code from being flagged. This is where the **baseline** feature of PHPStan comes into play.

**What is a Baseline?**

A baseline is a snapshot of all errors currently present in your codebase. PHPStan uses this to ignore existing issues while still detecting new ones in updated or added code. This allows teams to adopt stricter rules without immediately needing to fix historical errors, making the transition smoother.

**Steps to Add a Rule Without Affecting Legacy Code**

1. Reset the Baseline:
    - Use the script `run-stan-reset-baseline` defined in the project’s `composer.json` file.
    - This will generate a new baseline file that includes all currently detectable errors, including those triggered by the newly added rule.
2. Commit the Updated Baseline File:
    - After generating the new baseline, commit it to your repository. This ensures the baseline is shared with your team and reflects the latest set of ignored issues.
3. Validate the Changes:
    - Run PHPStan locally and in your CI pipeline to confirm that no new or unexpected errors are being reported.

By maintaining and updating the baseline, you can progressively enforce new standards without overwhelming the team with legacy issues. This approach allows for continuous improvement in code quality while keeping development workflows manageable.

**Steps to Create a Custom Rule**

1. Define the Rule:
    - Extend `\PHPStan\Rules\Rule` and bind the rule to a specific node type.
    - Use `RuleErrorBuilder::message()` to define error messages.
    - Example: A rule to disallow `var_dump()`:
        
        ```php
        public function processNode(Node $node, Scope $scope): array {
            if ($node->name->toString() === 'var_dump') {
                return [RuleErrorBuilder::message('var_dump is not allowed')->build()];
            }
            return [];
        }
        
        ```
        
2. Test the Rule:
    - Extend `PHPStan\Testing\RuleTestCase` and implement `getRule()` to return the custom rule instance.
    - Write test cases with code templates and expected errors.
    - Example: Detect `var_dump()` in a test file and confirm error reporting.
3. Run PHPStan:
    - Use `phpstan analyse` to validate the implementation.

For more detailed examples, refer to the sample implementations in PHPStan's [documentation on rules](https://phpstan.org/developing-extensions/rules).

### Custom Rules included within WP Rocket

- Discourage the use of `apply_filters`  in favour of `wpm_apply_filters_typed`
- Discourage the use of `update_option` in favour of `Option` object, with the `set()` method.
- Ensuring that no hooks are called within ORM
- Make sure that the callback set within `get_subscribed_events` from a `Subscriber` is a method that exists.