# How I Automated a Cloud

Not quite a whole Cloud, but a Juniper based Leaf/Spine EVPN fabric that supports a Red Hat OpenStack deployment. This repository contains a framework as opposed to a full deployment but gives the general idea as to how we used Ansible with Jinja2 to build and deploy the networking aspects of our Cloud environment. Whilst we have used Ansible before, we had never used it on this scale - we wanted Ansible to deploy everything, with no need for manual configuration (Network as Code).

The initial challenge was how do we get all the information about interfaces, IP addresses and BGP ASNs into the Jinja2 templates that we were going to use to generate our device configurations. Anyone who is familiar with Ansible knows information is provided in `group_vars` and `host_vars`, but when you have nearly a hundred devices per environment you don't want to be creating and updating them by hand. Excel is quite common with Engineers that deploy networks as it is usually used for patching schedules, which detail what needs to be connected to what. As we didn’t want to duplicate information in multiple places, we wanted to use this information as input to our Ansible automation, but it didn't contain everything we needed.

One important point that I can't emphasise enough when designing a large deployment, is to design it with automation in mind. Without standardisation you can't easily template it, which is key to automation unless you want lots and lots of lookup tables. Make things deterministic as much as possible, for example, design it so that leaf-01 connects to port 1 and leaf-02 connects to port 2 on your spine switches, allocate a point-to-point subnet per spine switch and use the leaf number to determine the subnet to use and allocate BGP ASNs based on your leaf or spine numbers. When allocating anything, think how can I devise a function that can work it out without having to perform a lookup.

We wanted to make it easy for someone to describe a whole environment without having to modify lots of different files, so we came up with the approach of using two files per environment (a CSV file that was based on a patching schedule and a YAML file which allowed us to specify subnets and other important bits of information). These two files were stored in a Git repository so they were version controlled and could be used to trigger a CI/CD pipeline if we decided it was ever a good idea to automatically push changes en masse to a Live production environment.

