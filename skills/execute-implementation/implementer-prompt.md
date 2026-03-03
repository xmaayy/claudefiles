# Implementer Subagent Prompt Template

Use this template when dispatching an implementer subagent.

```
Task tool (general-purpose):
  description: "Implement Task N: [task name]"
  prompt: |

    You are implementing Task N: [task name]

    ## Task Description

    [FULL TEXT of task from plan - paste it here, don't make subagent read file]

    ## Context

    [Scene-setting: where this fits, dependencies, architectural context]

    ## Before You Begin

    Read the files listed in the task's Files section and any imports or dependencies they reference to familiarize yourself with the landscape.

    At any point — before or during implementation — ask questions rather than making assumptions. This includes questions about requirements, approach, dependencies, or anything unclear in the task description.

    ## Your Job

    Once you're clear on requirements:

    1. Implement exactly what the task specifies
    2. Write tests (following TDD if task says to)
    3. Verify implementation works
    4. Commit your work
    5. Self-review (see below)
    6. Report back

    Work from: [directory]

    ## Before Reporting Back: Self-Review

    Review your work before reporting. The formal review comes later — this is your chance to catch obvious issues.

    - Did I implement everything in the spec? Did I miss any requirements?
    - Did I avoid overbuilding? Did I only build what was requested (YAGNI)?
    - Did I follow existing patterns in the codebase?
    - Do tests actually verify behavior (not just mock it)? Do they run and pass?
    - Would this pass code review without comments?

    If you find issues during self-review, fix them before reporting.

    ## Report Format

    When done, report:

    - What you implemented
    - What you tested and test results
    - Files changed
    - Commit SHA(s)
    - Self-review findings (if any)
    - Any issues or concerns

    ## Troubleshooting

    If you encounter missing tools or broken environments (e.g. pytest not found), look for an `init-env` skill or setup instructions in the README before attempting any fixes. Do not blindly install packages. If you can't find instructions to resolve the issue, stop and alert the user.
```
