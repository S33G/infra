# ansible

## Setup

```sh
pip install ansible
ansible-galaxy install -r requirements.yml --force
```

# 
ansible-playbook playbooks/update.yml
ansible-playbook playbooks/media.yml
ansible-playbook playbooks/update.yml -l media
