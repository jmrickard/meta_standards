This document is intended to be a standards document for the Ansible roles contained in the
Oasis Roles repositories.

Meta Best Practices
===================

A rationale should be included for best practices, with a reference if applicable. It is
really helpful to know not only how to do certain things, but why to do them in this
way. It will also help with further revisions of the standards as some items may become
obsolete or no longer applicable.  If the reason is not included, there is a risk of
keeping items that are no longer applicable, or alternatively blindly removing items that
should be kept. It also has great educational value for understanding how things actually
work (or how they don’t).

Background
==========

The goal of the Ansible Metateam project (specifically, the [Linux System Roles
project](https://github.com/linux-system-roles)) is to provide a stable and consistent user
interface to multiple operating systems (multiple versions of RHEL in the downstream RHEL System
Roles package, additionally CentOS, Fedora at least). Stable and consistent means that the same
Ansible playbook will be usable to manage the equivalent functionality in the supported versions
without the administrator (the user of the role) being forced to change anything in the playbook
(the roles should serve as abstractions to shield the administrator from differences). Of course,
this means that the interface of the roles should be itself stable (i.e. changing only in a backward
compatible way). This implies a great responsibility in the design of the interface, because the
interface, unlike the underlying implementation, can not be easily changed. 

The differences in the underlying operating systems that the roles need to compensate for are
basically of two types:
* Trivial differences like changed names of packages, services, changed location of configuration
  files. Roles must deals with those by using internal variables based on the OS defaults. This is
  fairly simple, but still it brings value to the user, because they then do not have to worry about
  keeping up with such trivial changes. 
* Change of the underlying implementation of a given functionality. Quite often, there are multiple
  packages/components implementing the same functionality. Classic examples are the various MTAs
  (sendmail, postfix, qmail, exim), FTP daemons, etc. In the context of Linux System Roles, we call
  them “providers”. The goal of the roles is to abstract even such differences, so that when the OS
  changes to a different component (provider), the role continues to work. An example is time
  synchronization, where RHEL used to use the ntpd package, then chrony was introduced and became
  the default, but both components have been shipped in RHEL 6 and RHEL 7, until finally ntpd was
  dropped from RHEL 8, leaving only chrony. A role covering time synchronization should therefore
  support both components with the same interface, and on systems which ship both components, both
  should be supported. The appropriate supported component should be automatically selected on
  systems that ship only one of them. This covers several related use cases:
  * Users that want to manage multiple major releases of the system simultaneously with a single playbook.
  * Users that want to migrate to a new version of the system without changing their automation (playbook).
  * Users who want to switch to a different provider in the same version of the OS (like switching
    from ntpd to chrony to RHEL 7) and keep the same playbook.

Designing the interface in the latter case is difficult because it has to be sufficiently abstract
to cover different providers. We, for example, do not provide an email role in the Linux System
Roles project, only a postfix role, because the underlying implementations (sendmail, postfix) were
deemed to be too divergent. Generally, an abstract interface should be something that should be
always aimed for though, especially if there are multiple providers in use already, and in
particular when the default provider is changing or is known to be likely to change in the next
major releases.

Basics
======

* Every repository in the oasis-roles namespace should be a valid Ansible Galaxy compatible role
  with the exception of any whose names begin with "meta\_", such as this one.
* New roles should be initiated in line with the skeleton directory, which has standard boilerplate
  code for a Galaxy-compatible Ansible role and some enforcement around these standards
* Use [semantic versioning](https://semver.org/) for Git release tags.  Use
  0.y.z before the role is declared stable (interface-wise).  Although it has
  not been a problem so far for linux system roles, since they use strict X.Y.Z
  versioning, you should be aware that there are some
  [restrictions](https://github.com/ansible/ansible/issues/67512) for Ansible
  Galaxy and Automation Hub.  The versioning must be in strict X.Y.Z[ab][W]
  format, where X, Y, and Z are integers.

Interface design considerations
===================================

What should a role do and how can a user tell it what to do.

## Basic design

Try to design the interface focused on the functionality, not on the software implementation behind
it. This will help abstracting differences between different providers (see above), and help the
user to focus on the functionality, not on technical details.

## Naming things

* All YAML or Python files, variables, arguments, repositories, and other such names should follow
  standard Python naming conventions of being in snake\_case\_naming\_schemes.
* Names should be mnemonic and descriptive and not strive to shorten more than necessary. Systems
  support long identifier names, so use them to be descriptive
* All defaults and all arguments to a role should have a name that begins with the role name to help
  avoid collision with other names. Avoid names like `packages` in favor of a name like `foo_packages`.
  (Rationale: Ansible has no namespaces, doing so reduces the potential for conflicts and makes
  clear what role a given variable belongs to.)
* Same argument applies for modules provided in the roles, they also need a `$ROLENAME_` prefix:
  `foo_module`. While they are usually implementation details and not intended for direct use in
  playbooks, the unfortunate fact is that importing a role makes them available to the rest of the
  playbook and therefore creates opportunities for name collisions.
* Moreover, internal variables (those that are not expected to be set by users) are to be prefixed
  by two underscores: `__foo_variable`. (Rationale: role variables, registered variables, custom
  facts are usually intended to be local to the role, but in reality are not local to the role - as
  such a concept does not exist, and pollute the global namespace. Using the name of the role
  reduces the potential for name conflicts and using the underscores clearly marks the variables as
  internals and not part of the common interface. The two underscores convention has prior art in
  some popular roles like
  [geerlingguy.ansible-role-apache](https://github.com/geerlingguy/ansible-role-apache/blob/f2b91ac84001db3fd4b43306a8f73f1a54f96f7d/vars/Debian.yml#L8)). This
  includes variables set by set_fact and register, because they persist in the namespace after the
  role has finished!
* Do not use special characters other than underscore in variable names, even if YAML/JSON allow
  them. (Using such variables in Jinja2 or Python would be then very confusing and probably not
  functional.)
  
## Providers

When there are multiple implementations of the same functionality, we call them “providers”. A role
supporting multiple providers should have an input variable called `$ROLENAME_provider`. If this
variable is not defined, the role should detect the currently running provider on the system, and
respect it. (Rationale: users can be surprised if the role changes the provider if they are running
one already.) If there is no provider currently running, the role should select one according to the
OS version. (E.g. on RHEL 7, chrony should be selected as the provider of time synchronization,
unless there is ntpd already running on the system, or user requests it specifically. Chrony should
be chosen on RHEL 8 as well, because it is the only provider available.) The role should set a
variable or custom fact called `$ROLENAME_provider_os_default` to the appropriate default value for
the given OS version. (Rationale: users may want to set all their managed systems to a consistent
state, regardless of the provider that has been used previously. Setting `$ROLENAME_provider` would
achieve it, but is suboptimal, because it requires selecting the appropriate value by the user, and
if the user has multiple system versions managed by a single playbook, a common value supported by
all of them may not even exist. Moreover, after a major upgrade of their systems, it may force the
users to change their playbooks to change their `$ROLENAME_provider` setting, if the previous value
is not supported anymore. Exporting `$ROLENAME_provider_os_default` allows the users to set
`$ROLENAME_provider: "{{ $ROLENAME_provider_os_default }}"` (thanks to the lazy variable evaluation
in Ansible) and thus get a consistent setting for all the systems of the given OS version without
having to decide what the actual value is - the decision is delegated to the role.)

Coding Conventions
===========

## Role Structure
Avoid testing for distribution and version in tasks. Rather add a variable file to "vars/"
for each supported distribution and version with the variables that need to change according
to the distribution and version. This way it is easy to add support to a new distribution by
simply dropping a new file in to "vars/", see below [Supporting multiple distributions and versions](#supporting-multiple-distributions-and-versions). See also
 (https://github.com/oasis-roles/meta_stanards#vars-vs-defaults) which mandates "Avoid embedding large lists or "magic values" directly into the playbook." Since
distribution-specific values are kind of "magic values", it applies to them. The same logic
applies for providers: a role can load a provider-specific variable file, include a provider-specific
task file, or both, as needed. Consider making paths to templates internal variables if you need
different templates for different distributions.

## Check Mode and Idempotency Issues

* The role should work in check mode, meaning that first of all, they should not fail check mode, and
  they should also not report changes when there are no changes to be done. If it is not possible
  to support it, please state the fact and provide justification in the documentation. 
  This applies to the first run of the role.

* Reporting changes properly is related to the other requirement: **idempotency**. Roles
 should not perform changes when applied a second time to the same system with the same parameters,
 and it should not report that changes have been done if they have not been done. Due to this,
 using `command:` is problematic, as it always reports changes. Therefore, override the result by
 using `changed_when:`

* Concerning check mode, one usual obstacle to supporting it are registered variables. If there
 is a task which registers a variable and this task does not get executed (e.g. because it is a
 `command:` or another task which is not properly idempotent), the variable will not get registered
 and further accesses to it will fail (or worse, use the previous value, if the role has been
 applied before in the play, because variables are global and there is no way to unregister them).
 To fix, either use a properly idempotent module to obtain the information (e.g. instead of
 using `command: cat` to read file into a registered variable, use `slurp` and apply `.content|b64decode`
 to the result like [here](https://github.com/linux-system-roles/kdump/pull/23/files#diff-d2414d4ec8ba189e1a244b0afc9aa81eL8)), or apply proper `check_mode:` and `changed_when:`
 attributes to the task. [more_info](https://github.com/ansible/molecule/issues/128#issue-135906202).

* Another problem are commands that you need to execute to make changes. In check mode, you
 need to test for changes without actually applying them. If the command has some kind of "--dry-run"
 flag to enable executing without making actual changes, use it in check_mode (use the variable
 `ansible_check_mode` to determine whether we are in check mode). But you then need to set `changed_when:`
 according to the command status or output to indicate changes. See (https://github.com/linux-system-roles/selinux/pull/38/files#diff-2444ad0870f91f17ca6c2a5e96b26823L101) for an example.

* Another problem is using commands that get installed during the install phase, which is
 skipped in check mode. This will make check mode fail if the role has not been executed
 before (and the packages are not there), but DTRT if check mode is executed after normal mode. 

* To view reasoning for supporting why check mode in first execution may not be worthwhile: see [here](https://github.com/ansible/molecule/issues/128#issuecomment-245009843). If this is to be supported, see hhaniel's proposal [here](https://github.com/linux-system-roles/timesync/issues/27#issuecomment-472466223), which seems to properly guard even against such cases.

## Supporting multiple distributions and versions 

If some tasks need to be parameterized according to distribution and version (name of packages,
configuration file paths, names of services), use this in the beginning of your `tasks/main.yml`:
```yaml
- name: Set version specific variables
  include_vars: "{{ item }}"
  with_first_found:
    - "{{ ansible_distribution }}-{{ ansible_distribution_version }}.yml"
    - "{{ ansible_distribution }}-{{ ansible_distribution_major_version }}.yml"
    - "{{ ansible_distribution }}.yml"
    - "{{ ansible_os_family }}.yml"
```

You can add "-default.yml" at the end to set the variables for unrecognized distributions from  a common default.yml file. If this is not done, the role will fail for unrecognized distribution (this may or may not be intended). Put the distribution-specific variables into appropriate files under "vars/". Unfortunately, this will lead to duplicate vars files for similiar distibutions (e.g. CentOS 7 and RHEL 7). In cases such as these, use symlinks. 

```If the user is on Windows and symlinks are not working, have them request the extension or point the user WSL to Galaxy to resolve symlink issue for Windows users. https://github.com/linux-system-roles/network/pull/64```


## Supporting multiple providers

Use a task file per provider and include it from the main task file, like this example from `storage:`
```yaml
- name: include the appropriate provider tasks
  include_tasks: "main-{{ storage_provider }}.yml"
```
The same process should be used for variables (not defaults, as defaults can be loaded according to a variable).


## Generating files from templates
* Comment with `{{ ansible_managed }}`at the top of the file. [more_info](https://docs.ansible.com/ansible/latest/modules/template_module.html#template-module) 
* When commenting, don't include anything like "Last modified: {{ date }}" 
* Use standard module parameters for backups, keep it on unconditionally (`backup: true`) (Until there is a user request to have it configurable)
* Make prominently clear in the HOWTO (at the top) what settings/configuration files are replaced by the role instead of just modified.

Implementation considerations
===========

## YAML and Jinja2 Syntax

* Indent at two spaces
* List contents should be indented beyond the list definition
* It is easy to split long Jinja2 expressions into [multiple
  lines](https://github.com/linux-system-roles/timesync/pull/47/files).  If the
  `when:` condition results in a line that is too long, and is an `and`
  expression, then break it into a list of conditions.  Ansible will `and` them
  together (see: [[1]](https://github.com/linux-system-roles/timesync/pull/36)
  [[2]](https://docs.ansible.com/ansible/latest/user_guide/playbooks_conditionals.html#the-when-statement)).
  Multiple conditions that all need to be true (a logical `and`) can also be
  specified as a list, but beware of bare variables in `when:`.
* All roles need to, minimally, pass a basic ansible-playbook syntax check run
* All task arguments should be spelled out in YAML style and not use `key=value` type of arguments
* All YAML files need to pass standard yamllint syntax with the modifications listed in tests/yamllint.yml
  in the meta\_skeleton role. These modifications are minimal: document starter characters (the initial
  `---` string at the top of a file) should not be used, and it is not necessary to start every comment
  with a space. Most comments should start with a space, but no space is allowed when a comment is
  documenting an optional variable with its default value.
  * As a result of being able to pass basic YAML lint, avoid the use of `True` and `False` for boolean values
   in playbooks. These values are sometimes used because they are the words Python uses. However, they are
   improper YAML and will be treated as either strigns or as booleans but generating a warning depending on
   the particular YAML implementation.
   * Do not use the Ansible-specific `yes` and `no` as boolean values in YAML as these are completely
   custom extensions used by Ansible and are not part of the YAML spec.
* Although it is not expressly forbidden, comments in playbooks should be avoided when possible. The task
  `name` value should be descriptive enough to tell what a task does. Variables should be well commented in
  the `defaults` and `vars` directories and should, therefore, not need explanation in the playbooks
  themselves.
* All Jinja2 template points should have a single space separating the template markers from the variable
  name inside. For instance, always write it as `{{ variable_name_here }}`. The same goes if the value is
  an expression. `{{ variable_name | default('hiya, doc') }}`
* When naming files, use the `.yml` extension and *not* `.yaml`.  `.yml` is what
  `ansible-galaxy init` does when creating a new role template.
* Double quotes should be used for YAML strings with the exception of Jinja2
  strings which will use single quotes.
* Do not use quotes unless you have to, especially for short strings like
  `present`, `absent`, etc.  This is how examples in module documentation
  are typically presented.

## Python Guidelines

* Review Ansible guidelines for
[modules](https://docs.ansible.com/ansible/latest/dev_guide/developing_modules_best_practices.html)
and [development](https://docs.ansible.com/ansible/latest/dev_guide/index.html).
* Use [PEP8](https://pep8.org/).
* File headers and functions should have comments for their intent.

## Ansible Best Practices
* Ansible variables use lazy evaluation. [more_info](https://github.com/ansible/ansible/issues/10374)
* All tags should be namespaced/prefixed with the role name. 
* Use the `command` module instead of the `shell` module when use of either is required.
  In all other cases use a dedicated module, if it exists. If not, see the [section](#check-mode-and-idempotency-issues) 
  about idempotency and check mode and make sure that you support them properly (your task
  will likely need conditionals such as `changed_when:` and maybe `check_mode:` ). 
  Anytime `command` or `shell` modules are used; a comment in the code with justificiation
  would help with future maintenance.
* Beware of bare variables in `when`. Use '`| bool`' if the variable is supposed to hold a boolean
  value. [more_info](https://github.com/ansible/ansible/issues/39414).
* Do not use `meta: end_play`. It aborts the whole play instead of a given host (with multiple
  hosts in the inventory) [more_info](https://github.com/ansible/ansible/issues/27973) - We may
  consider using `meta: end_host` but this was recently introduced in Ansible 2.8 [more_info](https://github.com/ansible/ansible/pull/47194)
* Put variables in Jinja2 templates and if reasonable into task names, this helps with reading
  the logs. Variables don't get expanded in play names properly.
* Do not override role defaults or vars or input parameters using `set_fact`. Use a different
  name instead. (Rationale: a fact set using `set_fact` can not be unset and it will override
  the role default or role variable in all subsequent invocations of the role in the same 
  playbook. A fact has a different priority than other variables and not the highest, so in
  some cases overriding a given parameter will not work because the parameter has a higher priority) [more_info](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable)
* Use the smallest scope for variables. Facts are global for playbook run, so it is preferable
  to use other types of variables. Therefore limit (preferably avoid) the use of `set_fact`.
  Role variables are exposed to the whole play when the role is applied using `roles:` or
  `import_role:`. A more restricted scope such as `task` or `block` variables is preferred. 
* Beware of `ignore_errors: yes`! Especially in tests. If you set on a block, it will ignore
  all the asserts in the block ultimately making them pointless. A comment in the code with 
  justification is required to use this statement.
* Do not use the `eq` (introduced in Jinja 2.10) or `equalto` (introduced in Jinja 2.8) Jinja
  Operators - or any other post-2.7 Jinja2 features. (RHEL 7 has Jinja 2.7.2)
  * https://github.com/linux-system-roles/storage/pull/26
  * https://github.com/linux-system-roles/storage/issues/49
* All tasks should be idempotent, with notable and rare exceptions such as the
  [OASIS reboot role](https://github.com/oasis-roles/reboot).
* Avoid the use of `when: foo_result is changed` whenever possible. Use
  handlers, and, if necessary, handler chains to achieve this same result. Exceptions are permitted
  but they should be avoided when possible
* Use the various include/import statements in Ansible when doing so can lead to simplified code and a
  reduction of repetition. This is the closest that Ansible comes to callable sub-routines, so use judgment
  about callable routines to know when to similarly include a sub playbook. Some examples of good times
  to do so are
  * When a set of multiple commands share a single `when` conditional
  * When a set of multiple commands are being looped together over a list of items
  * When a single large role is doing many complicated tasks and cannot easily be broken into multiple roles,
   but the process proceeds in multiple related stages
* Avoid calling the `package` module iteratively with the `{{ item }}` argument, as this is impressively
  more slow than calling it with the line `name: "{{ foo_packages }}"`.  The same can go for many other
  modules that can be given an entire list of items all at once.
* Use meta modules when possible. Instead of using the `upstart` and `systemd` modules, use the `service`
  module when at all possible. Similarly for package management, use `package` instead of `yum` or `dnf` or
  similar. This will allow our playbooks to run on the widest selection of operating systems possible without
  having to modify any more tasks than is necessary.
* Avoid excessive use of the `lineinefile` module. Slight miscalculations in how it is used can lead to a loss
  of idempotence. Additionally, modifying config files with it can become arcane and difficult to read,
  especially for someone not familiar with the file in question. If the file is not in a form that can be
  modified with a module (e.g. the `ini` module) then default to using the `template` module. This both makes
  it easier to see what is being configured and to add additional, more complex options in the future.
* Avoid the use of `lineinfile` wherever that might be feasible. Try editing files directly using other built
  in modules (e.g. `ini_file`, `blockinfile`, `xml`) or reading and parsing. If you are modifying more than a tiny number of lines or in a manner more than trivially complex, try leveraging the `template` module, instead. This will allow
  the entire structure of the file to be seen by later users and maintainers. The use of `lineinfile` should include a comment with justification.
* Limit use of the `copy` module to copying remote files and to uploading binary blobs. For all other file
  pushes, use the `template` module. Even if there is nothing in the file that is being templated at the
  current moment, having the file handled by the `template` module now makes adding that functionality much
  simpler than if the file is initially handled by the `copy` and then needs to be moved before it can be
  edited.
* When using the template module, refrain from appending .j2 to the file name. This alters
  the syntax highlighting in most  editors and will obscure the benefits of highlighting for
  the particular file type or filename. Anything under the templates directory of a role is
  assumed to be treated as a Jinja2 template, so adding the .j2 extension is redundant information
  that is not helpful
* Keep filenames and templates as close to the name on the destination system as possible. This will help with
  both editor highlighting as well as identifying source and destination versions of the file at a glance.
  Avoid duplicating the remote full path in the role directory, however, as that creates unnecessary depth in
  the file tree for the role. Grouping sets of similar files into a subdirectory of `templates` is allowable,
  but avoid unnecessary depth to the hierarchy.

### Vars vs Defaults

* Avoid embedding large lists or "magic values" directly into the playbook. Such static lists should be
  placed into the `vars/main.yml` file and named appropriately
* Every argument accepted from outside of the role should be given a default value in `defaults/main.yml`.
  This allows a single place for users to look to see what inputs are expected. Document these variables
  in the role's README.md file copiously
* Use the `defaults/main.yml` file in order to avoid use of the default Jinja2 filter within a playbook.
  Using the default filter is fine for optional keys on a dictionary, but the variable itself should be
  defined in `defaults/main.yml` so that it can have documentation written about it there and so that all
  arguments can easily be located and identified.
* Avoid giving default values in `vars/main.yml` as such values are very high in the precedence order and
  are difficult for users and consumers of a role to override.
* As an example, if a role requires a large number of packages to install, but could also accept a list of
  additional packages, then the required packages should be placed in `vars/main.yml` with a name such as
  `foo_packages`, and the extra packages should be passed in a variable named `foo_extra_packages`,
  which should default to an empty array in `defaults/main.yml` and be documented as such.
  
References
==========

Links that contain additional standardization information that provide context,
inspiration or contrast to the standards described above.

* https://github.com/debops/debops/blob/v0.7.2/docs/debops-policy/code-standards-policy.rst). For
  inspiration, as the DebOps project has some specific guidance that we do not necessarily
  want to follow.
* https://docs.adfinis-sygroup.ch/public/ansible-guide/overview.html 
* https://docs.openstack.org/openstack-ansible/latest/contributor/code-rules.html 
