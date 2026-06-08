# tasks

This directory contains non-role-related task files that support supplementary
actions within the rhis-builder-baremetal-init project.

Task files placed here can be invoked directly via `run_task.yml` at the project
root by passing the task file name (without extension) as the `task_name` variable:

```bash
ansible-playbook -i inventory run_task.yml -e "task_name=<filename>"
```

Add additional task files here for any standalone operations that do not belong
within a role — for example, validation checks, environment setup steps, or
one-off administrative actions.
