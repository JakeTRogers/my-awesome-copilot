# my-awesome-copilot

A personal, public notebook of the files that shape how AI coding assistants behave, kept as patterns actually in use.

## Language

**Skill**:
A portable, reusable playbook for one kind of task, packaged as a folder with a `SKILL.md`, that an assistant loads on demand or a user invokes by name.
_Avoid_: Prompt, prompt file, command

**Prompt**:
Text handed to a model as input, such as a user's request or what the promptly skill produces. Never a kind of file kept in this repo.
_Avoid_: Prompt file (as a repo artifact; superseded by Skill)

**Instruction**:
Always-on guidance that a host applies automatically to any file matching its `applyTo` glob. The trigger sets it apart from a Skill: an instruction loads when the file path matches, and a skill loads when the task calls for it.
_Avoid_: Rule, convention file
