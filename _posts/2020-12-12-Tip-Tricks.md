---
layout: post
title: Working environment tips for development on AWS Instances
comments: false
---

This post includes some tips and tricks for coding using a remote instance such as AWS EC2. It includes the some of the know-how that I could gather while I was developing deep learning models at Design AI. 

Before delving right into the tips and tricks, it might be a good idea to briefly mention the tools and why they are needed.

1. [Jupyter Lab](#jupyter-lab) 
- Since all the created objects live by the kernel’s lifetime, it is easy for quick prototyping
- Good for creating guides or demonstrations for purposes
- Lacks very important features like visual debugger, jumping to source or some smart checks before run-time
2. [PyCharm Professional Edition](#pycharm)  >= 2020.2
- Used development and maintain of the code
- Has debugging support
- Most importantly, makes it very easy to use remote interpreters which is essential in case of a cloud GPU use is required
- Has many other convenience functionalities like refactoring, running tests, code versioning and Jupiter notebook support (experimental if you ask me).
3. [Tmux](#tmux)
- For running processes, such as a training on the instance, you first ssh into the instance and run your training script with `python script.py`. However, the process dies when the current shell is closed. When you use Tmux, your shell is persistent even if you close the terminal until the instance itself is closed. 


## Jupyter Lab

a Jupyter Notebook/Lab can be ran on local or remote however it is mostly done on instance since we need the use GPUs.

**Remote usage:** 

- Run `jupyter lab` in remote instance (doing that in a tmux session would make it last longer)
- The jupyter lab uses the port 8888 by default
- Run `ssh -N -f -L <your desired local port #>:localhost:8888 <user name>@<remote instance ip>`
	- You redirect `<your desired local port #>` to `8888` of the remote instance on which the jupyter lab is running
	- You can use the remote jupyter lab with typing `localhost:<your desired local port #>`on browser


**Troubleshooting:**
- Sometimes the ssh connection has to be killed and reset. For that run `ps aux|grep ssh` and find the appropriate ssh process that connects to the instance you used, and run `kill <the id of process next to the username>` and reconnect using the command given in the *usage* part.


## Pycharm 
The main advantage of Pycharm is, it makes it easy to maintain the local code and remote code all together with a little amount of care.

**Some things to consider:**
- The git repository is only cloned into the local computer. No need to create a repository on the instance and keep track of the changes there. This makes things less complicated. So to sum up the process:
	- First clone the repository onto the local computer
	- Deploy the project (copy project files to the istance)
	- Then choose the interpreter (the `../bin/python` file of the virtual environment that is created on the instance)
	- Continue development

**Detailed steps:**

**Deployment:**
Deployment part is for managing the file transfers between local repository and the instance. We can explain it roughly as follows:

1. Click **+** in `Tools -> Deployment -> Configuration` to add a new remote path and give a server name (e.g. the ip so that it can be distunguished amongst different deployment paths)
2. Click **...** next to SSH configuration.
	- Again, click **+** to add a new SSH configuration for the instance
	- After writing the host(ip) and username(e.g. murat), choose,	*OpenSSH and PuTTY* for Authentication. If you have provided the default public key of yours for setting the AWS instance, then leave `id_rsa` not changed. Otherwise, choose the public key that is not the default one. Put your passphrase if you have used in the creation of your ssh key pair.
	- Test connection, Click OK.
3. *Root path* should be `/home/<user name>`. This is the main path that you can see within PyCharm when you do a `Tools->Deployment->Browse Remote Host`
4. In *Mappings* tab, *deployment path* will be the main folder of the project that will be used when you upload your files. e.g. `project`. This path is relative to the *Root path*
5. Choose the default remote server by clicking "✔" while the server configuration is chosen. 
6. Check `Tools->Deployment->Options` to adjust the settings according to your needs. Some more information in *Caveats* part.
7. Now the the whole project or any file/folder can be uploaded to instance by right clicking the file then `Deployment->Upload to <chosen server name>`. 
	- Assigning a shortcut such as ⌥⇪⌘U for this functionality to use it when i change a filet(as if saving a file after modifying it)  to make sure that it is uploaded even though it is supposed to be uploaded automatically to avoid any strange bugs for no reason.
	- Also assigning ⌥⇪⌘R for `Tools->Deployment->Browse Remote Host` is a time saver since browsing the *current chosen remote server* is needed a lot.

**Caveats:**
- When `Tools->Deployment->Automatic Upload` is chosen, if you change a file in PyCharm it will automatically upload the changed file to remote. 
- **However, when a file is externally changed (e.g. with a branch change, stashing, overriding file somehow), PyCharm will not upload it automatically**. And sometimes, when you run the code on PyCharm, it might be using the old code on the instance without you realizing it. This happens mostly in the cases where you confidently say "this is a really weird bug, i am sure it should work correctly."
- Good thing about that is, in `Tools->Deployment->Options` there is an option to include external changes by deselecting *Skip external changes*. Now if you change a branch, it will upload the affected files automatically.
	- Mostly, the flow of development is from local to instance. We develop in the local, deploy it(mostly automatically), and run the code via using the remote interpreter(that will be explained below). 
	- However, sometimes we develop on the instance. This happens mostly when we use the remote jupyter notebook by connecting to it in local as [explained](#jupyter-lab) . Any changes we made is directly made on remote at that point. The local notebooks are not updated. So we need to be careful about that. If you change a branch, say the new branch changes `example.ipynb`, it might override the same file on the instance and you might lose some work if you had on that file. For that you can choose *Compare timestamp & size* in the *Warn when uploading over newer file:* field and also keep an eye of what are going to be changed when there are some external changes to not lose any work.
	  - In that case, when a remote file has changed on the instance, you can `Tools->Deployment->Browse Remote Host`, and then right click the related file and say `Download from here` so that it downloads the file from remote and then overrides the local file. You can see that it took affect by `git status` in the project directory.

**Create a virtual environment in the instance:**
Before setting the remote interpreter we should have a python interpreter in a virtual environment that already has all the libraries we need to use in our project. 

1. Our project is likely to have a `requirements.txt` file to be used. If that is the case it can be uploaded to the instance easily by doing right click on the file and `Deployment->Upload to <chosen server name>`. So that it can be used to install the libraries after creating the virtual environment.
2. ssh into the instance by `ssh <user name>@<ip of the instance>`
3. Download miniconda using `wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O ~/miniconda.sh`
4. Install miniconda by `bash miniconda.sh`. Don't forget to let it init conda at the end.
5. `exit `and `ssh <user name>@<ip of the instance>` again to have conda initialized in the shell.
6. `conda create -n environment_name python=<python version>` to create an environment
7. `conda activate environment_name`
8. `pip install -r requirements.txt` if  `requirements.txt` is present.
  -	It is a good habbit to check `which pip` before running this command. If the `pip` is not from the environment, it can lead some undesired behaviours.



**Remote interpreter:**

1. Go to `PyCharm->Preferences->Python Interpreter` click to ⚙ on the top right and click to *add*.
2. Choose *SSH Interpreter* and check *Existing server configuration* since we already have added the SSH configuration while we set the deployment settings.
	- If you have a warning that says *Move this config to IDE Settings* click to the *move*.
3. Click *next*
4. De-select *Automatically upload project files to the server* since we already did it (in Deployment-7) to the project root we configured (in Deployment-4).
5. Click 📁 on right of the interpreter to choose the python interpreter we created within the `environment_name`environment
6. Select `home/<user name>/miniconda3/envs/<environment_name>/bin/python` and click *Finish*
7. Now you will be using the remote interpreter when you run a python file in PyCharm


## Tmux

Tmux lets you keep the shell session even if you disconnect from the server or close the shell when you run it locally.

**Usage:**

- After `ssh <user name>@<ip of the instance>`, run `tmux` command.
- The `tmux` command will create a new session already. A session can have different number of *windows* and a *window* can have different amount of *panes*. 
- All the commands in tmux is preceded by `ctrl + b`. You press `ctrl + b` together and then (quickly) press the other shortcut.
- For creating a vertical pane, do `(ctrl + b) + %` 
- For creating a horizontal pane, do `(ctrl + b) + "`
- To navigate between panes, do `(ctrl + b) + arrow keys`
- To exit from a pane, run `exit` in the related pane.
- To create new window within the same session, do `(ctrl + b) + c`
- To navigate between windows, use `(ctrl + b) + n` for next and `(ctrl + b) + p` for previous
- To detach from the session all together to go back to the shell just after ssh, `(ctrl + b) + d` . Detaching means closing(kind of minimising) the session however it is not lost, you just return where you ran the `tmux` command.
- After detaching, you can see your sessions by `tmux ls`
- And you can attach to the session by using `tmux attach-session -t <session id>`. `<session id>` is the first number you see when you do `tmux ls`
- Now, you can close the ssh connection or close the terminal but that sesion will remain there and you will be able to attach next time you ssh into the instance with `tmux ls` and `tmux attach-session -t <session id>`


### Additional Commands:

- `nvidia-smi` to check the status of the GPU on the instance
- `w` to check who are using the instance right now
- `netstat -tulnap | grep :<port number>` to check which process is using the given `<port number>`



**TODOS:** 

- Check the Docker functionality of Pycharm and extend the documentation for use of Docker