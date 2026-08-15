Ansible ad-hoc commands are quick, single-line commands you run directly from the terminal to perform simple tasks across your servers without writing a full YAML playbook.

Think of them as a way to do something fast—like rebooting servers, checking uptime, or quickly starting a service—across your entire infrastructure at once.

### The Basic Syntax

The structure of an ad-hoc command usually looks like this:

```bash
ansible [target] -i [inventory] -m [module] -a "[arguments]" -b

```

Here is what each part means:

* **`[target]`**: Which servers to run this on (e.g., `all`, `webservers`, or a specific IP).
* **`-i`**: Your inventory file (just like you used with `-i inventory`).
* **`-m`**: The module you want to use (like `ping`, `command`, `dnf`, or `service`).
* **`-a`**: The arguments you pass to the module.
* **`-b`**: Become (`sudo` privileges). Required for administrative tasks.

---

### Everyday Examples

Here are a few practical ad-hoc commands you might use regularly on your servers.

**1. Check if all servers are reachable**
This uses the `ping` module to verify Ansible can connect to and authenticate with your nodes.

```bash
ansible all -i inventory -m ping

```

**2. Check the uptime of all servers**
This uses the `command` module to execute a standard Linux command on the remote machines.

```bash
ansible all -i inventory -m command -a "uptime"

```

**3. Install a package quickly**
Since you are working with `dnf` on distributions like Rocky Linux, you can install a package across all machines instantly.

```bash
ansible all -i inventory -b -m dnf -a "name=git state=present"

```

**4. Start and enable a service**
If you just installed Nginx or Grafana and need to make sure the service is running across your servers:

```bash
ansible all -i inventory -b -m service -a "name=nginx state=started enabled=yes"

```

**5. Gather system information (Facts)**
This pulls all the hardware, IP, and OS information from your remote servers. It returns a lot of JSON data.

```bash
ansible all -i inventory -m setup

```

### When to use Ad-Hoc vs. Playbooks

| Feature | Ad-Hoc Commands | Playbooks (`.yml`) |
| --- | --- | --- |
| **Best For** | Quick, one-off tasks (restarts, updates, checks). | Complex, multi-step deployments and configuration management. |
| **Reusability** | Low (typed directly into the terminal). | High (saved in version control like Git). |
| **Execution** | Runs a single module. | Runs multiple tasks and modules in a specific sequence. |
