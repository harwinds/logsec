---
title: "Ansible - Making API calls idempotent"
date: 2023-01-28
description: "Use the Ansible URI module to call the Kibana Fleet API idempotently, choosing between POST and PUT based on which policies already exist."
tags: [ansible, automation, elastic, kibana, api]
ShowToc: false
---

This article is about the Ansible [URI module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/uri_module.html). In the last couple of months, I have spent a lot of time around automation of Elasticsearch and wanted to share some useful information about how to use Ansible to interact with REST API endpoints and make them idempotent.

We all know Ansible modules are [idempotent](https://docs.ansible.com/ansible/latest/reference_appendices/glossary.html#term-Idempotency) in nature, which means performing an operation once has exactly the same result as performing it repeatedly without any intervening actions. However, sometimes making the tasks idempotent requires additional work. For instance, in the URI module we have to decide when to use POST or PUT calls.

### Difference between POST and PUT?

Both `POST` and `PUT` are used to send data to a server to create/update a resource. The difference between POST and PUT is that PUT requests are idempotent. That is, calling the same PUT request multiple times will always produce the same result. In contrast, calling a POST request repeatedly has side effects of creating the same resource multiple times or failing with an error message that the resource already exists.

Let's start with the playbook:

```
---
- name: Kibana Fleet tasks
  hosts: localhost
  connection: local
  roles:
    - fleet
```

Directory structure of the `fleet` role:

```
fleet/
├── tasks       # Includes Ansible tasks using the URI module
├── policies    # Fleet policies
└── vars        # Variables
```

Variables `main.yml` in the vars directory:

```
---
kibana_url: https://<KIBANA_URL>:5601

fleet_username: <FLEET_USERNAME>
fleet_password: <FLEET_PASSWORD>

force_basic_auth: true
validate_certs: false
```

policies/elasticagent.json

```
{
    "name": "elasticagent",
    "description": "Elastic-Agent Endpoints",
    "namespace": "default",
    "monitoring_enabled": [
        "logs",
        "metrics"
    ]
}
```

Below are the List Policies related tasks under the `tasks` directory. We are making the `GET` call first to list the existing policies, so that later we can decide, based on the response of the GET call, whether we need to create a new policy or update an existing policy. For the new policies, we have created a new variable called `policies_to_create` based on the template files in the directory called policies. 

Later, we ran a delta using the `difference` filter between the two variables `policies_to_create` & `policies_to_update`. The result is the list of policies that we need to create.

More on difference filter: https://docs.ansible.com/ansible/devel/collections/ansible/builtin/difference_filter.html

```
---
- name: List Policies
  ansible.builtin.uri:
    url: "{{ kibana_url }}/api/fleet/agent_policies"
    method: GET
    body_format: json
    status_code: [200]
    user: "{{ fleet_username }}"
    password: "{{ fleet_password }}"
    force_basic_auth: "{{ force_basic_auth }}"
    validate_certs: "{{ validate_certs }}"
    headers:
      kbn-xsrf: "true"
  ignore_errors: true
  register: list_policies_response

- name: Extract all policies from the list_policies_response
  ansible.builtin.set_fact:
    policies: "{{ policies | default({}) | combine({item.name: item.id}) }}"
    policies_to_update: "{{ policies_to_update | default([]) + [item.name] }}"
  with_items: "{{ list_policies_response | json_query('json.items') }}"

- name: Extract all policies_to_create from the folder "policies/*.json"
  ansible.builtin.set_fact:
    policies_to_create: "{{ policies_to_create | default([]) + [item | basename | splitext | first] }}"
  with_fileglob: "policies/*.json"

- name: Gather all policies that need to be created
  ansible.builtin.set_fact:
    policies_to_create: "{{ policies_to_create | difference(policies_to_update) | list }}"
  when:
    - policies_to_update is defined
    - policies_to_update != []
```

- Create or Update Policies 

Based on the two variables `policies_to_create` & `policies_to_update`, we group our create and update tasks into two blocks.  

```
---
- name: Print required variables
  ansible.builtin.debug:
    msg:
      - "policies_to_create -- {{ policies_to_create | default([]) }}"
      - "policies_to_update -- {{ policies_to_update | default([]) }}"

- name: A group of tasks to be executed when policies_to_create is defined
  when:
    - policies_to_create is defined
    - policies_to_create != []
  block:
    - name: Create Agent Policies
      ansible.builtin.uri:
        url: "{{ kibana_url }}/api/fleet/agent_policies"
        method: POST
        body_format: json
        body: "{{ lookup('file', 'policies/{{ item }}.json') }}"
        status_code: [200]
        user: "{{ fleet_username }}"
        password: "{{ fleet_password }}"
        force_basic_auth: "{{ force_basic_auth }}"
        validate_certs: "{{ validate_certs }}"
        headers:
          kbn-xsrf: "true"
      ignore_errors: true
      register: agent_policies_response
      with_items: "{{ policies_to_create }}"

    - name: Response
      ansible.builtin.debug:
        msg: "{{ agent_policies_response }}"

    - name: Extract the policy_ids from the agent_policies_response variable
      ansible.builtin.set_fact:
        policy_ids: "{{ policy_ids | default({}) | combine ({item.0 : item.1}) }}"
      with_together:
        - "{{ agent_policies_response | json_query('results[*].json.item.name') }}"
        - "{{ agent_policies_response | json_query('results[*].json.item.id') }}"

    - name: Debug
      ansible.builtin.debug:
        msg: "POLICY IDS - {{ policy_ids }}"

- name: A group of tasks to be executed when policies_to_update is defined
  when:
    - policies_to_update is defined
    - policies_to_update != []

  block:
    - name: Update Agent Policies
      ansible.builtin.uri:
        url: "{{ kibana_url }}/api/fleet/agent_policies/{{ policies | json_query(item) }}"
        method: PUT
        body_format: json
        body: "{{ lookup('file', 'policies/{{ item }}.json') }}"
        status_code: [200]
        user: "{{ fleet_username }}"
        password: "{{ fleet_password }}"
        force_basic_auth: "{{ force_basic_auth }}"
        validate_certs: "{{ validate_certs }}"
        headers:
          kbn-xsrf: "true"
      ignore_errors: true
      register: agent_policies_response
      with_items: "{{ policies_to_update }}"

    - name: Response
      ansible.builtin.debug:
        msg: "{{ agent_policies_response }}"

    - name: Extract the policy_ids from the agent_policies_response variable
      ansible.builtin.set_fact:
        policy_ids: "{{ policy_ids | default({}) | combine ({item.0 : item.1}) }}"
      with_together:
        - "{{ agent_policies_response | json_query('results[*].json.item.name') }}"
        - "{{ agent_policies_response | json_query('results[*].json.item.id') }}"

    - name: Debug
      ansible.builtin.debug:
        msg: "POLICY IDS - {{ policy_ids }}"

```

> Note: As we can see, there is some repeated code between two blocks for module `set_fact`, I tried to move this code outside of the block to keep the code clean, but the variable value is still being updated when the tasks are being skipped. 

Ansible documentation on [conditionals](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_conditionals.html) states "Ansible always registers something in a registered variable for every host, even on hosts where a task fails or Ansible skips a task because a condition is not met." 

Is this the best approach to make API calls idempotent? I'm not sure. There are several things that can be improved, but for now this is good enough for our use case. One of the shortcomings is that if a user creates a Fleet policy through Kibana console, and we run this role again, it will fail, as it will attempt to update the existing policy, but will not find the policy in the policies directory or in the code itself.

Hope you find this post helpful.

---
### References

- [HTTP Request Methods](https://www.w3schools.com/tags/ref_httpmethods.asp)
- [Kibana Fleet APIs](https://petstore.swagger.io/?url=https://raw.githubusercontent.com/elastic/kibana/8.6/x-pack/plugins/fleet/common/openapi/bundled.json)
