# kafka-summit-cluster-linking

## Abstract

Cluster Linking enables you to directly connect clusters by mirroring topics from one cluster to another and makes it easy to build multi-datacenter, multi-region, and hybrid cloud deployments.

We have implemented Cluster Linking for enabling data sharing across environments, so we became very aware of all the necessary configurations needed for both meeting the business requirements and ensuring good performance, which will be discussed in this session. Additionally, this experience also enabled us to devise additional use cases for leveraging this functionality. 

Expect to leave this talk understanding more about Cluster Linking, including how to take advantage of this feature to unlock alternative solutions beyond data sharing.

## Main Pipeline

```yaml
trigger: none

schedules:
  - cron: "15 */2 * * *"
    displayName: scheduled mirror topics creation for Cluster Linking
    branches:
      include:
        - master
    always: true

parameters:
  - name: environments
    type: object
    default: [dev, tst, prod]
  - name: 'topics_to_mirror_input'
    type: string
    default: ''

resources:
  repositories:
    - repository: datahub-state
      type: git
      name: DataHub/datahub-state

stages:
  - ${{ each environment in parameters.environments }}:
      - stage: get_topics_to_mirror_${{ environment }}
        displayName: ${{ environment }} - topics to mirror
        dependsOn: []
        jobs:
          - job: listTopicsToMirror
            displayName: ${{ environment }} - topics to mirror
            steps:
              - checkout: datahub-state
              - bash: |
                  topics_directory_path=./src/environments/dk-${{ environment }}/topics/
                  topics_to_mirror=""
                  if [ -z "${{ parameters.topics_to_mirror_input }}" ]
                  then
                    for file in $(find $topics_directory_path -name '*.yaml' -o -name '*.yml');
                    do
                        topic_name=$(yq '.name' $file)
                        replicated=$(yq e 'keys | .[] | select(. == "*replicated")' $file)
                        if [ -z "$replicated" ]
                        then
                            if [ -z "$topics_to_mirror" ]
                            then
                              comma=""
                            else
                              comma=","
                            fi
                            echo "Replicated flag does not exist for $topic_name -> topic should be mirrored"
                            topics_to_mirror+="${comma}${topic_name}"
                        else
                            echo "Replicated flag exists for $topic_name  -> topic should not be mirrored"
                        fi
                    done;
                  fi
                  echo "##vso[task.setvariable variable=topics;isOutput=true]$topics_to_mirror"
                name: topics_to_mirror
  - ${{ each environment in parameters.environments }}:
      - stage: manage_cluster_linking_${{ environment }}
        displayName: ${{ environment }} - manage Cluster Linking
        dependsOn:
          - get_topics_to_mirror_${{ environment }}
        jobs:
          - job: manageClusterLinking
            displayName: ${{ environment }} - manage Cluster Linking
            pool:
              name: ALM-Linux-OnPrem
              demands:
                - docker
            container:
              image: artifacts.cf.saxo/docker/teams/datahub/infrastructure:3.3.1
            variables:
              - name: mirrors_to_create
                value: $[ stageDependencies.get_topics_to_mirror_${{ environment }}.listTopicsToMirror.outputs['topics_to_mirror.topics'] ]
              - template: templates/run_datahub_ansible_variables.yml
                parameters:
                  environment: az-${{ environment }}
                  operation: manageClusterLinking
                  ansible_command_extra_options: '-e "topics_to_mirror=${{ parameters.topics_to_mirror_input }}$(mirrors_to_create)"'
            steps:
              - template: templates/run_datahub_ansible_steps.yml
                parameters:
                  environment: az-${{ environment }}
                  operation: manageClusterLinking
```


## Ansible

### Playbook

```yaml
# Auxiliary playbook to manage Cluster Linking creation and configuration.
#
#
# Example usage:
#   ansible-playbook -i <hosts_file> manage_cluster_linking.yml

---
- name: Get list of hosts
  hosts: localhost
  gather_facts: false
  environment:
    https_proxy: http://dk.proxy.mid.dom:80
    REQUESTS_CA_BUNDLE: files/cacert.pem  # See https://github.com/Azure/azure-cli/blob/dev/doc/use_cli_effectively.md#working-behind-a-proxy
  vars:
    azure_operation: get_hosts
  pre_tasks:
    - name: Run play only for a cloud environment
      tags: always
      meta: end_play
      when: not (topology is defined) | bool
  roles:
    - provisioning

- name: Manage Cluster Linking Role
  hosts: localhost
  gather_facts: false
  remote_user: "{{ ansible_ssh_user }}"
  roles:
    - manage_cluster_linking
```

### Roles - Defaults

```yaml
cluster_linking_oauth_file_path: /tmp/cluster-linking.properties
cluster_linking_not_found_string: "No cluster links found."
cluster_link_name: datahub-cluster-link-{{ 'az' if deployment_location is search('on-prem') else 'dk' }}-{{ environment_type }}-{{ environment_name }}
alter_configs_file_path: /tmp/cluster-linking-alter-configs.properties
alter_configs_repo_file_path: ./inventories/cluster-linking/{{ environment_name }}.properties
```

### Roles - Meta

```yaml
dependencies:
  - { role: common }
  - { role: predeployment }
```

### Roles - Tasks

