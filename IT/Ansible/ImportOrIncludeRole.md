___
___
# Tags
#ansible 
___
Фан факт из мира Ansible. Есть различия между `import_role` и `include_role`.

Пример:

```yaml
- name: Deploy process_engine
  hosts: process_engine_engine
  become_user: "{{ privilege_user | default('root') }}"
  tasks:
    - name: Deploy engine
      import_role:
        name: app/deploy
      tags: [engine, deploy]

    - name: Configure engine
      import_role:
        name: app/configure
      tags: [engine, configure]

    - name: Extra conf
      ****_role: |<-
        name: meta_data/extra_conf/download
      tags: [configure, extra_conf]
      when: download_extra_conf is defined
```

- `import_role` — **статический импорт**. Все таски роли читаются на этапе парсинга плейбука (до того, как выполняются условия). То есть Ansible уже успел залезть в meta_data/extra_conf/download/tasks/main.yml, увидел там `include_tasks`, и свалился на download_extra_conf_classifiers is undefined — ещё до проверки твоего `when`.
    
- Чтобы условие реально отработало, нужно использовать **динамический импорт** (`include_role`), потому что он читается только во время выполнения и учитывает `when`.

Из [документации](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html#applying-conditions-to-roles):

> Conditional statements applied to roles in the roles: section are only evaluated at the task level. The role itself will still be included and its variables may still be loaded.
> If you want to conditionally include an entire role, use `import_role` or `include_role` in a task with a when statement.