Instead of writing a throwaway Python script which parsed these two files (as lots of people have done), we decided to write a generic tool [JinjaFx](https://pypi.org/project/jinjafx/) which took CSV and/or YAML as input and combined it with a Jinja2 template to generate our `group_vars` and `host_vars` that Ansible would then use:

```
.
└── /contrib
    ├── GenerateSiteVars.j2
    ├── site_a.csv
    └── site_a.yml
```

With these files, we can then run the following commands:

```
python3 -m pip install --user jinjafx
 
python3 -m jinjafx -g contrib/site_a.yml -d contrib/site_a.csv -t contrib/GenerateSiteVars.j2
```

This results in the following files being generated - the `GenerateSiteVars.j2` template will need to be tailored to your environment as it currently only adds a subset of the information required to deploy all the networking aspects. If you look at `site_a.csv` you will also notice it doesn't include all the connections required - JinjaFx supports something that I term dynamic CSV, where we can use counters and RegEx style character classes and groups to expand rows into lots of rows if there is a pattern - the following CSV will get expanded to over 200 lines:

DEVICE | INTERFACE | HOST | PORT | TAG
--- | --- | --- | --- | ---
`mx-{1-2:1}%2` | `et-0/1/\1` | `spine-({1-3:1})%2` | `et-0/0/{0\|63:2}` | `Underlay`
`spine-({1-3:1})%2` | `et-0/0/{0\|63}` | `mx-{1-2:1}%2` | `et-0/1/\1` | `Underlay`
`leaf-({1-32:1})%2` | `et-0/0/\2` | `spine-({1-3:1})%2` | `et-0/0/\1` | `Underlay`
`spine-({1-3:1})%2` | `et-0/0/\2` | `leaf-({1-32:1})%2` | `et-0/0/\1` | `Underlay`

The syntax `{1-32:1}` is a counter which will create rows using the values 1 to 32 with an increment of 1. The `%2` syntax will ensure the number is zero padded to 2 digits. The `{0|63}` syntax will alternate the value between 0 and 63 for subsequent rows. The `()` and `\n` syntax are standard Regex style capture groups that allows you to copy a value and use it elsewhere.

You should never modify anything within the `sites` directory - only ever modify the two site-specific files within the `contrib` directory and then re-run the above command to re-generate the files within the `sites` directory.

```
894B > sites/site_a/hosts.yml
49B > sites/site_a/group_vars/all.yml
429B > sites/site_a/host_vars/ukabc-prd-mx-01.yml
432B > sites/site_a/host_vars/ukabc-prd-mx-02.yml
4.73kB > sites/site_a/host_vars/ukabc-prd-spine-01.yml
4.77kB > sites/site_a/host_vars/ukabc-prd-spine-02.yml
4.74kB > sites/site_a/host_vars/ukabc-prd-spine-03.yml
444B > sites/site_a/host_vars/ukabc-prd-leaf-01.yml
444B > sites/site_a/host_vars/ukabc-prd-leaf-02.yml
...
449B > sites/site_a/host_vars/ukabc-prd-leaf-31.yml
449B > sites/site_a/host_vars/ukabc-prd-leaf-32.yml
```

Now we have a way to generate our site-specific `group_vars` and `host_vars` on demand, we need to look at how we are going to structure all of this. One of the good features of Ansible is the ability to define `group_vars` and `host_vars` at different levels to create global ones and site-specific ones:

```
.
├── playbook-build.yml
├── playbook-deploy.yml
|
├── /group_vars
│   ├── all.yml
│   ├── leaf.yml
│   ├── spine.yml
|   └── mx.yml
|
└── /sites
    └── /site_a
        ├── hosts.yml
        |
        ├── /group_vars
        │   └── all.yml
        |
        └── /host_vars
            └── <hostname>.yml
```

Ansible will first process `group_vars` and `host_vars` at the same level as our Playbooks and then it will process `group_vars` and `host_vars` at the same level as the site-specific inventory `hosts.yml`. Anything that is defined in `group_vars/all.yml` can be overridden by defining the same variable within `sites/site_a/group_vars/all.yml`. We also have global `group_vars` that define variables that are specific to the device group (i.e. "leaf", "spine" or "mx") as defined in the inventory.

The final piece of the puzzle was our Jinja2 templates used to generate our device configurations which we place within a `templates` directory (the main templates will include different re-usable bits of configuration from the `includes` directory):

```
.
└── /templates
    ├── fabric.j2
    |
    └── /includes
        └── junos.<*>.j2
```

The Ansible Playbook `playbook-build.yml` defines a build task using the `template` module - the required template is defined within `group_vars/leaf.yml`, `group_vars/spine.yml` or `group_vars/mx.yml` depending on the device type. This template will then import other templates from the `templates/includes` directory as required. These templates will reference variables which are defined within the `group_vars` and `host_vars`.

```yaml
- hosts: all
  connection: local
  gather_facts: no

  tasks:
    - name: "create local build directory"
      file:
        path: "build/"
        state: directory
      run_once: True

    - name: "build device configurations"
      template:
        src: "templates/{{ template }}"
        dest: "build/{{ inventory_hostname }}.cfg"
```

By running the following command it will build all the configurations for a specific site:

```
ansible-playbook playbook-build.yml -i sites/site_a/hosts.yml
```

We then used the `juniper.device.config` module from the `juniper.device` Ansible collection in the following Playbook to deploy our configuration to our Juniper devices over NETCONF - we started off by deploying the configuration using `load: replace` so we automated subsets of it, but we have now migrated to using `load: override` as all network configuration is generated by our Ansible templates (this is defined as `deploy_method` within our `group_vars`).

```yaml
- hosts: all
  connection: local
  gather_facts: no

  tasks:
    - name: "junos commit confirmed"
      juniper.device.config:
        load: "{{ deploy_method }}"
        format: "text"
        src: "build/{{ inventory_hostname }}.cfg"
        check_commit_wait: 10
        confirmed: 5
        diff: true
      register: response

    - name: "diff"
      debug: var=response.diff_lines
      when: response.diff_lines is defined

    - name: "junos commit"
      juniper.device.config:
        commit: true
      when: response.diff_lines is defined
```

We then run the following command to deploy the changes for a specific site:

```
ansible-playbook playbook-deploy.yml -i sites/site_a/hosts.yml -u <USER> -k
```
