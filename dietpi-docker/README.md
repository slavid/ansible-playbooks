# Update docker socket to bind on port 2375

### Manual way:

Edit file `/usr/lib/systemd/system/docker.service` adding `-H tcp://0.0.0.0:2375` at the end of the line that starts with `ExecStart=` if not present.

It will look like this: `ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock -H tcp://0.0.0.0:2375`

Then:\
`sudo systemctl daemon-reload`\
`sudo systemctl restart docker`

### Ansible way

```yml
- hosts: all
  vars:
    file_to_edit: /usr/lib/systemd/system/docker.service
    text_to_add: "-H tcp:\/\/0.0.0.0:2375"
  become: yes
  tasks:

    - name: Is docker daemon already bind to port 2375
      ansible.builtin.command: /bin/grep "\{{ text_to_add }}" '{{ file_to_edit }}'
      ignore_errors: true
      register: result
      failed_when: "result.rc == 1"                 # Grep command returns 0 if string found and 1 if didn't find the string.


    - name: Bind docker daemon to port 2375
      lineinfile:
        path: '{{ file_to_edit }}'
        state: present
        backrefs: true
        regex: '^ExecStart=(.*)'
        line: 'ExecStart=\1 {{  text_to_add }}'
      when: result is failed                        # Only execute this task if grep command failed (didn't find the string).
      register: file_change

    - name: Restart Docker
      systemd:
        daemon_reload: true
        name: docker
        state: restarted
      when: file_change.changed                     # Only execute this task if the previous one was executed (changed) and not skipped.
```