# Auto Formatter for php projects

> This code uses the PHP_CodeSniffer library from [squizlabs](https://github.com/squizlabs/PHP_CodeSniffer]).

### Requirements
Create a new ssh public/private keypair, that has access to the repository. (Add the public key to your ssh keys on the gitlab [account page](https://gitlab.com/-/profile/keys).)

Add these variabled to your Gitlab CI/CD cofiguration, with either the web GUI or write them into the .gitlab-ci.yml file.
- `SSH_PRIVATE_KEY` - The private key from the generated keypair.
- `SSH_PUBLIC_KEY` - The public key from the generated keypair.
- `PHP_VERSION` - The php version your project uses.
- `FORMATTER_EMAIL` - Custom e-mail address for the formatter it uses to push the code to the origin (usually: `formatter@example.com`)
- `FORMATTER_NAME` - Custom name for the formatter it uses to push the code to the origin (usually: `Formatter`)
- `FORMATTER_ORIGIN_URL` - Usually `gitlab.com`, but if you host your own instance, change this to your private gitlab url.
- `FORMATTER_MODE` - Set the mode for formatting (valid values: `push` - pushes the modifications to the branch, `branch` - creates a new branch, and pushes the modifications there, `merge` - creates a new branch, pushes the modifications there and creates a merge request).
- `FORMATTER_BRANCH_NAME` - In `branch` or `merge` mode, the suffix for the new branch.
- `FORMATTER_COMMIT_MESSAGE` - The commit message for the modifications.

Create a file called `phpcs.xml` in the root of your code. In this file, you can define custom rules for the CodeSniffer.
An example configuration shown here:
```xml
<?xml version="1.0"?>

<ruleset name="Laravel Standards">
    <description>The Laravel Coding Standards</description>

    <!-- Define the folders here, you want to check -->
    <file>app</file>

    <rule ref="Generic.Classes.DuplicateClassName"/>
    <rule ref="Generic.CodeAnalysis.EmptyStatement"/>
    <rule ref="Generic.CodeAnalysis.ForLoopShouldBeWhileLoop"/>
    <rule ref="Generic.CodeAnalysis.ForLoopWithTestFunctionCall"/>
    <rule ref="Generic.CodeAnalysis.JumbledIncrementer"/>
    <rule ref="Generic.CodeAnalysis.UnconditionalIfStatement"/>
    <rule ref="Generic.CodeAnalysis.UnnecessaryFinalModifier"/>
    <rule ref="Generic.CodeAnalysis.UnusedFunctionParameter"/>
    <rule ref="Generic.CodeAnalysis.UselessOverridingMethod"/>
    <rule ref="Generic.Commenting.Todo"/>
    <rule ref="Generic.Commenting.Fixme"/>
    <rule ref="Generic.ControlStructures.InlineControlStructure"/>
    <rule ref="Generic.Files.ByteOrderMark"/>
    <rule ref="Generic.Files.LineEndings"/>
    <rule ref="Generic.Files.LineLength">
        <properties>
            <property name="lineLimit" value="120"/>
            <property name="absoluteLineLimit" value="150"/>
        </properties>
    </rule>
    <rule ref="Generic.Formatting.DisallowMultipleStatements"/>
    <rule ref="Generic.Formatting.MultipleStatementAlignment"/>
    <rule ref="Generic.Formatting.SpaceAfterCast"/>
    <rule ref="Generic.Functions.CallTimePassByReference"/>
    <rule ref="Generic.Functions.FunctionCallArgumentSpacing"/>
    <rule ref="Generic.Functions.OpeningFunctionBraceBsdAllman"/>
    <rule ref="Generic.Metrics.CyclomaticComplexity">
        <properties>
            <property name="complexity" value="50"/>
            <property name="absoluteComplexity" value="100"/>
        </properties>
    </rule>
    <rule ref="Generic.Metrics.NestingLevel">
        <properties>
            <property name="nestingLevel" value="10"/>
            <property name="absoluteNestingLevel" value="30"/>
        </properties>
    </rule>
    <rule ref="Generic.NamingConventions.ConstructorName"/>
    <rule ref="Generic.PHP.LowerCaseConstant"/>
    <rule ref="Generic.PHP.DeprecatedFunctions"/>
    <rule ref="Generic.PHP.DisallowShortOpenTag"/>
    <rule ref="Generic.PHP.ForbiddenFunctions"/>
    <rule ref="Generic.PHP.NoSilencedErrors"/>
    <rule ref="Generic.Strings.UnnecessaryStringConcat"/>
    <rule ref="Generic.WhiteSpace.DisallowTabIndent"/>
    <rule ref="Generic.WhiteSpace.ScopeIndent">
        <properties>
            <property name="indent" value="4"/>
            <property name="tabIndent" value="true"/>
        </properties>
    </rule>
    <rule ref="Generic">
        <exclude-pattern>*.php</exclude-pattern>
        <exclude name="Generic.Strings.UnnecessaryStringConcat.Found)"/>
    </rule>
    <rule ref="MySource.PHP.EvalObjectFactory"/>
    <rule ref="PSR1.Classes.ClassDeclaration"/>
    <rule ref="PSR1.Files.SideEffects"/>
    <rule ref="PSR1">
        <exclude-pattern>*.php</exclude-pattern>
        <exclude name="PSR1.Methods.CamelCapsMethodName.NotCamelCaps"/>
    </rule>
    <rule ref="Zend.Files.ClosingTag"/>
    <rule ref="Zend.NamingConventions.ValidVariableName"/>
    <rule ref="Squiz.Strings.ConcatenationSpacing">
        <properties>
            <property name="spacing" value="1"/>
        </properties>
    </rule>
    <rule ref="PEAR.Commenting.FunctionComment"/>
    <rule ref="Squiz.Commenting.DocCommentAlignment"/>
    <rule ref="Squiz.Commenting.BlockComment"/>
    <rule ref="Squiz.WhiteSpace.MemberVarSpacing"/>
    <rule ref="PSR2">
        <exclude name="PSR2.Classes.PropertyDeclaration"/>
    </rule>
    <rule ref="PSR12"/>

    <arg name="colors"/>
    <arg name="tab-width" value="4"/>
    <ini name="memory_limit" value="128M"/>
</ruleset>
```

### Install:
Add this snippet to your `.gitlab-ci.yml` file:
```yml
formatter:
  stage: formatting
  image: "php:${PHP_VERSION}-cli-alpine"
  except:
    - "${FORMATTER_BRANCH_NAME}"
  before_script:
    - php -v
    - apk add --no-cache curl git openssh-client
    - eval `ssh-agent -s`
    - echo "${SSH_PRIVATE_KEY}" | tr -d '\r' | ssh-add - > /dev/null
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - echo "${SSH_PUBLIC_KEY}" >> ~/.ssh/id_rsa.pub
    - '[[ -f /.dockerenv ]] && echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config'
    - git config --global user.email "${FORMATTER_EMAIL}"
    - git config --global user.name "${FORMATTER_NAME}"
    - curl -s https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin/ --filename=composer
    - composer global require "squizlabs/php_codesniffer=*"
  script:
    - set +e
    - /root/.composer/vendor/bin/phpcbf
    - set -e
    - git remote show origin
    - git remote set-url --push origin "git@${FORMATTER_ORIGIN_URL}:$CI_PROJECT_PATH"
    - git remote show origin
    - git add .
    - git update-index -q --refresh
    - test -z "$(git diff-index --name-only HEAD --)" && echo "Nothing changed! Aborting..." && exit 0
    - echo "Has changes. Committing..."
    - set +e
    - if [[ "$FORMATTER_MODE" == "branch" ]]; then git push origin --delete "$CI_COMMIT_REF_NAME-${FORMATTER_BRANCH_NAME}"; fi
    - if [[ "$FORMATTER_MODE" == "merge" ]]; then git push origin --delete "$CI_COMMIT_REF_NAME-${FORMATTER_BRANCH_NAME}"; fi
    - set -e
    - if [[ "$FORMATTER_MODE" == "branch" ]]; then git checkout -b "$CI_COMMIT_REF_NAME-${FORMATTER_BRANCH_NAME}"; fi
    - if [[ "$FORMATTER_MODE" == "merge" ]]; then git checkout -b "$CI_COMMIT_REF_NAME-${FORMATTER_BRANCH_NAME}"; fi
    - git commit -m "${FORMATTER_COMMIT_MESSAGE}"
    - if [[ "$FORMATTER_MODE" == "push" ]]; then git push --follow-tags origin HEAD:$CI_COMMIT_REF_NAME; fi
    - if [[ "$FORMATTER_MODE" == "branch" ]]; then git push -u origin "$CI_COMMIT_REF_NAME-${FORMATTER_BRANCH_NAME}"; fi
    - if [[ "$FORMATTER_MODE" == "merge" ]]; then git push -u -o merge_request.create -o merge_request.target=$CI_COMMIT_REF_NAME -o merge_request.remove_source_branch origin "$CI_COMMIT_REF_NAME-${FORMATTER_BRANCH_NAME}"; fi
```
