---
title: "Securing kubectl config with 1Password"
date: 2023-12-20T07:26:13+01:00
draft: false
---

## The problem

When you access your Kubernetes cluster your `~/.kube/config` file contains a lot of sensitive information. It contains the credentials to access your cluster, the cluster certificate and the cluster endpoint. This is typically fine if you're encrypting your harddrive and everything, but to be extra safe you shouldn't keep sensitive plain text files on your harddrive at all.

## The solution

There are industry standard solutions to problems like this like running Vault and getting credentials through there, but what if you're just a single person business and Vault is overkill for your usecase? Well, you can use 1Password to store your credentials and then use a script to fetch them and write them to your kubeconfig file.

In my case, I have a cluster that we'll call "my-cluster" and I keep my kubeconfig file in a cluster project directory along with various deployments, services and other definition files.

1Password has a nifty set of commandline tools for extracting data from your vault. [You can find them here](https://support.1password.com/command-line-getting-started/) and they're available for most platforms. As I'm using macOS so I installed them with Homebrew.

```bash
brew install 1password-cli
```

Now let's process our kubernetes config file a bit. Here's a garbled version of my original file:

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

The first step is to transfer these secret values to 1Password. You can go crazy here and transfer every single value if you wish, but the important ones - as far as I know - are the client-certificate-data, client-key-data and the certificate-authority-data. Open 1Password and create a new Password item. Add custom fields for each of the values you want to store. I named mine `client-certificate-data`, `client-key-data` and `certificate-authority-data`. Now copy the values from your kubeconfig file into the corresponding fields in 1Password.

![Store Kubeconfig values in 1Password](/images/1password-store-kubeconfig-values.png)

As you can see, I also put a backup of the original kubeconfig file in 1Password. This is optional, but I like to have it there just in case.

After you've saved your new Password item, you can get the secret reference for each of the values.

![Store Kubeconfig values in 1Password](/images/1password-copy-secret-reference.png)

Rename your kubeconfig file by adding `.tpl` to the end of the filename and open it in an editor so you can replace each of the secret values with their corresponding secret reference. Here's what my file looks like after I've replaced the values:

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

Now it's time for the magic sauce.

When ever I want to play with my kuberentes cluster, I open a terminal. `kubectl` uses an environment variable called `KUBECONFIG`. Very fancy name. In my setup, this file should not exist when I don't have a terminal window open, and it should magically appear when ever I need it.

First, let's solve the magic apperance of the file using the very fancy `op` cli tool.

```zsh
my_cluster_kubeconfig_tpl=$HOME/Projects/my-cluster/kubeconfig-my-cluster.yaml.tpl
my_cluster_kubeconfig=$HOME/Projects/my-cluster/kubeconfig-my-cluster.yaml
echo "Creating kubeconfig for my-cluster"
op inject -i $my_cluster_kubeconfig_tpl -o $my_cluster_kubeconfig
export KUBECONFIG=$my_cluster_kubeconfig
```

This will create and configure my kubeconfig yaml file and set the environment variable. All the secrets are fetched from 1Password and injected into the file, replacing the reference strings with the actual values.

When `op` runs, 1Password appears and asks for either a fingerprint reader touch, or the master password. This is annoying if it happens when it's not needed, so let's add this simple if statement to the script:

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

Now, when I close my terminal window, I want to remove the file again. I also want to remove it if I close the terminal window and I'm the last one using it. To do this, let's leverage the `trap` command. This command allows us to run a function when the shell exits. We'll use this to remove the kubeconfig file when the shell exits.

```zsh
function cleanup_kubeconfig {
    rm -f $my_cluster_kubeconfig
}

trap cleanup_kubeconfig EXIT
```

This will remove the generated kubeconfig when the terminal session ends. But what if I have multiple terminal windows open? I don't want to remove the kubeconfig file if I'm still using it in another window. To solve this, we'll create a temporary file for each session and check if there are any other session files remaining when the shell exits. If there are no other session files, we'll remove the kubeconfig file.

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

The only flaw with this approach is that if the terminal window crashes, the kubeconfig file will not be removed. I'm not sure how to solve this, but I'm open to suggestions.

Different terminals behave differently, but both iTerm (my favorite MacOS terminal) and the built-in terminal app seem to behave in a self-cleaning way in this scenaro. iTerm recreates the crashed session and keeps the reference to the trap file, and the built-in terminal app seems to remove the trap file when the session crashes but fails to remove the kubeconfig file. I'm not sure why this is. But this way, the insecure state only lasts until the next terminal session in my tests.

Now to tie this all together, I added the script to my `.zshrc` file.

Here's the result:

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

I hope this is useful to someone. If you have any suggestions for improvements, please let me know in the comments.
