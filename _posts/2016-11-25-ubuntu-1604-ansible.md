---
layout: post
title: "Ubuntu 16.04を対象にAnsibleを実行する"
description: ""
date: 2016-11-25
tags: ["ansible"]
comments: true
share: true
---

これまでサーバーを構築するときはUbuntu14.04を使ってきましたがそろそろ16.04にしようかなー、と思ってAnsibleを使うときにちょっとハマったのでメモ。

## 問題点
何も考えずにUbuntu 16.04を対象にAnsibleを走らせると

```
TASK [setup] ************************
fatal: [ubuntu-test]: FAILED! => {"changed": false, "failed": true, "module_stderr": "", "module_stdout": "\r\n/bin/sh: 1: /usr/bin/python: not found\r\n", "msg": "MODULE FAILURE"}
```

このようにpythonが見つからないので失敗する。  
それもそのはずで、Ubuntu 16.04ではPython3がデフォルトとなりUbuntu Server等のイメージではPython2がインストールされていない。

## 解決策
できれば手動でPython2をインストールするのは避けたいので、その処理自体を自動化する方法を模索する。  
これについて調べると、[この回答のように](http://stackoverflow.com/a/34402816)`gather_facts`を切って`pre_tasks`としてpython-simplejsonをインストールする方法が見つかる

```yaml
- hosts: my_hosts
  become_user: root
  become: yes
  gather_facts: no
  pre_tasks:
    - name: Install python-simplejson
      raw: sudo apt-get -y install python-simplejson
  roles:
    - some_sole
```

上記の方法により実行するtaskより先にPython2をインストールすることが出来るのだが、`gather_facts`を切ったことにより`ansible_os_family`などの変数が使用できなくなる。  
これでは困るのでPython2をインストールした後で明示的に`gather_facts`を行うため[setupモジュール](http://docs.ansible.com/ansible/setup_module.html)を呼び出す。

```yaml
- hosts: my_hosts
  become_user: root
  become: yes
  gather_facts: no
  pre_tasks:
    - name: Install python-simplejson
      raw: sudo apt-get -y install python-simplejson
    - name: Gathers facts
      setup:
  roles:
    - some_sole
```

これでUbuntu 16.04でも問題なくAnsibleを実行できた。


ちなみに、Ansible2.2から[Python3の対応がされるようになった](http://docs.ansible.com/ansible/python_3_support.html)らしい(まだ試験段階)  
Ansible2.3になれば今回の問題は意識する必要がなくなるかも...
