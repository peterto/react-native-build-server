## React Native Ansible Build and Dev Playbook

Ansible Playbook that will setup a build server for [Debian Trixie 13](https://www.debian.org/releases/trixie/). I have only verified testing on Debian Trixie, but presumbly other debian based systems will work too.

### Quickstart

<code>
sudo apt install python3.13-venv<br>
python3 -m venv venv<br>
source venv/bin/activate<br>
pip3 install -r requirements.txt<br>
<!-- ansible-galaxy install -r requirements.yml<br> -->
ansible-playbook -i inventory setup_react_native_environment.yml<br>
</code>

<br>

See [Running the playbook](#running-the-playbook) section for more information on running this playbook.

Main playbook is [setup_react_native_environment.yml](setup_react_native_environment.yml), it runs a set of tasks in [roles](.roles). You can omit any tasks if you do not need it in the main playbook. If running locally, remove the first task from "Copy SSH Key Into Remote Host" in the base role.


### How Is This Different From Others Playbooks

- No hardcoded values - This playbook doesn't use any hardcoded versions of any software except for JDK 17 ([Adoptium Temurin 17 is used here](https://adoptium.net/temurin/releases/?version=17)), which is a [requirement for a React Native setup](https://reactnative.dev/docs/set-up-your-environment?os=linux), so this should be pretty future proof with limited amount of changes needed. I use, either, homebrew or aptitude for package installations to install the latest versions of required build software.
- Stock npm usage - This playbook uses npm instead of yarn, bun or something else, for nodejs package management.
- Right now this is for linux systems, so no iOS local build available, but that is a work in progress.
- No docker container image yet.

### Memory Considerations
You will need at least 12GB of RAM to be able to build with `eas build`. Debian Trixie will, by default, allocate half of your RAM to the temporary file system (tmpfs), so I recommend using at least 24GB of memory for a build server.

You might be able to use less if you are just using `./gradlew` to build for specific architectures instead of all architecture binary, see [React Native documentation here](https://reactnative.dev/docs/build-speed). You will need to run `npx expo prebuild` before running `./gradlew` to generate the android project folder. Ensure that you have the `@expo/cli` component installed in your project before running. 

Alternative to using a 24GB system or running `./gradlew`, is allocating more RAM to the tmpfs by running the following:

`sudo systemctl edit tmp.mount` 

Follow the instructions and add the following lines, you might need to allocate more depending on the size of your build:

<code>
[Mount]<br>
<br>
What=tmpfs<br>
Where=/tmp<br>
Type=tmpfs<br>
Options=mode=1777,strictatime,nosuid,nodev,size=12G,nr_inodes=1m<br>
</code>

<br>

Just be aware that I was only occassionally able to run a build successfully even after doing this. 

### inventory and ansible assets and variables

Take a look at [inventory.example](inventory.example), make a copy to `inventory` with your server assets. [Generate SSH key](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent) for key authentication to your server.

Change `hosts` variable in [setup_react_native_environment.yml](setup_react_native_environment.yml) to match your `inventory` file. By default I am using `buildservers`. You will need to generate a ssh key pair for authentication, then have this copied over to the `authorized_keys` in the `.ssh` folder in the build user's home directory.

Edit [all.yml](group_vars/all.yml) with your server's admin user, for this playbook I used `serveradmin` by default.

### Running The Playbook

If using ansible vault for secrets:

ansible-playbook --ask-vault-pass setup_react_native_environment.yml

If you aren't using secrets:

`ansible-playbook -i inventory setup_react_native_environment.yml`

### Useful Commands to Monitor tmpfs

If you are running into issues building, errors might state "Gradle build daemon disappeared unexpectedly", that probably means you ran of out memory and you will have to work on the [workarounds above](#memory-considerations). This command run every 5 seconds to check usage of tmpfs:

`while :; do sleep 5; df -h /tmp; done`

### WSL Host:

I had some permissions issues running this in a Windows host using [WSL](https://learn.microsoft.com/en-us/windows/wsl/install). If you are having permission issues with your ssh key, see [this github issue](https://github.com/geerlingguy/ansible-for-devops/issues/234). I used venv to setup a python virtual environment. Running the playbook:

<code>
sudo apt install python3.13-venv<br>
python3 -m venv venv<br>
source venv/bin/activate<br>
pip3 install -r requirements.txt<br>
<!-- ansible-galaxy install -r requirements.yml<br> -->
ansible-playbook -i inventory setup_react_native_environment.yml
</code>

<br>

<!-- ansible-playbook --ask-vault-pass run.yml -->
<!-- ssh -i ~/.ssh/homelab serveradmin@192.168.1.74 -->
<!-- docker run hello-world -->

### Local React Native (Expo) Build 
After running playbook, ensure that you are logged into EAS services.

`eas login` 

Building a local project:

`eas build --local --platform android --profile development`

`eas build --local --platform android --profile preview`

`eas build --local --platform android --profile production`

I tested local development, preview and production android builds successfully on a fresh install of Debian Trixie. It takes roughly 8 minutes to build on a VM in Debian Trixie with 24GB of RAM using a passthrough host CPU of AMD 5950x.


