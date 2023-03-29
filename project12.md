# ANSIBLE REFACTORING AND STATIC ASSIGNMENTS (IMPORTS AND ROLES)

In this project, I will be enhancing the architecture from project 11 by refactoring the ansible codes and adding import functionality.

## Step 1 – Jenkins job enhancement

The previous design consumes space on Jenkins servers with each subsequent change. Let us enhance it by introducing a new Jenkins project/job – which will require ***Copy Artifact plugin*** as in the steps below:

1. On Jenkins-Ansible server ceate a new directory called `ansible-config-artifact` – we will store there all artifacts after each build.

`sudo mkdir /home/ubuntu/ansible-config-artifact`

2. Change permissions to this directory, so Jenkins could save files there – 
`chmod -R 0777 /home/ubuntu/ansible-config-artifact`

3. In Jenkins web console ***-> Manage Jenkins -> Manage Plugins -> on Available tab search for Copy Artifact and install this plugin without restarting Jenkins***

4. Create a new Freestyle projec, name it `save_artifacts` and set it to discard builds and with the following configs as below:


![old-builds.PNG](./images/old-builds.PNG)



![build-trigger.PNG](./images/build-trigger.PNG)




![build-step.PNG](./images/build-step.PNG)



![target-directory.PNG](./images/target-directory.PNG)



![ansible.PNG](./images/ansible.PNG)


save-artifact-error.PNG


artifact-error.PNG


updated-in-var-directory.PNG

This failed for me a couple of tries and after a lot of tweeks with the `/home/ubuntu/ansible-config-artifact` directory


So I created the `ansible-config-artifact` directory in the `/var/lib/jenkins` directory and updated both ownership and permission. Also updated the directory in the save_artifats job in jenkins:


![target-directory-updated.PNG](./images/target-directory-updated.PNG)


![jen-ownership-perm.PNG](./images/jen-ownership-perm.PNG)



![ansible-artifact.PNG](./images/ansible-artifact.PNG)



![artifacts-console-output.PNG](./images/artifacts-console-output.PNG)


## Step 2 – Refactor Ansible code by importing other playbooks into site.yml

In ***Project 11*** I wrote all tasks in a single playbook `common.yml`, pretty simple because the set of instructions were for only 2 types of OS, but imagine I have many more tasks and need to apply the playbook to other servers with different requirements. In this case, I will have to read through the whole playbook to check if all tasks written there are applicable and is there anything that needs to be added for certain server/OS families. Very fast it will become a tedious exercise and my playbook will become messy with many commented parts. DevOps colleagues will not appreciate such organization of the codes and it will be difficult for them to use my playbook.

1. create a new branch "refactor" and switch to the branch using : 
`git checkout -b refactor`

2. Within playbooks folder, create a new file and name it `site.yml`. This file will now be the parent to all other playbooks


`touch playbooks/site.yml`

3. Create a new folder in root of the repository and name it `static-assignments`. The static-assignments folder is where all other children playbooks will be stored. 

4. Move `common.yml` file into the newly created static-assignments folder.

5. Inside `site.yml` file, import `common.yml` playbook. This code uses built in `import_playbook` Ansible module.

```
---
- hosts: all
- import_playbook: ../static-assignments/common.yml
```

6. Run `ansible-playbook` command against the dev environment


Since I would need to apply some tasks to the ***dev servers*** and ***wireshark*** is already installed – I would go ahead and create another playbook under ***static-assignments*** and name it `common-del.yml`pasting the code below. This is to configure deletion of wireshark utility.

```
---
- name: update nfs and web servers
  hosts: nfs, webservers
  remote_user: ec2-user
  become: yes
  become_user: root
  tasks:
  - name: delete wireshark
    yum:
      name: wireshark
      state: removed

- name: update LB and DB server
  hosts: lb, db
  remote_user: ubuntu
  become: yes
  become_user: root
  tasks:
  - name: delete wireshark
    apt:
      name: wireshark-qt
      state: absent
      autoremove: yes
      purge: yes
      autoclean: yes
```

Now update `site.yml` with `- import_playbook: ../static-assignments/common-del.yml` instead of `common.yml` and run it against ***dev*** servers:

```
cd /home/ubuntu/ansible-config-mgt/

ansible-playbook -i inventory/dev.yml playbooks/site.yaml
```

Now, I would commit my code update to my GitHub branch ***(refactor)*** and then create pull request, compare, and merge pull request to the ***main branch***

`git status`

`git add .`

`git commit -m "commit message"`

`git push origin refactor`



![commit-refactor.PNG](./images/commit-refactor.PNG)


* On merging the code to the main branch, my Jenkins job updated a build.


![merged.PNG](./images/merged.PNG)


Now, run the playbook:
```
ansible-playbook -i /var/lib/jenkins/jobs/ansible/builds/<builds-number>/archive/inventory/dev.yml /var/lib/jenkins/jobs/ansible/builds/<builds-number>/archive/playbooks/site.yml
```

In my case:
```
ansible-playbook -i /var/lib/jenkins/jobs/ansible/builds/17/archive/inventory/dev.yml /var/lib/jenkins/jobs/ansible/builds/17/archive/playbooks/site.yml
```


![play-result.PNG](./images/play-result.PNG)


Then, check to ensure that ***wireshark*** is deleted on all the servers by running `wireshark --version`

**Before:**

![wireshark-before.PNG](./images/wireshark-before.PNG)

**After:**

![wireshark-after.PNG](./images/wireshark-after.PNG)


## Step 3 – Configure UAT Webservers with a role ‘Webserver’


