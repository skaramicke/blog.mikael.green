---
title: "Securing kubectl config with 1Password"
date: 2023-12-20T07:26:13+01:00
draft: false
categories: ["Kubernetes", "1Password", "Security"]
---

Your `~/.kube/config` file is a treasure trove of sensitive information when accessing your Kubernetes cluster. It holds the keys to the kingdom: credentials, cluster certificate, and endpoint. While encrypting your hard drive offers a degree of protection, the ideal scenario is to avoid storing sensitive plain text files on your hard drive altogether.

<!--more-->

## Understanding the Basics

Here are some foundational concepts for beginners:

- **Kubernetes (`kubectl`)**: A tool for managing applications on clusters of hosts.
- **1Password**: A password manager for securely storing and accessing sensitive information.
- **Shell Scripting**: The practice of writing scripts to automate tasks in Unix-like systems.

But if you were unaware of this, I bet this guide is not for you 😅

## A Tailored Solution

For larger organizations, industry-standard solutions like Vault are common. But what if you're a solo entrepreneur or a small team? Vault might be overkill for your needs. Enter 1Password, a convenient way to store credentials and integrate them into your kubeconfig file.

I manage a cluster named "my-cluster" and store my kubeconfig file in a project directory that includes various deployment and definition files. 1Password's command-line tools are a godsend for extracting data from your vault. [Read about it here](https://developer.1password.com/docs/cli/get-started). Since I'm on macOS, I installed them via Homebrew:

```bash
brew install 1password-cli
```

Let's take a look at a sanitized version of my original kubeconfig file:

```yaml
apiVersion: v1
clusters:
  - cluster:
      certificate-authority-data: "lots of fun characters"
      server: https://123.123.123.123:1337
    name: my-cluster
contexts:
  - context:
      cluster: my-cluster
      user: my-cluster-admin
    name: my-cluster-admin@my-cluster
current-context: my-cluster-admin@my-cluster
kind: Config
preferences: {}
users:
  - name: my-cluster-admin
    user:
      client-certificate-data: "lots of fun characters again"
      client-key-data: "lots of fun characters a third time"
```

The first task is to transfer the crucial data - primarily the `client-certificate-data`, `client-key-data`, and `certificate-authority-data` - to 1Password. After creating a new Password item in 1Password, add custom fields for these values and fill them with data from your kubeconfig file.

![Store Kubeconfig values in 1Password](/images/1password-store-kubeconfig-values.png)

As a precaution, I also store a backup of the original kubeconfig file in 1Password, though this is optional.

Next, retrieve the secret reference for each value from 1Password:

![Get references to Kubeconfig values in 1Password](/images/1password-copy-secret-reference.png)

Rename your kubeconfig file, appending `.tpl` to its filename. Then, replace the secret values in this template file with their 1Password references:

```yaml
apiVersion: v1
clusters:
  - cluster:
      certificate-authority-data: "op://omitted-vault-id/My Kubernetes Cluster/certificate-authority-data"
      server: https://123.123.123.123:1337
    name: my-cluster
contexts:
  - context:
      cluster: my-cluster
      user: my-cluster-admin
    name: my-cluster-admin@my-cluster
current-context: my-cluster-admin@my-cluster
kind: Config
preferences: {}
users:
  - name: my-cluster-admin
    user:
      client-certificate-data: "op://omitted-vault-id/My Kubernetes Cluster/client-certificate-data"
      client-key-data: "op://omitted-vault-id/My Kubernetes Cluster/client-key-data"
```

### Automating the Magic

When I need to work on my Kubernetes cluster, I use a terminal. The `kubectl` command relies on an environment variable named `KUBECONFIG`. In my system, this file is ephemeral, appearing when needed and vanishing afterward.

To automate its appearance, I use the `op` CLI tool:

```zsh
my_cluster_kubeconfig_tpl=$HOME/Projects/my-cluster/kubeconfig-my-cluster.yaml.tpl
my_cluster_kubeconfig=$HOME/Projects/my-cluster/kubeconfig-my-cluster.yaml
echo "Creating kubeconfig for my-cluster"
op inject -i $my_cluster_kubeconfig_tpl -o $my_cluster_kubeconfig
export KUBECONFIG=$my_cluster_kubeconfig
```

To avoid unnecessary authentication prompts from 1Password, I implemented a simple check:

```zsh
my_cluster_kubeconfig_tpl=$HOME/Projects/my-cluster/kubeconfig-my-cluster.yaml.tpl
my_cluster_kubeconfig=$HOME/Projects/my-cluster/kubeconfig-my-cluster.yaml
if [ -f $my_cluster_kubeconfig ]; then
    echo "kubeconfig for my-cluster already exists"
else
    echo "Creating kubeconfig for my-cluster"
    op inject -i $my_cluster_kubeconfig_tpl -o $my_cluster_kubeconfig
fi
```

Ensuring the kubeconfig file disappears after use involves leveraging the `trap` command, which executes a function upon shell exit:

```zsh
function cleanup_kubeconfig {
    rm -f $my_cluster_kubeconfig
}

trap cleanup_kubeconfig EXIT
```

To handle multiple terminal sessions, the cleanup function was modified to check for other active sessions before removing the kubeconfig file:

```zsh
function cleanup_kubeconfig {
    # Remove the temporary file for this session
    rm -f "$session_file"

    # Check if there are any other session files remaining
    if [ $(ls /tmp/zsh_session_* 2> /dev/null | wc -l) -eq 0 ]; then
        echo "Removing kubeconfig for my-cluster"
        rm -f $my_cluster_kubeconfig
    else
        echo "kubeconfig for my-cluster still in use"
    fi
}

trap cleanup_kubeconfig EXIT
```

There's a minor caveat: if the terminal window crashes, the kubeconfig file might not get removed. I'm exploring solutions for this.

Interestingly, iTerm and macOS's built-in terminal handle crashed sessions differently, with iTerm maintaining the trap file and the built-in terminal failing to remove the kubeconfig file. However, this security gap is temporary and resolves with the next terminal session.

Finally, I incorporated the script into my `.zshrc` file for seamless integration:

```zsh
# Create a unique temporary file for the session
session_file="/tmp/zsh_session_$$"
touch "$session_file"

my_cluster_kubeconfig_tpl=$HOME/Projects/my-cluster/kubeconfig-my-cluster.yaml.tpl
my_cluster_kubeconfig=$HOME/Projects/my-cluster/kubeconfig-my-cluster.yaml
if [ -f $my_cluster_kubeconfig ]; then
    echo "kubeconfig for my-cluster already exists"
else
    echo "Creating kubeconfig for my-cluster"
    op inject -i $my_cluster_kubeconfig_tpl -o $my_cluster_kubeconfig
fi

export KUBECONFIG=$my_cluster_kubeconfig

function cleanup_kubeconfig {
    # Remove the temporary file for this session
    rm -f "$session_file"

    # Check if there are any other session files remaining
    if [ $(ls /tmp/zsh_session_* 2> /dev/null | wc -l) -eq 0 ]; then
        echo "Removing kubeconfig for my-cluster"
        rm -f $my_cluster_kubeconfig
    else
        echo "kubeconfig for my-cluster still in use"
    fi
}

trap cleanup_kubeconfig EXIT
```

I hope this method proves helpful. If you have suggestions or improvements, I'd love to hear them in the comments.
