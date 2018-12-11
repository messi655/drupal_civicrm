# 1. What will playbook do?

- Create SSL/TLS certificate.
- Install PHP packages and dependencies.
- Install MySQL Server
- Install and Configure Nginx, PHP-FPM.
- Prepare Drupal for installation.

## Required
- Ansible 2.7
- Make sure you can access to Drupal server by SSH key
- Make sure the account that you use to login must have sudo permission (without password)
- To get more: https://www.ansible.com/

## How to run playbook 

- Edit group_vars/all.yml to change the domain, email,.. for matching with your requirement
- Edit inventories/hosts for matching with your Drupal server
- Run

```
ansible-playbook --private-key /path/to/private_key -i inventories/hosts roles.yml
```

# 2. What will you do?

*After finish `1` You will do some of remain steps by manually:*

- Create database for Drupal and CiViCRM 
- Access http://your_domain to install drupal
- Install CiViCRM https://docs.civicrm.org/sysadmin/en/latest/install/drupal7/