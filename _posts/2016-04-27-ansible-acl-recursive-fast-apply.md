---
layout: post
title:  "Ansible recursive ACLs: workaround for fast apply"
tags: [ansible,acl]
---
If you are setting POSIX acls recursive for directory with a lot of files, playbook is applying very slow, because it checks acl for each file. If you use default acls, then every newly created file will get right acl. So only time when you really want recursion to work is when you are applying playbook is first time. Here is a workround to do so:

```c
- name: set acls
  acl: path=/some/path state=present etype=user entity=www-data permissions="rX" recursive=no
  register: dirs_acls_updated

# Output of this task is realy huge, so it's suppressed
- name: set directory acls for user www-data in /some/path with recursion
  acl: path=/some/path state=present etype=user entity=www-data permissions="rX" recursive=yes
  when: app_dirs_acls is defined and app_dirs_acls.changed
  no_log: True
```