```yaml
- name: Prepare operation
  block:
    - name: Gather facts
      setup:
    - name: Get jumpboxes names and IPs
      include_tasks: ../../../tasks/set-delegation-jumpboxes.yml

- name: Create Cluster Linking base config file
  delegate_to: "{{ broker_jumpbox_ip }}"
  template:
    src: cluster-linking.properties.j2
    dest: "{{ cluster_linking_oauth_file_path }}"
    mode: 0400

- name: List Cluster Links
  command: >
    kafka-cluster-links \
      --list \
      --bootstrap-server {{ broker_jumpbox_name }}:{{ common.broker.config.port }} \
      --command-config {{ cluster_linking_oauth_file_path }}
  delegate_to: "{{ broker_jumpbox_ip }}"
  changed_when: false
  vars:
    ansible_become: true
  register: list_cluster_link
  until: list_cluster_link.rc == 0
  retries: 5
  delay: 3

- name: Setup Cluster Link
  when: list_cluster_link is search(cluster_linking_not_found_string)
  delegate_to: "{{ broker_jumpbox_ip }}"
  block:
    - name: Copy file with Cluster Link configurations
      ansible.builtin.copy:
        src: "{{ alter_configs_repo_file_path }}"
        dest: "{{ alter_configs_file_path }}"
        mode: 0400
    - name: Create Cluster Link
      command: >
        kafka-cluster-links \
          --create \
          --bootstrap-server {{ broker_jumpbox_name }}:{{ common.broker.config.port }} \
          --link {{ cluster_link_name }} \
          --command-config {{ cluster_linking_oauth_file_path }} \
          --config-file {{ cluster_linking_oauth_file_path }}
      changed_when: false
      vars:
        ansible_become: true
      register: create_cluster_link
      until: create_cluster_link.rc == 0
      retries: 5
      delay: 3
    - name: Apply Cluster Link configurations
      command: >
        kafka-configs \
          --alter \
          --bootstrap-server {{ broker_jumpbox_name }}:{{ common.broker.config.port }} \
          --cluster-link {{ cluster_link_name }} \
          --add-config-file {{ alter_configs_file_path }} \
          --command-config {{ cluster_linking_oauth_file_path }}
      changed_when: false
      vars:
        ansible_become: true
      register: alter_cluster_link_config
      until: alter_cluster_link_config.rc == 0
      retries: 5
      delay: 3

- name: Get list of mirrors
  command: >
    kafka-mirrors \
      --list \
      --bootstrap-server {{ broker_jumpbox_name }}:{{ common.broker.config.port }} \
      --link {{ cluster_link_name }} \
      --command-config {{ cluster_linking_oauth_file_path }}
  delegate_to: "{{ broker_jumpbox_ip }}"
  changed_when: false
  vars:
    ansible_become: true
  register: mirror_list
  until: mirror_list.rc == 0
  retries: 5
  delay: 3

- name: Create mirrors
  when: item not in mirror_list.stdout_lines
  command: >
    kafka-mirrors \
      --create \
      --bootstrap-server {{ broker_jumpbox_name }}:{{ common.broker.config.port }} \
      --link {{ cluster_link_name }} \
      --mirror-topic {{ item }} \
      --command-config {{ cluster_linking_oauth_file_path }}
  delegate_to: "{{ broker_jumpbox_ip }}"
  changed_when: false
  vars:
    ansible_become: true
  register: create_mirrors
  until: create_mirrors.rc == 0
  loop: "{{ topics_to_mirror.split(',') }}"
  loop_control:
    pause: 3
  retries: 5
  delay: 3
```

### Roles - Templates

```yaml
{# This is not the proper way to do this, bootstrap servers for the Cluster Linking configuration will need to be in the inventories #}

{%- if environment_type == 'dev' %}

bootstrap.servers=DEVKAFKA4-DK1.sys.dom:9093,DEVKAFKA5-DK1.sys.dom:9093,DEVKAFKA6-DK1.sys.dom:9093

{% elif environment_type == 'tst' %}

bootstrap.servers=TSTDHBBRK1-DK1.tst2.dom:9093,TSTDHBBRK2-DK1.tst2.dom:9093,TSTDHBBRK3-DK1.tst2.dom:9093

{% elif environment_type == 'prod' %}

bootstrap.servers=dimbrk1-dk1.mid.dom:9093,dimbrk2-dk2.mid.dom:9093,dimbrk3-dk1.mid.dom:9093,dimbrk4-dk2.mid.dom:9093,dimbrk5-dk1.mid.dom:9093,dimbrk6-dk2.mid.dom:9093,dimbrk7-dk6.mid.dom:9093,dimbrk8-dk6.mid.dom:9093,dimbrk9-dk6.mid.dom:9093

{%- endif %}

sasl.login.callback.handler.class=com.saxobank.datahub.oauth.oauthbearer.OAuthAuthenticateLoginCallbackHandler
sasl.jaas.config=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required OAUTH_LOGIN_SERVER="login.microsoftonline.com/" OAUTH_LOGIN_ENDPOINT="/oauth2/token" OAUTH_LOGIN_GRANT_TYPE=client_credentials OAUTH_AUTHORIZATION="Basic {{ common.oauth_broker_client_secret }}" OAUTH_TENANT_ID="48794f31-2f6d-4909-8b2f-53b64c7f3199" OAUTH_CLIENT_ID="{{ common.oauth_broker_client_id }}" OAUTH_UNSECURE_HTTP_CONNECTION=false;
sasl.mechanism=OAUTHBEARER
security.protocol=SASL_SSL

ssl.truststore.location=/etc/kafka/broker-certificates/broker_all.truststore.jks
ssl.truststore.password={{ common.certificates_password }}

```

