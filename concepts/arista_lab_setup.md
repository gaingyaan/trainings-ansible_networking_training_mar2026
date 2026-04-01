The below guide helps you prepare an Arista emulator setup.




### Getting the Arista Image:
**Step 1 — Create Arista Account**

1. Go to `arista.com`
2. Click **Login** → **Create Account**
3. Fill in your details — work email, name, company
4. Verify your email
5. Log back in

---

**Step 2 — Download cEOS Image**

Once logged in:

1. Click **Support → Software Download**
2. Click **EOS**
3. Click **Active Releases**
4. Pick the latest stable release — look for one ending in `F` (e.g. `4.32.2F`)
5. Inside that release folder, look for **cEOS-lab**
6. Download: `cEOS64-lab-4.32.2F.tar.xz`
   - Must be `cEOS64` — not `cEOS` (32-bit)
   - File size around 600–900MB
   
The same image is also available in the home directory of the emulator setup VM. Since, you have the needed images, now let's set up Docker and Containerlab on that Ubuntu VM, then import the image.

**Step 3 — Install Docker:**

```bash
sudo apt update
sudo apt install -y docker.io

# Add your user to docker group
sudo usermod -aG docker $USER

# Apply group change without logout
newgrp docker

# Verify
docker --version
docker ps
```

**Step 4 — Install Containerlab:**

```bash
bash -c "$(curl -sL https://get.containerlab.dev)"

# Verify
containerlab version
```

**Step 5 — Import cEOS image into Docker:**

```bash
cd ~
docker import cEOS-lab-4.35.3F.tar.xz ceos:4.35.3F

# Verify image imported
docker image ls | grep ceos
```

Expected output:
```
ceos    4.35.3F    <id>    <size>
```

Incase if you do see the name of the repository or the version of the image, you'll need to tag your image. Follow the below steps for the same.

```bash
# Get the image ID
docker image ls

# Tag it correctly using the ID from the output
docker tag <IMAGE_ID> ceos:4.35.3F
```

Replace `<IMAGE_ID>` with the long hex string from the first column of `docker image ls`.

Then verify:

```bash
docker image ls | grep ceos
```

Should now show:
```
ceos    4.35.3F    <id>    <size>
```

Now let's create and start a basic two-node topology to verify everything works.

**Step 6 — Create topology file:**

```bash
mkdir ~/arista-lab
cd ~/arista-lab

cat > topology.clab.yml << 'EOF'
name: ansible-lab

topology:
  nodes:
    R1:
      kind: arista_ceos
      image: ceos:4.35.3F
      startup-config: |
        username admin privilege 15 secret admin
        !
        management api http-commands
         no shutdown
        !

    R2:
      kind: arista_ceos
      image: ceos:4.35.3F
      startup-config: |
        username admin privilege 15 secret admin
        !
        management api http-commands
         no shutdown
        !

  links:
    - endpoints: ["R1:eth1", "R2:eth1"]
EOF
```

**Step 7 — Deploy topology:**

```bash
sudo containerlab deploy -t topology.clab.yml
```

---

This takes about 60-90 seconds. It will print a table at the end showing both nodes and their IP addresses. With this the setup is ready. But its good to always test the setup before moving ahead with creating/executing playbooks on the same.

- Check that the arista emulators are running

```
sudo containerlab inspect -t ~/arista-lab/topology.clab.yml
```

- Make sure that you have the needed ansible modules required for configuring arista setup using ansible. Follow the below commands to install the modules. Best practice is to have them added in the requirements.yml document.
```
ansible-galaxy collection install arista.eos
ansible-galaxy collection install ansible.netcommon
```

- Sample device details for the inventory file. As can be seen there is no difference where we add a Linux machine or an Arista device to the inventory file.

```
[arista_nodes]
R1  ansible_host=<R1_IP>
R2  ansible_host=<R2_IP>

[network_devices:children]
arista_nodes
```

- Sample set of group vars needed specifically for Arista setups

```
---
ansible_connection: ansible.netcommon.network_cli
ansible_network_os: arista.eos.eos
ansible_user: admin
ansible_password: admin
ansible_become: true
ansible_become_method: enable
```

- Gathering arista device facts using ansible adhoc command. this helps to confirm that the setup is accessible

```
ansible arista_nodes -m arista.eos.eos_facts -a "gather_subset=min"
```

And with this your Arista setup is ready. 