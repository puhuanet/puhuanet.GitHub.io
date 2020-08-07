# Ansible roles远程安装配置Windows的Zabbix agent


```
│  zabbix_agent.yml
│
└─zabbix_agent
    ├─files
    │  └─win_zabbix
    │      │  zabbix_agentd.exe
    │      │  zabbix_get.exe
    │      │  zabbix_sender.dll
    │      │  zabbix_sender.exe
    │      │  zabbix_sender.lib
    │      │
    │      └─zabbix_agentd.d
    │              custom.net.tcp.service.bat
    │              custom.net.tcp.service.conf
    │              custom.vfs.file.cksum.bat
    │              custom.vfs.file.cksum.conf
    │
    ├─handlers
    │      main.yml
    │
    ├─tasks
    │      Linux.yml
    │      main.yml
    │      userparameter.yml
    │      Windows.yml
    │
    ├─templates
    │      zabbix_agentd.conf.j2
    │
    └─vars
            metadata.yml
            RedHat.yml
            Windows.yml
```

## tasks\Windows.yml
```
---
# jacky@puhua.net
# zabbix_agent ip地址
- name: "Set default ip address for zabbix_agent_ip"
  set_fact:
    zabbix_agent_ip: "{{ hostvars[inventory_hostname]['ansible_ip_addresses'][0] }}"
  when:
    - zabbix_agent_ip is not defined
    - "'ansible_ip_addresses' in hostvars[inventory_hostname]"

# - debug: var=zabbix_agent_ip

# - debug: var=metadata

- name: "Set zabbix_agent_hostmetadata"
  set_fact:
    # zabbix_agent_hostmetadata: "{{ metadata['10.0.20.235'] }}"
    zabbix_agent_hostmetadata: "{{ metadata[zabbix_agent_ip] }}"
  ignore_errors: true

- debug: var=zabbix_agent_hostmetadata

# Windows架构
# - name: "Windows | Set default architecture"
#   set_fact:
#     windows_arch: 32

# - name: "Windows | Override architecture if 64-bit"
#   set_fact:
#     windows_arch: 64
#   when:
#     - ansible_architecture == "64-bit"

# 服务是否存在
- name: "Windows | Check if Zabbix agent service is present"
  win_service:
    name: "{{zabbix_win_agent_service}}"
  register: agent_service_status

- debug: var=agent_service_status.exists
    
# agent文件是否存在
- name: "Windows | Check if Zabbix agent is present"
  win_stat:
    path: '{{ zabbix_win_exe_path }}'
  register: agent_file_info

- debug: var=agent_file_info.stat.exists

# 停止服务
- name: "Windows | Stop Zabbix agent service"
  win_service:
    name: "{{zabbix_win_agent_service}}"
    state: stopped
    start_mode: auto
  when:
    - agent_service_status.exists
    - agent_file_info.stat.exists
  notify: restart win zabbix agent

# 卸载服务
- name: "Windows | Uninstall Zabbix agent service"
  win_command: '"{{ agent_service_status.path }}" --uninstall'
  when:
    - agent_service_status.exists
    - not agent_file_info.stat.exists
  ignore_errors: true

# 拷贝文件
- name: "Windows | Copy Zabbix agent files"
  win_copy:
    src: "win_zabbix/"
    dest: "{{zabbix_win_install_dir}}"

# 配置文件
- name: "Windows | Configure Zabbix agent"
  win_template:
    src: zabbix_agentd.conf.j2
    dest: '{{ zabbix_win_install_dir }}\zabbix_agentd.conf'
    backup: yes
  notify: restart win zabbix agent

# 安装服务
- name: "Windows | Install Zabbix agent service"
  win_command: '"{{ zabbix_win_exe_path }}" --config "{{ zabbix_win_install_dir }}\zabbix_agentd.conf" --install'
  when: not agent_service_status.exists

# 配置Windows防火墙策略
- name: "Windows | Firewall rule"
  win_firewall_rule:
    name: Zabbix Agent
    localport: "{{ zabbix_agent_listenport }}"
    action: allow
    direction: in
    protocol: tcp
    state: present
    enabled: yes
```

