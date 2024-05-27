# Workshop Exercise - Three Tier App

## Table of Contents

- [Workshop Exercise - Three Tier App](#workshop-exercise---three-tier-app)
  - [Table of Contents](#table-of-contents)
  - [Objectives](#objectives)
  - [Guide](#guide)
    - [Three Tier App](#three-tier-app)
    - [Step 1 - Set Instance Tags](#step-1---set-instance-tags)
    - [Step 2 - Update Ansible Inventory](#step-2---update-ansible-inventory)
    - [Step 3 - Install Three Tier Application](#step-3---install-three-tier-application)
    - [Step 4 - Smoke Test Three Tier Application](#step-4---open-a-terminal-session)
  - [Conclusion](#conclusion)

## Objectives

* Install a three tier application stack to provide example of application/workload functionality testing pre/post conversion.

## Guide

### Three Tier App

This use-case will focus on conversion from CentOS (though this could be another RHEL derivitive) to RHEL while maintaining a 3 tier application stack (do no harm). We will utilize an additional project in Ansible Automation Platform, "Three Tier App / Dev", which will allow us to install a three tier application stack, consisting of HAProxy, Tomcat, and PostgreSQL, across the three CentOS nodes. Additionally, the project also provides a means to test/verify functionality of the application components, which we will perform before and after CentOS to RHEL conversions.

| Role                                   | Inventory name | 
| ---------------------------------------| ---------------|
| Automation controller                  | ansible-1      |
| Satellite Server                       | satellite      |
| CentOS/OracleLinux Host 4 - HAProxy    | node4          |
| CentOS/OracleLinux Host 5 - Tomcat     | node5          |
| CentOS/OracleLinux Host 6 - PostgreSQL | node6          |

| **A Note about using Satellite vs. Ansible Automation Platform for this...**<br>  |
| ------------- |
| Out of the box, Satellite 6 supports [RHEL systems roles](https://access.redhat.com/articles/3050101) (a collection of Ansible Roles) for a limited set of administration tasks. Satellite can be used to do OS conversions and upgrades, however an Ansible Automation Platform Subscription is required to execute complicated OS conversions and upgrades that require logic to meet up-time requirements.  Using these two solutions together ensures you have the best tool for the job for:<br>- Content Management (Satellite)<br>- OS Patching & Standardized Operating Environments (Satellite)<br>- Provisioning: OS, Infra Services and Applications/Other (Satellite and/or Ansible Automation Platform)<br>- Configuration of Infra and Apps (Ansible Automation Platform)<br><br>Reference: [Converting CentOS to RHEL with Red Hat Satellite 6](https://www.redhat.com/en/blog/steps-converting-centos-linux-convert2rhel-and-red-hat-satellite) and [Leapp Upgrade with Satellite 6](https://www.redhat.com/en/blog/leapp-upgrade-using-red-hat-satellite-6)|

### Step 1 - Set Instance Tags 

- Return to the AAP Web UI browser tab you opened in step 3 of the previous exercise. Navigate to Resources > Templates by clicking on "Templates" under the "Resources" group in the navigation menu. This will bring up a list of job templates that can be used to run playbook jobs on target hosts:

  ![Job templates filtered list](images/set_instance_tags_01.png)

- In the filter box enter **Set instance tag** and click the magnifying glass.

  ![Access job template details](images/set_instance_tags_02.png)

- Click on the job template named **EC2 / Set instance tag - AnsibleGroup** to view the job template details.

  ![View job template details](images/set_instance_tags_03.png)

- Click on the variable section expansion icon to view the entire variable definition for the job template.

  ![Expanded job template variables](images/set_instance_tags_04.png)

- Note the **group_tag_map**...each node is being mapped to a group name which corresponds to the particular application tier role that the node is to serve. Looking at the Ansible playbook that corresonds to this job template:

  ![Ansible playbook for group tags](images/set_instance_tags_06.png)

- We can see that the **group_tag_map** dictionary is looped through, selecting a particular instace via the *resource: "{{ host_ec2_instance_id[item.key] }}"* filter and then setting the "AnsibleGroup" tag via *AnsibleGroup: "{{ item.value }}"*

- Click "Done" and then click "Launch" on the **EC2 / Set instance tag - AnsibleGroup** job template screen.

  ![View job run details](images/set_instance_tags_05.png)

- The job run should complete in 8 to 10 seconds. Note the item --> key, value output listing, which corresponds to the instance tag "AnsibleGroup" that is being set respectively on each instance.

### Step 2 - Update Ansible Inventory

- Now that we have application tier group tags set, we can update the Ansible inventory to include inventory groups associated with our application stack tiers. However, before we do, let's review how the Ansible inventory update works behind the scenes.

  ![Controller inventories](images/update_controller_inventory_01.png)

- On the left menu bar, select "Inventories" and then click on "EC2 Dynamic Inventory".

  ![Controller inventories details](images/update_controller_inventory_02.png)

- Initially, the **EC2 Dynamic Inventory** inventory details tab will be displayed. Click on the "Sources" tab.

  ![Controller inventories sources](images/update_controller_inventory_03.png)

- The **EC2 Dynamic Inventory** inventory sources tab will be displayed. Click on the "CentOS7 Development" inventory source.

  ![Controller inventories details expand](images/update_controller_inventory_04.png)

- The **CentOS7 Development** inventory source details will be displayed. Click on the variables expansion button on the side right.

  ![Controller inventories keyed_groups](images/update_controller_inventory_05.png)

- Scroll down the source variables section until you see "keyed_groups". [Keyed groups](https://docs.ansible.com/ansible/latest/plugins/inventory.html#:~:text=with%20the%20constructed-,keyed_groups,-option.%20The%20option) are where you can define dynamic inventory groups based on instance tags. In this case, when a dynamic inventory generation event is executed, if the EC2 inventory plugin comes across an instance with the "AnsibleGroup" tag, then it will create an inventory group with the name prefixed by "AnsibleGroup" then the default separator "_" (underscore) and then the value assigned to the "AnsibleGroup" tag...so in this case, if the "AnsibleGroup" tag is currently set to "appdbs", then the inventory group "AnsibleGroup_appdbs" will be created (or confirmed if already existing) and that instance will be assigned to the group.

- Click on "Done" in the Source variables exapanded view and then click on "Sync". The sync should complete in a few seconds. Now, let's verify our dynamic inventory group.

  ![Controller inventories groups](images/update_controller_inventory_06.png)

- In the menu bar on the left, select "Inventories" --> "EC2 Dynamic Inventories". Once the **EC2 Dynamic Inventory** details screen is displayed, click on the "Groups" tab.

  ![Controller inventories group](images/update_controller_inventory_07.png)

- The **Groups** defined for the **EC2 Dynamic Inventory** screen is displayed. Click on the "AnsibleGroups_appdbs" group.

  ![Controller inventories group](images/update_controller_inventory_08.png)

- Initially, the "Details" tab will display. Click on the "Hosts" tab and we will see that `node6.example.com` is present in the "AnsibleGroups_appdbs" group. Remember earlier, when we reviewed the variables section of the **EC2 / Set instance tag - AnsibleGroup** job template? The **group_tag_map** matches up:

```
  "group_tag_map": {
    "node4.example.com": "frontends",
    "node5.example.com": "apps",
    "node6.example.com": "appdbs"
  }
```

- Rather than having to navigate to **Inventories > EC2 Dynamic Inventory > Sources > _inventory_source_name_** and clicking "Sync" to initiate sync events, we have put together a job template with surveys and associated Ansible playbook that provides a more convenient method to utilize for this task. Let's review:

- Navigate to Resources > Templates by clicking on "Templates" under the "Resources" group in the navigation menu. This will bring up a list of job templates that can be used to run playbook jobs on target hosts:

  ![Job templates listed on AAP Web UI](images/update_inventory_01.png)

- Click ![launch](images/convert2rhel-aap2-launch.png) to the right of **CONTROLLER / Update inventories via dynamic sources**.  

  ![Inventory update job survey prompt on AAP Web UI](images/update_inventory_02.png)

- Next we see the job template survey prompt. A survey is a customizable set of prompts that can be configured from the Survey tab of the job template. For this job template, the survey allows for choosing which part of the inventory will be updated. For "Select inventory to update" choose "CentOS7" from the drop-down and for "Choose Environment" select "Dev" and click the "Next" button.

  ![Inventory update job launch details on AAP Web UI](images/update_inventory_03.png)

- This will bring you to a preview of the selected job options and variable settings. Click "Launch".

  ![Inventory update job output on AAP Web UI](images/update_inventory_04.png)

- Survey the job output details...this job run only takes about 5 to 6 seconds to complete.

### Step 3 - Install Three Tier Application

- Return to the AAP Web UI browser tab you opened in step 3 of the previous exercise. Navigate to Resources > Templates by clicking on "Templates" under the "Resources" group in the navigation menu. This will bring up a list of job templates that can be used to run playbook jobs on target hosts:

  ![Job templates listed on AAP Web UI](images/aap_templates.png)

- Use the side pane menu on the left to select **Templates**.

- Click ![launch](images/convert2rhel-aap2-launch.png) to the right of **CONVERT2RHEL / 98 - Three Tier App deployment** to launch the job.  This will take ~2 minutes to complete.

![3tier-install](images/convert2rhel-3tier-install.png)

### Step 4 - Smoke Test Three Tier Application

Now that our three tier application is installed, we will look at how we can automate the testing of the application stack functionality.

  ![Job templates listed on AAP Web UI 2](images/aap_templates_2.png)

- Use the side pane menu on the left to select **Templates**.

- Click ![launch](images/convert2rhel-aap2-launch.png) to the right of **CONVERT2RHEL / 99 - Three Tier App smoke test** to launch the application test job.  This should take ~15 seconds to complete.

  ![3tier-smoke-test-output](images/convert2rhel-3tier-smoke-output.png)

- If the job template completes successfully, then we have verified that our three tier application stack is functioning correctly. If it fails, we know that something with the application stack is malfunctioning and we can begin the debugging process to determine where the problem resides.

## Conclusion

In this exercise, we learned about how to set instance tags to assist with identifying instances. We then turned to looking into how Ansible Automation Platform dynamic inventory sources can be utilized to generate various host groups with a given inventory. We followed that with performing an automated installation of an example three tier application stack. Finally, we verified the three tier application stack functionality via an automated application smoke test.

Use the link below to move on the the next exercise.

---

**Navigation**

[Previous Exercise](../1.1-setup/README.md) - [Next Exercise](../1.3-analysis/README.md)

[Home](../README.md)
