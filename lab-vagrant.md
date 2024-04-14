# Lab - Vagrant

## Build the VM

Build the VM by running the following command in the same directory as `Vagrantfile`.

```shell
vagrant up
```

## Connect to the VM

Connect to the VM by running the following command.

```shell
vagrant ssh
```

Login credentials:

- Username: `vagrant`

- Password: `vagrant`

## Sudo mode in VM

Use the VM in sudo mode to avoid permission issues.

```shell
sudo -s
```

## Access synced directory in VM

The directory containing the `Vagrantfile` is accessible inside the VM.

```shell
cd /vagrant
```
